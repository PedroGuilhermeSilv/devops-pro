# Estratégia Recreate (Deployment do Kubernetes)

## O que é

A estratégia `Recreate` é uma das formas de atualização de um Deployment no Kubernetes. Quando uma nova versão é aplicada, o Kubernetes **encerra todos os Pods antigos primeiro** e só depois **cria os novos Pods**.

## Como funciona

1. Você altera o Deployment (ex.: troca a imagem do container).
2. O Kubernetes **derruba todos os Pods da versão atual**.
3. Após todos serem encerrados, ele **sobe os Pods da nova versão**.

Ou seja, não há Pods antigos e novos rodando ao mesmo tempo.

## Como configurar

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 10
  strategy:
    type: Recreate
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
        image: fabricioveronez/web-color:blue
        ports:
        - containerPort: 80
```

## Vantagens

- Simples e previsível.
- Garante que **apenas uma versão** da aplicação esteja em execução por vez.
- Útil quando a aplicação **não pode** ter versões diferentes rodando juntas (ex.: incompatibilidade de banco de dados ou de schema).

## Desvantagens

- Gera **indisponibilidade (downtime)**: durante a troca, nenhum Pod está disponível.
- Não recomendada para aplicações que precisam de alta disponibilidade.

## Recreate x RollingUpdate

| Característica        | Recreate                  | RollingUpdate (padrão)        |
|----------------------|---------------------------|-------------------------------|
| Downtime             | Sim                       | Não (atualização gradual)     |
| Versões simultâneas  | Não                       | Sim (temporariamente)         |
| Uso recomendado      | Apps que não toleram mix  | Maioria dos casos             |
