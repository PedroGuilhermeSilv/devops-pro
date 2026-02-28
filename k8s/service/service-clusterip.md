# Kubernetes Service - ClusterIP

ClusterIP é o tipo **padrão** de Service no Kubernetes. Ele expõe o Service em um IP interno do cluster, tornando-o acessível apenas dentro do cluster.

## O que é ClusterIP?

- 🔒 **IP interno do cluster**: Acessível apenas de dentro do cluster
- 🎯 **Tipo padrão**: Se não especificar `type`, será ClusterIP
- 🔄 **Load balancing interno**: Distribui tráfego entre os Pods
- 📡 **Service discovery**: Acessível via DNS interno
- 🚀 **Caso de uso**: Comunicação entre serviços (microserviços)

## Características

| Característica | Descrição |
|----------------|-----------|
| **Visibilidade** | Apenas dentro do cluster |
| **IP** | Interno, faixa do cluster (ex: 10.96.0.0/12) |
| **DNS** | `<service-name>.<namespace>.svc.cluster.local` |
| **Porta** | Qualquer porta disponível |
| **Custo** | Gratuito |
| **Load Balancing** | Automático entre os Pods |

## Arquitetura

```
┌─────────────────────────────────────────────┐
│              Kubernetes Cluster             │
│                                             │
│  ┌──────────┐                               │
│  │   Pod A  │ ──┐                           │
│  └──────────┘   │                           │
│                 │    ┌────────────────┐     │
│  ┌──────────┐   ├───▶│ ClusterIP      │     │
│  │   Pod B  │ ──┤    │ 10.96.0.10:80  │     │
│  └──────────┘   │    └────────────────┘     │
│                 │            ▲              │
│  ┌──────────┐   │            │              │
│  │   Pod C  │ ──┘            │              │
│  └──────────┘                │              │
│                               │              │
│  ┌──────────────────┐         │              │
│  │ Client Pod       │─────────┘              │
│  │ curl service:80  │                        │
│  └──────────────────┘                        │
│                                             │
└─────────────────────────────────────────────┘
```

## Quando Usar ClusterIP

### ✅ Use ClusterIP para:

1. **Comunicação entre microserviços**
   - API Gateway → Auth Service
   - Backend → Database
   - App → Cache (Redis)

2. **Serviços internos**
   - Bancos de dados (PostgreSQL, MySQL)
   - Message queues (RabbitMQ, Kafka)
   - Caches (Redis, Memcached)
   - Monitoring interno (Prometheus)

3. **APIs privadas**
   - APIs administrativas
   - Serviços de autenticação
   - Processamento de background

4. **Padrão de produção**
   - Expor externamente apenas via Ingress
   - Manter serviços internos privados

### ❌ NÃO use ClusterIP quando:

- Precisa acessar de fora do cluster → Use **LoadBalancer** ou **Ingress**
- Desenvolvimento local e quer acessar diretamente → Use **NodePort** ou `kubectl port-forward`
- Integração com sistemas externos → Use **ExternalName** ou Service sem selector

## Service Discovery

### Via DNS (Recomendado)

ClusterIP Services são automaticamente registrados no DNS do cluster:

```bash
# Dentro do mesmo namespace
http://myapp-service
http://myapp-service:80

# De outro namespace
http://myapp-service.production
http://myapp-service.production.svc.cluster.local

# FQDN completo
http://myapp-service.production.svc.cluster.local:80
```

**Formato DNS**:
```
<service-name>.<namespace>.svc.<cluster-domain>
```

**Padrões**:
- `cluster-domain`: `cluster.local` (padrão)
- Pode omitir partes do FQDN se estiver no mesmo namespace/cluster

### Via Variáveis de Ambiente (Legacy)

Quando um Pod é criado, Kubernetes injeta variáveis de ambiente para Services **já existentes**:

```bash
# Para Service chamado "myapp-service" na porta 80
MYAPP_SERVICE_SERVICE_HOST=10.96.0.10
MYAPP_SERVICE_SERVICE_PORT=80
MYAPP_SERVICE_PORT=tcp://10.96.0.10:80
MYAPP_SERVICE_PORT_80_TCP=tcp://10.96.0.10:80
MYAPP_SERVICE_PORT_80_TCP_PROTO=tcp
MYAPP_SERVICE_PORT_80_TCP_PORT=80
MYAPP_SERVICE_PORT_80_TCP_ADDR=10.96.0.10
```

⚠️ **Limitações**:
- Service deve existir **antes** do Pod ser criado
- Muito verboso e difícil de gerenciar
- **Use DNS ao invés disso!**

## Acessando ClusterIP de Fora do Cluster

ClusterIP é interno, mas há formas de acessar para desenvolvimento/debug:

### 1. kubectl port-forward (Recomendado para Dev)

```bash
# Forward porta local para o Service
kubectl port-forward service/myapp 8080:80

# Acesse em: http://localhost:8080
```

**Características**:
- Rápido e fácil
- Não expõe publicamente
- Apenas sua máquina pode acessar
- Ótimo para debug

### 2. kubectl proxy

```bash
# Inicia proxy para API Server
kubectl proxy --port=8001

# Acesse em:
# http://localhost:8001/api/v1/namespaces/default/services/myapp:80/proxy/
```

**Características**:
- Acesso via API Server do Kubernetes
- URL mais verbosa
- Útil para explorar API

### 3. Pod de Debug

```bash
# Criar Pod temporário para testar
kubectl run curl --image=curlimages/curl -i --tty --rm -- sh

# Dentro do Pod
curl http://myapp-service:80
```

### 4. Ingress (Produção)

```yaml
# Expor ClusterIP externamente via Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service  # ClusterIP Service
            port:
              number: 80
```

## ClusterIP Allocation

### Automático (Recomendado)

Kubernetes atribui um IP automaticamente da faixa de ClusterIPs:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: ClusterIP  # ou omita, é o padrão
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

### Manual (Raramente necessário)

Especificar um ClusterIP fixo:

```yaml
spec:
  type: ClusterIP
  clusterIP: 10.96.0.100  # IP específico
  selector:
    app: myapp
  ports:
  - port: 80
```

⚠️ **Cuidados**:
- IP deve estar na faixa válida do cluster
- IP não deve estar em uso
- Geralmente não é necessário (use DNS!)

### None (Headless Service)

Service sem ClusterIP, retorna IPs dos Pods diretamente:

```yaml
spec:
  clusterIP: None  # Headless
  selector:
    app: myapp
  ports:
  - port: 80
```

**Quando usar**:
- StatefulSets (acesso direto aos Pods)
- Descobrir todos os IPs dos Pods
- Client-side load balancing

## Load Balancing

ClusterIP distribui tráfego entre os Pods usando iptables ou IPVS:

### Round-Robin (Padrão)

Distribui requisições aleatoriamente/uniformemente:

```
Request 1 → Pod A
Request 2 → Pod C
Request 3 → Pod B
Request 4 → Pod A
```

### Session Affinity (Sticky Sessions)

Mantém cliente sempre no mesmo Pod:

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 horas
```

**Quando usar**:
- Aplicações com estado na memória
- WebSockets que precisam persistência
- Shopping carts que não usam Redis/DB

**Desvantagens**:
- Desbalanceamento de carga
- Pod com problema continua recebendo tráfego do mesmo cliente

## Políticas de Tráfego

### internalTrafficPolicy (K8s 1.22+)

Controla roteamento de tráfego interno:

```yaml
spec:
  internalTrafficPolicy: Local  # ou Cluster (padrão)
```

**Cluster (padrão)**:
- Roteia para qualquer Pod no cluster
- Melhor distribuição de carga
- Pode fazer hops entre nodes

**Local**:
- Apenas Pods no mesmo node
- Menor latência
- Pode causar desbalanceamento
- Útil para DaemonSets

## Multi-Port Services

Service pode expor múltiplas portas:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - name: http      # Nome obrigatório com múltiplas portas
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
  - name: metrics
    port: 9090
    targetPort: 9090
    protocol: TCP
```

**Acesso**:
```bash
# HTTP
curl http://myapp:80

# HTTPS
curl https://myapp:443

# Metrics
curl http://myapp:9090/metrics
```

## Troubleshooting ClusterIP

### Service não resolve no DNS

```bash
# 1. Testar DNS de dentro de um Pod
kubectl run debug --image=busybox -i --tty --rm -- sh
nslookup myapp-service

# 2. Verificar CoreDNS está rodando
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 3. Ver logs do CoreDNS
kubectl logs -n kube-system -l k8s-app=kube-dns

# 4. Verificar configuração DNS do Pod
kubectl exec -it <pod> -- cat /etc/resolv.conf
```

### Service não alcança os Pods

```bash
# 1. Verificar Endpoints
kubectl get endpoints myapp-service
# Se vazio, o selector não corresponde aos labels dos Pods!

# 2. Verificar labels dos Pods
kubectl get pods -l app=myapp --show-labels

# 3. Verificar selector do Service
kubectl get svc myapp-service -o yaml | grep -A 5 selector

# 4. Comparar labels
# Service selector DEVE ser subset das labels dos Pods
```

### Tráfego não balanceia

```bash
# 1. Verificar readiness dos Pods
kubectl get pods -l app=myapp
# Apenas Pods Ready recebem tráfego

# 2. Verificar readiness probe
kubectl describe pod <pod-name> | grep -A 10 Readiness

# 3. Verificar se há sessionAffinity
kubectl get svc myapp -o yaml | grep sessionAffinity
```

### Latência alta

```bash
# 1. Verificar se Pods estão no mesmo node
kubectl get pods -o wide -l app=myapp

# 2. Usar internalTrafficPolicy: Local
kubectl patch svc myapp -p '{"spec":{"internalTrafficPolicy":"Local"}}'

# 3. Verificar recursos dos Pods
kubectl top pods -l app=myapp

# 4. Verificar se há throttling
kubectl describe pod <pod-name> | grep -i throttl
```

## Exemplos Práticos

### Backend API

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-api
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: backend
    tier: api
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

**Acesso de outros Pods**:
```bash
# Mesmo namespace
curl http://backend-api/api/users

# Outro namespace
curl http://backend-api.production/api/users
```

### Banco de Dados

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: databases
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
  - name: postgres
    port: 5432
    targetPort: 5432
```

**Connection string**:
```
postgresql://user:pass@postgres.databases:5432/mydb
```

### Redis Cache

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: cache
spec:
  type: ClusterIP
  selector:
    app: redis
  ports:
  - name: redis
    port: 6379
    targetPort: 6379
```

**Acesso**:
```python
import redis
r = redis.Redis(host='redis.cache', port=6379)
```

## Boas Práticas

### 1. Use DNS, não variáveis de ambiente
```python
# ✅ Bom
db_host = "postgres.databases"

# ❌ Evite
db_host = os.getenv("POSTGRES_SERVICE_HOST")
```

### 2. Use FQDN em ambientes multi-namespace
```yaml
# ✅ Bom - evita ambiguidade
DATABASE_URL: postgresql://user:pass@postgres.databases.svc.cluster.local:5432/db

# ⚠️ Funciona apenas no mesmo namespace
DATABASE_URL: postgresql://user:pass@postgres:5432/db
```

### 3. Nomeie as portas
```yaml
# ✅ Bom - facilita referência
ports:
- name: http
  port: 80
- name: metrics
  port: 9090

# ❌ Evite
ports:
- port: 80
- port: 9090
```

### 4. Configure readiness probes nos Pods
```yaml
# Service só roteia para Pods Ready
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 5. Use labels consistentes
```yaml
# Service
selector:
  app: myapp
  component: backend

# Deployment
template:
  metadata:
    labels:
      app: myapp
      component: backend
```

### 6. Monitore Endpoints
```bash
# Alerte quando Service ficar sem Endpoints
kubectl get endpoints myapp -o json | jq '.subsets[].addresses | length'
```

### 7. Prefira ClusterIP + Ingress sobre LoadBalancer
```yaml
# ✅ Bom - 1 LoadBalancer, N Services
ClusterIP Services + Ingress Controller

# ❌ Caro - N LoadBalancers
N LoadBalancer Services
```

## Comparação com Outros Tipos

| Aspecto | ClusterIP | NodePort | LoadBalancer |
|---------|-----------|----------|--------------|
| **Acesso** | Interno | Externo (Node) | Externo (LB) |
| **Complexidade** | Simples | Média | Média |
| **Custo** | Grátis | Grátis | Pago |
| **Produção** | ✅ Sim | ⚠️ Limitado | ✅ Sim |
| **Segurança** | ✅ Alta | ⚠️ Média | ✅ Alta |
| **Latência** | ✅ Baixa | ⚠️ Média | ⚠️ Média |

## Monitoramento

### Métricas Prometheus

```promql
# Service existe
kube_service_info{service="myapp",type="ClusterIP"}

# Número de Endpoints
kube_endpoint_address_available{service="myapp"}

# Alerta: Service sem Endpoints
kube_endpoint_address_available{service="myapp"} == 0

# Portas do Service
kube_service_spec_port{service="myapp"}
```

### Health Check

```bash
# Script de monitoramento
#!/bin/bash
SERVICE_NAME="myapp"
ENDPOINTS=$(kubectl get endpoints $SERVICE_NAME -o json | jq '.subsets[].addresses | length')

if [ "$ENDPOINTS" -eq 0 ]; then
  echo "CRITICAL: Service $SERVICE_NAME has no endpoints!"
  exit 2
fi

echo "OK: Service $SERVICE_NAME has $ENDPOINTS endpoints"
```

## Links Úteis

- [Kubernetes Service - ClusterIP](https://kubernetes.io/docs/concepts/services-networking/service/#type-clusterip)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Connecting Applications with Services](https://kubernetes.io/docs/tutorials/services/connect-applications-service/)
