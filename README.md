# Infraestrutura Vaultwarden — Documentação e Governança

Repositório de documentação, configurações reproduzíveis e procedimentos operacionais da infraestrutura dedicada do **Vaultwarden** (gerenciador de senhas *self-hosted*), implantado em ambiente residencial privado sobre virtualização Proxmox VE.

---

## 1. Objetivo do Projeto

O objetivo principal desta infraestrutura é prover um serviço seguro, privado e altamente confiável de gerenciamento de senhas e credenciais para uso pessoal e familiar, guiando-se pelos seguintes pilares de engenharia:

- **Mínima Superfície de Ataque:** Nenhuma porta aberta diretamente na Internet; isolamento de portas internas no firewall.
- **Isolamento Forte de Privilégios:** Execução em Máquina Virtual (VM) dedicada no Proxmox, separando o serviço de senhas de outros serviços da rede.
- **Configuração Explícita e Reprodutível:** Infraestrutura declarada e documentada, permitindo recriação total a partir da documentação, templates versionados e backups.
- **Backups Consistentes e Restauração Validada:** Rotina de backup atômica com testes práticos de restauração realizados em produção.
- **Sem Segredos no Versionamento:** Políticas rigorosas de proteção e exclusão de credenciais em conformidade com o `.gitignore`.

---

## 2. Visão Geral da Arquitetura

A aplicação opera sobre uma VM dedicada com Debian 13, gerenciada por Docker Compose com banco de dados SQLite local. O acesso público é intermediado pela Cloudflare através de um túnel encriptado (*Cloudflare Tunnel*), dispensando qualquer redirecionamento de portas no roteador de borda.

```text
       [ Internet ]
            │
            ▼
     [ Cloudflare ] (DNS Anycast / Terminação TLS / WAF)
            │
            ▼
   [ Cloudflare Tunnel ] (Túnel seguro outbound)
            │
            ▼
   [ Cloudflared LXC ] (192.168.15.253)
            │
       HTTP :8080 (LAN privada)
            │
            ▼
 [ VM Debian: 192.168.15.200:8080 ]
            │
 [ DOCKER-USER / VW-DOCKER ] (Filtro iptables: autoriza APENAS 192.168.15.253)
            │
 [ Container Vaultwarden :80 ] (v1.37.3 - digest imutável)
            │
 [ /opt/vaultwarden/data ] (Bind mount no host)
            │
      [ SQLite ] (db.sqlite3)
```

---

## 3. Componentes Utilizados

| Componente | Especificação / Versão | Função / Observação |
| :--- | :--- | :--- |
| **Hypervisor** | Proxmox VE (KVM) | Hospeda a VM do Vaultwarden e o LXC do Cloudflared |
| **Agente de VM** | `qemu-guest-agent` | Comunicação com o hypervisor para backups consistentes e quiescing |
| **Sistema Operacional** | Debian GNU/Linux 13 (trixie) amd64 | Kernel `Linux 6.12.107+deb13-amd64` |
| **Recursos da VM** | 2 vCPU, ~2 GB RAM, ~32 GB Disco | Alocação enxuta e dimensionada para uso pessoal |
| **Rede da VM** | IP `192.168.15.200/24` (ens18) | Gateway `192.168.15.1`, IPv6 bloqueado para SSH |
| **Motor de Containers** | Docker Engine `29.8.1` / Compose `5.5.1` | Repositório oficial Docker; storage driver `overlayfs` |
| **Configuração Docker** | `/etc/docker/daemon.json` | Log local limitado (20MB x 5) e `live-restore: true` |
| **Aplicação** | Vaultwarden `1.37.3` | Fixado por digest SHA-256 (`sha256:4ecafc9049c7...`) |
| **Banco de Dados** | SQLite 3 | Base local em `/opt/vaultwarden/data/db.sqlite3` |
| **Túnel de Ingress** | Cloudflared em LXC (`192.168.15.253`) | Roteamento de `vault.rufonex.com.br` para porta 8080 da VM |
| **Firewall do Host** | `iptables-nft` / `netfilter-persistent` | `nftables.service` mascarado; integração via `DOCKER-USER` |
| **Sincronização Temporal** | NTP ativo (`America/Sao_Paulo`) | Essencial para validação de tokens TOTP e TLS |

---

## 4. Fluxo de Acesso

1. **Acesso Externo:**
   - Usuários e clientes Bitwarden conectam-se em `https://vault.rufonex.com.br`.
   - A Cloudflare encerra o TLS na borda e encaminha a requisição via túnel outbound ao container LXC Cloudflared (`192.168.15.253`).
   - O Cloudflared encaminha o tráfego HTTP para a interface local da VM em `http://192.168.15.200:8080`.
   - O firewall do host (`VW-DOCKER` em `DOCKER-USER`) valida via `conntrack` se a origem é rigorosamente `192.168.15.253`. Se for, permite o pacote; se não for, aplica `DROP`.
   - O tráfego entra na bridge interna `vaultwarden_net` e atinge a porta 80 do container.

2. **Acesso Administrativo (SSH):**
   - Restrito à rede local `192.168.15.0/24` na porta TCP 22.
   - Autenticação exclusivamente por chave pública Ed25519; login como `root` e autenticação por senha estão desativados.

---

## 5. Estrutura do Projeto

A organização dos arquivos e documentação do repositório é apresentada a seguir:

```text
vaultwarden/
├── README.md                      # Visão geral, requisitos, status e navegação do projeto
├── PROJECT_CONTEXT.md             # Fonte de verdade exaustiva da infraestrutura
├── ARCHITECTURE.md                # Detalhamento técnico da arquitetura, rede e fluxos
├── DECISIONS.md                   # Registro de Decisões Arquiteturais (ADRs)
├── SECURITY.md                    # Medidas e controles de segurança reais implementados
├── .gitignore                     # Regras de exclusão de segredos, bancos e backups
├── .env.example                   # Modelo público de variáveis de ambiente sem segredos
├── docker/                        # Arquivos e configurações do Docker (Compose, daemon.json)
├── firewall/                      # Scripts e serviços systemd de filtragem de rede
├── backup/                        # Scripts e serviços da rotina de backup local
├── docs/                          # Manuais operacionais práticos
│   ├── OPERATIONS.md              # Comandos do dia a dia, logs, status e upgrades
│   ├── BACKUP.md                  # Procedimentos detalhados da rotina de backup
│   ├── RESTORE.md                 # Procedimento de disaster recovery e teste validado
│   └── DOCKER-FIREWALL.md         # Explicação aprofundada da integração DOCKER-USER
└── diagrams/                      # Diagramas textuais e esquemáticos da topologia
    └── architecture.md            # Diagramas textuais e de sequência de rede
```

---

## 6. Requisitos do Ambiente

Para hospedar ou reconstruir esta infraestrutura, são necessários:
- **Hypervisor:** Proxmox VE com suporte a KVM e LXC.
- **Máquina Virtual:** 2 vCPU, 2 GB RAM, 32 GB de armazenamento SSD/NVMe.
- **Rede Privada:** Sub-rede IPv4 `192.168.15.0/24` com IP estático reservado para a VM (`192.168.15.200`).
- **Sistema Operacional:** Debian GNU/Linux 13 (trixie) com pacotes básicos e `qemu-guest-agent`.
- **Docker Engine:** Instalação oficial Docker (>= 29.x) com plugin Docker Compose.
- **Ingress:** Instância externa do Cloudflared (LXC `192.168.15.253`) vinculada à conta Cloudflare correspondente.

---

## 7. Visão Geral do Deployment

A aplicação opera sob controle estrito:
- **Isolamento de Portas:** A porta `8080` é publicada unicamente na interface da VM (`192.168.15.200:8080`), sem exposição em `0.0.0.0`.
- **Segurança do Container:** Uso obrigatório de `security_opt: [no-new-privileges:true]`, política de reinicialização `unless-stopped` e parada graciosa de `30s`.
- **Healthcheck Integrado:** Monitoramento contínuo da integridade da aplicação através do endpoint `/alive`.
- **Armazenamento de Dados:** Montagem persistente de `/opt/vaultwarden/data` mapeada para `/data` no container.

---

## 8. Estratégia de Backup

O processo de proteção dos dados opera em camadas:
1. **Backup da Aplicação:** O script `/usr/local/sbin/vaultwarden-backup` invoca a rotina atômica embutida no binário (`docker exec vaultwarden /vaultwarden backup`), interrompe o container limparemte, empacota `/opt/vaultwarden/data` em `/var/backups/vaultwarden/*.tar.gz` (excluindo arquivos WAL voláteis e temporários), gera hash SHA-256 e valida a integridade do arquivo. A rotina é automatizada via serviço systemd (`vaultwarden-backup.service`, Type=oneshot) acionado por timer diário (`vaultwarden-backup.timer`, diariamente às 03:00, Persistent=true) com execução noturna real já confirmada (`vaultwarden_20260923_030040.tar.gz`). Inclui política de retenção local de 10 dias (`RETENTION_DAYS=10`), que remove arquivos `.tar.gz` e `.sha256` antigos somente após a criação e validação bem-sucedida do novo backup. A verificação operacional é realizada via logs do systemd e inspeção de arquivos; o monitoramento centralizado via Zabbix é classificado como melhoria futura.
2. **Restauração Testada:** A restauração dos dados foi executada e homologada em ambiente temporário isolado, confirmando a recuperação dos cofres e das credenciais sem afetar a produção.
3. **Backup da VM:** Snapshot de baseline da VM Debian validado via Proxmox Backup Server (PBS).
4. **Backup Off-site em Nuvem (OCI via Restic):** Repositório Restic inicializado com sucesso (ID `7bbbe221`) em bucket privado no Oracle Cloud Infrastructure (`sa-saopaulo-1`, bucket `vaultwarden-offsite`, API compatível com S3) com criptografia client-side. O upload de um backup real (`vaultwarden_20260923_181123.tar.gz`, snapshot `d5f61547`) foi executado manualmente e validado com `restic check`. A recuperação foi homologada em ambiente temporário isolado (`/tmp/restic-vaultwarden-restore`), confirmando 17 itens restaurados, integridade do banco SQLite e chave RSA, e SHA-256 estritamente idêntico ao original, sem substituir ou alterar a produção. A automação diária do upload OCI e a política de retenção remota são melhorias futuras.

Consulte [`docs/BACKUP.md`](file:///home/juniorrufo/projetos/vaultwarden/docs/BACKUP.md) e [`docs/RESTORE.md`](file:///home/juniorrufo/projetos/vaultwarden/docs/RESTORE.md) para detalhes operacionais.

---

## 9. Controles de Segurança

- **Acesso SSH:** Chave pública obrigatória, root desabilitado, senha desabilitada, restrição `AllowUsers junior` e acesso liberado apenas para a LAN.
- **Firewall Base:** Política `DROP` para tráfego não solicitado em IPv4 e IPv6; SSH IPv6 bloqueado.
- **Isolamento Docker:** Filtragem conntrack na cadeia `DOCKER-USER` / `VW-DOCKER` permitindo conexões na porta 8080 apenas a partir do LXC Cloudflared (`192.168.15.253`).
- **Endurecimento da Aplicação:** `SIGNUPS_ALLOWED=false`, `INVITATIONS_ALLOWED=false`, `PASSWORD_HINTS_ALLOWED=false`.
- **Autenticação:** Segundo fator (TOTP) configurado e testado com sucesso na conta administrativa; códigos de recuperação salvos em cofre offline físico.
- **Imagens:** Paginadas exclusivamente por digest SHA-256 imutável.

Consulte [`SECURITY.md`](file:///home/juniorrufo/projetos/vaultwarden/SECURITY.md) para a relação detalhada.

---

## 10. Manutenção e Operação

- **Checagem de Saúde:** `curl http://192.168.15.200:8080/alive` (a partir do host autorizado) e inspeção via `docker inspect`.
- **Atualizações:** Toda atualização de versão segue política formal de verificação prévia de changelogs, execução de backup com validação de hash, atualização do digest SHA-256 no Compose e testes funcionais no Web Vault e extensões.
- **Auditoria de Regras:** Verificação de contadores e regras de firewall via `iptables -S VW-DOCKER`.

Consulte [`docs/OPERATIONS.md`](file:///home/juniorrufo/projetos/vaultwarden/docs/OPERATIONS.md) para o guia completo.

---

## 11. Observações Arquiteturais Importantes

- **Por que o Nginx Proxy Manager (NPM) não é utilizado:** O NPM existe na LAN (`192.168.15.251`), mas foi intencionalmente descartado da rota do Vaultwarden para evitar pontos únicos de falha adicionais e acoplamento entre serviços.
- **Por que SQLite em vez de PostgreSQL:** SQLite elimina o overhead operacional e o consumo de recursos de um SGBD adicional, tornando os backups atômicos e simples para o perfil familiar da infraestrutura.
- **Diferenciação de Disponibilidade:** Container em execução não significa aplicação saudável; endpoint `/alive` saudável não garante que a rota externa da Cloudflare esteja operando; arquivo `.tar.gz` gerado não comprova que os dados são restauráveis sem testes práticos.

---

## 12. Estado Atual

A tabela a seguir consolida os itens já concluídos em produção e os itens planejados para as próximas etapas:

### ✅ Implementado e Validado em Produção
- [x] Provisionamento da VM Debian 13 (trixie) no Proxmox VE com IP estático `192.168.15.200`.
- [x] Hardening completo do SSH (chaves Ed25519, `PermitRootLogin no`, `PasswordAuthentication no`, `AllowUsers junior`).
- [x] Firewall base do host (`iptables-nft` / `netfilter-persistent`) com políticas padrão DROP em IPv4 e IPv6.
- [x] Desativação e mascaramento do `nftables.service` para compatibilidade com o Docker.
- [x] QEMU Guest Agent instalado e em execução no canal virtio.
- [x] Sincronização de relógio ativa via NTP no fuso `America/Sao_Paulo`.
- [x] Instalação do Docker Engine e Compose com `daemon.json` endurecido (rotação de logs local e live-restore).
- [x] Usuário `junior` mantido fora do grupo `docker` (execução com `sudo`).
- [x] Implantação do Vaultwarden v1.37.3 pinado por digest SHA-256 com volume `/opt/vaultwarden/data`.
- [x] Regras de firewall `DOCKER-USER` e `VW-DOCKER` com `conntrack --ctorigdst` restringindo acesso HTTP à origem `192.168.15.253`.
- [x] Persistência do firewall Docker pós-boot e pós-restart via `vaultwarden-docker-firewall.service`.
- [x] Configuração do Cloudflare Tunnel no LXC `192.168.15.253` com rota pública `vault.rufonex.com.br`.
- [x] Criação de conta de usuário, testes de login, logout e sincronização na extensão do navegador Bitwarden.
- [x] Configuração e validação com sucesso do segundo fator de autenticação (TOTP) na conta administrativa e guarda offline dos códigos de recuperação.
- [x] Bloqueio de novos cadastros (`SIGNUPS_ALLOWED=false`) e convites (`INVITATIONS_ALLOWED=false`).
- [x] Backup local diário via systemd (`vaultwarden-backup.service` Type=oneshot e `vaultwarden-backup.timer`, execução noturna observada em 23/09/2026 com `vaultwarden_20260923_030040.tar.gz`).
- [x] Retenção local de 10 dias (`RETENTION_DAYS=10`, expurgo automático de pares `.tar.gz` e `.sha256` pós-backup, validado em teste com par fictício de 15 dias).
- [x] Integridade SHA-256 (validação automatizada de integridade estrutural `tar -tzf` e hash `sha256sum -c`).
- [x] Restore local testado (validação funcional realizada em ambiente temporário isolado sem impacto na produção).
- [x] Snapshot de baseline da VM validado via Proxmox Backup Server (PBS).
- [x] Restic repository OCI (inicializado com sucesso em bucket privado `vaultwarden-offsite`, ID `7bbbe221`).
- [x] Upload real para OCI (backup real `vaultwarden_20260923_181123.tar.gz` enviado com snapshot `d5f61547`, executado manualmente).
- [x] `restic check` (verificações inicial, pós-prune e pós-upload concluídas com `no errors were found`).
- [x] Restore de backup real a partir do OCI (recuperação do snapshot `d5f61547` para `/tmp/restic-vaultwarden-restore` com 17 itens, sem substituir ou alterar a produção).
- [x] Validação do SHA-256 do backup recuperado (`fc40c0ae6da319aa89283632fb0aea58ab4f2ce286e578beb6dc167631b1ce40` idêntico ao `.sha256` armazenado).

### ⚠️ Melhorias Futuras / Evolução (Planejado)
- [ ] **Automação do upload off-site:** Criação de timer e serviço systemd para envio diário automatizado ao repositório OCI.
- [ ] **Política de retenção do repositório Restic:** Automação de expurgo (`restic forget --prune`) de snapshots antigos na nuvem.
- [ ] **Monitoramento centralizado:** Configuração de monitoramento centralizado e alertas do timer de backup e métricas de integridade (melhoria futura; atualmente não existe servidor Zabbix no ambiente).
- [ ] **Teste completo de disaster recovery:** Simulação ponta a ponta de perda total da VM e reconstrução em outro hypervisor.
- [ ] **Verificação periódica de restore:** Formalização e execução de rotinas regulares de testes de recuperação.
- [ ] **Hardening de Credenciais OCI:** Configuração de chaves de API com privilégios mínimos de escrita sem permissão de exclusão pública.
- [ ] **Runbook Formal de Atualização e Testes Periódicos:** Formalização de cronograma de revisões periódicas.

### 🔒 Informações que NUNCA Devem ir para o Git
- Arquivo `/opt/vaultwarden/.env` real contendo segredos de produção.
- Chaves privadas SSH (`id_ed25519`, `id_rsa`) e certificados com chaves privadas (`*.key`, `*.pem`).
- Chave privada de aplicação `/opt/vaultwarden/data/rsa_key.pem`.
- Banco de dados de produção `/opt/vaultwarden/data/db.sqlite3` e journals (`-wal`, `-shm`).
- Arquivos de backup reais gerados (`/var/backups/vaultwarden/*.tar.gz`).
- Tokens de autenticação do Cloudflare Tunnel (`/etc/cloudflared/token`).
- Chaves de API, credenciais ou segredos de bucket do Oracle Cloud (OCI) e credenciais de acesso S3.
- Senha do repositório Restic (`RESTIC_PASSWORD` ou arquivos de chave/senha).
- Códigos de recuperação do Vaultwarden, chaves mestras e segredos TOTP.

---

## 13. Referência e Navegação na Documentação

Para aprofundar em cada área do projeto, consulte a documentação especializada:

- **Fonte de Verdade Exaustiva:** [`PROJECT_CONTEXT.md`](file:///home/juniorrufo/projetos/vaultwarden/PROJECT_CONTEXT.md)
- **Especificação Técnica Detalhada:** [`ARCHITECTURE.md`](file:///home/juniorrufo/projetos/vaultwarden/ARCHITECTURE.md)
- **Registro de Decisões de Engenharia:** [`DECISIONS.md`](file:///home/juniorrufo/projetos/vaultwarden/DECISIONS.md)
- **Matriz de Segurança Efetiva:** [`SECURITY.md`](file:///home/juniorrufo/projetos/vaultwarden/SECURITY.md)
- **Guia Operacional e Comandos:** [`docs/OPERATIONS.md`](file:///home/juniorrufo/projetos/vaultwarden/docs/OPERATIONS.md)
- **Rotinas de Backup:** [`docs/BACKUP.md`](file:///home/juniorrufo/projetos/vaultwarden/docs/BACKUP.md)
- **Procedimento de Restauração:** [`docs/RESTORE.md`](file:///home/juniorrufo/projetos/vaultwarden/docs/RESTORE.md)
- **Integração de Firewall e Docker:** [`docs/DOCKER-FIREWALL.md`](file:///home/juniorrufo/projetos/vaultwarden/docs/DOCKER-FIREWALL.md)
- **Diagramas de Topologia:** [`diagrams/architecture.md`](file:///home/juniorrufo/projetos/vaultwarden/diagrams/architecture.md)
- **Exemplo de Variáveis de Ambiente:** [`.env.example`](file:///home/juniorrufo/projetos/vaultwarden/.env.example)
- **Regras de Exclusão do Repositório:** [`.gitignore`](file:///home/juniorrufo/projetos/vaultwarden/.gitignore)
