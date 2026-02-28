# Kubernetes Restart Policy

A política de reinicialização (`restartPolicy`) define como o Kubernetes deve se comportar quando os containers de um Pod param de funcionar. Ela é configurada no nível do Pod e se aplica a todos os containers contidos nele.

## Opções de Configuração

Existem três valores possíveis para o campo `restartPolicy`:

| Valor | Comportamento | Uso Comum |
| :--- | :--- | :--- |
| **`Always`** | O container é reiniciado automaticamente sempre que para, seja por erro ou por finalização bem-sucedida. Este é o **valor padrão**. | Deployments, StatefulSets, DaemonSets. |
| **`OnFailure`** | O container é reiniciado apenas se o processo terminar com um código de erro (diferente de zero). | Jobs e tarefas que devem rodar até o sucesso. |
| **`Never`** | O container nunca é reiniciado, independente do motivo da parada. | Scripts de manutenção única ou diagnósticos. |

## Funcionamento e Back-off

Quando um container precisa ser reiniciado, o Kubernetes não o faz instantaneamente de forma infinita. Ele utiliza um algoritmo de **atraso exponencial** (exponential back-off delay):

- O tempo de espera entre as tentativas aumenta: 10s, 20s, 40s, 80s...
- O limite máximo de espera é de **5 minutos**.
- O contador de back-off é resetado quando o container roda com sucesso por pelo menos 10 minutos.

## Exemplos Práticos

### 1. Pod com Restart Policy `Always` (Padrão)
Ideal para serviços que devem estar sempre online.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-always
spec:
  containers:
  - name: nginx
    image: nginx
  restartPolicy: Always
```

### 2. Job com Restart Policy `OnFailure`
Ideal para processamento de dados onde você quer que a tarefa tente novamente se falhar, mas pare se terminar corretamente.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-processor
spec:
  containers:
  - name: processor
    image: busybox
    command: ["sh", "-c", "exit 1"] # Simulando falha
  restartPolicy: OnFailure
```

## Resumo de Comportamento por Tipo de Recurso

- **Deployments/ReplicaSets:** Só aceitam `restartPolicy: Always`.
- **Jobs/CronJobs:** Aceitam `OnFailure` ou `Never`.
- **Pods isolados:** Aceitam qualquer uma das três opções.
