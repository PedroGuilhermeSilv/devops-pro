# Metrics Server

## O que é?

O **Metrics Server** é um componente oficial do Kubernetes que coleta métricas de recursos (CPU e memória) dos nodes e pods no cluster. Ele funciona como uma fonte de dados agregada e em tempo real sobre o consumo de recursos.

## Para que serve?

O Metrics Server é **essencial** para várias funcionalidades do Kubernetes:

1. **Comandos `kubectl top`**
   - `kubectl top nodes` - Ver uso de CPU/memória dos nodes
   - `kubectl top pods` - Ver uso de CPU/memória dos pods

2. **Horizontal Pod Autoscaler (HPA)**
   - Permite escalar automaticamente pods baseado em métricas de CPU/memória
   - Sem o Metrics Server, o HPA não funciona

3. **Vertical Pod Autoscaler (VPA)**
   - Ajusta automaticamente os limites de recursos dos containers

4. **Scheduler baseado em recursos**
   - Ajuda o Kubernetes a tomar decisões de agendamento mais inteligentes

## Como funciona?

- Coleta métricas via **kubelet** de cada node
- Armazena métricas em **memória** (não persiste histórico)
- Expõe dados através da **Metrics API** (`metrics.k8s.io`)
- Atualiza métricas a cada **60 segundos** por padrão

## Instalação

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

## Verificar se está funcionando

```bash
# Ver status do metrics-server
kubectl get deployment metrics-server -n kube-system

# Testar comandos
kubectl top nodes
kubectl top pods -A
```

## Projeto Metrics Server

https://github.com/kubernetes-sigs/metrics-server