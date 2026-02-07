# Docker Capabilities

Linux capabilities são uma forma de dividir os privilégios tradicionalmente associados ao usuário root em unidades menores e distintas. Ao executar containers Docker, você pode adicionar ou remover capabilities específicas para aumentar a segurança, seguindo o princípio do menor privilégio.

## O que são Capabilities?

Capabilities permitem que processos tenham apenas os privilégios necessários para funcionar, sem precisar rodar como root completo. Por exemplo, um processo pode ter permissão para fazer bind em portas privilegiadas (< 1024) sem ter todos os poderes do root.

## Capabilities Padrão no Docker

Por padrão, o Docker adiciona um conjunto limitado de capabilities aos containers:
- `CHOWN` - alterar ownership de arquivos
- `DAC_OVERRIDE` - ignorar permissões de leitura/escrita/execução
- `FSETID` - não limpar bits setuid/setgid quando arquivo é modificado
- `FOWNER` - ignorar permissões em operações que requerem ownership
- `MKNOD` - criar arquivos especiais usando mknod
- `NET_RAW` - usar sockets RAW e PACKET
- `SETGID` - manipular GIDs de processos
- `SETUID` - manipular UIDs de processos
- `SETFCAP` - definir file capabilities
- `SETPCAP` - modificar capabilities de processos
- `NET_BIND_SERVICE` - fazer bind em portas < 1024
- `SYS_CHROOT` - usar chroot()
- `KILL` - enviar sinais para processos
- `AUDIT_WRITE` - escrever logs de auditoria

## Usar Capabilities no Docker

### Remover todas as capabilities (máxima segurança)
```bash
docker run --cap-drop=ALL nginx
```

### Adicionar capabilities específicas
```bash
# Adicionar NET_ADMIN para configurar interfaces de rede
docker run --cap-add=NET_ADMIN nginx

# Adicionar múltiplas capabilities
docker run --cap-add=NET_ADMIN --cap-add=SYS_TIME nginx
```

### Remover capabilities específicas
```bash
# Remover NET_RAW para prevenir packet sniffing
docker run --cap-drop=NET_RAW nginx
```

### Combinar operações
```bash
# Remover todas e adicionar apenas as necessárias (abordagem recomendada)
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE --cap-add=CHOWN nginx
```

## Docker Compose

```yaml
services:
  web:
    image: nginx
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
      - CHOWN
```

## Capabilities Comuns

| Capability | Descrição | Uso Comum |
|-----------|-----------|-----------|
| `NET_ADMIN` | Configurar interfaces de rede, routing, firewall | VPNs, proxies, ferramentas de rede |
| `NET_BIND_SERVICE` | Bind em portas < 1024 | Web servers, servidores que precisam de portas privilegiadas |
| `SYS_TIME` | Modificar relógio do sistema | Servidores NTP |
| `SYS_ADMIN` | Várias operações administrativas | **Evitar!** Quase equivalente a root |
| `DAC_READ_SEARCH` | Ignorar permissões de leitura de arquivos/diretórios | Backup tools, scanners de arquivos |
| `CHOWN` | Alterar ownership de arquivos | Aplicações que gerenciam permissões |

## Listar Capabilities de um Container

```bash
# Ver capabilities de um processo dentro do container
docker exec <container_id> grep Cap /proc/1/status

# Ou usar capsh (se disponível no container)
docker exec <container_id> capsh --print
```

## Verificar Capabilities com getpcaps

```bash
# Instalar libcap2-bin (Debian/Ubuntu)
apt-get install libcap2-bin

# Ver capabilities de um processo
getpcaps <pid>
```

## Boas Práticas de Segurança

1. **Princípio do Menor Privilégio**: Sempre comece com `--cap-drop=ALL` e adicione apenas as capabilities necessárias
2. **Evite SYS_ADMIN**: Esta capability é muito poderosa e deve ser evitada
3. **Teste Extensivamente**: Depois de restringir capabilities, teste todos os cenários de uso
4. **Documente**: Mantenha documentação sobre quais capabilities sua aplicação precisa e por quê
5. **Use User Namespaces**: Combine com user namespaces para camadas adicionais de segurança
6. **Auditoria Regular**: Revise periodicamente as capabilities concedidas aos containers

## Debugging

Se sua aplicação falhar após remover capabilities:

1. Verifique os logs do container: `docker logs <container_id>`
2. Procure por erros relacionados a "permission denied" ou "operation not permitted"
3. Use `strace` para identificar syscalls falhando
4. Adicione capabilities uma por uma até identificar a necessária

## Links Úteis

- [Linux Capabilities Man Page](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [Docker Run Reference - Runtime Privilege](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities)
