# LimitRange

## O que é?

O **LimitRange** é um objeto do Kubernetes que define restrições de consumo de recursos (CPU e memória) para objetos dentro de um **namespace**. Ele atua como uma política de governança que garante que nenhum container ou pod consuma recursos além do permitido — ou abaixo de um mínimo aceitável.

## Para que serve?

O LimitRange resolve três problemas principais em ambientes Kubernetes:

1. **Definir valores padrão de requests/limits**
   - Se um container for criado sem especificar `requests` ou `limits`, o LimitRange aplica os valores padrão automaticamente

2. **Impor limites mínimos e máximos**
   - Impede que um container seja criado com recursos excessivos (ex: 100Gi de memória) ou insuficientes (ex: 1m de CPU)

3. **Controlar a proporção entre request e limit**
   - Define a razão máxima permitida entre `limit` e `request` de um recurso (ex: limit não pode ser mais que 2x o request)

## Quando usar?

- Em ambientes **multi-tenant** onde múltiplos times compartilham o mesmo cluster
- Para evitar que um **pod mal configurado** consuma todos os recursos de um node
- Quando você quer garantir **QoS (Quality of Service)** previsível nos pods
- Em namespaces de **desenvolvimento/homologação** para evitar desperdício de recursos
- Para complementar o **ResourceQuota**, que limita o total do namespace, enquanto o LimitRange limita por objeto individual

## Tipos de escopo

| Tipo        | Descrição                                      |
|-------------|------------------------------------------------|
| `Container` | Limites por container dentro de um pod         |
| `Pod`       | Limites para o somatório de todos os containers do pod |
| `PersistentVolumeClaim` | Limites de tamanho para PVCs       |

## Exemplo de configuração

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limite-recursos
  namespace: meu-namespace
spec:
  limits:
    - type: Container
      default:          # valor de limit aplicado se não for especificado
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:   # valor de request aplicado se não for especificado
        cpu: "100m"
        memory: "128Mi"
      max:              # valor máximo permitido
        cpu: "2"
        memory: "1Gi"
      min:              # valor mínimo permitido
        cpu: "50m"
        memory: "64Mi"
```

## Como funciona na prática?

```
Pod criado sem limits/requests
        ↓
Kubernetes verifica LimitRange do namespace
        ↓
Aplica os valores de `default` e `defaultRequest`
        ↓
Valida se os valores estão dentro de `min` e `max`
        ↓
✅ Pod criado com valores garantidos
❌ Pod rejeitado se violar os limites
```

## Impacto no QoS (Quality of Service)

O LimitRange influencia diretamente a classe de QoS dos pods:

| Classe         | Condição                                  | Prioridade de eviction |
|----------------|-------------------------------------------|------------------------|
| `Guaranteed`   | request == limit para todos os containers | Última a ser removida  |
| `Burstable`    | request < limit                           | Intermediária          |
| `BestEffort`   | Sem request e limit definidos             | Primeira a ser removida|

## Comandos úteis

```bash
# Criar um LimitRange a partir de um arquivo
kubectl apply -f limitrange.yaml

# Listar LimitRanges de um namespace
kubectl get limitrange -n meu-namespace

# Ver detalhes de um LimitRange
kubectl describe limitrange limite-recursos -n meu-namespace

# Deletar um LimitRange
kubectl delete limitrange limite-recursos -n meu-namespace
```

## Diferença entre LimitRange e ResourceQuota

| Feature        | LimitRange                        | ResourceQuota                       |
|----------------|-----------------------------------|-------------------------------------|
| Escopo         | Por objeto (container, pod, PVC)  | Total do namespace                  |
| Objetivo       | Limitar recursos individuais      | Limitar consumo agregado            |
| Valores padrão | Sim, aplica automaticamente       | Não                                 |
| Uso combinado  | ✅ São complementares — use os dois juntos para controle completo |

## Documentação oficial

https://kubernetes.io/docs/concepts/policy/limit-range/
