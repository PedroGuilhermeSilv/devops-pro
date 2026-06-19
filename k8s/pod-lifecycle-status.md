# Status e Ciclo de Vida do Pod

Este documento detalha as **fases** (`phase`), **condições** (`conditions`) e **estados dos containers** que compõem o ciclo de vida de um Pod no Kubernetes.

> Para visão geral de Pods, probes e troubleshooting, veja [pods.md](./pods.md).

## Visão geral do fluxo

```
                    ┌─────────────┐
                    │   Pending   │
                    └──────┬──────┘
                           │ agendado + containers criados
                           ▼
                    ┌─────────────┐
         ┌─────────│   Running   │─────────┐
         │         └──────┬──────┘         │
         │                │                │
         ▼                ▼                ▼
  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
  │  Succeeded  │  │   Failed    │  │   Unknown   │
  └─────────────┘  └─────────────┘  └─────────────┘
```

Um Pod **não volta** de `Succeeded` ou `Failed` para `Running`. Para workloads contínuos, um controller (Deployment, ReplicaSet, etc.) cria um **novo** Pod.

---

## Fase do Pod (`status.phase`)

A fase é um resumo de alto nível do estado do Pod. Valores possíveis:

| Fase | Significado |
|------|-------------|
| `Pending` | O Pod foi aceito pelo API server, mas ainda não está totalmente em execução (aguardando schedule, pull de imagem, montagem de volume, init containers, etc.). |
| `Running` | O Pod foi agendado a um node e pelo menos um container está em execução (ou está sendo iniciado/reiniciado). |
| `Succeeded` | Todos os containers terminaram com sucesso (exit code 0) e a `restartPolicy` não exige novo start (`Never` ou `OnFailure` sem falha). |
| `Failed` | Pelo menos um container terminou com falha ou o Pod foi removido do node antes de concluir normalmente. |
| `Unknown` | O estado do Pod não pôde ser obtido (geralmente problema de comunicação com o kubelet do node). |

### O que acontece em cada fase

**Pending** — etapas comuns antes de `Running`:

1. Scheduler escolhe um node (`PodScheduled=True`)
2. Kubelet faz pull da imagem
3. Volumes são montados
4. Init containers rodam em sequência
5. Containers principais são criados

**Running** — o Pod está ativo no node. Pode haver reinícios de container (CrashLoopBackOff) e a fase continua `Running` enquanto pelo menos um container estiver ativo ou sendo reiniciado.

**Succeeded / Failed** — estado terminal para Pods que não devem ficar rodando indefinidamente (Jobs, Pods com `restartPolicy: Never`, etc.).

---

## Condições do Pod (`status.conditions`)

Condições descrevem **por que** o Pod está ou não pronto. Cada condição tem `type`, `status` (`True`/`False`/`Unknown`), `reason` e `message`.

| Condition | `True` significa |
|-----------|------------------|
| `PodScheduled` | O Pod foi atribuído a um node. |
| `PodReadyToStartContainers` | (K8s 1.29+) Infraestrutura do Pod no node está pronta para iniciar containers. |
| `Initialized` | Todos os init containers terminaram com sucesso. |
| `ContainersReady` | Todos os containers do Pod passaram na readiness (ou não têm readiness probe). |
| `Ready` | O Pod pode receber tráfego via Service (soma de `ContainersReady` + regras do controller). |

### Relação entre condições e tráfego

```
Initialized=True  →  init containers OK
        ↓
ContainersReady=True  →  readiness probe OK (ou ausente)
        ↓
Ready=True  →  Pod entra nos Endpoints do Service
```

Se a readiness probe falhar, `Ready=False` e o Pod é **removido** dos endpoints — mas a fase pode continuar `Running`.

---

## Estado dos containers (`status.containerStatuses`)

Cada container tem seu próprio ciclo, independente da fase do Pod.

### Campos importantes

| Campo | Descrição |
|-------|-----------|
| `state.waiting` | Container ainda não iniciou (ex.: `ContainerCreating`, `ImagePullBackOff`, `CrashLoopBackOff`). |
| `state.running` | Container em execução (`startedAt`). |
| `state.terminated` | Container parou (`exitCode`, `reason`, `finishedAt`). |
| `ready` | Resultado da readiness probe (`true`/`false`). |
| `restartCount` | Quantas vezes o container foi reiniciado. |
| `lastState` | Estado anterior (útil após restart). |

### Razões comuns em `waiting`

| Reason | Causa típica |
|--------|----------------|
| `ContainerCreating` | Criando container, montando volumes, configurando rede. |
| `PodInitializing` | Init containers ainda em execução. |
| `ImagePullBackOff` / `ErrImagePull` | Falha ao baixar a imagem. |
| `CreateContainerConfigError` | Secret, ConfigMap ou volume referenciado não existe. |
| `CrashLoopBackOff` | Container inicia e cai repetidamente; kubelet aumenta o intervalo entre tentativas. |

### Razões comuns em `terminated`

| Reason | Causa típica |
|--------|----------------|
| `Completed` | Processo terminou com exit code 0. |
| `Error` | Processo terminou com exit code ≠ 0. |
| `OOMKilled` | Container excedeu o limite de memória. |
| `ContainerCannotRun` | Erro ao iniciar o binário (permissão, arquivo ausente). |

---

## Init containers

Init containers têm status em `status.initContainerStatuses` e seguem as mesmas transições `waiting` → `running` → `terminated`.

- Rodam **em ordem** (um após o outro).
- Enquanto init containers não terminam com sucesso, a condition `Initialized` permanece `False`.
- Falha em um init container impede o start dos containers principais.

---

## Transições típicas (workload long-running)

Exemplo: Deployment com app web.

```
1. Pod criado                    → phase: Pending
2. Agendado no node              → PodScheduled=True
3. Pull da imagem                → waiting: ContainerCreating
4. Container sobe                → running, restartCount: 0
5. Readiness OK                  → Ready=True, entra no Service
6. Rolling update / delete       → terminating (ver sigterm-sigkill.md)
7. Novo Pod substitui o antigo   → controller cria outro Pod
```

---

## Transições típicas (Job)

```
1. Pod Pending → Running
2. Container executa a tarefa
3. Exit 0        → phase: Succeeded (restartPolicy: Never/OnFailure sem restart)
   Exit ≠ 0      → phase: Failed (ou restart se OnFailure)
```

---

## Inspecionar status na prática

```bash
# Fase e readiness resumidos
kubectl get pods

# Fase, conditions, events, container states
kubectl describe pod <nome>

# Apenas a fase
kubectl get pod <nome> -o jsonpath='{.status.phase}'

# Conditions em formato legível
kubectl get pod <nome> -o jsonpath='{range .status.conditions[*]}{.type}={.status} ({.reason}){"\n"}{end}'

# Estado de um container específico
kubectl get pod <nome> -o jsonpath='{.status.containerStatuses[0].state}'

# Logs do container anterior (após crash/restart)
kubectl logs <nome> --previous
```

---

## Tabela rápida: sintoma → onde olhar

| Sintoma (`kubectl get`) | Onde investigar |
|-------------------------|-----------------|
| `Pending` | `describe` → Events (schedule, PVC, recursos, taints) |
| `Running` mas `0/1 Ready` | Readiness probe, logs, `containerStatuses[].ready` |
| `CrashLoopBackOff` | `logs --previous`, command/args, liveness agressivo |
| `ImagePullBackOff` | Nome da imagem, registry, `imagePullSecrets` |
| `OOMKilled` | `limits.memory`, consumo real (`kubectl top pod`) |
| `Unknown` | Saúde do node / kubelet |

---

## Relação com outros recursos

| Recurso | Impacto no ciclo |
|---------|------------------|
| [restart-policy](./policy/restart-policy.md) | Define se o kubelet reinicia containers após `terminated`. |
| [liveness-probe](./probes/liveness-probe.md) | Falha → kill + restart do container. |
| [readiness-probe](./probes/readiness-probe.md) | Falha → `Ready=False`, sem restart. |
| [sigterm-sigkill](./sigterm-sigkill.md) | Encerramento gracioso ao deletar ou substituir o Pod. |

---

## Links oficiais

- [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Pod Phase](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase)
- [Pod Conditions](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-conditions)
