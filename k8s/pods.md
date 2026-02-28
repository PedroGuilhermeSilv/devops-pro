# Kubernetes Pods

Pods são a menor unidade de deploy no Kubernetes. Um Pod encapsula um ou mais containers, armazenamento compartilhado (volumes) e configurações sobre como executar os containers.

## O que é um Pod?

- **Unidade atômica**: Menor objeto que você pode criar ou deployar no Kubernetes
- **Containers compartilhados**: Containers no mesmo Pod compartilham namespace de rede e podem compartilhar volumes
- **IP único**: Cada Pod recebe seu próprio endereço IP
- **Efêmero**: Pods são projetados para serem descartáveis e substituíveis
- **Colocation**: Containers fortemente acoplados devem rodar no mesmo Pod

## Ciclo de Vida do Pod

```
Pending → Running → Succeeded/Failed
```

### Fases do Pod

| Fase | Descrição |
|------|-----------|
| `Pending` | Pod aceito pelo cluster, mas containers ainda não estão rodando |
| `Running` | Pod foi associado a um node e todos os containers foram criados |
| `Succeeded` | Todos os containers terminaram com sucesso |
| `Failed` | Todos os containers terminaram e pelo menos um falhou |
| `Unknown` | Estado do Pod não pode ser obtido |

## Pod Conditions

| Condition | Descrição |
|-----------|-----------|
| `PodScheduled` | Pod foi agendado para um node |
| `ContainersReady` | Todos os containers estão prontos |
| `Initialized` | Todos os init containers completaram com sucesso |
| `Ready` | Pod pode servir requisições |

## Quando Usar Pods Diretamente

### ✅ Use Pods quando:
- Testando configurações localmente
- Debugging rápido
- Jobs ou tarefas únicas
- Desenvolvimento local com Kubernetes

### ❌ NÃO use Pods em produção quando:
- Precisa de alta disponibilidade
- Requer escalabilidade
- Necessita rolling updates
- Quer self-healing automático

> **Use Deployments, StatefulSets ou DaemonSets em produção!**

## Comandos Úteis

```bash
# Criar Pod a partir de YAML
kubectl apply -f pod.yaml

# Listar Pods
kubectl get pods
kubectl get pods -o wide
kubectl get pods --all-namespaces

# Descrever Pod (detalhes, eventos)
kubectl describe pod <pod-name>

# Ver logs do Pod
kubectl logs <pod-name>
kubectl logs <pod-name> -f  # follow
kubectl logs <pod-name> -c <container-name>  # container específico

# Executar comando no Pod
kubectl exec <pod-name> -- ls /app
kubectl exec -it <pod-name> -- /bin/bash

# Port forward para acessar localmente
kubectl port-forward pod/<pod-name> 8080:80

# Ver YAML do Pod em execução
kubectl get pod <pod-name> -o yaml

# Deletar Pod
kubectl delete pod <pod-name>
kubectl delete -f pod.yaml

# Ver recursos utilizados
kubectl top pod <pod-name>

# Ver eventos do Pod
kubectl get events --field-selector involvedObject.name=<pod-name>
```

## Padrões de Múltiplos Containers

### 1. Sidecar Pattern
Container auxiliar que estende/melhora o container principal.

**Exemplos**:
- Container de logging agregando logs
- Service mesh proxy (Istio, Linkerd)
- Config sync mantendo configurações atualizadas

### 2. Ambassador Pattern
Container proxy que intermedia acesso a serviços externos.

**Exemplos**:
- Proxy para diferentes ambientes de banco de dados
- Load balancing para serviços externos
- Circuit breaker

### 3. Adapter Pattern
Container que padroniza/normaliza dados do container principal.

**Exemplos**:
- Converter logs para formato padrão
- Normalizar métricas para Prometheus
- Transformar dados antes de enviar

## Init Containers

Init containers rodam antes dos containers principais e são usados para:
- Setup inicial
- Aguardar dependências estarem prontas
- Clonar repositórios Git
- Gerar configurações
- Registrar o Pod em sistemas externos

**Características**:
- Rodam sequencialmente (um após o outro)
- Devem completar com sucesso antes dos containers principais iniciarem
- Não suportam readiness, liveness ou startup probes

## Quality of Service (QoS)

Kubernetes atribui uma classe QoS baseada em resource requests/limits:

| Classe | Condição | Prioridade |
|--------|----------|------------|
| `Guaranteed` | Todos os containers têm requests = limits | Mais alta |
| `Burstable` | Pelo menos um container tem requests ou limits | Média |
| `BestEffort` | Nenhum container tem requests ou limits | Mais baixa |

Em caso de pressão de recursos, Kubernetes termina Pods nesta ordem: BestEffort → Burstable → Guaranteed

## Probes (Health Checks)

### Liveness Probe
Verifica se o container está rodando. Se falhar, Kubernetes mata e reinicia o container.

**Use quando**: Detectar deadlocks ou aplicação travada

### Readiness Probe
Verifica se o container está pronto para receber tráfego. Se falhar, remove o Pod dos endpoints do Service.

**Use quando**: Aplicação está iniciando, fazendo warmup, ou temporariamente indisponível

### Startup Probe
Verifica se a aplicação dentro do container iniciou. Desabilita liveness/readiness até que suceda.

**Use quando**: Aplicação tem startup lento (> 30s)

## Troubleshooting

### Pod está em Pending
```bash
kubectl describe pod <pod-name>
# Verifique eventos:
# - Recursos insuficientes no cluster
# - PVC não está bound
# - Node selector não encontra nodes
# - Taints impedindo schedule
```

### Pod está em CrashLoopBackOff
```bash
kubectl logs <pod-name> --previous
# Container está crashando ao iniciar
# Verifique:
# - Logs do container anterior
# - Liveness probe muito agressivo
# - Aplicação não inicia corretamente
# - Recursos insuficientes
```

### Pod está em ImagePullBackOff
```bash
kubectl describe pod <pod-name>
# Não consegue fazer pull da imagem
# Verifique:
# - Nome da imagem correto
# - Tag existe
# - Credenciais do registry (imagePullSecrets)
# - Node tem acesso à internet/registry
```

### Container está em Running mas não responde
```bash
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/bash
# Verifique:
# - Aplicação iniciou corretamente
# - Porta correta exposta
# - Readiness probe configurado corretamente
# - Service aponta para porta correta
```

## Boas Práticas

### 1. **Sempre defina Resource Requests e Limits**
```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "500m"
```

### 2. **Configure Health Checks**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 3. **Use Labels e Annotations**
```yaml
metadata:
  labels:
    app: myapp
    version: v1.0.0
    environment: production
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
```

### 4. **Configure Security Context**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
```

### 5. **Use Namespaces para Isolamento**
```bash
kubectl create namespace production
kubectl apply -f pod.yaml -n production
```

### 6. **Implemente Graceful Shutdown**
```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
```

### 7. **Configure Log Adequadamente**
- Use stdout/stderr (não arquivos)
- Logs estruturados (JSON)
- Inclua context (request ID, user ID, etc.)

### 8. **Use Init Containers para Dependências**
```yaml
initContainers:
- name: wait-for-db
  image: busybox
  command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 1; done']
```

## Links Úteis

- [Kubernetes Pods Documentation](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
