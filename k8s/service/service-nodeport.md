# Kubernetes Service - NodePort

NodePort expõe o Service em uma porta estática em cada Node do cluster, tornando-o acessível externamente através de `<NodeIP>:<NodePort>`.

## O que é NodePort?

- 🌐 **Acesso externo**: Acessível de fora do cluster
- 🔢 **Porta fixa**: Mesma porta em todos os nodes (30000-32767)
- 🔄 **ClusterIP incluído**: NodePort cria automaticamente um ClusterIP também
- 📡 **Roteamento**: Qualquer node pode receber e rotear tráfego
- 🎯 **Caso de uso**: Desenvolvimento, bare-metal, debugging

## Características

| Característica | Descrição |
|----------------|-----------|
| **Visibilidade** | Externa (qualquer node do cluster) |
| **Porta** | 30000-32767 (configurável) |
| **IP** | IP de qualquer node do cluster |
| **ClusterIP** | Criado automaticamente |
| **Load Balancing** | Entre nodes + entre pods |
| **Custo** | Gratuito |

## Arquitetura

```
┌───────────────────────────────────────────────────────┐
│                   Internet/External                    │
└───────────────────────────────┬───────────────────────┘
                                │
                ┌───────────────┼───────────────┐
                │               │               │
        ┌───────▼──────┐ ┌──────▼──────┐ ┌─────▼───────┐
        │   Node 1     │ │   Node 2    │ │   Node 3    │
        │ :30000       │ │ :30000      │ │ :30000      │
        └───────┬──────┘ └──────┬──────┘ └─────┬───────┘
                │               │               │
                └───────────────┼───────────────┘
                                │
                        ┌───────▼───────┐
                        │   NodePort    │
                        │   Service     │
                        │ :30000 → :80  │
                        └───────┬───────┘
                                │
                ┌───────────────┼───────────────┐
                │               │               │
        ┌───────▼──────┐ ┌──────▼──────┐ ┌─────▼───────┐
        │   Pod A      │ │   Pod B     │ │   Pod C     │
        │   :8080      │ │   :8080     │ │   :8080     │
        └──────────────┘ └─────────────┘ └─────────────┘
```

## Fluxo de Tráfego

```
Cliente Externo
    ↓
NodeIP:30000 (qualquer node)
    ↓
NodePort Service
    ↓
ClusterIP:80
    ↓
Pod:8080
```

**Importante**: Tráfego pode entrar em um node e ser roteado para Pod em outro node!

## Quando Usar NodePort

### ✅ Use NodePort para:

1. **Desenvolvimento local**
   - Minikube, Kind, K3s
   - Testar aplicações localmente
   - Acesso rápido sem configuração complexa

2. **Clusters bare-metal**
   - Sem cloud provider (sem LoadBalancer)
   - Integrar com load balancer externo (HAProxy, Nginx)
   - Datacenters on-premise

3. **Debugging e testes**
   - Acesso temporário para troubleshooting
   - Testes de integração
   - Demos e PoCs

4. **Load balancer externo customizado**
   - HAProxy/Nginx roteando para NodePorts
   - F5, Citrix, ou outros load balancers

### ❌ NÃO use NodePort quando:

- Em produção com cloud provider → Use **LoadBalancer** ou **Ingress**
- Precisa de TLS termination → Use **Ingress**
- Múltiplos serviços HTTP → Use **Ingress**
- Apenas acesso interno → Use **ClusterIP**
- Portas conflitantes (limitado a 30000-32767)

## Range de Portas

Por padrão, NodePort usa portas entre **30000-32767**:

```bash
# Ver range configurado no cluster
kubectl cluster-info dump | grep service-node-port-range

# Padrão: --service-node-port-range=30000-32767
```

**Modificar range** (no API Server):
```yaml
# kube-apiserver.yaml
--service-node-port-range=30000-35000
```

⚠️ **Atenção**: Aumentar range pode conflitar com outras aplicações no node!

## Acessando NodePort

### Obter NodePort e IP

```bash
# Ver NodePort atribuído
kubectl get svc myapp
# NAME    TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
# myapp   NodePort   10.96.0.10    <none>        80:30080/TCP   5m

# Obter IPs dos nodes
kubectl get nodes -o wide
# NAME     STATUS   ROLES    INTERNAL-IP    EXTERNAL-IP
# node-1   Ready    master   192.168.1.10   203.0.113.10
# node-2   Ready    worker   192.168.1.11   203.0.113.11
# node-3   Ready    worker   192.168.1.12   203.0.113.12
```

### Acessar o Service

```bash
# Via IP externo do node
curl http://203.0.113.10:30080

# Via IP interno do node (de dentro da rede)
curl http://192.168.1.10:30080

# Qualquer node funciona!
curl http://203.0.113.11:30080
curl http://203.0.113.12:30080

# Via navegador
http://203.0.113.10:30080
```

## NodePort Allocation

### Automático (Recomendado)

Kubernetes atribui uma porta disponível automaticamente:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - port: 80         # Porta do ClusterIP
    targetPort: 8080 # Porta do container
    # nodePort: automático (30000-32767)
```

### Manual (Porta Específica)

Especificar NodePort manualmente:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # Porta específica
```

⚠️ **Cuidados**:
- Porta deve estar no range permitido (30000-32767)
- Porta não deve estar em uso
- Pode causar conflitos em outros ambientes

## ClusterIP Incluído

NodePort **automaticamente cria um ClusterIP**:

```bash
kubectl get svc myapp -o yaml
```

```yaml
spec:
  type: NodePort
  clusterIP: 10.96.0.10      # Criado automaticamente
  ports:
  - port: 80                 # Porta do ClusterIP
    targetPort: 8080         # Porta do Pod
    nodePort: 30080          # Porta do Node
```

**Acesso duplo**:
```bash
# De dentro do cluster (ClusterIP)
curl http://10.96.0.10:80
curl http://myapp:80

# De fora do cluster (NodePort)
curl http://203.0.113.10:30080
```

## Load Balancing

### Entre Nodes

Qualquer node pode receber tráfego e rotear para qualquer Pod:

```
Cliente → Node1:30080 → Pod em Node2
Cliente → Node2:30080 → Pod em Node3
Cliente → Node3:30080 → Pod em Node1
```

### Entre Pods

Tráfego é distribuído entre todos os Pods, independente do node:

```
Request 1 → Node1:30080 → Pod A (Node2)
Request 2 → Node2:30080 → Pod C (Node3)
Request 3 → Node1:30080 → Pod B (Node1)
```

## Políticas de Tráfego

### externalTrafficPolicy

Controla como tráfego externo é roteado:

```yaml
spec:
  type: NodePort
  externalTrafficPolicy: Cluster  # ou Local
```

#### Cluster (Padrão)

- ✅ Distribui para **qualquer Pod** no cluster
- ✅ Melhor balanceamento de carga
- ✅ Alta disponibilidade
- ❌ Perde IP do cliente original
- ❌ Hop extra entre nodes

```
Cliente → Node1 → Pod em Node2 (hop extra)
```

#### Local

- ✅ Apenas Pods **no mesmo node**
- ✅ Preserva IP do cliente
- ✅ Sem hops extras (menor latência)
- ❌ Pode causar desbalanceamento
- ❌ Se node não tem Pods, conexão falha

```
Cliente → Node1 → Pod em Node1 (apenas)
Cliente → Node2 → FALHA (se não tem Pods)
```

**Exemplo**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: NodePort
  externalTrafficPolicy: Local  # Preserva source IP
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

## Firewall e Segurança

### Abrir Portas no Firewall

NodePort requer que as portas estejam abertas no firewall:

```bash
# Ubuntu/Debian (ufw)
sudo ufw allow 30080/tcp

# CentOS/RHEL (firewalld)
sudo firewall-cmd --permanent --add-port=30080/tcp
sudo firewall-cmd --reload

# iptables
sudo iptables -A INPUT -p tcp --dport 30080 -j ACCEPT
```

### Cloud Provider Security Groups

**AWS**:
```bash
# Adicionar regra ao Security Group
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 30080 \
  --cidr 0.0.0.0/0
```

**GCP**:
```bash
# Criar firewall rule
gcloud compute firewall-rules create allow-nodeport \
  --allow tcp:30080 \
  --source-ranges 0.0.0.0/0
```

**Azure**:
```bash
# Adicionar regra NSG
az network nsg rule create \
  --resource-group myResourceGroup \
  --nsg-name myNSG \
  --name allow-nodeport \
  --protocol tcp \
  --priority 1000 \
  --destination-port-range 30080 \
  --access allow
```

### Restringir Acesso

```yaml
# Usar LoadBalancerSourceRanges (alguns providers)
spec:
  type: NodePort
  loadBalancerSourceRanges:
  - 203.0.113.0/24  # Apenas este range
  - 198.51.100.0/24
```

⚠️ Isso funciona apenas em alguns cloud providers!

## Integração com Load Balancer Externo

NodePort é comumente usado com load balancers externos:

### HAProxy

```bash
# /etc/haproxy/haproxy.cfg
frontend http_front
  bind *:80
  default_backend k8s_backend

backend k8s_backend
  balance roundrobin
  server node1 192.168.1.10:30080 check
  server node2 192.168.1.11:30080 check
  server node3 192.168.1.12:30080 check
```

### Nginx

```nginx
upstream k8s_backend {
  server 192.168.1.10:30080;
  server 192.168.1.11:30080;
  server 192.168.1.12:30080;
}

server {
  listen 80;
  location / {
    proxy_pass http://k8s_backend;
  }
}
```

## Multi-Port NodePort

Service pode expor múltiplas portas:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - name: http
    port: 80
    targetPort: 8080
    nodePort: 30080
  - name: https
    port: 443
    targetPort: 8443
    nodePort: 30443
  - name: metrics
    port: 9090
    targetPort: 9090
    nodePort: 30090
```

**Acesso**:
```bash
# HTTP
curl http://203.0.113.10:30080

# HTTPS
curl https://203.0.113.10:30443

# Metrics
curl http://203.0.113.10:30090/metrics
```

## Troubleshooting NodePort

### NodePort não acessível

```bash
# 1. Verificar Service foi criado
kubectl get svc myapp
# Verificar TYPE=NodePort e PORT(S) mostra mapeamento

# 2. Verificar Endpoints
kubectl get endpoints myapp
# Se vazio, selector não corresponde aos Pods

# 3. Verificar Pods estão Running e Ready
kubectl get pods -l app=myapp

# 4. Testar de dentro do cluster (ClusterIP)
kubectl run curl --image=curlimages/curl -i --tty --rm -- sh
curl http://myapp:80

# 5. Testar do próprio node
ssh node-1
curl localhost:30080

# 6. Verificar firewall
sudo ufw status
sudo iptables -L -n | grep 30080

# 7. Ver logs dos Pods
kubectl logs -l app=myapp
```

### Porta já em uso

```bash
# Erro: "provided port is already allocated"

# 1. Listar NodePorts em uso
kubectl get svc --all-namespaces -o json | \
  jq '.items[] | select(.spec.type=="NodePort") | {name:.metadata.name, namespace:.metadata.namespace, ports:.spec.ports[].nodePort}'

# 2. Escolher outra porta ou deixar automático
```

### Connection refused

```bash
# Possíveis causas:

# 1. Pods não estão prontos
kubectl get pods -l app=myapp
# Aguardar status Running e Ready

# 2. Readiness probe falhando
kubectl describe pod <pod-name>

# 3. Aplicação não escuta na porta correta
kubectl exec <pod-name> -- netstat -tlnp | grep 8080

# 4. targetPort incorreto no Service
kubectl get svc myapp -o yaml | grep targetPort
```

### Tráfego não balanceia

```bash
# 1. Verificar número de Endpoints
kubectl get endpoints myapp

# 2. Verificar sessionAffinity
kubectl get svc myapp -o yaml | grep sessionAffinity

# 3. Testar múltiplas requisições
for i in {1..10}; do curl http://203.0.113.10:30080; done
```

## Exemplos Práticos

### Aplicação Web Simples

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - name: http
    port: 80
    targetPort: 8080
    nodePort: 30080
```

**Acesso**:
```bash
curl http://<node-ip>:30080
```

### API com Metrics

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: NodePort
  selector:
    app: api
  ports:
  - name: api
    port: 8080
    targetPort: 8080
    nodePort: 30800
  - name: metrics
    port: 9090
    targetPort: 9090
    nodePort: 30900
```

### Com externalTrafficPolicy

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  type: NodePort
  externalTrafficPolicy: Local  # Preserva source IP
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

## Boas Práticas

### 1. Use LoadBalancer ou Ingress em Produção
```yaml
# ✅ Produção (cloud)
type: LoadBalancer

# ✅ Produção (HTTP/HTTPS)
ClusterIP + Ingress

# ⚠️ Apenas dev/bare-metal
type: NodePort
```

### 2. Deixe Kubernetes escolher a porta
```yaml
# ✅ Bom - evita conflitos
spec:
  type: NodePort
  ports:
  - port: 80

# ⚠️ Pode causar conflitos
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30080
```

### 3. Configure externalTrafficPolicy conforme necessário
```yaml
# Se precisa do IP do cliente
externalTrafficPolicy: Local

# Se precisa de melhor balanceamento
externalTrafficPolicy: Cluster  # padrão
```

### 4. Documente as portas usadas
```yaml
metadata:
  annotations:
    description: "Web application - NodePort 30080"
    exposed-port: "30080"
```

### 5. Use com load balancer externo
```yaml
# NodePort como backend de HAProxy/Nginx
type: NodePort
# Frontend: HAProxy/Nginx (porta 80/443)
# Backend: NodePort (30080)
```

### 6. Configure health checks nos Pods
```yaml
# NodePort só roteia para Pods Ready
readinessProbe:
  httpGet:
    path: /health
    port: 8080
```

### 7. Monitore disponibilidade
```bash
# Verificar se todas as portas NodePort estão acessíveis
for node in $(kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'); do
  curl -s -o /dev/null -w "%{http_code}" http://$node:30080 || echo "FAIL: $node"
done
```

## Comparação com Outros Tipos

| Aspecto | ClusterIP | NodePort | LoadBalancer |
|---------|-----------|----------|--------------|
| **Acesso** | Interno | Externo (Node) | Externo (LB) |
| **Porta** | Qualquer | 30000-32767 | Qualquer |
| **Firewall** | Não necessário | Necessário | Gerenciado |
| **Custo** | Grátis | Grátis | Pago (cloud) |
| **Produção** | ✅ Sim | ⚠️ Limitado | ✅ Sim |
| **Bare-metal** | - | ✅ Funciona | ❌ Requer setup |
| **HA** | ✅ | ⚠️ Depende | ✅ |

## Migração de NodePort para LoadBalancer

```bash
# 1. Mudança simples de tipo
kubectl patch svc myapp -p '{"spec":{"type":"LoadBalancer"}}'

# 2. Porta será preservada (80), NodePort será mantido mas não mais usado

# 3. Aguardar External-IP
kubectl get svc myapp -w

# 4. Atualizar DNS/clients para usar novo IP
```

## Monitoramento

### Métricas Prometheus

```promql
# Services do tipo NodePort
kube_service_spec_type{type="NodePort"}

# NodePorts específicos
kube_service_spec_node_port{service="myapp"}

# Endpoints disponíveis
kube_endpoint_address_available{service="myapp"}
```

### Health Check

```bash
#!/bin/bash
# health-check-nodeport.sh
SERVICE="myapp"
NODEPORT=$(kubectl get svc $SERVICE -o jsonpath='{.spec.ports[0].nodePort}')
NODES=$(kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}')

for node in $NODES; do
  if ! curl -sf http://$node:$NODEPORT/health > /dev/null; then
    echo "FAIL: $node:$NODEPORT"
    exit 1
  fi
done

echo "OK: All nodes responding"
```

## Links Úteis

- [Kubernetes Service - NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)
- [Publishing Services - Service Types](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)
- [Source IP for Services with Type=NodePort](https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-type-nodeport)
