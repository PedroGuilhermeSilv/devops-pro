# Kubernetes Service - LoadBalancer

LoadBalancer é um tipo de Service que provisiona um load balancer externo (geralmente fornecido pelo cloud provider) e expõe o Service externamente através de um IP público.

## O que é LoadBalancer?

- ☁️ **Cloud-native**: Integra com load balancers do cloud provider
- 🌍 **IP público**: Recebe IP externo acessível da internet
- 🔄 **NodePort + ClusterIP incluídos**: Cria toda a cadeia automaticamente
- 💰 **Custo**: Geralmente cobrado pelo cloud provider
- 🎯 **Caso de uso**: Produção, aplicações públicas, alta disponibilidade

## Características

| Característica | Descrição |
|----------------|-----------|
| **Visibilidade** | Externa (internet) |
| **IP** | Público, provisionado pelo cloud provider |
| **DNS** | Pode configurar DNS para o IP externo |
| **Porta** | Qualquer porta (80, 443, etc.) |
| **Custo** | Pago por hora/mês (cloud providers) |
| **HA** | Alta disponibilidade gerenciada |
| **Health Check** | Automático pelo load balancer |

## Arquitetura

```
┌─────────────────────────────────────────────────┐
│                   Internet                       │
└────────────────────┬────────────────────────────┘
                     │
              ┌──────▼─────┐
              │ Load       │  External IP
              │ Balancer   │  (203.0.113.50)
              │ (Cloud)    │
              └──────┬─────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼────┐    ┌─────▼────┐    ┌────▼────┐
│ Node 1  │    │ Node 2   │    │ Node 3  │
│ :30080  │    │ :30080   │    │ :30080  │
└────┬────┘    └─────┬────┘    └────┬────┘
     │               │               │
     └───────────────┼───────────────┘
                     │
             ┌───────▼───────┐
             │ LoadBalancer  │  ClusterIP
             │   Service     │  (10.96.0.10)
             │   :80         │
             └───────┬───────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼────┐    ┌─────▼────┐    ┌────▼────┐
│ Pod A   │    │ Pod B    │    │ Pod C   │
│ :8080   │    │ :8080    │    │ :8080   │
└─────────┘    └──────────┘    └─────────┘
```

## Fluxo de Tráfego

```
Cliente (Internet)
    ↓
External Load Balancer (203.0.113.50:80)
    ↓
NodePort (Node1:30080, Node2:30080, Node3:30080)
    ↓
ClusterIP (10.96.0.10:80)
    ↓
Pod (10.1.1.5:8080, 10.1.1.6:8080, 10.1.1.7:8080)
```

## Quando Usar LoadBalancer

### ✅ Use LoadBalancer para:

1. **Produção em cloud**
   - AWS, GCP, Azure, DigitalOcean
   - Aplicações públicas na internet
   - APIs externas

2. **Alta disponibilidade**
   - Load balancing automático
   - Health checks integrados
   - Failover automático

3. **Serviços não-HTTP**
   - Bancos de dados externos
   - Serviços TCP/UDP customizados
   - Quando Ingress não é adequado

4. **IP estático externo**
   - Aplicações que precisam de IP fixo
   - Integrações com serviços externos
   - DNS apontando para IP fixo

### ❌ NÃO use LoadBalancer quando:

- **Múltiplos serviços HTTP** → Use **Ingress** (mais econômico)
- **Apenas acesso interno** → Use **ClusterIP**
- **Desenvolvimento local** → Use **NodePort** ou port-forward
- **Cluster on-premise sem suporte** → Use **NodePort** + LB externo
- **Custo é preocupação** → Use **Ingress** (1 LB para N services)

## Cloud Providers

### AWS (ELB/ALB/NLB)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  annotations:
    # Classic Load Balancer (legado)
    service.beta.kubernetes.io/aws-load-balancer-type: "classic"
    
    # Network Load Balancer (recomendado para TCP)
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    
    # Application Load Balancer (HTTP/HTTPS)
    service.beta.kubernetes.io/aws-load-balancer-type: "alb"
    
    # Interno (VPC apenas)
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
    
    # SSL Certificate (ARN)
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:..."
    
    # Health check path
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/health"
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

**Tipos de Load Balancer**:
| Tipo | Camada | Uso | Custo |
|------|--------|-----|-------|
| Classic (CLB) | L4/L7 | Legado | Médio |
| Network (NLB) | L4 | Alto throughput, TCP/UDP | Mais caro |
| Application (ALB) | L7 | HTTP/HTTPS, path-based routing | Médio |

### GCP (Cloud Load Balancing)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  annotations:
    # Interno (VPC apenas)
    cloud.google.com/load-balancer-type: "Internal"
    
    # Backend configuration
    cloud.google.com/backend-config: '{"default": "my-backend-config"}'
    
    # Firewall rules
    cloud.google.com/firewall-rule-name: "allow-myapp"
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

**GCP Load Balancer**:
- Externo: Global (anycast IP) ou Regional
- Interno: Regional apenas
- Health checks automáticos
- Suporte HTTP/HTTPS, TCP, UDP

### Azure (Azure Load Balancer)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  annotations:
    # Interno (VNet apenas)
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    
    # Subnet específica
    service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "mysubnet"
    
    # IP específico
    service.beta.kubernetes.io/azure-load-balancer-ipv4: "20.30.40.50"
    
    # Resource group
    service.beta.kubernetes.io/azure-load-balancer-resource-group: "myResourceGroup"
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

**Azure Load Balancer**:
- Basic: Gratuito, funcionalidades limitadas
- Standard: Pago, HA, availability zones, mais backends

### DigitalOcean

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  annotations:
    # Health check path
    service.beta.kubernetes.io/do-loadbalancer-healthcheck-path: "/health"
    
    # Health check protocol
    service.beta.kubernetes.io/do-loadbalancer-healthcheck-protocol: "http"
    
    # Certificado SSL (ID)
    service.beta.kubernetes.io/do-loadbalancer-certificate-id: "cert-id"
    
    # Redirect HTTP para HTTPS
    service.beta.kubernetes.io/do-loadbalancer-redirect-http-to-https: "true"
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8080
```

## LoadBalancer em Ambientes Locais

### MetalLB (Bare-Metal)

Para clusters on-premise sem cloud provider:

```bash
# Instalar MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

# Configurar pool de IPs
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default
EOF
```

Agora LoadBalancer Services funcionam em bare-metal!

### Minikube

```bash
# Habilitar tunnel (terminal precisa ficar aberto)
minikube tunnel

# Services LoadBalancer receberão IPs (127.0.0.1 ou similar)
```

### Kind

```bash
# Kind não suporta LoadBalancer nativamente
# Use NodePort ou configure MetalLB
```

## IP Externo

### Automático (Padrão)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

Após criar, aguarde o IP externo:

```bash
kubectl get svc myapp -w
# NAME    TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)
# myapp   LoadBalancer   10.96.0.10    <pending>     80:30080/TCP
# myapp   LoadBalancer   10.96.0.10    203.0.113.50  80:30080/TCP
```

### IP Específico (Reservado)

```yaml
spec:
  type: LoadBalancer
  loadBalancerIP: 203.0.113.50  # IP previamente reservado
```

⚠️ **Nota**: 
- IP deve ser reservado no cloud provider primeiro
- Nem todos os providers suportam
- Campo `loadBalancerIP` está deprecated (use annotations do provider)

## Políticas de Tráfego

### externalTrafficPolicy

Controla como tráfego externo é roteado:

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Cluster  # ou Local
```

#### Cluster (Padrão)

- ✅ Melhor balanceamento de carga
- ✅ Alta disponibilidade
- ✅ Tráfego roteado para qualquer Pod no cluster
- ❌ Perde IP do cliente (SNAT)
- ❌ Hop extra entre nodes

```
Load Balancer → Node1 → Pod em Node2 (hop extra)
```

#### Local

- ✅ Preserva IP do cliente
- ✅ Sem hops extras (menor latência)
- ✅ Reduz tráfego inter-node
- ❌ Pode causar desbalanceamento
- ❌ Se node não tem Pods, health check falha

```
Load Balancer → Node1 → Pod em Node1 (apenas local)
Load Balancer → Node2 → Fail (se não tem Pods)
```

**Recomendação**: Use `Local` se precisar do IP do cliente (logs, analytics, geo-location).

## Health Checks

Load balancers fazem health checks automáticos:

### Configuração no Pod (Readiness Probe)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0
        readinessProbe:
          httpGet:
            path: /health      # Endpoint de health check
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
```

### Health Check do Load Balancer

Cada cloud provider configura automaticamente:

**AWS NLB**:
```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: "HTTP"
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/health"
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "10"
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "5"
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "2"
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "2"
```

## LoadBalancer Source Ranges

Restringir IPs que podem acessar o LoadBalancer:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: LoadBalancer
  loadBalancerSourceRanges:
  - 203.0.113.0/24    # Apenas este range
  - 198.51.100.0/24   # E este
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

**Implementação**:
- AWS: Security Group rules
- GCP: Firewall rules
- Azure: Network Security Group rules

⚠️ Suporte varia por cloud provider!

## SSL/TLS Termination

### Opção 1: Load Balancer (Recomendado)

Certificado configurado no Load Balancer:

```yaml
# AWS
annotations:
  service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:us-east-1:123456:certificate/abc123"
  service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
  service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
spec:
  ports:
  - name: https
    port: 443
    targetPort: 8080  # Backend é HTTP
```

**Vantagens**:
- Offload de SSL (menos CPU nos Pods)
- Certificados gerenciados centralmente
- Renovação automática (com ACM, Let's Encrypt)

### Opção 2: Pod (End-to-End Encryption)

```yaml
spec:
  ports:
  - name: https
    port: 443
    targetPort: 8443  # Pod escuta HTTPS
```

**Vantagens**:
- Criptografia end-to-end
- Mais seguro (tráfego interno criptografado)

**Desvantagens**:
- Mais overhead nos Pods
- Gerenciamento de certificados mais complexo

## Multi-Port LoadBalancer

Expor múltiplas portas:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  - name: grpc
    port: 50051
    targetPort: 50051
```

**Acesso**:
```bash
# HTTP
curl http://203.0.113.50:80

# HTTPS
curl https://203.0.113.50:443

# gRPC
grpcurl 203.0.113.50:50051 list
```

## Troubleshooting LoadBalancer

### External-IP fica em <pending>

```bash
# Verificar status
kubectl describe svc myapp

# Causas comuns:
# 1. Cloud provider não configurado
kubectl get events --sort-by=.metadata.creationTimestamp

# 2. Quotas excedidas (AWS, GCP, Azure)
# Verificar quotas no console do cloud provider

# 3. Permissões insuficientes
# Service account do cluster precisa de permissões para criar LBs

# 4. Cluster local sem suporte
# Instalar MetalLB ou usar NodePort
```

### LoadBalancer criado mas não acessa

```bash
# 1. Verificar External-IP
kubectl get svc myapp
EXTERNAL_IP=$(kubectl get svc myapp -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# 2. Testar conectividade
curl http://$EXTERNAL_IP

# 3. Verificar Pods estão Running e Ready
kubectl get pods -l app=myapp

# 4. Verificar Endpoints
kubectl get endpoints myapp

# 5. Verificar Security Groups/Firewall
# AWS: Security Groups do Load Balancer
# GCP: Firewall rules
# Azure: Network Security Groups

# 6. Verificar health checks
kubectl describe svc myapp | grep -i health

# 7. Testar de dentro do cluster (ClusterIP)
kubectl run curl --image=curlimages/curl -i --tty --rm -- sh
curl http://myapp
```

### Health checks falhando

```bash
# 1. Verificar readiness probe dos Pods
kubectl describe pod <pod-name> | grep -A 10 Readiness

# 2. Verificar endpoint de health
kubectl exec <pod-name> -- curl http://localhost:8080/health

# 3. Verificar logs do Pod
kubectl logs <pod-name>

# 4. Verificar configuração do health check no LB
# AWS Console → Load Balancer → Health checks
# GCP Console → Load Balancer → Backend services

# 5. Ajustar thresholds
kubectl annotate svc myapp \
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold=2
```

### Custo muito alto

```bash
# Problema: Múltiplos LoadBalancer Services

# Solução: Usar Ingress
# 1 LoadBalancer + Ingress Controller
# N Services (ClusterIP) → Ingress → 1 LoadBalancer

# Exemplo: 10 Services LoadBalancer = 10 LBs
# Com Ingress: 10 Services ClusterIP + 1 Ingress = 1 LB

# Economia: ~90% (dependendo do provider)
```

## Migração para Ingress

Reduzir custos usando Ingress ao invés de múltiplos LoadBalancers:

```yaml
# Antes: 3 LoadBalancer Services (3 LBs = $$$$)
---
apiVersion: v1
kind: Service
metadata:
  name: app1
spec:
  type: LoadBalancer  # LB #1
  selector:
    app: app1
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: app2
spec:
  type: LoadBalancer  # LB #2
  selector:
    app: app2
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: app3
spec:
  type: LoadBalancer  # LB #3
  selector:
    app: app3
  ports:
  - port: 80
```

```yaml
# Depois: ClusterIP + Ingress (1 LB = $)
---
apiVersion: v1
kind: Service
metadata:
  name: app1
spec:
  type: ClusterIP  # Interno
  selector:
    app: app1
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: app2
spec:
  type: ClusterIP
  selector:
    app: app2
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: app3
spec:
  type: ClusterIP
  selector:
    app: app3
  ports:
  - port: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: apps-ingress
spec:
  rules:
  - host: app1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1
            port:
              number: 80
  - host: app2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2
            port:
              number: 80
  - host: app3.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app3
            port:
              number: 80
```

**Economia**: De 3 Load Balancers para 1 (do Ingress Controller)!

## Boas Práticas

### 1. Use Ingress para HTTP/HTTPS
```yaml
# ❌ Caro - LoadBalancer para cada app HTTP
type: LoadBalancer

# ✅ Econômico - 1 LoadBalancer + Ingress
type: ClusterIP + Ingress
```

### 2. Configure externalTrafficPolicy conforme necessidade
```yaml
# Se precisa IP do cliente
externalTrafficPolicy: Local

# Se precisa melhor balanceamento
externalTrafficPolicy: Cluster
```

### 3. Implemente health checks robustos
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

### 4. Use loadBalancerSourceRanges para segurança
```yaml
loadBalancerSourceRanges:
- 203.0.113.0/24  # Apenas IPs conhecidos
```

### 5. Reserve IPs para produção
```bash
# AWS
aws ec2 allocate-address --domain vpc

# GCP
gcloud compute addresses create myapp-ip --region=us-central1

# Use IP reservado no Service
```

### 6. Configure SSL no Load Balancer
```yaml
# Offload SSL no LB (melhor performance)
annotations:
  service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:..."
```

### 7. Monitore custos
```bash
# Auditar Services LoadBalancer
kubectl get svc --all-namespaces -o json | \
  jq '.items[] | select(.spec.type=="LoadBalancer") | {name:.metadata.name, namespace:.metadata.namespace}'

# Considere consolidar com Ingress
```

### 8. Use annotations do cloud provider
```yaml
# Aproveite recursos específicos (health checks, SSL, etc)
annotations:
  service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/health"
```

## Comparação dos Tipos

| Aspecto | ClusterIP | NodePort | LoadBalancer |
|---------|-----------|----------|--------------|
| **Acesso** | Interno | Externo (Node) | Externo (Internet) |
| **IP** | Cluster interno | Node IP | IP público |
| **HA** | Via cluster | Depende | Gerenciado |
| **Custo** | Grátis | Grátis | Pago |
| **Produção** | ✅ | ⚠️ | ✅ |
| **Cloud** | Qualquer | Qualquer | Requer suporte |
| **Health Check** | K8s | K8s | LB + K8s |

## Monitoramento

### Métricas Prometheus

```promql
# Services LoadBalancer
kube_service_spec_type{type="LoadBalancer"}

# External IPs atribuídos
kube_service_status_load_balancer_ingress

# Services sem External-IP (alerta!)
kube_service_spec_type{type="LoadBalancer"} 
  unless on(service,namespace) 
  kube_service_status_load_balancer_ingress
```

### Custos

```bash
# AWS: ver custos de ELB/ALB/NLB no Cost Explorer
# GCP: ver custos de Load Balancing no Billing
# Azure: ver custos de Load Balancer no Cost Management
```

## Links Úteis

- [Kubernetes Service - LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [GKE LoadBalancer Service](https://cloud.google.com/kubernetes-engine/docs/concepts/service-load-balancer)
- [Azure Load Balancer](https://learn.microsoft.com/en-us/azure/aks/load-balancer-standard)
- [MetalLB](https://metallb.universe.tf/)
