# Kubernetes Services

Services são uma abstração que define um conjunto lógico de Pods e uma política de acesso a eles. Services permitem que seus Pods sejam acessíveis na rede, mesmo que os Pods sejam efêmeros e tenham IPs dinâmicos.

## O que é um Service?

Um Service é um recurso que:
- ✅ Fornece um **endpoint estável** (IP fixo, DNS) para um grupo de Pods
- ✅ Faz **load balancing** automático entre os Pods
- ✅ Permite **service discovery** via DNS
- ✅ Abstrai a localização física dos Pods
- ✅ Suporta diferentes tipos de exposição (interna/externa)

## Tipos de Services

| Tipo | Descrição | Acesso | Uso Comum |
|------|-----------|--------|-----------|
| **ClusterIP** | IP interno do cluster (padrão) | Apenas dentro do cluster | Comunicação interna entre serviços |
| **NodePort** | Expõe em porta específica em cada Node | Externa via `<NodeIP>:<NodePort>` | Desenvolvimento, bare-metal |
| **LoadBalancer** | Provisiona load balancer externo | Externa via IP do LB | Produção em cloud providers |
| **ExternalName** | Mapeia para DNS externo | Via CNAME | Integração com serviços externos |

## Como Services Funcionam

### Seletor de Labels
Services usam labels para identificar quais Pods devem receber tráfego:

```yaml
# Service
selector:
  app: myapp
  tier: backend

# Pods que correspondem
metadata:
  labels:
    app: myapp
    tier: backend
```

### Endpoints
Kubernetes cria automaticamente um objeto **Endpoints** que lista os IPs dos Pods:

```bash
kubectl get endpoints myapp-service
# NAME            ENDPOINTS
# myapp-service   10.1.1.5:8080,10.1.1.6:8080,10.1.1.7:8080
```

### Service Discovery

**Via DNS** (recomendado):
```bash
# Dentro do mesmo namespace
curl http://myapp-service:80

# De outro namespace
curl http://myapp-service.production.svc.cluster.local:80
```

**Via Variáveis de Ambiente** (legacy):
```bash
MYAPP_SERVICE_HOST=10.96.0.10
MYAPP_SERVICE_PORT=80
```

## Port Terminology

Entender a diferença entre as portas é crucial:

```yaml
spec:
  ports:
  - port: 80           # Porta do SERVICE (onde clientes se conectam)
    targetPort: 8080   # Porta do CONTAINER (onde app escuta)
    nodePort: 30000    # Porta do NODE (apenas NodePort/LoadBalancer)
    protocol: TCP
    name: http
```

| Campo | Descrição | Exemplo |
|-------|-----------|---------|
| `port` | Porta exposta pelo Service | 80 |
| `targetPort` | Porta no container do Pod | 8080 |
| `nodePort` | Porta no node (30000-32767) | 30000 |

**Fluxo de tráfego**:
```
Cliente → Service:port → targetPort:Container
Cliente:NodeIP:30000 → Service:80 → Pod:8080
```

## Session Affinity

Por padrão, Services distribuem tráfego aleatoriamente. Para manter cliente sempre no mesmo Pod:

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 horas
```

Valores:
- `None`: Sem afinidade (padrão)
- `ClientIP`: Mesmo cliente → mesmo Pod (baseado no IP do cliente)

## Headless Services

Service sem ClusterIP, retorna IPs dos Pods diretamente:

```yaml
spec:
  clusterIP: None  # Headless
  selector:
    app: myapp
```

**Quando usar**:
- StatefulSets (acesso direto aos Pods)
- Service discovery customizado
- Controle total sobre load balancing

**DNS retorna**:
```bash
# Service normal
nslookup myapp-service
# 10.96.0.10 (ClusterIP)

# Headless service
nslookup myapp-service
# 10.1.1.5, 10.1.1.6, 10.1.1.7 (IPs dos Pods)
```

## Service sem Selector

Para integrar serviços externos ou bancos de dados fora do cluster:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  ports:
  - port: 3306
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-db  # Mesmo nome do Service
subsets:
- addresses:
  - ip: 192.168.1.100
  ports:
  - port: 3306
```

## Comandos Úteis

```bash
# Criar Service
kubectl apply -f service.yaml
kubectl expose deployment myapp --port=80 --target-port=8080 --type=ClusterIP

# Listar Services
kubectl get services
kubectl get svc
kubectl get svc -o wide
kubectl get svc --all-namespaces

# Descrever Service
kubectl describe service myapp

# Ver Endpoints
kubectl get endpoints myapp
kubectl describe endpoints myapp

# Testar Service de dentro do cluster
kubectl run curl --image=curlimages/curl -i --tty --rm -- sh
curl http://myapp-service:80

# Port forward para acessar localmente
kubectl port-forward service/myapp 8080:80
# Acesse: http://localhost:8080

# Ver DNS do Service
kubectl exec -it <pod-name> -- nslookup myapp-service

# Ver detalhes do Service
kubectl get service myapp -o yaml

# Editar Service
kubectl edit service myapp

# Deletar Service
kubectl delete service myapp
kubectl delete -f service.yaml

# Ver logs de um Pod atrás do Service
kubectl logs -l app=myapp --all-containers=true

# Ver eventos relacionados
kubectl get events --field-selector involvedObject.name=myapp
```

## Load Balancing Algorithms

Kubernetes usa **iptables** ou **IPVS** para load balancing:

### iptables (padrão)
- Round-robin aleatório
- Baseado em probabilidade
- Overhead menor
- Sem health checking real-time

### IPVS (avançado)
- Algoritmos: rr (round-robin), lc (least connection), dh (destination hashing)
- Melhor performance com muitos Services
- Health checking mais preciso
- Requer módulo kernel IPVS

**Habilitar IPVS**:
```yaml
# kube-proxy ConfigMap
mode: "ipvs"
ipvs:
  scheduler: "rr"  # ou lc, dh, sh, sed, nq
```

## Políticas de Tráfego

### externalTrafficPolicy

Controla como tráfego externo é roteado:

**Cluster (padrão)**:
- Distribui para qualquer Pod no cluster
- Pode fazer hops entre nodes
- Preserva disponibilidade
- Perde source IP do cliente

**Local**:
- Apenas Pods no mesmo node
- Sem hops entre nodes
- Preserva source IP do cliente
- Pode causar desbalanceamento

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local  # ou Cluster
```

### internalTrafficPolicy (Kubernetes 1.22+)

Similar a externalTrafficPolicy, mas para tráfego interno:

```yaml
spec:
  internalTrafficPolicy: Local  # ou Cluster (padrão)
```

## Multi-Port Services

Service pode expor múltiplas portas:

```yaml
spec:
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  - name: metrics
    port: 9090
    targetPort: 9090
```

**Importante**: Quando múltiplas portas, `name` é obrigatório!

## Service Topology (Deprecated → Topology Aware Hints)

Rotear tráfego baseado em topologia (zona, região):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  annotations:
    service.kubernetes.io/topology-mode: Auto
spec:
  # ...
```

Kubernetes tenta rotear para Pods na mesma zona para reduzir latência e custos de cross-zone traffic.

## Troubleshooting

### Service não alcança Pods

```bash
# 1. Verificar se Service existe
kubectl get svc myapp

# 2. Verificar Endpoints
kubectl get endpoints myapp
# Se vazio, labels não correspondem!

# 3. Verificar labels dos Pods
kubectl get pods -l app=myapp --show-labels

# 4. Verificar selector do Service
kubectl get svc myapp -o yaml | grep -A 5 selector

# 5. Testar de dentro do cluster
kubectl run debug --image=busybox -i --tty --rm -- sh
wget -O- http://myapp:80
```

### DNS não resolve

```bash
# Testar DNS de dentro de um Pod
kubectl exec -it <pod-name> -- nslookup myapp-service

# Verificar CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# Verificar configuração DNS do Pod
kubectl exec -it <pod-name> -- cat /etc/resolv.conf
```

### LoadBalancer fica em Pending

```bash
# Ver eventos
kubectl describe service myapp

# Causas comuns:
# - Cloud provider não suporta LoadBalancer
# - Quotas excedidas
# - Configuração incorreta do cloud controller
# - Cluster local (minikube, kind) sem suporte

# Alternativa: usar NodePort ou Ingress
```

### NodePort não acessível

```bash
# 1. Verificar se porta está no range (30000-32767)
kubectl get svc myapp

# 2. Verificar firewall do node
# Porta precisa estar aberta no firewall

# 3. Verificar se Pods estão Running
kubectl get pods -l app=myapp

# 4. Testar de dentro do node
ssh node-1
curl localhost:30000
```

### Tráfego não balanceia corretamente

```bash
# Verificar número de Endpoints
kubectl get endpoints myapp

# Verificar se todos os Pods estão Ready
kubectl get pods -l app=myapp

# Verificar readiness probes
kubectl describe pod <pod-name>

# Verificar sessionAffinity
kubectl get svc myapp -o yaml | grep sessionAffinity
```

## Boas Práticas

### 1. **Use ClusterIP por padrão**
```yaml
# Para comunicação interna
spec:
  type: ClusterIP
```

### 2. **Nomeie suas portas**
```yaml
# Facilita referência em Ingress e outras configs
ports:
- name: http
  port: 80
- name: https
  port: 443
```

### 3. **Configure Health Checks nos Pods**
```yaml
# Service só roteia para Pods Ready
readinessProbe:
  httpGet:
    path: /health
    port: 8080
```

### 4. **Use Ingress para HTTP/HTTPS**
```yaml
# Ao invés de LoadBalancer para cada serviço
# Use 1 LoadBalancer com Ingress Controller
```

### 5. **Defina Resource Limits nos Pods**
```yaml
# Evita Pods sobrecarregados recebendo tráfego
resources:
  requests:
    cpu: "100m"
  limits:
    cpu: "500m"
```

### 6. **Use Labels Consistentes**
```yaml
# Service
selector:
  app: myapp
  version: v1

# Deployment
template:
  metadata:
    labels:
      app: myapp
      version: v1
```

### 7. **Configure Session Affinity quando necessário**
```yaml
# Para aplicações stateful
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
```

### 8. **Use externalTrafficPolicy=Local quando precisar do IP do cliente**
```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
```

### 9. **Monitore Endpoints**
```yaml
# Alerte quando Endpoints ficarem vazios
kube_endpoint_address_available{service="myapp"} == 0
```

### 10. **Use Headless Services para StatefulSets**
```yaml
# StatefulSets precisam de acesso direto aos Pods
spec:
  clusterIP: None
```

## Comparação dos Tipos

| Aspecto | ClusterIP | NodePort | LoadBalancer |
|---------|-----------|----------|--------------|
| **Acesso** | Interno | Externo | Externo |
| **IP** | Cluster interno | Node IP | Externo (provisionado) |
| **Porta** | Qualquer | 30000-32767 | Qualquer |
| **Load Balancing** | Interno | Nos nodes | Externo + Interno |
| **Custo** | Grátis | Grátis | Pago (cloud) |
| **Produção** | ✅ Sim | ⚠️ Limitado | ✅ Sim |
| **Desenvolvimento** | ✅ Sim | ✅ Sim | ❌ Overkill |
| **High Availability** | ✅ Sim | ⚠️ Depende | ✅ Sim |

## Quando Usar Cada Tipo

### ClusterIP
✅ Comunicação entre microserviços  
✅ APIs internas  
✅ Bancos de dados  
✅ Caches (Redis, Memcached)  
✅ Message queues  

### NodePort
✅ Desenvolvimento local  
✅ Clusters bare-metal sem LoadBalancer  
✅ Acesso temporário para debug  
✅ Integração com load balancers externos  
❌ Produção (use LoadBalancer ou Ingress)  

### LoadBalancer
✅ Produção em cloud (AWS, GCP, Azure)  
✅ Aplicações públicas  
✅ APIs externas  
✅ Quando precisa de IP estático externo  
❌ Desenvolvimento (custo desnecessário)  
❌ Múltiplos serviços HTTP (use Ingress)  

## Service Mesh Integration

Services são a base para Service Mesh (Istio, Linkerd):

```yaml
# Service normal
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80

# Istio adiciona sidecar proxy automaticamente
# E fornece:
# - Mutual TLS
# - Traffic shaping
# - Observability
# - Circuit breaking
```

## Métricas e Monitoramento

### Prometheus Metrics

```promql
# Service existe
kube_service_info{service="myapp"}

# Número de Endpoints
kube_endpoint_address_available{service="myapp"}

# Service sem Endpoints (alerta!)
kube_endpoint_address_available{service="myapp"} == 0

# Service do tipo LoadBalancer
kube_service_spec_type{type="LoadBalancer"}
```

## Links Úteis

- [Kubernetes Services Documentation](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Service Types](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Connecting Applications with Services](https://kubernetes.io/docs/tutorials/services/connect-applications-service/)
