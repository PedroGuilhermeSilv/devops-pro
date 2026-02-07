# Imagens Distroless

Imagens distroless são imagens de container minimalistas que contêm apenas a aplicação e suas dependências de runtime, sem um sistema operacional completo, shell, package managers ou outras ferramentas típicas de distribuições Linux.

## O que são Imagens Distroless?

Desenvolvidas pelo Google, imagens distroless removem tudo que não é essencial para executar sua aplicação:
- ❌ Sem shell (bash, sh)
- ❌ Sem package managers (apt, yum, apk)
- ❌ Sem utilitários do sistema (curl, wget, netcat)
- ✅ Apenas runtime da linguagem + aplicação
- ✅ Bibliotecas essenciais do sistema

## Vantagens

### 1. **Segurança Aprimorada**
- Superfície de ataque drasticamente reduzida
- Menos vulnerabilidades (menos software = menos CVEs)
- Impossível executar shell para exploits
- Dificulta ataques de container escape

### 2. **Tamanho Reduzido**
- Imagens muito menores (geralmente < 50MB)
- Deploy mais rápido
- Menos uso de banda e armazenamento
- Pull de imagens mais veloz

### 3. **Compliance e Auditoria**
- Apenas código que você controla está presente
- Fácil de auditar (menos componentes)
- Melhor para ambientes regulados

### 4. **Melhor Performance**
- Menos overhead
- Startup mais rápido
- Menor uso de recursos

## Desvantagens

- 🔴 **Debugging difícil**: Sem shell, sem ferramentas de diagnóstico
- 🔴 **Curva de aprendizado**: Requer mudança de mentalidade
- 🔴 **Limitações em dev**: Não adequado para desenvolvimento local

## Imagens Distroless Disponíveis

Google mantém imagens distroless para as principais linguagens:

| Imagem | Descrição | Tamanho Aproximado |
|--------|-----------|-------------------|
| `gcr.io/distroless/static-debian12` | Binários estáticos (Go, Rust) | ~2MB |
| `gcr.io/distroless/base-debian12` | glibc + runtime básico | ~20MB |
| `gcr.io/distroless/cc-debian12` | Com libstdc++ (C/C++) | ~25MB |
| `gcr.io/distroless/java17-debian12` | Java 17 JRE | ~150MB |
| `gcr.io/distroless/java21-debian12` | Java 21 JRE | ~160MB |
| `gcr.io/distroless/python3-debian12` | Python 3 | ~50MB |
| `gcr.io/distroless/nodejs20-debian12` | Node.js 20 | ~120MB |

## Exemplos Práticos

### Go Application (Static Binary)

```dockerfile
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Runtime stage - imagem distroless
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/main /
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/main"]
```

### Java Application

```dockerfile
# Build stage
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Runtime stage - imagem distroless
FROM gcr.io/distroless/java21-debian12
COPY --from=builder /app/target/app.jar /app.jar
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### Python Application

```dockerfile
# Build stage
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Runtime stage - imagem distroless
FROM gcr.io/distroless/python3-debian12
COPY --from=builder /root/.local /root/.local
COPY app.py /app/
WORKDIR /app
ENV PATH=/root/.local/bin:$PATH
ENV PYTHONUNBUFFERED=1
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["python3", "app.py"]
```

### Node.js Application

```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

# Runtime stage - imagem distroless
FROM gcr.io/distroless/nodejs20-debian12
COPY --from=builder /app /app
WORKDIR /app
USER nonroot:nonroot
EXPOSE 3000
CMD ["index.js"]
```

### Rust Application

```dockerfile
# Build stage
FROM rust:1.75 AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
RUN rm -rf src
COPY src ./src
RUN touch src/main.rs
RUN cargo build --release

# Runtime stage - imagem distroless
FROM gcr.io/distroless/cc-debian12
COPY --from=builder /app/target/release/app /
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]
```

## Variantes de Imagens

Cada imagem distroless tem variantes com tags específicas:

```bash
# Versão base
gcr.io/distroless/base-debian12

# Com debug tools (shell e busybox) - NÃO use em produção
gcr.io/distroless/base-debian12:debug

# Versão não-root (recomendado para segurança extra)
gcr.io/distroless/base-debian12:nonroot

# Debug + nonroot
gcr.io/distroless/base-debian12:debug-nonroot

# Versão específica (recomendado para prod)
gcr.io/distroless/base-debian12:latest-amd64
```

## Debugging de Containers Distroless

### Opção 1: Usar Imagem de Debug Localmente

```bash
# Build com tag debug para desenvolvimento
docker build --target debug -t myapp:debug .
docker run myapp:debug
```

```dockerfile
# Adicionar stage de debug no Dockerfile
FROM gcr.io/distroless/base-debian12:debug AS debug
COPY --from=builder /app/main /
ENTRYPOINT ["/busybox/sh"]

FROM gcr.io/distroless/base-debian12 AS production
COPY --from=builder /app/main /
ENTRYPOINT ["/main"]
```

### Opção 2: Ephemeral Debug Container (Kubernetes)

```bash
# Anexar container debug temporário
kubectl debug -it pod/mypod --image=busybox --target=mypod
```

### Opção 3: Docker Exec com Imagem Debug

```bash
# Inspecionar filesystem com container compartilhado
docker run --rm -it \
  --pid=container:<container_id> \
  --net=container:<container_id> \
  --cap-add SYS_PTRACE \
  gcr.io/distroless/base-debian12:debug
```

### Opção 4: Logs e Observabilidade

Como não há shell, invista em:
- Logging estruturado (JSON)
- Métricas (Prometheus, StatsD)
- Tracing distribuído (OpenTelemetry)
- Health checks HTTP

```go
// Exemplo: logging estruturado em Go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("server started", "port", 8080, "env", "production")
```

## Verificação de Vulnerabilidades

Imagens distroless têm significativamente menos vulnerabilidades:

```bash
# Comparar imagens
trivy image ubuntu:22.04
# CRITICAL: 5, HIGH: 20, MEDIUM: 50, LOW: 100

trivy image gcr.io/distroless/base-debian12
# CRITICAL: 0, HIGH: 0, MEDIUM: 1, LOW: 3

# Scan de imagem distroless
docker scout cves gcr.io/distroless/static-debian12
```

## Boas Práticas

### 1. **Use Multi-Stage Builds**
Sempre separe build e runtime para imagens mínimas:
```dockerfile
FROM builder AS build
# ... build steps ...

FROM gcr.io/distroless/static-debian12
COPY --from=build /app/binary /
```

### 2. **Especifique Usuário Não-Root**
```dockerfile
USER nonroot:nonroot
# ou
USER 65532:65532
```

### 3. **Use Tags Específicas**
Evite `:latest`, prefira versões fixas:
```dockerfile
FROM gcr.io/distroless/base-debian12:latest-amd64
# melhor ainda: use digest SHA256
FROM gcr.io/distroless/base-debian12@sha256:abc123...
```

### 4. **Configure Health Checks**
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD ["/app", "healthcheck"]
```

### 5. **Otimize Dependências**
```dockerfile
# Python: instale apenas o necessário
RUN pip install --no-cache-dir --user gunicorn==21.2.0

# Node: use npm ci ao invés de npm install
RUN npm ci --only=production
```

### 6. **Use Certificados SSL**
Distroless inclui ca-certificates, mas verifique se sua app precisa de certificados personalizados:
```dockerfile
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
```

## Comparação de Tamanhos

| Base Image | Tamanho | Vulnerabilidades (aprox.) |
|-----------|---------|---------------------------|
| `ubuntu:22.04` | 77MB | 80+ |
| `alpine:3.19` | 7MB | 0-5 |
| `distroless/static` | 2MB | 0 |
| `distroless/base` | 20MB | 0-2 |
| `scratch` | 0MB | 0 (vazio) |

## Quando Usar Distroless

### ✅ Use Distroless Quando:
- Aplicação está pronta para produção
- Segurança é prioridade máxima
- Você tem boa observabilidade configurada
- Aplicação é stateless/cloud-native
- Compliance requer superfície mínima de ataque

### ❌ Não Use Distroless Quando:
- Está desenvolvendo/debugando ativamente
- Precisa executar scripts shell no container
- Aplicação depende de ferramentas do sistema
- Equipe não tem experiência com debugging avançado
- Ainda está prototipando

## Alternativas

- **Alpine Linux**: Menor que distribuições completas, mas com shell e package manager
- **Scratch**: Imagem completamente vazia (apenas para binários 100% estáticos)
- **Chainguard Images**: Similar a distroless, com foco em supply chain security
- **Wolfi**: Base minimalista da Chainguard com apk package manager

## Docker Compose com Distroless

```yaml
services:
  api:
    build:
      context: .
      target: production
    image: myapp:distroless
    ports:
      - "8080:8080"
    environment:
      - ENV=production
    # Health check via HTTP (sem shell)
    healthcheck:
      test: ["CMD", "/app", "healthcheck"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    # Sem shell, usar array format para command
    command: ["/app", "--port=8080"]
```

## Links Úteis

- [GitHub - GoogleContainerTools/distroless](https://github.com/GoogleContainerTools/distroless)
- [Distroless Container Images](https://github.com/GoogleContainerTools/distroless#why-should-i-use-distroless-images)
- [Kubernetes Ephemeral Containers](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#ephemeral-container)
- [Chainguard Images](https://www.chainguard.dev/chainguard-images)
