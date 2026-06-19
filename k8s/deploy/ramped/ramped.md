# Estratégia Ramped / RollingUpdate (Deployment do Kubernetes)

## O que é

A estratégia **Ramped** (chamada de `RollingUpdate` no Kubernetes) atualiza a aplicação de forma **gradual**: novos Pods sobem aos poucos enquanto os antigos são encerrados, mantendo o serviço disponível durante o deploy.

É a estratégia **padrão** do Deployment quando nenhuma outra é definida.

## Como funciona

1. Você altera o Deployment (ex.: troca a imagem do container).
2. O Kubernetes **cria alguns Pods da nova versão**.
3. Quando os novos Pods ficam prontos, **encerra alguns Pods da versão antiga**.
4. Repete o processo até **todas as réplicas** estarem na nova versão.

Durante a atualização, **versões antiga e nova podem rodar ao mesmo tempo**.

### Exemplo com 10 réplicas (padrão: 25%)

Com `maxSurge: 25%` e `maxUnavailable: 25%`:

- Até **3 Pods extras** podem ser criados (`maxSurge`)
- Até **2 Pods** podem ficar indisponíveis (`maxUnavailable`)
- Mínimo disponível: **8 Pods** (10 − 2)
- Máximo total: **13 Pods** (10 + 3)

```
Inicial:  [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1]

Passo 1:  sobe novos Pods (v2) até o limite de maxSurge
Passo 2:  encerra Pods antigos (v1) conforme os novos ficam prontos
...
Final:    [v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2]
```

## Como configurar

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # máximo de Pods extras durante o update
      maxUnavailable: 25%  # máximo de Pods indisponíveis durante o update
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: fabricioveronez/web-color:green
        ports:
        - containerPort: 80
```

### Parâmetros principais

| Parâmetro          | O que controla                                      | Default |
|--------------------|-----------------------------------------------------|---------|
| `maxSurge`         | Quantos Pods **a mais** podem existir no rollout    | 25%     |
| `maxUnavailable`   | Quantos Pods podem ficar **fora do ar** no rollout  | 25%     |

Valores aceitos: número absoluto (ex.: `1`) ou porcentagem (ex.: `25%`).

Para **zero downtime**, use `maxUnavailable: 0` — o Kubernetes só encerra Pods antigos depois que os novos estão prontos.

## Vantagens

- **Sem downtime** (ou com impacto mínimo, dependendo dos parâmetros).
- Recomendada para **produção** e alta disponibilidade.
- Permite **rollback** rápido se a nova versão apresentar problemas.
- Controle fino da velocidade do deploy via `maxSurge` e `maxUnavailable`.

## Desvantagens

- **Versões antiga e nova rodam juntas** por um período — a aplicação precisa tolerar isso.
- Mais lento que `Recreate`.
- Rollout pode falhar se os novos Pods não passarem nos health checks.

## Ramped x Recreate

| Característica        | Ramped (RollingUpdate)     | Recreate                      |
|-----------------------|----------------------------|-------------------------------|
| Downtime              | Não (atualização gradual)  | Sim                           |
| Versões simultâneas   | Sim (temporariamente)      | Não                           |
| Velocidade            | Mais lento                 | Mais rápido                   |
| Uso recomendado       | Produção / alta disponib.  | Apps que não toleram mix      |
