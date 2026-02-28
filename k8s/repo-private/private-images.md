# Trabalhando com Imagens Privadas no Kubernetes

Para utilizar imagens de containers que estão hospedadas em registros privados (como Docker Hub privado, AWS ECR, Google Artifact Registry, ou um registro próprio), o Kubernetes precisa de credenciais para autenticar e baixar essas imagens.

## 1. Criando um Secret de Registro Docker

A forma mais comum de fornecer essas credenciais é através de um `Secret` do tipo `kubernetes.io/dockerconfigjson`.

### Comando via CLI:
```bash
kubectl create secret docker-registry <NOME_DO_SECRET> \
  --docker-server=<REGISTRY_SERVER> \
  --docker-username=<USUARIO> \
  --docker-password=<SENHA> \
  --docker-email=<EMAIL>
```

*   **`<REGISTRY_SERVER>`**: Endereço do registro (ex: `https://index.docker.io/v1/` para Docker Hub ou `ghcr.io` para GitHub).
*   **`<NOME_DO_SECRET>`**: O nome que você usará no arquivo YAML (ex: `my-registry-key`).

---

## 2. Utilizando o Secret no Pod

Após criar o Secret, você deve informar ao Kubernetes para usá-lo através do campo `imagePullSecrets` na especificação do Pod.

### Exemplo de Pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-nginx
spec:
  containers:
  - name: internal-app
    image: <USUARIO>/private-repo:latest
  imagePullSecrets:
  - name: my-registry-key
```

### Exemplo em Deployment:
O campo `imagePullSecrets` fica dentro do `template.spec`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: private-app
  template:
    metadata:
      labels:
        app: private-app
    spec:
      containers:
      - name: app
        image: my-private-registry.com/app:v1
      imagePullSecrets:
      - name: my-registry-key
```

---

## 3. Configurando via ServiceAccount (Automático)

Para evitar ter que adicionar `imagePullSecrets` em todos os seus arquivos YAML, você pode associar o Secret à `ServiceAccount` padrão do namespace.

1.  **Edite a ServiceAccount:**
    ```bash
    kubectl edit serviceaccount default
    ```
2.  **Adicione o campo `imagePullSecrets`:**
    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: default
    secrets:
    - name: default-token-abcde
    imagePullSecrets:
    - name: my-registry-key
    ```

Desta forma, qualquer novo Pod criado usando esta ServiceAccount terá o segredo de pull injetado automaticamente.

---

## Resumo de Comandos Úteis

| Ação | Comando |
| :--- | :--- |
| Criar Secret | `kubectl create secret docker-registry ...` |
| Listar Secrets | `kubectl get secrets` |
| Ver detalhes | `kubectl describe secret <nome>` |
| Testar Pull | `kubectl describe pod <nome>` (Verificar eventos de erro de Pull) |
