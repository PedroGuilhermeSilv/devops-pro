# Readiness Probe

O **Readiness Probe** (Sonda de Prontidão) serve para determinar se um container está pronto para começar a aceitar tráfego de usuários.

## Funcionamento

Diferente do Liveness Probe, o Readiness **não reinicia o container** em caso de falha. Em vez disso, o Kubernetes remove o endereço IP do Pod de todos os **Services** (Endpoints) aos quais ele pertence. O tráfego só volta a ser direcionado ao Pod quando a sonda passar novamente.

## Quando usar?

- Durante o startup da aplicação (carregamento de arquivos grandes, migração de banco de dados).
- Quando a aplicação precisa verificar dependências externas antes de atender requisições.
- Para gerenciar picos de carga onde a aplicação pode ficar "temporariamente ocupada".

## Exemplo de Configuração (TCP)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-example
spec:
  containers:
  - name: my-app
    image: my-app-image
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
      successThreshold: 1      # Precisa de 1 sucesso para ser considerado "Ready"
```

## Diferença Chave: Liveness vs Readiness

| Característica | Liveness Probe | Readiness Probe |
| :--- | :--- | :--- |
| **Ação na Falha** | Reinicia o Container | Remove do Service (para de receber tráfego) |
| **Objetivo** | Recuperar de travamentos | Garantir que o app consegue responder |
| **Uso em Escala** | Evita Pods "zumbis" | Evita enviar usuários para um app que ainda está subindo |
