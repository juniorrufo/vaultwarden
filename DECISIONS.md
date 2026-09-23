# Registro de Decisões Arquiteturais (ADR) — Vaultwarden

Este documento registra formalmente as principais decisões arquiteturais tomadas durante o planejamento, provisionamento e operação da infraestrutura do Vaultwarden, com base no histórico de engenharia do projeto.

---

## DECISÃO: VM dedicada para Vaultwarden

- **Contexto:** O Vaultwarden gerencia credenciais e senhas críticas. O hypervisor Proxmox VE permite a implantação tanto em containers LXC quanto em Máquinas Virtuais (KVM).
- **Motivo:** O isolamento de segurança e a independência de kernel são vitais para o serviço de senhas. Em um container LXC, o kernel é compartilhado com o host e com outros containers. Uma vulnerabilidade de escape de container comprometeria todo o ambiente.
- **Escolha:** Implantar o Vaultwarden em uma Máquina Virtual (VM) KVM dedicada, isolando seus privilégios de kernel, memória e subsistema de rede.
- **Alternativas consideradas:**
  - Container LXC compartilhado (descartado por menor isolamento de segurança).
  - VM compartilhada com múltiplos outros serviços da rede (descartado para evitar que falhas de outros serviços afetem o cofre).
- **Consequências:** Leve overhead de virtualização de recursos (2 vCPU, ~2 GB RAM, ~32 GB disco), mas garantia de isolamento total de privilégios e capacidade de restauração independente de snapshot via Proxmox/PBS.

---

## DECISÃO: Debian como sistema operacional da VM

- **Contexto:** Necessidade de selecionar uma distribuição Linux para atuar como sistema base da VM do Vaultwarden.
- **Motivo:** O Debian oferece confiabilidade excepcional, suporte estável e previsível de longo prazo, ausência de telemetria indesejada e suporte de primeira classe ao repositório oficial da Docker Engine.
- **Escolha:** Debian GNU/Linux 13 (trixie) amd64, com kernel estável.
- **Alternativas consideradas:**
  - Ubuntu Server (descartado por maior footprint de pacotes desnecessários e serviços extras em segundo plano).
  - Alpine Linux (descartado devido a eventuais diferenças de biblioteca musl libc em scripts administrativos e compatibilidade com certas ferramentas de backup).
- **Consequências:** Ambiente enxuto, alta estabilidade do kernel, integração facilitada com `qemu-guest-agent` e comandos padronizados de administração.

---

## DECISÃO: Docker como gerenciador de containers

- **Contexto:** O Vaultwarden é um binário escrito em Rust, disponibilizado primariamente como imagem de container OCI.
- **Motivo:** O uso de containers isola as dependências da aplicação em relação ao sistema operacional base, permite atualizações reprodutíveis e facilita a definição da infraestrutura em arquivos versionáveis (`docker-compose.yaml`).
- **Escolha:** Utilizar Docker Engine oficial (v29.8.1) gerenciado via Docker Compose (v5.5.1), com parâmetros de segurança explícitos (`no-new-privileges`, limites de log no `daemon.json`).
- **Alternativas consideradas:**
  - Execução como serviço direto do systemd em bare-metal (descartado pela complexidade de gerenciar compilações e bibliotecas no SO).
  - Kubernetes / K3s (descartado pelo excesso desnecessário de complexidade e consumo de recursos para uma instância pessoal).
- **Consequências:** Facilidade de gestão e portabilidade, exigindo atenção à integração com as regras de firewall do kernel (`DOCKER-USER`).

---

## DECISÃO: SQLite como banco de dados

- **Contexto:** O Vaultwarden oferece suporte a motores de banco de dados SQLite, PostgreSQL e MySQL/MariaDB.
- **Motivo:** O deployment atende a uma demanda pessoal/familiar. Introduzir um SGBD externo como PostgreSQL adicionaria consumo substancial de memória, complexidade operacional de gerenciamento de containers adicionais, manutenção de conexões de rede e rotinas complexas de dump/restore, sem gerar nenhum ganho perceptível de performance nesta volumetria.
- **Escolha:** Utilizar o banco embutido SQLite armazenado em `/opt/vaultwarden/data/db.sqlite3`.
- **Alternativas consideradas:**
  - PostgreSQL em container secundário (descartado por excesso de complexidade e consumo de recursos).
  - MariaDB / MySQL (descartado pelas mesmas razões operacionais).
- **Consequências:** Todo o estado da base reside em um único arquivo consistente; simplifica drasticamente a estratégia de backup e disaster recovery sem dependências de rede interna.

---

## DECISÃO: Cloudflare Tunnel como ponto de entrada público

- **Contexto:** A aplicação necessita de terminação HTTPS válida publicamente para viabilizar a comunicação com aplicativos móveis e extensões de navegadores fora da rede local.
- **Motivo:** Expor diretamente o IP residencial com redirecionamento de portas (port forwarding) no roteador vulnerabiliza o IP público residencial a varreduras automatizadas e ataques direcionados.
- **Escolha:** Utilizar o Cloudflare Tunnel operando em um container LXC dedicado (`192.168.15.253`), conectando-se à borda da Cloudflare via conexão outbound segura, sem abrir portas de entrada no roteador.
- **Alternativas consideradas:**
  - Redirecionamento de portas no roteador (TCP 80/443) associado a Dynamic DNS (descartado por expor o IP público e o roteador).
  - VPN exclusiva (WireGuard / Tailscale) sem acesso público (descartado por inviabilizar o uso amigável por familiares e em navegadores em que não se possa instalar cliente de VPN).
- **Consequências:** Nenhuma porta externa aberta no roteador; proteção contra ataques DDoS e terminação TLS automática na borda; dependência da disponibilidade da rede da Cloudflare.

---

## DECISÃO: Não utilizar Nginx Proxy Manager (NPM) para o Vaultwarden

- **Contexto:** A rede local já possui uma instância de Nginx Proxy Manager em execução no IP `192.168.15.251` para outros serviços residenciais.
- **Motivo:** Inserir o NPM na cadeia de requisições do Vaultwarden criaria um intermediário redundante, pois a Cloudflare já efetua a terminação TLS e o Cloudflare Tunnel pode se conectar diretamente ao IP e porta da VM do Vaultwarden. Além disso, tornaria o cofre de senhas dependente da disponibilidade de um proxy compartilhado com outros serviços.
- **Escolha:** Conectar o túnel do Cloudflared diretamente ao IP interno da VM (`192.168.15.200:8080`), sem passar pelo NPM.
- **Alternativas consideradas:**
  - Rota `Cloudflare -> Cloudflared LXC -> NPM -> Vaultwarden` (descartada por adicionar camadas e pontos de falha supérfluos).
- **Consequências:** Arquitetura mais enxuta, menor latência, menor superfície de ataque e desacoplamento completo do serviço de senhas em relação aos demais serviços da rede.

---

## DECISÃO: iptables-nft em vez de regras nftables gerenciadas diretamente

- **Contexto:** O Debian 13 utiliza `nftables` como backend moderno de filtragem no kernel, mas o Docker Engine ainda gera regras dinamicamente através do conjunto de ferramentas `iptables`.
- **Motivo:** Se o administrador configurasse regras diretamente em `/etc/nftables.conf`, haveria conflitos de precedência entre as chains do `nftables.service` e as tabelas injetadas pelo Docker, resultando em comportamentos indeterminados.
- **Escolha:** Operar com a camada de compatibilidade `iptables-nft` e `ip6tables-nft`, mascarando o serviço nativo `nftables.service` do Debian e salvando as regras via `netfilter-persistent`.
- **Alternativas consideradas:**
  - Gerenciamento direto via `nftables` (descartado por incompatibilidade com as rotinas internas do Docker).
- **Consequências:** Coexistência harmônica e previsível entre as regras de segurança do host e as regras de encaminhamento do daemon Docker.

---

## DECISÃO: Firewall DOCKER-USER para isolamento de portas publicadas

- **Contexto:** Por padrão, portas mapeadas no Docker utilizam DNAT na tabela `nat`, fazendo com que os pacotes de entrada pulem a cadeia `INPUT` do firewall do host e cheguem desprotegidos ao container a partir de qualquer endereço da LAN.
- **Motivo:** A porta 8080 do Vaultwarden deve ser acessada unicamente pelo LXC do Cloudflared (`192.168.15.253`), impedindo acessos diretos de outros computadores ou dispositivos comprometidos na rede local.
- **Escolha:** Criar regras na cadeia padrão `DOCKER-USER` apontando para a cadeia `VW-DOCKER`, utilizando correspondência conntrack (`--ctorigdst 192.168.15.200 --ctorigdstport 8080`) para aceitar apenas pacotes originados de `192.168.15.253/32` e descartar (`DROP`) qualquer outra origem.
- **Alternativas consideradas:**
  - Binding apenas em `127.0.0.1` (descartado porque o Cloudflared reside em outro host da rede e precisa alcançar a VM via LAN).
  - Deixar a porta aberta para toda a LAN (descartado por violar o princípio de privilégio mínimo e segurança em profundidade).
- **Consequências:** Isolamento de rede estrito mantido ativamente mesmo após reboots e reinicializações do Docker através de serviço systemd dedicado.

---

## DECISÃO: TOTP como segundo fator de autenticação em vez de WebAuthn

- **Contexto:** Necessidade de reforçar a autenticação da conta administrativa além da senha mestre.
- **Motivo:** O padrão TOTP (*Time-based One-Time Password*) é universalmente suportado em dispositivos móveis e desktops, permitindo implementação simples, exportação de códigos de recuperação e confiabilidade no dia a dia familiar, sem exigir chaves físicas FIDO2 para todos os acessos.
- **Escolha:** Configurar e testar com sucesso o TOTP na conta administrativa como segundo fator de autenticação, armazenando os códigos de recuperação em cofre físico offline, sem definir o TOTP como política global obrigatória para todos os usuários da instância.
- **Alternativas consideradas:**
  - WebAuthn / Chaves de segurança de hardware FIDO2 (descartado para a fase atual para manter compatibilidade ampla e simplicidade de contingência sem exigir chaves FIDO2 físicas).
  - Sem segundo fator na conta administrativa (descartado por representar risco de segurança).
- **Consequências:** Mecanismo seguro, estável e testado com sucesso no login da conta administrativa.

---

## DECISÃO: Desativação de cadastros públicos (SIGNUPS_ALLOWED=false)

- **Contexto:** O endpoint do Vaultwarden está exposto publicamente na Internet via Cloudflare Tunnel para permitir sincronização de clientes remotos.
- **Motivo:** Se o registro de usuários permanecesse ativado, qualquer visitante na Internet poderia criar um cofre na instância, consumindo armazenamento e utilizando o serviço indevidamente.
- **Escolha:** Provisionar o usuário administrador com o cadastro temporariamente aberto e, imediatamente após o primeiro login bem-sucedido, fixar `SIGNUPS_ALLOWED=false` e `INVITATIONS_ALLOWED=false` no arquivo `.env`.
- **Alternativas consideradas:**
  - Permitir cadastros com confirmação manual (desnecessário para um cofre familiar).
  - Restringir por regex de e-mail (inseguro contra usuários maliciosos que possuam contas em domínios autorizados).
- **Consequências:** Fechamento completo da superfície de criação de contas; apenas os usuários autorizados permanecem no sistema.

---

## DECISÃO: Estratégia de backup local com verificação de integridade

- **Contexto:** Garantir a salvaguarda dos dados criptografados sem provocar inconsistências no banco durante a gravação.
- **Motivo:** Copiar o banco SQLite a quente com o container em execução pode capturar transações incompletas no arquivo `-wal`. Além disso, criar backups sem verificar estruturalmente o arquivo e sem testar a restauração em laboratório cria uma ilusão de segurança perigosa.
- **Escolha:** Criar o script `/usr/local/sbin/vaultwarden-backup` que executa o comando atômico de backup embutido no Vaultwarden, para o container de forma limpa, empacota `/opt/vaultwarden/data` em `.tar.gz`, gera e valida o checksum SHA-256, confirma a estrutura com `tar -tzf`, aplica a política de retenção local de 10 dias (`RETENTION_DAYS=10`) para expurgo de arquivos antigos somente após validação bem-sucedida, automatizado via systemd (`vaultwarden-backup.timer` e `vaultwarden-backup.service`); somado à execução de um teste prático de restauração em ambiente temporário isolado.
- **Alternativas consideradas:**
  - Script simples de `tar` sobre a pasta `/data` ativa sem parar a aplicação (descartado pelo risco de inconsistência no SQLite).
  - Apenas snapshot de VM no Proxmox (descartado porque o backup em nível de aplicação provê granularidade para restaurar apenas o banco sem precisar reinstalar a VM inteira).
  - Retenção sem validação prévia do novo arquivo (descartado pelo risco de expurgar backups antigos válidos caso o processo atual falhe).
- **Consequências:** Processo de backup seguro, íntegro, com retenção local automatizada de 10 dias, agendado via systemd e com restauração já validada em ambiente de testes.

---

## DECISÃO: Imagem Docker pinada por digest SHA-256

- **Contexto:** Definição da referência da imagem do container no arquivo de orquestração `docker-compose.yaml`.
- **Motivo:** Utilizar tags mutáveis como `latest` ou mesmo tags de versão sem digest como `1.37.3` expõe o ambiente ao risco de pulls não intencionais que tragam versões com bugs ou comprometidas no registro público, além de impedir o rollback determinístico.
- **Escolha:** Fixar a imagem Docker com seu digest imutável:
  `vaultwarden/server@sha256:4ecafc9049c7d878c7717d1ce4f9059d706758c78b8fa42e5ead21f4b2dfc770`.
- **Alternativas consideradas:**
  - Utilizar a tag `latest` com atualizadores automáticos como Watchtower (descartado expressamente para evitar quebras silenciosas em produção).
  - Utilizar apenas a tag de versão `1.37.3` (descartado porque tags de imagens podem ser republicadas no Docker Hub).
- **Consequências:** Garantia absoluta de imutabilidade e reprodutibilidade do container; toda atualização passa por processo manual e documentado de homologação.

---

## DECISÃO: Backup off-site em nuvem via Restic e OCI Object Storage

- **Contexto:** Necessidade de proteção contra cenários de desastre catastrófico físico (perda total da residência, queima física do host Proxmox ou corrupção generalizada do armazenamento local).
- **Motivo:** Backups locais em `/var/backups/vaultwarden/` e snapshots no PBS residem sob o mesmo teto físico e compartilham riscos locais. A salvaguarda em nuvem precisa garantir total privacidade dos dados através de criptografia client-side ponta a ponta e independência de fornecedor, sem risco de expor credenciais em repositórios de código.
- **Escolha:** Utilizar o **Restic** com backend S3 para o bucket privado `vaultwarden-offsite` no **Oracle Cloud Infrastructure (OCI Object Storage)** na região `sa-saopaulo-1` (compartment `Backups`, ID do repositório `7bbbe221`). O Restic encripta os dados localmente antes do envio, mantendo as chaves privadas e senha de acesso sob controle exclusivo do administrador fora do Git. O repositório foi inicializado, verificado, o backup real `vaultwarden_20260923_181123.tar.gz` (snapshot `d5f61547`) foi enviado manualmente e a recuperação foi homologada com sucesso em ambiente isolado (`/tmp/restic-vaultwarden-restore`), confirmando 17 arquivos restaurados e integridade SHA-256 perfeita.
- **Alternativas consideradas:**
  - Cópia remota simples via scp/rsync para VPS ou servidor de terceiros (descartado por manter complexidade de gestão de host adicional).
  - AWS S3 / Google Cloud Storage (descartado em favor da infraestrutura OCI com bucket no compartment `Backups` e compatibilidade com API S3).
  - Upload direto de arquivos `.tar.gz` sem ferramenta especializada de snapshots (descartado porque o Restic provê criptografia nativa no cliente, deduplicação em blocos e integridade verificável via `restic check`).
- **Consequências:** Capacidade real e homologada de recuperação de desastres fora do ambiente local; garantia de privacidade por criptografia client-side; automação do envio diário via systemd e retenção do repositório remoto permanecem registradas como evoluções futuras.
