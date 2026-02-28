# Kubernetes ReplicaSets

ReplicaSet é um controlador que garante que um número especificado de réplicas de Pod estejam rodando a qualquer momento. É a próxima geração do ReplicationController (deprecated).

## O que é um ReplicaSet?

Um ReplicaSet mantém um conjunto estável de Pods réplicas rodando a qualquer momento. É usado para garantir a disponibilidade de um número específico de Pods idênticos.

**Características**:
- ✅ Garante número desejado de réplicas
- ✅ Self-healing: recria Pods que falham
- ✅ Scaling horizontal
- ✅ Seletor de labels flexível (set-based)
- ⚠️ **Não fornece rolling updates** - use Deployments!

## Hierarquia

```
Deployment (recomendado)
    ↓ (gerencia)
ReplicaSet
    ↓ (gerencia)
Pods
```

## ReplicaSet vs ReplicationController

| Característica | ReplicaSet | ReplicationController (deprecated) |
|----------------|-----------|-----------------------------------|
| Seletor | Set-based (`In`, `NotIn`, `Exists`) | Equality-based (`=`, `!=`) |
| Flexibilidade | Alta | Limitada |
| Status | Atual e recomendado | Deprecated |
| API Group | `apps/v1` | `v1` |

**Exemplo de Seletor Set-based**:
```yaml
selector:
  matchExpressions:
    - key: environment
      operator: In
      values:
        - production
        - staging
```

## Como Funciona

ReplicaSet monitora constantemente os Pods no cluster:

1. **Conta os Pods** que correspondem ao seu seletor
2. **Compara** com o número desejado de réplicas
3. **Cria novos Pods** se houver menos que o desejado
4. **Deleta Pods** se houver mais que o desejado

```
Desejado: 3
Atual: 2
Ação: Criar 1 novo Pod

Desejado: 3
Atual: 4
Ação: Deletar 1 Pod
```

## Quando Usar (ou não usar)

### ❌ NÃO use ReplicaSets diretamente quando:
- Precisa de rolling updates → Use **Deployment**
- Precisa de rollback → Use **Deployment**
- Precisa de histórico de versões → Use **Deployment**
- Aplicação stateful → Use **StatefulSet**
- Precisa de um Pod por node → Use **DaemonSet**
- Tarefa que roda até completar → Use **Job**

### ✅ Use ReplicaSets diretamente quando:
- Precisa de controle muito granular sobre Pods
- Está criando seu próprio controlador customizado
- Casos muito específicos onde Deployment não serve
- **Raramente necessário na prática!**

> **Recomendação**: Use **Deployments** ao invés de ReplicaSets diretamente em 99% dos casos!

## Comandos Úteis

```bash
# Criar ReplicaSet
kubectl apply -f replicaset.yaml

# Listar ReplicaSets
kubectl get replicasets
kubectl get rs
kubectl get rs -o wide

# Descrever ReplicaSet
kubectl describe rs <name>

# Escalar ReplicaSet
kubectl scale rs/<name> --replicas=5

# Autoscaling
kubectl autoscale rs/<name> --min=2 --max=10 --cpu-percent=80

# Ver Pods gerenciados
kubectl get pods -l app=<label>

# Editar ReplicaSet
kubectl edit rs/<name>

# Ver YAML
kubectl get rs/<name> -o yaml

# Deletar ReplicaSet
kubectl delete rs/<name>
kubectl delete -f replicaset.yaml

# Deletar ReplicaSet mas manter Pods
kubectl delete rs/<name> --cascade=orphan

# Ver eventos
kubectl get events --field-selector involvedObject.name=<name>

# Ver métricas
kubectl top pods -l app=<label>
```

## ReplicaSet Template

O `template` dentro do ReplicaSet define como os Pods devem ser criados:

```yaml
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:  # Pod template
    metadata:
      labels:
        app: myapp  # DEVE corresponder ao selector
    spec:
      containers:
      - name: app
        image: nginx
```

**Importante**: Labels no `template.metadata.labels` **DEVEM** corresponder ao `selector`!

## Scaling

### Manual Scaling

```bash
# Via comando
kubectl scale rs/myapp --replicas=5

# Via patch
kubectl patch rs/myapp -p '{"spec":{"replicas":5}}'

# Via edit
kubectl edit rs/myapp
# Altere spec.replicas e salve
```

### Horizontal Pod Autoscaler (HPA)

```bash
# Criar HPA baseado em CPU
kubectl autoscale rs/myapp --min=2 --max=10 --cpu-percent=80

# Ver HPAs
kubectl get hpa

# Descrever HPA
kubectl describe hpa myapp

# YAML do HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: ReplicaSet
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

## Seletores

### Equality-based (simples)
```yaml
selector:
  matchLabels:
    app: myapp
    tier: frontend
# Equivalente a: app=myapp AND tier=frontend
```

### Set-based (avançado)
```yaml
selector:
  matchExpressions:
    - key: app
      operator: In
      values:
        - myapp
        - yourapp
    - key: environment
      operator: NotIn
      values:
        - dev
    - key: tier
      operator: Exists
# app IN (myapp, yourapp) AND environment NOT IN (dev) AND tier EXISTS
```

### Operadores Disponíveis
- `In`: Label está na lista de valores
- `NotIn`: Label não está na lista de valores
- `Exists`: Label existe (não importa o valor)
- `DoesNotExist`: Label não existe

## Adotando Pods Órfãos

ReplicaSet pode "adotar" Pods que correspondem ao seu seletor:

```bash
# Criar Pods manualmente
kubectl run pod1 --image=nginx --labels=app=myapp
kubectl run pod2 --image=nginx --labels=app=myapp

# Criar ReplicaSet com selector app=myapp e replicas=3
kubectl apply -f replicaset.yaml

# ReplicaSet adota os 2 Pods existentes e cria apenas 1 novo
kubectl get pods -l app=myapp
# pod1, pod2, myapp-xxxxx
```

## Diferença entre ownerReferences

### Pod gerenciado por ReplicaSet
```yaml
metadata:
  ownerReferences:
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: myapp-rs
    uid: ...
```

### ReplicaSet gerenciado por Deployment
```yaml
metadata:
  ownerReferences:
  - apiVersion: apps/v1
    kind: Deployment
    name: myapp
    uid: ...
```

**Implicação**: Deletar o owner deleta os recursos filho (cascading delete).

## Troubleshooting

### Pods não são criados

```bash
# Verificar status do ReplicaSet
kubectl describe rs/<name>

# Verificar eventos
kubectl get events --sort-by=.metadata.creationTimestamp

# Causas comuns:
# - Seletor não corresponde às labels do template
# - Recursos insuficientes no cluster
# - Quotas de namespace excedidas
# - Node taints impedindo schedule
```

### ReplicaSet cria Pods indefinidamente

```bash
# Verificar se os Pods estão crashando
kubectl get pods -l app=<label>

# Se Pods estão em CrashLoopBackOff, ReplicaSet continua tentando
# Solução: corrigir a imagem/configuração do Pod
```

### Pods não são deletados ao reduzir réplicas

```bash
# Verificar finalizers no Pod
kubectl get pod <pod-name> -o yaml | grep finalizers -A 5

# Pods com finalizers não são deletados até que sejam removidos
# Remover finalizer manualmente (CUIDADO!)
kubectl patch pod <pod-name> -p '{"metadata":{"finalizers":null}}'
```

### ReplicaSet não respeita labels do Selector

```bash
# Verificar se labels do template correspondem ao selector
kubectl get rs/<name> -o yaml

# spec.selector.matchLabels DEVE ser subset de spec.template.metadata.labels
```

## ReplicaSet vs Deployment - Comparação Detalhada

| Funcionalidade | ReplicaSet | Deployment |
|----------------|-----------|-----------|
| Mantém réplicas | ✅ | ✅ |
| Self-healing | ✅ | ✅ |
| Scaling | ✅ | ✅ |
| Rolling updates | ❌ | ✅ |
| Rollback | ❌ | ✅ |
| Update strategy | ❌ | ✅ (Rolling/Recreate) |
| Histórico de versões | ❌ | ✅ |
| Pause/Resume rollout | ❌ | ✅ |
| Gerencia ReplicaSets | ❌ | ✅ |
| Uso recomendado | Raramente | Sempre |

### Exemplo: Update sem Deployment

**Com ReplicaSet (manual, complexo)**:
```bash
# 1. Criar novo ReplicaSet com nova versão
kubectl apply -f replicaset-v2.yaml

# 2. Escalar novo ReplicaSet
kubectl scale rs/myapp-v2 --replicas=3

# 3. Escalar antigo ReplicaSet
kubectl scale rs/myapp-v1 --replicas=0

# 4. Deletar antigo ReplicaSet
kubectl delete rs/myapp-v1

# Se algo der errado, processo reverso manual!
```

**Com Deployment (automático, simples)**:
```bash
# 1. Update
kubectl set image deployment/myapp app=nginx:1.22

# Se algo der errado:
kubectl rollout undo deployment/myapp

# Pronto!
```

## Boas Práticas

### 1. **Use Deployments ao invés de ReplicaSets**
```bash
# Ao invés de:
kubectl apply -f replicaset.yaml

# Use:
kubectl apply -f deployment.yaml
```

### 2. **Se usar ReplicaSets, sempre defina Resources**
```yaml
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
```

### 3. **Use Labels consistentes**
```yaml
metadata:
  labels:
    app: myapp
    version: v1.0.0
spec:
  selector:
    matchLabels:
      app: myapp
      version: v1.0.0
  template:
    metadata:
      labels:
        app: myapp
        version: v1.0.0
```

### 4. **Configure Pod Disruption Budget**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: myapp
```

### 5. **Monitore o número de réplicas**
```bash
# Via kubectl
kubectl get rs -w

# Via Prometheus
kube_replicaset_status_replicas{replicaset="myapp"}
kube_replicaset_spec_replicas{replicaset="myapp"}
```

## Controladores Customizados

ReplicaSets são úteis quando você está construindo seus próprios controladores:

```go
// Exemplo em Go (simplificado)
// Criar ReplicaSet programaticamente
rs := &appsv1.ReplicaSet{
    ObjectMeta: metav1.ObjectMeta{
        Name: "myapp",
    },
    Spec: appsv1.ReplicaSetSpec{
        Replicas: int32Ptr(3),
        Selector: &metav1.LabelSelector{
            MatchLabels: map[string]string{
                "app": "myapp",
            },
        },
        Template: corev1.PodTemplateSpec{
            ObjectMeta: metav1.ObjectMeta{
                Labels: map[string]string{
                    "app": "myapp",
                },
            },
            Spec: corev1.PodSpec{
                Containers: []corev1.Container{{
                    Name:  "app",
                    Image: "nginx",
                }},
            },
        },
    },
}

clientset.AppsV1().ReplicaSets("default").Create(context.TODO(), rs, metav1.CreateOptions{})
```

## Monitoramento

### Métricas Prometheus
```promql
# Réplicas desejadas
kube_replicaset_spec_replicas{replicaset="myapp"}

# Réplicas atuais
kube_replicaset_status_replicas{replicaset="myapp"}

# Réplicas prontas
kube_replicaset_status_ready_replicas{replicaset="myapp"}

# ReplicaSets com réplicas faltando
kube_replicaset_status_replicas{replicaset="myapp"} 
  < kube_replicaset_spec_replicas{replicaset="myapp"}
```

## Links Úteis

- [Kubernetes ReplicaSet Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [ReplicationController to ReplicaSet Migration](https://kubernetes.io/docs/tasks/run-application/upgrade-deployment/)
