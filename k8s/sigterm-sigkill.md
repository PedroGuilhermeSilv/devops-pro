# SIGTERM e SIGKILL no Kubernetes

Quando um Pod é **deletado**, **substituído** (rolling update) ou o **node é drenado**, o Kubernetes precisa encerrar os processos nos containers. Esse processo usa sinais POSIX: primeiro **SIGTERM** (pedido de encerramento), depois **SIGKILL** (encerramento forçado) se o processo não sair a tempo.

> Para fases e status durante o shutdown, veja [pod-lifecycle-status.md](./pod-lifecycle-status.md).

---

## Fluxo de terminação

```
Evento de delete / eviction / rolling update
        │
        ▼
Pod marcado para remoção (deletionTimestamp definido)
        │
        ▼
Pod removido dos Endpoints do Service (para de receber tráfego novo)
        │
        ▼
preStop hook (se configurado) ──► pode atrasar o SIGTERM
        │
        ▼
SIGTERM enviado ao PID 1 de cada container
        │
        ▼
Aguarda terminationGracePeriodSeconds (padrão: 30s)
        │
        ├── Processo terminou ──► container removido
        │
        └── Ainda rodando ──► SIGKILL (não pode ser ignorado)
        │
        ▼
Pod removido do API server
```

---

## SIGTERM

**O que é:** sinal que pede ao processo que encerre de forma ordenada (flush de buffers, fechar conexões, completar requisições em andamento).

**Quem recebe:** o processo com **PID 1** dentro do container (não necessariamente sua aplicação, se o entrypoint for um shell ou um init wrapper).

**Comportamento esperado da aplicação:**

1. Parar de aceitar novas requisições
2. Finalizar trabalho em andamento (dentro do tempo disponível)
3. Fechar conexões com banco, filas, etc.
4. Sair com exit code 0

### Problema comum: PID 1 errado

Se o container inicia assim:

```dockerfile
CMD ["./app"]
```

O `./app` é PID 1 e recebe SIGTERM diretamente — ideal.

Se inicia assim:

```dockerfile
CMD ["sh", "-c", "./app"]
```

O **shell** é PID 1. Ele pode não repassar SIGTERM para `./app`, e o kubelet acaba enviando SIGKILL após o grace period.

**Soluções:**

- Usar `exec` no script: `exec ./app`
- Usar imagem com init leve (`tini`, `dumb-init`) como entrypoint
- Definir `command` no manifest para chamar o binário diretamente

---

## SIGKILL

**O que é:** encerramento **imediato** e **irreversível**. O processo não pode capturar nem ignorar esse sinal.

**Quando ocorre:** após esgotar `terminationGracePeriodSeconds` (contado desde o envio do SIGTERM, considerando também o tempo do `preStop`).

**Efeitos:**

- Conexões TCP cortadas abruptamente
- Escritas em disco podem ficar incompletas
- Requisições HTTP em andamento falham para o cliente

---

## `terminationGracePeriodSeconds`

Tempo máximo que o kubelet espera **após o SIGTERM** antes de enviar SIGKILL.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-com-grace
spec:
  terminationGracePeriodSeconds: 60  # padrão: 30
  containers:
  - name: app
    image: myapp:1.0
```

| Valor | Efeito |
|-------|--------|
| Muito baixo (ex.: 5s) | Risco de cortar requisições longas ou shutdown lento. |
| Adequado (30–60s) | Balanceia disponibilidade e tempo para drenar. |
| Muito alto (ex.: 600s) | Rolling updates e node drain demoram mais. |

O grace period vale para **todo o Pod** (não por container individual).

---

## Hook `preStop`

Executado **antes** do SIGTERM. Útil para drenar tráfego ou esperar o Service remover o Pod dos endpoints.

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 15"]
```

### Por que um `sleep` no preStop?

Após o delete, há uma janela em que o Pod ainda pode receber tráfego até o control plane atualizar os Endpoints. Um `sleep` curto (5–15s) dá tempo da remoção propagar antes do SIGTERM.

> O tempo do `preStop` **conta** dentro do `terminationGracePeriodSeconds`. Se `preStop` dorme 15s e o grace é 30s, restam ~15s para a aplicação encerrar após o SIGTERM.

Alternativas ao sleep:

- Chamar endpoint de shutdown da própria app
- Usar `preStop` com `httpGet` para health que falha e acelera remoção do load balancer (depende do ingress/controller)

---

## Cenários que disparam o shutdown

| Cenário | O que acontece |
|---------|----------------|
| `kubectl delete pod` | Fluxo completo SIGTERM → SIGKILL |
| Rolling update (Deployment) | Pod antigo deletado; novo Pod já criado conforme estratégia |
| Scale down | Pods excedentes deletados |
| Node drain (`kubectl drain`) | Pods evicted com grace period |
| Liveness probe falha | **Apenas o container** é reiniciado (SIGTERM no container), não delete do Pod inteiro |
| OOMKilled | Kernel mata o processo; não é SIGTERM/SIGKILL do kubelet |

---

## Graceful shutdown em aplicações

### Node.js

```javascript
const server = app.listen(3000);

process.on('SIGTERM', () => {
  server.close(() => {
    process.exit(0);
  });
});
```

### Go

```go
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
<-quit
// ctx com timeout para shutdown do http.Server
```

### Java / Spring Boot

Spring Boot trata SIGTERM e inicia shutdown do contexto se `server.shutdown=graceful` estiver configurado.

### Nginx

`STOPSIGNAL SIGQUIT` na imagem ou config `worker_shutdown_timeout`; para Kubernetes, garantir que PID 1 seja o nginx master.

---

## Exemplo completo (Deployment)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  template:
    spec:
      terminationGracePeriodSeconds: 45
      containers:
      - name: web
        image: myapp:2.0
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10 && kill -SIGTERM 1"]
        # readiness garante que tráfego só vai para Pods prontos
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
```

---

## Linha do tempo (exemplo numérico)

Configuração: `terminationGracePeriodSeconds: 30`, `preStop: sleep 10`.

| Tempo | Evento |
|-------|--------|
| T+0s | `kubectl delete pod` → Pod removido dos Endpoints |
| T+0s | `preStop` inicia (`sleep 10`) |
| T+10s | `preStop` termina → **SIGTERM** enviado ao PID 1 |
| T+10s–T+30s | App deve encerrar (20s restantes do grace) |
| T+30s | Se ainda rodando → **SIGKILL** |
| T+30s+ | Container e Pod removidos |

Se o `preStop` durar 35s e o grace for 30s, o SIGKILL pode ocorrer **durante** o preStop — planeje os tempos juntos.

---

## Boas práticas

1. **Trate SIGTERM na aplicação** — não dependa só do `preStop sleep`.
2. **PID 1 deve ser o processo da app** (ou um init que repassa sinais).
3. **Alinhe grace period com o pior caso** — requisição mais longa + tempo de flush.
4. **Combine preStop + readiness** — readiness tira tráfego de Pods não saudáveis; preStop cobre a janela no delete.
5. **Teste o shutdown** — `kubectl delete pod` e observe logs, métricas e erros 502 no load balancer.
6. **Evite grace period excessivo** em clusters com muitos deploys — atrasa rollouts e drains.

---

## Comandos úteis

```bash
# Ver grace period configurado
kubectl get pod <nome> -o jsonpath='{.spec.terminationGracePeriodSeconds}'

# Ver se o Pod está terminando
kubectl get pod <nome> -o jsonpath='{.metadata.deletionTimestamp}'

# Acompanhar eventos de terminação
kubectl describe pod <nome>

# Deletar e observar tempo até sumir
kubectl delete pod <nome> --wait=false
kubectl get pod <nome> -w
```

---

## Relação com outros documentos

| Documento | Tema relacionado |
|-----------|------------------|
| [pod-lifecycle-status.md](./pod-lifecycle-status.md) | Fases e estados durante e após o shutdown |
| [pods.md](./pods.md) | Visão geral e troubleshooting |
| [liveness-probe](./probes/liveness-probe.md) | Restart de container (SIGTERM local, não delete do Pod) |
| [deployments.md](./deployments.md) | Rolling update e graceful shutdown em Deployments |

---

## Links oficiais

- [Terminate a Container](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)
- [Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
- [Configure Graceful Shutdown](https://kubernetes.io/docs/tutorials/services/connect-applications-service/#graceful-shutdown)
