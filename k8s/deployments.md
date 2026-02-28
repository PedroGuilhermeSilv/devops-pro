# Kubernetes Deployments

Deployments são o recurso mais comum para gerenciar aplicações stateless no Kubernetes. Eles fornecem atualizações declarativas para Pods e ReplicaSets, com capacidades de rollout, rollback e scaling.

## O que é um Deployment?

Um Deployment gerencia ReplicaSets que, por sua vez, gerenciam Pods. Ele fornece:

- ✅ **Declarative Updates**: Descreve o estado desejado e o Kubernetes faz acontecer
- ✅ **Rolling Updates**: Atualiza Pods gradualmente sem downtime
- ✅ **Rollback**: Reverte para versões anteriores facilmente
- ✅ **Scaling**: Aumenta ou diminui réplicas facilmente
- ✅ **Self-Healing**: Recria Pods que falham automaticamente
- ✅ **Histórico de Revisões**: Mantém histórico de alterações

## Hierarquia

```
Deployment
    ↓ (gerencia)
ReplicaSet
    ↓ (gerencia)
Pods
```

## Estratégias de Deployment

### 1. RollingUpdate (Padrão)
Atualiza Pods gradualmente, mantendo disponibilidade.

**Vantagens**:
- Zero downtime
- Rollback rápido se houver problemas
- Controle fino sobre o processo

**Desvantagens**:
- Versões antiga e nova rodam simultaneamente
- Mais lento que Recreate

**Parâmetros**:
- `maxSurge`: Máximo de Pods extras durante update (default: 25%)
- `maxUnavailable`: Máximo de Pods indisponíveis durante update (default: 25%)

### 2. Recreate
Termina todos os Pods antigos antes de criar novos.

**Vantagens**:
- Simples e rápido
- Apenas uma versão roda por vez

**Desvantagens**:
- Downtime durante a atualização
- Não adequado para produção na maioria dos casos

**Quando usar**:
- Aplicações que não suportam múltiplas versões simultaneamente
- Desenvolvimento/staging
- Migrações de banco de dados que requerem downtime

## Ciclo de Vida do Deployment

```
Create → Progressing → Complete
                ↓
            Failed (pode fazer Rollback)
```

### Conditions

| Condition | Descrição |
|-----------|-----------|
| `Progressing` | Deployment está executando rollout |
| `Complete` | Todas as réplicas foram atualizadas e estão disponíveis |
| `Failed` | Rollout falhou (ex: ImagePullBackOff, CrashLoopBackOff) |
| `Available` | Deployment tem pelo menos minReadySeconds disponível |

## Comandos Úteis

```bash
# Criar Deployment
kubectl create deployment nginx --image=nginx:1.21
kubectl apply -f deployment.yaml

# Listar Deployments
kubectl get deployments
kubectl get deploy -o wide

# Descrever Deployment
kubectl describe deployment <name>

# Escalar Deployment
kubectl scale deployment <name> --replicas=5

# Autoscaling
kubectl autoscale deployment <name> --min=2 --max=10 --cpu-percent=80

# Atualizar imagem
kubectl set image deployment/<name> container-name=nginx:1.22
kubectl edit deployment <name>

# Ver status do rollout
kubectl rollout status deployment/<name>

# Ver histórico de revisões
kubectl rollout history deployment/<name>
kubectl rollout history deployment/<name> --revision=2

# Fazer rollback
kubectl rollout undo deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=2

# Pausar/Retomar rollout
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>

# Restart (recria todos os Pods)
kubectl rollout restart deployment/<name>

# Ver ReplicaSets gerenciados
kubectl get rs -l app=<label>

# Deletar Deployment
kubectl delete deployment <name>
kubectl delete -f deployment.yaml

# Ver logs de todos os Pods
kubectl logs -l app=<label> --all-containers=true
kubectl logs -l app=<label> -f  # follow

# Ver recursos utilizados
kubectl top pods -l app=<label>
```

## Rolling Update - Como Funciona

Exemplo com 3 réplicas, `maxSurge=1` e `maxUnavailable=1`:

```
Inicial: [v1] [v1] [v1]

Passo 1: [v1] [v1] [v1] [v2]  (cria 1 novo - maxSurge)
Passo 2: [v1] [v1] [v2]       (termina 1 antigo - maxUnavailable)
Passo 3: [v1] [v1] [v2] [v2]  (cria 1 novo)
Passo 4: [v1] [v2] [v2]       (termina 1 antigo)
Passo 5: [v1] [v2] [v2] [v2]  (cria 1 novo)
Passo 6: [v2] [v2] [v2]       (termina último antigo)

Final: [v2] [v2] [v2]
```

Durante todo o processo:
- Mínimo disponível: 2 Pods (3 - maxUnavailable 1)
- Máximo total: 4 Pods (3 + maxSurge 1)

## Deployment vs StatefulSet vs DaemonSet

| Característica | Deployment | StatefulSet | DaemonSet |
|----------------|-----------|-------------|-----------|
| **Use case** | Apps stateless | Apps stateful | Agentes de sistema |
| **Pod names** | Aleatórios | Previsíveis (app-0, app-1) | Um por node |
| **Ordem de criação** | Paralela | Sequential | Paralela |
| **Persistent storage** | Não garantido | Garantido por Pod | Possível |
| **Network identity** | Dinâmico | Estável | Por node |
| **Scaling** | Qualquer ordem | Ordenado | Automático (1/node) |
| **Updates** | Rolling/Recreate | Rolling/OnDelete | Rolling/OnDelete |

**Exemplos**:
- **Deployment**: APIs REST, web servers, workers stateless
- **StatefulSet**: Bancos de dados, Kafka, Zookeeper, Elasticsearch
- **DaemonSet**: Log collectors, monitoring agents, network plugins

## Progressive Delivery

### Blue-Green Deployment
Duas versões completas, troca instantânea via Service.

```bash
# Criar deployment blue
kubectl apply -f app-blue.yaml

# Criar deployment green
kubectl apply -f app-green.yaml

# Service aponta para blue
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'

# Trocar para green
kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'
```

### Canary Deployment
Libera nova versão para pequeno percentual de tráfego.

```bash
# 90% da versão estável (9 réplicas)
kubectl scale deployment app-stable --replicas=9

# 10% da versão canary (1 réplica)
kubectl scale deployment app-canary --replicas=1

# Service roteia para ambos
# Monitore métricas e gradualmente aumente canary
```

## Troubleshooting

### Deployment não progride (stuck)

```bash
# Ver status detalhado
kubectl describe deployment <name>

# Verificar eventos
kubectl get events --sort-by=.metadata.creationTimestamp

# Comum: Pod não fica Ready
kubectl get pods -l app=<label>
kubectl logs <pod-name>
kubectl describe pod <pod-name>

# Verificar readiness probe
# Verificar recursos insuficientes
# Verificar imagem pull errors
```

### Rollout muito lento

```bash
# Ajustar progressDeadlineSeconds
kubectl patch deployment <name> -p '{"spec":{"progressDeadlineSeconds":600}}'

# Aumentar maxSurge e maxUnavailable para rollouts mais rápidos
# (trade-off: menos controle, mais recursos temporários)
```

### Rollback não funciona

```bash
# Ver histórico
kubectl rollout history deployment/<name>

# Histórico limitado por revisionHistoryLimit (default: 10)
# Se revision foi removida, não pode fazer rollback

# Verificar se há erro de configuração no deployment
kubectl describe deployment <name>
```

### Pods não são criados

```bash
# Verificar quotas de recursos
kubectl describe resourcequota -n <namespace>

# Verificar LimitRange
kubectl describe limitrange -n <namespace>

# Verificar disponibilidade de nodes
kubectl get nodes
kubectl describe node <node-name>
```

## Boas Práticas

### 1. **Use Labels Adequadas**
```yaml
metadata:
  labels:
    app: myapp
    version: v1.2.3
    component: backend
    tier: api
    managed-by: helm
```

### 2. **Configure Resource Requests e Limits**
```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### 3. **Implemente Health Checks**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

### 4. **Configure minReadySeconds**
Evita rollout rápido de versões com bugs que só aparecem após alguns segundos:
```yaml
spec:
  minReadySeconds: 30
```

### 5. **Ajuste progressDeadlineSeconds**
Tempo máximo para rollout antes de marcar como falho:
```yaml
spec:
  progressDeadlineSeconds: 600  # 10 minutos
```

### 6. **Mantenha Histórico de Revisões**
```yaml
spec:
  revisionHistoryLimit: 10  # default, ajuste conforme necessário
```

### 7. **Use Rolling Update com Valores Adequados**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # ou 25%
    maxUnavailable: 0  # zero downtime
```

### 8. **Configure Graceful Shutdown**
```yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: app
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
```

### 9. **Use Pod Disruption Budget**
Garante disponibilidade mínima durante manutenções:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
```

### 10. **Configure Node Affinity/Anti-Affinity**
Distribui Pods em diferentes nodes/zones para alta disponibilidade:
```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - myapp
        topologyKey: kubernetes.io/hostname
```

## Estratégias Avançadas

### Canary com Flagger (Progressive Delivery)
```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  progressDeadlineSeconds: 60
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
    - name: request-duration
      thresholdRange:
        max: 500
```

### A/B Testing
Use Ingress/Service Mesh para rotear baseado em headers:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-canary
            port:
              number: 80
```

## Monitoramento

### Métricas Importantes
```bash
# Taxa de disponibilidade
kubectl get deployment <name> -o jsonpath='{.status.availableReplicas}/{.status.replicas}'

# Status do rollout
kubectl rollout status deployment/<name>

# Ver annotations com change-cause
kubectl annotate deployment/<name> kubernetes.io/change-cause="Update to v1.2.3"
kubectl rollout history deployment/<name>
```

### Prometheus Metrics
```promql
# Réplicas disponíveis
kube_deployment_status_replicas_available{deployment="myapp"}

# Réplicas desejadas vs atuais
kube_deployment_spec_replicas{deployment="myapp"}
kube_deployment_status_replicas{deployment="myapp"}

# Deployments com réplicas insuficientes
kube_deployment_status_replicas_available / kube_deployment_spec_replicas < 1
```

## Links Úteis

- [Kubernetes Deployments Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Rolling Update Strategy](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)
- [Blue-Green Deployments](https://kubernetes.io/blog/2018/04/30/zero-downtime-deployment-kubernetes-jenkins/)
- [Flagger - Progressive Delivery](https://flagger.app/)
