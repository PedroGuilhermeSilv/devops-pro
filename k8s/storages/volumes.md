# Volumes

## O que é?

Um **Volume** no Kubernetes é um diretório acessível aos containers de um Pod, usado para persistir ou compartilhar dados. Diferente do sistema de arquivos efêmero de um container (que some ao reiniciar), um volume tem seu ciclo de vida atrelado ao **Pod** — e não ao container individualmente.

## Por que usar?

- Containers dentro de um Pod podem **compartilhar dados** entre si via volume
- Dados **sobrevivem ao reinício do container**, mas não ao término do Pod (para isso, use PV/PVC)
- Permite montar **ConfigMaps**, **Secrets**, dados de nós, e armazenamentos externos

## Tipos de Volume mais usados

| Tipo               | Descrição                                                                 |
|--------------------|---------------------------------------------------------------------------|
| `emptyDir`         | Diretório vazio criado quando o Pod inicia. Destruído quando o Pod termina |
| `hostPath`         | Monta um caminho do nó (node) dentro do container                         |
| `configMap`        | Expõe dados de um ConfigMap como arquivos                                  |
| `secret`           | Expõe dados de um Secret como arquivos                                     |
| `persistentVolumeClaim` | Usa um PVC para armazenamento persistente externo                   |
| `nfs`              | Monta um compartilhamento NFS                                             |
| `projected`        | Combina múltiplas fontes (secret, configMap, serviceAccountToken) em um único diretório |

## emptyDir

O tipo mais simples. O diretório é criado junto com o Pod e deletado quando ele termina. Muito útil para compartilhar dados entre containers do mesmo Pod (padrão sidecar).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-emptydir
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: dados-compartilhados
      mountPath: /app/dados
  - name: sidecar
    image: busybox
    command: ["sh", "-c", "echo 'hello' > /dados/msg.txt && sleep 3600"]
    volumeMounts:
    - name: dados-compartilhados
      mountPath: /dados
  volumes:
  - name: dados-compartilhados
    emptyDir: {}
```

> Por padrão `emptyDir` usa disco. Para usar memória RAM: `emptyDir: { medium: Memory }`

## hostPath

Monta um diretório ou arquivo do nó host dentro do container. Útil para acessar logs do sistema ou sockets do Docker, mas **não recomendado para produção** pois cria acoplamento com o nó.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hostpath
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: logs-do-host
      mountPath: /host-logs
  volumes:
  - name: logs-do-host
    hostPath:
      path: /var/log
      type: Directory  # ou File, DirectoryOrCreate, FileOrCreate
```

## configMap como Volume

Permite montar as chaves de um ConfigMap como arquivos dentro do container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap-volume
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: meu-configmap
```

## secret como Volume

Monta as chaves de um Secret como arquivos (armazenados em `tmpfs`, ou seja, memória).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-volume
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: credenciais
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: credenciais
    secret:
      secretName: meu-secret
```

## Ciclo de vida do Volume vs. do Container

```
Container reinicia  →  Volume permanece (dados mantidos)
Pod é deletado      →  Volume emptyDir é destruído (dados perdidos)
Pod é deletado      →  PVC/PV permanecem (dados persistidos)
```

## Quando usar cada tipo?

| Necessidade                                    | Tipo recomendado              |
|------------------------------------------------|-------------------------------|
| Compartilhar dados entre containers no Pod     | `emptyDir`                    |
| Acessar arquivos do nó (logs, sockets)         | `hostPath`                    |
| Injetar configurações                          | `configMap`                   |
| Injetar senhas e chaves                        | `secret`                      |
| Persistência real entre Pods                   | `persistentVolumeClaim`       |

## Comandos úteis

```bash
# Ver volumes montados em um Pod
kubectl describe pod <nome-do-pod>

# Ver eventos relacionados a volumes
kubectl get events --field-selector reason=FailedMount
```

## Documentação oficial

https://kubernetes.io/docs/concepts/storage/volumes/
