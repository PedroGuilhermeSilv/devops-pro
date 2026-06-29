# Estratégia Blue-Green (Deployment do Kubernetes)

## O que é

Blue-Green é um **padrão de deploy**, não uma estratégia nativa do Kubernetes. Você mantém **dois ambientes completos** rodando em paralelo — **Blue** (versão atual) e **Green** (nova versão) — e troca o tráfego de um para o outro alterando o **selector do Service**.

No exemplo desta pasta, o tráfego começa no **Blue**; quando a versão **Green** estiver pronta, basta apontar o Service para ela.

## Como funciona

1. **Blue** (produção): Deployment `web-blue` com 10 réplicas recebe todo o tráfego via Service.
2. **Green** (nova versão): Deployment `web-green` sobe em paralelo, com a mesma capacidade, mas **sem receber tráfego**.
3. Você valida a versão Green (testes, smoke tests, acesso direto aos Pods).
4. Troca o tráfego **de uma vez** alterando o selector do Service de `version: blue` para `version: green`.
5. Se houver problema, **rollback instantâneo**: volta o selector para `version: blue`.

```
Estado inicial (tráfego → Blue):

  Service [web] ──→ web-blue  (10 Pods, imagem blue)
                  web-green   (10 Pods, imagem green — ocioso)

Após o switch (tráfego → Green):

  Service [web] ──→ web-green (10 Pods, imagem green)
                  web-blue    (10 Pods, imagem blue — standby)
```

## Como configurar

Dois Deployments separados, diferenciados pelo label `version`, e um Service que aponta para apenas um deles:

```yaml
# Deployment Blue (versão atual em produção)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-blue
spec:
  replicas: 10
  selector:
    matchLabels:
      app: web
      version: blue
  template:
    metadata:
      labels:
        app: web
        version: blue
    spec:
      containers:
      - name: web
        image: fabricioveronez/web-color:blue
        ports:
        - containerPort: 80
---
# Deployment Green (nova versão)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-green
spec:
  replicas: 10
  selector:
    matchLabels:
      app: web
      version: green
  template:
    metadata:
      labels:
        app: web
        version: green
    spec:
      containers:
      - name: web
        image: fabricioveronez/web-color:green
        ports:
        - containerPort: 80
---
# Service aponta para a versão ativa (Blue)
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
    version: blue    # troque para "green" para mudar o tráfego
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30000
  type: NodePort
```

## Trocar o tráfego

```bash
# Aplicar os recursos
kubectl apply -f deployment.yaml

# Verificar que ambos os Deployments estão prontos
kubectl get deployments web-blue web-green
kubectl get pods -l app=web

# Trocar tráfego para Green
kubectl patch service web -p '{"spec":{"selector":{"app":"web","version":"green"}}}'

# Rollback para Blue (se necessário)
kubectl patch service web -p '{"spec":{"selector":{"app":"web","version":"blue"}}}'
```

Também é possível editar o Service manualmente:

```bash
kubectl edit service web
# Altere version: blue → version: green
```

## Vantagens

- **Troca instantânea** de tráfego — sem rollout gradual.
- **Rollback imediato**: basta reverter o selector do Service.
- Permite **testar a versão Green** com tráfego real antes do switch (via Service temporário ou port-forward).
- **Zero downtime** na troca, desde que Green esteja saudável antes do switch.

## Desvantagens

- **Dobra o consumo de recursos** enquanto ambas as versões estão ativas (10 + 10 Pods).
- Não é estratégia nativa do Deployment — exige **dois Deployments** e gestão manual (ou automação externa).
- Ambas as versões precisam ser **compatíveis com o mesmo backend** (banco, cache, filas).
- Após o switch, a versão antiga continua rodando até você decidir removê-la.

## Blue-Green x Ramped x Recreate

| Característica        | Blue-Green                  | Ramped (RollingUpdate)     | Recreate                      |
|-----------------------|-----------------------------|----------------------------|-------------------------------|
| Downtime              | Não                         | Não                        | Sim                           |
| Troca de tráfego      | Instantânea (Service)       | Gradual (Pod a Pod)        | Tudo de uma vez               |
| Recursos extras       | Sim (2 ambientes completos) | Poucos Pods temporários    | Não                           |
| Rollback              | Instantâneo                 | `kubectl rollout undo`     | Redeploy da versão anterior   |
| Uso recomendado       | Releases críticos / validação prévia | Produção geral      | Apps que não toleram mix      |
