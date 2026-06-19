# Persistent Volume (PV)

## O que é?

Um **Persistent Volume (PV)** é um recurso de armazenamento no cluster provisionado pelo administrador ou dinamicamente pelo Kubernetes. Ele representa um pedaço de armazenamento físico (disco, NFS, cloud storage) abstraído em um objeto Kubernetes com ciclo de vida **independente de qualquer Pod**.

## Por que usar?

Volumes comuns (`emptyDir`, `hostPath`) morrem com o Pod. O PV garante que os dados sobrevivam ao ciclo de vida dos Pods — essencial para bancos de dados, uploads de usuários, e qualquer estado que precisa ser preservado.

## Como funciona?

```
Administrador cria PV  →  PVC solicita armazenamento  →  Kubernetes faz o bind  →  Pod usa o PVC
```

O PV define **o que existe** (capacidade, tipo de disco, modo de acesso). O PVC define **o que a aplicação precisa**. O Kubernetes faz o casamento entre eles.

## Modos de Acesso (accessModes)

| Modo                  | Abreviação | Descrição                                                |
|-----------------------|------------|----------------------------------------------------------|
| `ReadWriteOnce`       | RWO        | Leitura e escrita por **um único nó** por vez            |
| `ReadOnlyMany`        | ROX        | Leitura por **múltiplos nós** simultaneamente            |
| `ReadWriteMany`       | RWX        | Leitura e escrita por **múltiplos nós** simultaneamente  |
| `ReadWriteOncePod`    | RWOP       | Leitura e escrita por **um único Pod** (k8s >= 1.22)     |

> Nem todo tipo de armazenamento suporta todos os modos. NFS suporta RWX; AWS EBS suporta apenas RWO.

## Política de Reclaim (reclaimPolicy)

Define o que acontece com o PV depois que o PVC que o usava é deletado.

| Política    | Comportamento                                                     |
|-------------|-------------------------------------------------------------------|
| `Retain`    | PV fica no estado `Released`. Dados preservados. Admin deve limpar manualmente |
| `Delete`    | PV e o armazenamento subjacente são deletados automaticamente     |
| `Recycle`   | **Depreciado.** Fazia um `rm -rf` básico nos dados                |

> Para produção, prefira `Retain` para evitar perda acidental de dados.

## Fases (Status) do PV

| Fase        | Descrição                                                   |
|-------------|-------------------------------------------------------------|
| `Available` | PV disponível, ainda não vinculado a nenhum PVC             |
| `Bound`     | PV vinculado a um PVC                                       |
| `Released`  | PVC foi deletado, mas o PV ainda não foi reclamado          |
| `Failed`    | Falha automática no processo de reclaim                     |

## Exemplo de PV estático (provisionamento manual)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-dados-mysql
  labels:
    tipo: ssd
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  reclaimPolicy: Retain
  storageClassName: manual
  hostPath:              # apenas para testes locais/minikube
    path: /mnt/dados-mysql
```

## Exemplo de PV com NFS

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-compartilhado
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany
  reclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    server: 192.168.1.100
    path: /exports/dados
```

## Provisionamento Dinâmico

Em vez de criar PVs manualmente, o Kubernetes pode provisionar PVs automaticamente usando uma **StorageClass**. Quando um PVC é criado referenciando uma StorageClass, o provisioner cria o PV (e o disco subjacente) automaticamente.

```yaml
# StorageClass para AWS EBS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

> Com provisionamento dinâmico, você **não precisa criar PVs manualmente** — apenas defina a StorageClass e use o PVC.

## Estático vs. Dinâmico

| Característica       | Provisionamento Estático        | Provisionamento Dinâmico          |
|----------------------|---------------------------------|-----------------------------------|
| Quem cria o PV?      | Administrador                   | Kubernetes (via StorageClass)     |
| Flexibilidade        | Baixa — tamanho fixo            | Alta — PV criado sob demanda      |
| Indicado para        | On-premise, hardware dedicado   | Cloud (AWS, GCP, Azure)           |
| Configuração         | PV criado manualmente           | StorageClass + PVC suficiente     |

## Comandos úteis

```bash
# Listar todos os PVs do cluster
kubectl get pv

# Ver detalhes de um PV específico
kubectl describe pv pv-dados-mysql

# Ver o status e o PVC vinculado
kubectl get pv -o wide

# Deletar um PV (apenas se Released ou Available)
kubectl delete pv pv-dados-mysql
```

## Documentação oficial

https://kubernetes.io/docs/concepts/storage/persistent-volumes/
