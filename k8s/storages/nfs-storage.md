# Armazenamento NFS no Kubernetes

## O que é?

**NFS (Network File System)** expõe um diretório de um servidor de arquivos pela rede para que vários clientes montem o mesmo sistema de arquivos. No Kubernetes, isso aparece como:

- **Volume inline `nfs`** no Pod (ciclo de vida ligado ao Pod; menos comum para dados críticos)
- **`nfs` em um PersistentVolume (PV)** estático, consumido via **PersistentVolumeClaim (PVC)** — padrão mais usado em on-premise e homologação

O cluster monta o export NFS nos nós onde os Pods são agendados; a aplicação vê um diretório normal no `mountPath`.

## Por que usar?

- **ReadWriteMany (RWX)**: vários Pods em nós diferentes podem ler e escrever no mesmo volume — útil para uploads compartilhados, artefatos, caches de leitura, workloads que precisam do mesmo diretório
- **Servidor centralizado**: um único export pode atender muitas aplicações; backup e políticas de disco ficam no servidor NFS
- **Custo e simplicidade** em ambientes próprios onde não há disco gerenciado por volume (EBS, etc.)

## Requisitos de rede e infraestrutura

- Os **nós do cluster** (kubelet) precisam **alcançar o servidor NFS** nas portas usadas (em geral **TCP/UDP 2049**; versões antigas podem usar `rpcbind` 111 — alinhe com o time de rede)
- **Latência e largura de banda** impactam I/O; NFS não é equivalente a disco local ou SAN de baixa latência
- **Alta disponibilidade** do NFS costuma ser feita no servidor (VIP, DRBD, storage appliance), não pelo Kubernetes sozinho

## Modo de acesso

NFS no Kubernetes é tipicamente declarado com **`ReadWriteMany`**, pois o export é compartilhado. Confirme com o administrador do storage se o export permite múltiplos clientes simultâneos com as permissões desejadas.

## PV estático com `spec.nfs`

O administrador cria o PV apontando para `server` e `path` do export. O tamanho em `capacity` é **informativo** para o scheduler; o NFS não impõe quota por esse campo — quotas reais vêm do servidor (export options, quotas no FS, etc.).

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
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    server: 192.168.1.100
    path: /exports/dados
```

PVC correspondente (mesmo `storageClassName` e modo compatível):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dados-compartilhados
  namespace: producao
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs
  resources:
    requests:
      storage: 50Gi
```

Uso no Pod via `persistentVolumeClaim` — igual a qualquer outro PVC; ver [persistent-volume-claim.md](./persistent-volume-claim.md).

## Volume inline `nfs` no Pod (sem PVC)

Útil para testes ou quando não se quer PV/PVC. O compartilhamento morre com o Pod no sentido de configuração (novo Pod precisa declarar de novo); os **dados permanecem no servidor NFS**.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nfs-inline
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: nfs-share
      mountPath: /usr/share/nginx/html
  volumes:
  - name: nfs-share
    nfs:
      server: 192.168.1.100
      path: /exports/site-estatico
```

## Provisionamento dinâmico (NFS Subdir External Provisioner / CSI)

O manifesto nativo `spec.nfs` em PV **não** provisiona pastas automaticamente. Para **um PVC gerar um subdiretório** no mesmo servidor, equipes costumam usar:

- [NFS Subdir External Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner) (in-tree legado / comunidade), ou
- Drivers **CSI** específicos do fornecedor ou [nfs.csi.k8s.io](https://github.com/kubernetes-csi/csi-driver-nfs) (quando adotado no cluster)

Nesses casos a **StorageClass** referencia o provisionador; o desenvolvedor só cria o PVC. O procedimento exato depende da instalação do cluster.

## Segurança

- **NFSv3** é amplamente usado mas tráfego e autenticação são fracos comparados a opções modernas; prefira **NFSv4.x** e segmentação de rede quando possível
- **Export options** no servidor (`rw`/`ro`, `root_squash`, clientes permitidos) definem quem pode escrever e como o root do cliente é mapeado — alinhe com o administrador do NFS
- Evite expor o servidor NFS à internet; use firewall restrito aos CIDRs dos nós

## Problemas comuns

| Sintoma | Verificação |
|---------|-------------|
| `MountVolume.SetUp failed`, timeout | Do **nó** onde o Pod rodou: `showmount -e <server>`, `rpcinfo -p <server>`, firewall entre nó e NFS |
| Permission denied | UID/GID do container vs ownership no export; opções `all_squash` / `no_root_squash` no servidor |
| PVC `Pending` | PV com `storageClassName`, capacidade e `accessModes` compatíveis com o PVC; labels/selectors |
| Dados “somem” | Outro export ou subpath errado; `reclaimPolicy: Delete` em algum layer que apaga subdirs (depende do provisionador) |

```bash
# Onde o volume falhou ao montar — eventos do Pod
kubectl describe pod <nome> -n <namespace>

# PV/PVC ligados ao NFS
kubectl get pv,pvc -A | grep -i nfs

# Inspecionar spec do PV
kubectl get pv <nome-pv> -o yaml
```

## Quando **não** priorizar NFS

- Latência muito baixa e I/O síncrono pesado (alguns bancos preferem disco local ou block storage **ReadWriteOnce**)
- Aplicações que assumem semântica POSIX estrita de lock em todos os cenários (NFS pode expor edge cases)
- Quando a equipe precisa apenas de volumes **RWO** e já tem EBS/GCE Persistent Disk com operação mais simples na nuvem

## Relação com os outros guias

- Conceitos gerais de PV: [persistent-volume.md](./persistent-volume.md)
- PVC e uso em Deployments/StatefulSets: [persistent-volume-claim.md](./persistent-volume-claim.md)
- Tipo de volume `nfs` na lista de volumes do Pod: [volumes.md](./volumes.md)

## Documentação oficial

- Persistent Volumes: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- Volume NFS: https://kubernetes.io/docs/concepts/storage/volumes/#nfs
