# Image Pull Policy no Kubernetes

A política `imagePullPolicy` define quando o Kubernetes deve baixar (pull) uma imagem de container do registry.

## Tipos de imagePullPolicy

### 1. Always
```yaml
spec:
  containers:
  - name: app
    image: myapp:v1
    imagePullPolicy: Always
```
- **Comportamento**: Sempre baixa a imagem do registry, mesmo que ela já exista localmente
- **Quando usar**: 
  - Em produção com tags mutáveis (como `latest`)
  - Quando você quer garantir que está usando a versão mais recente
- **Desvantagem**: Mais lento, consome mais banda

### 2. IfNotPresent
```yaml
spec:
  containers:
  - name: app
    image: myapp:v1
    imagePullPolicy: IfNotPresent
```
- **Comportamento**: Só baixa a imagem se ela não existir localmente no node
- **Quando usar**:
  - Com tags versionadas específicas (ex: `v1.0.0`, `v2.3.1`)
  - Para economizar banda e acelerar o startup
- **Vantagem**: Mais rápido quando a imagem já existe

### 3. Never
```yaml
spec:
  containers:
  - name: app
    image: myapp:v1
    imagePullPolicy: Never
```
- **Comportamento**: Nunca tenta baixar a imagem, usa apenas o que está localmente
- **Quando usar**:
  - Desenvolvimento local
  - Imagens buildadas diretamente no node
  - Ambientes air-gapped (sem acesso à internet)
- **Atenção**: Se a imagem não existir localmente, o pod falhará

## Comportamento Padrão

Se você **não especificar** `imagePullPolicy`, o Kubernetes usa as seguintes regras:

| Tag da Imagem | imagePullPolicy Padrão |
|---------------|------------------------|
| `:latest` ou sem tag | `Always` |
| Tag específica (ex: `:v1`) | `IfNotPresent` |

### Exemplos do comportamento padrão:

```yaml
# Usa imagePullPolicy: Always (padrão para :latest)
image: nginx:latest

# Usa imagePullPolicy: Always (padrão quando não tem tag)
image: nginx

# Usa imagePullPolicy: IfNotPresent (padrão para tags específicas)
image: nginx:1.21.0
```

## Boas Práticas

1. **Produção**: Use tags versionadas específicas + `IfNotPresent`
   ```yaml
   image: myapp:v1.2.3
   imagePullPolicy: IfNotPresent
   ```

2. **Desenvolvimento**: Use `Always` se estiver testando mudanças frequentes
   ```yaml
   image: myapp:dev
   imagePullPolicy: Always
   ```

3. **Evite `latest` em produção**: Use sempre versões específicas para ter controle e reprodutibilidade

4. **Air-gapped**: Use `Never` quando não há acesso a registry externo
   ```yaml
   image: myapp:v1
   imagePullPolicy: Never
   ```

## Exemplo Completo

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: guilhermesilv/myapp:v1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
```

## Troubleshooting

### Problema: "ImagePullBackOff"
- Verifique se o nome da imagem está correto
- Verifique credenciais do registry (imagePullSecrets)
- Verifique conectividade com o registry

### Problema: "ErrImageNeverPull"
- A política está como `Never` mas a imagem não existe localmente
- Solução: Mude para `IfNotPresent` ou `Always`, ou faça o pull manual da imagem no node
