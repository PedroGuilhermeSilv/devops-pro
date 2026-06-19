# Persistent Volume Claim (PVC)

## O que é?

Um **Persistent Volume Claim (PVC)** é uma **requisição de armazenamento** feita por uma aplicação (Pod). É a forma como os desenvolvedores solicitam armazenamento persistente sem precisar saber os detalhes do hardware ou do provedor de nuvem.

O PVC funciona como um "ticket de pedido": você define o quanto precisa e qual modo de acesso quer, e o Kubernetes encontra (ou cria dinamicamente) um PV compatível para atender a requisição.

## Relação entre PV e PVC

```
PV (Persistent Volume)         →  O armazenamento real que existe no cluster
PVC (Persistent Volume Claim)  →  A requisição de armazenamento da aplicação
Bind                           →  O Kubernetes associa o PVC ao PV compatível
```

Um PVC sempre está vinculado a **exatamente um PV**. Um PV pode satisfazer **apenas um PVC** por vez.

## Fases (Status) do PVC

| Fase      | Descrição                                                         |
|-----------|-------------------------------------------------------------------|
| `Pending` | Aguardando um PV compatível (ou provisionamento dinâmico)         |
| `Bound`   | Vinculado a um PV — pronto para ser usado pelo Pod                |
| `Lost`    | O PV associado foi deletado; dados podem estar inacessíveis       |

## Exemplo básico de PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dados-app
  namespace: producao
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual   # omita para usar o StorageClass padrão do cluster
```

## Como usar o PVC em um Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-com-pvc
spec:
  containers:
  - name: app
    image: postgres:15
    env:
    - name: PGDATA
      value: /var/lib/postgresql/data/pgdata
    volumeMounts:
    - name: dados-postgres
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: dados-postgres
    persistentVolumeClaim:
      claimName: pvc-dados-app   # referencia o PVC pelo nome
```

## PVC com provisionamento dinâmico

Quando você não cria PVs manualmente, basta referenciar uma **StorageClass** no PVC e o Kubernetes provisiona o disco automaticamente.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-banco-dados
  namespace: producao
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: ebs-ssd   # StorageClass que aciona o provisionador AWS EBS
```

## PVC em Deployments e StatefulSets

### Deployment (PVC compartilhado)

Todos os Pods do Deployment compartilham o mesmo PVC — cuidado com conflitos de escrita se o PV não suportar `ReadWriteMany`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-com-storage
spec:
  replicas: 1   # use 1 réplica com RWO para evitar conflitos
  selector:
    matchLabels:
      app: minha-app
  template:
    metadata:
      labels:
        app: minha-app
    spec:
      containers:
      - name: app
        image: minha-app:latest
        volumeMounts:
        - name: storage
          mountPath: /data
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: pvc-dados-app
```

### StatefulSet (volumeClaimTemplates)

StatefulSets criam um PVC **por réplica** automaticamente usando `volumeClaimTemplates` — ideal para bancos de dados clusterizados.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: dados-mysql
          mountPath: /var/lib/mysql
  volumeClaimTemplates:          # cria um PVC por réplica automaticamente
  - metadata:
      name: dados-mysql
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: ebs-ssd
      resources:
        requests:
          storage: 10Gi
```

> O StatefulSet criará PVCs com nomes como `dados-mysql-mysql-0`, `dados-mysql-mysql-1`, etc.

## Redimensionamento de PVC

Se a StorageClass suportar `allowVolumeExpansion: true`, é possível aumentar o tamanho de um PVC sem recriá-lo.

```bash
# Editar o PVC e aumentar o storage
kubectl edit pvc pvc-dados-app

# ou via patch
kubectl patch pvc pvc-dados-app -p '{"spec": {"resources": {"requests": {"storage": "20Gi"}}}}'
```

> **Redução de tamanho não é suportada.** Só é possível aumentar.

## Como o Kubernetes faz o Bind?

O Kubernetes procura um PV que satisfaça **todos** os critérios do PVC:

1. `storage` do PV >= `storage` solicitado no PVC
2. `accessModes` do PV inclui o modo solicitado no PVC
3. `storageClassName` é igual nos dois (se especificado)
4. `selector` do PVC (se usado) bate com os `labels` do PV

Se nenhum PV satisfazer, o PVC fica em `Pending` até que um PV compatível apareça.

## Comandos úteis

```bash
# Listar PVCs de um namespace
kubectl get pvc -n producao

# Ver detalhes e o PV vinculado
kubectl describe pvc pvc-dados-app -n producao

# Ver todos os PVCs do cluster
kubectl get pvc -A

# Deletar um PVC (cuidado: pode deletar os dados dependendo da reclaimPolicy)
kubectl delete pvc pvc-dados-app -n producao

# Forçar deleção de PVC travado em Terminating
kubectl patch pvc pvc-dados-app -p '{"metadata":{"finalizers":null}}' -n producao
```

## Diferença entre Volume, PV e PVC

| Conceito  | Ciclo de vida     | Quem gerencia     | Persistência         |
|-----------|-------------------|-------------------|----------------------|
| `Volume`  | Igual ao Pod      | Desenvolvedor     | Não (exceto PVC)     |
| `PV`      | Independente      | Administrador     | Sim                  |
| `PVC`     | Independente      | Desenvolvedor     | Sim (via PV)         |

## Documentação oficial

https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims
