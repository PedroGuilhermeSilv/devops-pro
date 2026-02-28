# Startup Probe

O **Startup Probe** serve para verificar se a aplicação dentro do container foi iniciada com sucesso.

## Funcionamento

Enquanto o Startup Probe não passar, todas as outras probes (Liveness e Readiness) ficam **desativadas**. Isso impede que o Liveness Probe reinicie o container antes mesmo dele terminar de carregar, o que causaria um loop de reinicialização eterno.

## Quando usar?

- **Aplicações Legadas:** Apps que demoram muito tempo para subir (ex: 30s a 2 minutos).
- Evitar que o Liveness Probe mate o processo prematuramente durante o boot inicial.

## Exemplo de Configuração

Neste exemplo, a aplicação tem até 300 segundos (30 tentativas * 10 segundos) para terminar seu startup:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-example
spec:
  containers:
  - name: slow-app
    image: slow-app-image
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      periodSeconds: 10
```

## Fluxo de Execução

1. O container sobe.
2. O **Startup Probe** começa a rodar.
3. Liveness e Readiness ficam em espera.
4. Se o Startup Probe passar: o Liveness Probe assume a vigilância contínua.
5. Se o Startup Probe atingir o limite de falhas: o Kubernetes mata o container e tenta novamente.
