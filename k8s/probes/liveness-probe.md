# Liveness Probe

O **Liveness Probe** (Sonda de Sobrevivência) serve para determinar se um container ainda está em execução e saudável.

## Funcionamento

Se o Liveness Probe falhar, o Kubernetes entende que o container entrou em um estado de "travamento" (deadlock) ou erro irrecuperável e **reinicia o container** automaticamente, seguindo a `restartPolicy`.

## Quando usar?

- Para detectar processos que ainda estão rodando mas não conseguem mais progredir (ex: deadlock).
- Quando a aplicação não consegue se recuperar sozinha de um erro interno.

## Exemplo de Configuração (HTTP)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-example
spec:
  containers:
  - name: my-app
    image: my-app-image
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 3   # Espera 3s antes da primeira checagem
      periodSeconds: 5         # Checa a cada 5s
      failureThreshold: 3      # Considera falha após 3 tentativas malsucedidas
```

## Mecanismos de Checagem

1. **httpGet:** Requisição HTTP GET. Sucesso se o status for entre 200 e 399.
2. **exec:** Executa um comando dentro do container. Sucesso se retornar status 0.
3. **tcpSocket:** Tenta abrir uma conexão TCP em uma porta específica.
4. **grpc:** Faz uma chamada RPC usando gRPC.
