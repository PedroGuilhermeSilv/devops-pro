# Integração com Gerenciadores de Secrets (Vault, AWS SM, etc.)

Se você optar por usar um gerenciador de secrets externo como **HashiCorp Vault**, **AWS Secrets Manager**, ou **Azure Key Vault**, a dinâmica de gerenciamento muda, mas o consumo pelo Kubernetes permanece similar por questões de arquitetura.

## O Desafio: Kubelet vs. External Vaults
O componente do Kubernetes responsável por baixar a imagem é o **Kubelet** (que roda no nó). O Kubelet não tem integração nativa para se autenticar diretamente no Vault ou em APIs de nuvem para buscar uma senha de registro no momento do pull.

Ele espera encontrar um **Kubernetes Secret** do tipo `docker-registry`.

## A Solução: Operadores de Sincronização
Para usar um Vault, você utiliza um operador que atua como uma "ponte", sincronizando o valor do Vault para um Secret dentro do Kubernetes. O mais utilizado hoje é o **External Secrets Operator (ESO)**.

### O Fluxo de Trabalho:
1.  **Vault**: Você armazena o JSON de autenticação do Docker no Vault.
2.  **SecretStore**: Você cria um recurso no K8s que ensina o cluster a se autenticar no seu Vault.
3.  **ExternalSecret**: Você cria um recurso que diz: *"Pegue o segredo 'prod/docker-hub' do Vault e crie um Kubernetes Secret chamado 'registry-credentials' neste namespace"*.

---

## O que muda na prática?

### 1. No YAML da Aplicação (Deployment/Pod)
**Nada muda.**
Seu Pod continuará sendo configurado assim:
```yaml
spec:
  imagePullSecrets:
  - name: registry-credentials # Este nome é criado/gerido pelo Operador
```

### 2. Na Manutenção
*   **Sem Vault**: Você precisa rodar `kubectl create secret` manualmente ou via pipeline de CI sempre que a senha expirar.
*   **Com Vault**: Você atualiza a senha apenas no Vault. O Operador detecta a mudança e atualiza o Secret no Kubernetes automaticamente em todos os namespaces necessários.

### 3. Exemplo de ExternalSecret (Conceitual)
Em vez de criar o segredo manualmente, você aplica um YAML como este:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-registry-external-secret
spec:
  refreshInterval: 1h # Rotação/Sincronização automática
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: registry-credentials # Nome do Secret que será criado no K8s
    template:
      type: kubernetes.io/dockerconfigjson # Garante o tipo correto
  data:
  - secretKey: .dockerconfigjson
    remoteRef:
      key: secret/data/common/docker-hub # Caminho no Vault
```

---

## Resumo: Por que usar um Gerenciador?

| Aspecto | Manual (Kubernetes Secret) | Com Gerenciador (Vault + ESO) |
| :--- | :--- | :--- |
| **Origem da Senha** | Memória ou Arquivo local | Cofre centralizado e criptografado |
| **Rotação** | Manual / Requer nova execução de CI | Automática (Operador sincroniza) |
| **Segurança** | Secret fica no etcd (precisa de criptografia em repouso) | Secret ainda fica no etcd, mas a gestão é externa e auditável |
| **Multi-cluster** | Precisa replicar manualmente | O Operador em cada cluster busca do mesmo Vault |

---

## Outras Alternativas
*   **Secret Store CSI Driver**: Monta secrets como volumes. *Nota: Difícil de usar para imagePullSecrets pois o volume só é montado DEPOIS que a imagem já baixou.*
*   **Autenticação Nativa de Cloud**: Se você usa EKS (AWS) ou GKE (Google), os nós podem ter permissões IAM para baixar do ECR/GCR sem precisar de Secrets no YAML (o Kubelet usa a role do nó).
