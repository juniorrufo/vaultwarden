# Arquitetura Técnica — Infraestrutura Vaultwarden

Este documento apresenta a especificação técnica detalhada da infraestrutura que suporta o deployment do Vaultwarden, descrevendo cada componente, sua configuração em produção e as interações de rede.

---

## 1. Topologia Geral e Visão de Componentes

A infraestrutura foi projetada para ambiente doméstico/pessoal, priorizando isolamento de privilégios, estabilidade operacional e mínima superfície de ataque.

```text
                             INTERNET
                                 │
                                 ▼
                        ┌────────────────┐
                        │   Cloudflare   │
                        │ DNS / TLS      │
                        └───────┬────────┘
                                │
                                ▼
                         Cloudflare Tunnel
                                │
                                ▼
                         192.168.15.253
                         Cloudflared LXC
                                │
                           HTTP :8080
                                │
                                ▼
                         192.168.15.200
                         Vaultwarden VM
                                │
                          Docker Engine
                                │
                                ▼
                         Vaultwarden :80
                                │
                                ▼
                             SQLite
```

---

## 2. Camada de Virtualização (Proxmox VE)

- **Hypervisor:** Proxmox VE rodando KVM.
- **Isolamento da Aplicação:** O Vaultwarden executa em uma Máquina Virtual (VM) dedicada, não compartilhando o sistema operacional nem o espaço de usuários com outros serviços ou containers de terceiros.
- **QEMU Guest Agent:**
  - Pacote `qemu-guest-agent` instalado como serviço estático do systemd (`qemu-guest-agent.service`).
  - Canal de comunicação com o hypervisor: `/dev/virtio-ports/org.qemu.guest_agent.0`.
  - Status em produção: `active (running)`.
  - Finalidade: Permite comandos de quiescing de disco, gerenciamento limpo de shutdown/reboot e telemetria de rede para o Proxmox.
- **Camada de Backup da VM:** Snapshot de baseline da VM provisionada com SO, rede, Docker e hardening gerado e validado via Proxmox Backup Server (PBS).

---

## 3. Sistema Operacional (VM Debian)

- **Hostname:** `vaultwarden`
- **Distribuição:** Debian GNU/Linux 13 (trixie)
- **Arquitetura:** `x86-64 / amd64`
- **Kernel em Produção:** `Linux 6.12.107+deb13-amd64`
- **Recursos Alocados:**
  - CPU: 2 vCPU
  - Memória: ~2 GB RAM
  - Disco: ~32 GB
- **Rede Local:**
  - Interface primária: `ens18`
  - Endereço IPv4: `192.168.15.200/24` (estático/reservado)
  - Gateway padrão: `192.168.15.1`
  - IPv6: Habilitado na interface pela rede local, porém expressamente bloqueado para tráfego de entrada não solicitado (inclusive SSH).
- **Fuso Horário e NTP:**
  - Timezone: `America/Sao_Paulo`
  - Sincronização de relógio via NTP: ativo e sincronizado (crítico para validação temporal de TOTP, tokens de sessão e certificados TLS).
- **Modelo de Privilégios:**
  - Usuário administrativo: `junior`.
  - O usuário `junior` possui acesso `sudo`, mas **não** pertence ao grupo `docker` (exigência de execução explícita via `sudo docker`).

---

## 4. Rede Local (LAN) e Comunicação entre Hosts

A sub-rede local é `192.168.15.0/24`. Os nós relevantes para esta arquitetura são:

| Host / Instância | Endereço IP | Função na Infraestrutura |
| :--- | :--- | :--- |
| **Vaultwarden VM** | `192.168.15.200` | Hospeda o container da aplicação e armazenamento SQLite |
| **Cloudflared LXC** | `192.168.15.253` | Ponto de saída do túnel gerenciado para a Cloudflare |
| **Nginx Proxy Manager** | `192.168.15.251` | Reverse proxy para outros serviços da LAN (**NÃO utilizado pelo Vaultwarden**) |
| **Gateway / Roteador** | `192.168.15.1` | Roteamento interno da rede residencial |

### Por que o Nginx Proxy Manager (NPM) não é utilizado:
A arquitetura conecta o Cloudflare Tunnel diretamente à VM do Vaultwarden (`Cloudflare -> Cloudflare Tunnel -> Vaultwarden`), descartando o uso de NPM para esta aplicação:
- Elimina uma camada redundante de proxy reverso e pontos únicos de falha.
- Reduz a complexidade de manutenção e a superfície de ataque.
- A Cloudflare já entrega o certificado TLS na ponta e proteção WAF.
- Evita que uma eventual indisponibilidade do NPM afete o gerenciador de senhas.

---

## 5. Ponto de Entrada Público (Cloudflare Tunnel)

- **URL Pública Oficial:** `https://vault.rufonex.com.br`
- **Certificado TLS e Terminação:** Gerenciados na borda da Cloudflare (*Cloudflare Edge*).
- **Execução do Cloudflared:** Container LXC dedicado (`192.168.15.253`).
- **Autenticação do Túnel:** Gerenciado remotamente via token em `/etc/cloudflared/token` (não necessita de arquivo `cert.pem` ou `config.yml` local).
- **Destino do Ingress (Origin):** `http://192.168.15.200:8080`.
- **Ausência de Port Forwarding:** Não existem portas TCP abertas ou redirecionadas no roteador residencial para a Internet. O tráfego entra através da conexão outbound segura mantida pelo daemon `cloudflared`.

---

## 6. Arquitetura de Firewall e Integração com Docker

O host **não** utiliza um arquivo `/etc/nftables.conf` gerenciado diretamente, pois o Docker Engine gerencia suas regras dinâmicas por meio da interface `iptables-nft` (v1.8.11). O serviço `nftables.service` do Debian foi intencionalmente mascarado (`masked`).

### 6.1. Política Base do Host (netfilter-persistent)
- **IPv4:**
  - `INPUT`: `DROP`
  - `FORWARD`: `DROP`
  - `OUTPUT`: `ACCEPT`
  - SSH (`TCP/22`): Permitido exclusivamente a partir de `192.168.15.0/24`.
- **IPv6:**
  - `INPUT`: `DROP`
  - `FORWARD`: `DROP`
  - `OUTPUT`: `ACCEPT`
  - SSH por IPv6 expressamente descartado.
- Tráfego básico permitido: Loopback (`lo`), estados `ESTABLISHED,RELATED`, ICMP (IPv4) e mensagens essenciais de ICMPv6 (Neighbor Discovery).

### 6.2. Cadeia DOCKER-USER e Regras VW-DOCKER
Como o Docker realiza DNAT no estágio `PREROUTING` da tabela `nat` antes de encaminhar o tráfego para a cadeia `FORWARD`, a política padrão `INPUT DROP` não impede acessos indesejados às portas publicadas pelo Docker.

Para solucionar isso sem quebrar a integração do Docker, a filtragem administrativa ocorre na cadeia oficial `DOCKER-USER`:

```text
DOCKER-USER
    │
    ▼
VW-DOCKER
```

Regras da cadeia `VW-DOCKER`:
```text
1. -A VW-DOCKER -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
2. -A VW-DOCKER -s 192.168.15.253/32 -p tcp -m conntrack --ctorigdst 192.168.15.200 --ctorigdstport 8080 -j ACCEPT
3. -A VW-DOCKER -p tcp -m conntrack --ctorigdst 192.168.15.200 --ctorigdstport 8080 -j DROP
4. -A VW-DOCKER -j RETURN
```

**Papel do Conntrack (`--ctorigdst`):**
Como o IP de destino já foi reescrito para o IP da bridge interna antes de chegar em `FORWARD`, a correspondência deve ser feita com o destino original (`--ctorigdst 192.168.15.200` e `--ctorigdstport 8080`).

Isso garante que:
- O LXC Cloudflared (`192.168.15.253`) consegue se conectar em `http://192.168.15.200:8080`.
- Qualquer outro host da LAN que tente acessar `http://192.168.15.200:8080` recebe timeout imediato.
- A persistência é assegurada por `/usr/local/sbin/vaultwarden-docker-firewall` e `vaultwarden-docker-firewall.service`.

---

## 7. Fluxo de Tráfego Completo

### 7.1. Fluxo Externo (Acesso à Aplicação)
```text
Internet (Usuário / Extensão / Web Vault)
   ↓ (HTTPS / Porta 443)
Cloudflare Edge (Terminação TLS, WAF, Proteção DDoS)
   ↓ (Túnel Outbound Seguro)
Cloudflared LXC (192.168.15.253)
   ↓ (HTTP / Porta 8080 na rede LAN privada)
Interface ens18 da VM Vaultwarden (192.168.15.200:8080)
   ↓ (PREROUTING nat do Docker -> DNAT para bridge 172.x)
FORWARD (iptables)
   ↓ (Avaliado na cadeia DOCKER-USER -> VW-DOCKER)
Filtro conntrack verifica se origem é 192.168.15.253:
   ├── Se SIM: ACCEPT
   └── Se NÃO: DROP (timeout)
Rede bridge vaultwarden_net
   ↓ (Porta 80)
Container Vaultwarden (Processo da aplicação)
   ↓
Banco SQLite (/opt/vaultwarden/data/db.sqlite3)
```

### 7.2. Fluxo Interno (Gerenciamento e SSH)
```text
Estação de Trabalho Administrativa (192.168.15.x na LAN)
   ↓ (TCP / Porta 22)
Firewall da VM (Regra INPUT para 192.168.15.0/24: ACCEPT)
   ↓
Serviço SSHd (junior@192.168.15.200)
   ↓ (Autenticação obrigatória por chave pública Ed25519)
Shell do Sistema
   ↓ (sudo docker ...)
Docker Daemon / Arquivos em /opt/vaultwarden
```

---

## 8. Motor de Containers (Docker)

- **Versões dos Componentes:**
  - Docker Engine: `29.8.1`
  - Docker Compose: `5.5.1`
  - containerd: `2.3.5`
  - runc: `1.5.1`
  - Buildx: `0.37.1`
- **Configurações de Engine:**
  - Storage Driver: `overlayfs`
  - Cgroup Driver: `systemd` (Cgroup v2)
  - Firewall Backend: `iptables`
  - Diretório raiz: `/var/lib/docker`
- **Configuração do Daemon (`/etc/docker/daemon.json`):**
  ```json
  {
    "log-driver": "local",
    "log-opts": {
      "max-size": "20m",
      "max-file": "5"
    },
    "live-restore": true
  }
  ```
- **Rede do Compose:**
  - Rede bridge dedicada declarada: `vaultwarden_net`.
  - Publicação de porta com amarração estrita de IP: `192.168.15.200:8080:80` (não exposto em `0.0.0.0:8080`).

---

## 9. Container Vaultwarden

- **Versão da Aplicação:** `1.37.3`
- **Imagem Paginada por Digest:**
  ```text
  vaultwarden/server@sha256:4ecafc9049c7d878c7717d1ce4f9059d706758c78b8fa42e5ead21f4b2dfc770
  ```
  *(A tag `latest` não é utilizada para evitar quebras silenciosas em pull automatizado).*
- **Arquitetura do Container:** Contêiner único, sem serviços auxiliares (sem PostgreSQL, sem Redis, sem Watchtower, sem Portainer).
- **Parâmetros de Segurança:**
  - `restart: unless-stopped`
  - `security_opt: [no-new-privileges:true]`
  - `stop_grace_period: 30s`
  - Healthcheck nativo em `/alive` (HTTP 200).

---

## 10. Persistência de Dados e Banco SQLite

- **Diretório no Host:** `/opt/vaultwarden/data`
- **Ponto de Montagem no Container:** `/opt/vaultwarden/data:/data`
- **Motor do Banco de Dados:** SQLite (`db.sqlite3`).
- **Motivação do SQLite:**
  - Ideal para instâncias pessoais e familiares.
  - Elimina a complexidade de operar um SGBD adicional (PostgreSQL).
  - Facilidade direta de backup e recuperação atômica.
  - Baixo consumo de memória e CPU.
- **Artefatos Armazenados em `/opt/vaultwarden/data`:**
  - `db.sqlite3`: Banco de dados principal.
  - `rsa_key.pem`: Chave criptográfica da instância.
  - `icon_cache/`: Cache local de ícones dos serviços cadastrados.
  - `tmp/`: Diretório temporário interno.

---

## 11. Estratégia de Backup e Relação entre os Componentes

A resiliência da infraestrutura apoia-se em níveis complementares e desacoplados:

1. **Aplicação (Local):** O script `/usr/local/sbin/vaultwarden-backup` aciona o backup nativo do SQLite (`docker exec vaultwarden /vaultwarden backup`), interrompe o container graciosamente, empacota `/opt/vaultwarden/data` em `/var/backups/vaultwarden/*.tar.gz` (excluindo journals e temporários), gera hash SHA-256, valida integridade estrutural via `tar -tzf`, reinicia o container e aplica a retenção local de 10 dias (`RETENTION_DAYS=10`). A automação via `vaultwarden-backup.timer` (03:00 diário) teve execução noturna real validada (`vaultwarden_20260923_030040.tar.gz`).
2. **Sistema Operacional (Proxmox):** Snapshot completo do estado baseline da VM validado via Proxmox Backup Server (PBS).
3. **Desastre Externo (Off-site na Nuvem - OCI via Restic):** Repositório Restic (ID `7bbbe221`) hospedado em bucket privado no Oracle Cloud Infrastructure (`sa-saopaulo-1`, namespace `groqo9fbzuaz`, compartment `Backups`, bucket `vaultwarden-offsite`, via API S3-compatible). O Restic aplica criptografia client-side antes da transmissão. A rotina é orquestrada pelo script `/usr/local/sbin/vaultwarden-offsite-backup`, serviço systemd `vaultwarden-offsite-backup.service` (`Environment=HOME=/root`) e timer diário `vaultwarden-offsite-backup.timer` (03:30, `Persistent=true`). O script consome o backup local de `/var/backups/vaultwarden/`, valida o checksum SHA-256 antes da transmissão, envia os dados, aplica retenção remota de 10 dias (`forget --keep-within 10d --prune`) e audita o repositório com `restic check`. A execução manual do serviço foi validada com sucesso; o disparo automático do timer às 03:30 aguarda observação noturna. A restauração foi homologada com sucesso em ambiente isolado (`/tmp/restic-vaultwarden-restore`), com validação exata do hash SHA-256 e integridade de `./db.sqlite3` e `./rsa_key.pem`, sem afetar a produção.

### Diagrama de Relacionamento de Backup:

```text
+--------------------------------------------------------------------+
|                         VM 192.168.15.200                          |
|                                                                    |
|  /opt/vaultwarden/data (db.sqlite3, rsa_key.pem)                   |
|         │                                                          |
|         ▼ (vaultwarden-backup.service via timer 03:00)             |
|  /var/backups/vaultwarden/*.tar.gz                                 |
|         │  (Retenção local: 10 dias)                               |
|         │                                                          |
|         ▼ (vaultwarden-offsite-backup.service via timer 03:30)     |
|         │ (Restic: criptografia client-side + retenção remota 10d) |
+---------┼----------------------------------------------------------+
          │
          ▼ (HTTPS / API S3-compatible)
+--------------------------------------------------------------------+
|                Oracle Cloud Infrastructure (OCI)                   |
|  Região: sa-saopaulo-1 | Compartment: Backups                      |
|  Bucket privado: vaultwarden-offsite                               |
|  Restic Repository ID: 7bbbe221 (Retenção remota: 10 dias)         |
+--------------------------------------------------------------------+
```
