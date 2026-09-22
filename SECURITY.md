# Política e Controles de Segurança — Vaultwarden

Este documento registra exclusivamente as medidas e controles de segurança efetivamente implementados na infraestrutura do Vaultwarden. Nenhuma medida não testada ou fictícia foi incluída.

---

## 1. Segurança e Hardening de Acesso SSH

O acesso administrativo ao sistema operacional Debian 13 (`192.168.15.200`) é rigorosamente restrito através das seguintes diretivas ativas no `sshd_config`:

- **Autenticação Exclusiva por Chave Pública:**
  `PubkeyAuthentication yes`.
  Apenas chaves baseadas em algoritmos modernos (Ed25519) foram autorizadas para o usuário administrativo.
- **Bloqueio Total de Login como Root:**
  `PermitRootLogin no`.
  O acesso direto com o usuário `root` é proibido por SSH.
- **Autenticação por Senha Desabilitada:**
  `PasswordAuthentication no` e `KbdInteractiveAuthentication no`.
  Não é possível realizar autenticação interativa com senha, eliminando riscos de força bruta de credenciais.
- **Restrição de Usuários Autorizados (AllowUsers):**
  `AllowUsers junior`.
  Apenas a conta de usuário `junior` possui permissão para negociar sessões SSH com o daemon.
- **Controle de Tentativas:**
  `MaxAuthTries 3` e `UsePAM yes`.
  Sessões que falhem na negociação de chaves em 3 tentativas são sumariamente encerradas.
- **Armazenamento de Chaves Privadas:**
  As chaves privadas SSH residem unicamente nas estações de administração (WSL e PowerShell) e possuem cópia de contingência offline segura. Nenhuma chave privada é mantida no repositório Git ou exposta na rede.

---

## 2. Segurança de Rede e Firewall de Borda

### 2.1. Ausência de Exposição Direta à Internet
- Não há redirecionamento de portas (*port forwarding*) configurado no roteador ou firewall de borda da rede local (`192.168.15.1`).
- O IP público da conexão residencial não escuta requisições de entrada para o serviço do Vaultwarden.

### 2.2. Acesso Externo via Cloudflare Tunnel
- A entrada pública é intermediada pela infraestrutura da Cloudflare (`vault.rufonex.com.br`).
- O tráfego trafega encapsulado por um túnel outbound seguro estabelecido pelo container LXC `192.168.15.253` com a rede Anycast da Cloudflare.
- A Cloudflare atua como terminador TLS, provendo mitigação de DDoS e regras de inspeção WAF na borda.

### 2.3. Firewall do Host (IPv4)
- **Política Padrão:** `INPUT DROP`, `FORWARD DROP`, `OUTPUT ACCEPT`.
- **Regra de Gerenciamento:** Conexões SSH (porta 22) são autorizadas estritamente se originadas da rede local `192.168.15.0/24`.
- **Loopback e Conexões Estabelecidas:** Tráfego de loopback (`lo`) e estados `ESTABLISHED,RELATED` são explicitamente liberados.

### 2.4. Firewall do Host (IPv6)
- **Política Padrão:** `INPUT DROP`, `FORWARD DROP`, `OUTPUT ACCEPT`.
- **SSH IPv6 Bloqueado:** Acesso SSH através de endereços IPv6 é terminantemente proibido e descartado pelo firewall.
- **Tráfego Permitido:** Apenas mensagens indispensáveis do protocolo ICMPv6 (como Neighbor Discovery) e estados `ESTABLISHED,RELATED` são aceitos.

---

## 3. Isolamento de Rede Docker via DOCKER-USER

O Docker insere regras de DNAT que contornam a tabela `INPUT` do host. Para fechar essa brecha de segurança, foi criada uma política na cadeia `DOCKER-USER` delegando o tráfego à cadeia dedicada `VW-DOCKER`:

- **Restrição de Origem por Conntrack:**
  Utiliza `--ctorigdst 192.168.15.200 --ctorigdstport 8080` para inspecionar o destino real pré-DNAT.
- **Liberação Exclusiva para o LXC Cloudflared:**
  Apenas pacotes provenientes de `192.168.15.253/32` com destino original à porta 8080 são aceitos (`ACCEPT`).
- **Bloqueio de Demais Hosts da LAN:**
  Qualquer outro host da rede interna que tente conectar diretamente em `192.168.15.200:8080` sofre descarte imediato (`DROP`).
- **Binding Local no Docker Compose:**
  A porta do container não é vinculada a `0.0.0.0`, mas explicitamente ao IP da interface da VM: `192.168.15.200:8080:80`.

---

## 4. Segurança do Container e Privilégios do Sistema

- **Usuário Fora do Grupo Docker:**
  O usuário `junior` **não** pertence ao grupo Unix `docker`. Comandos de gerenciamento de containers exigem elevação explícita com `sudo`, prevenindo escalonamento de privilégios acidental no host.
- **Prevenção de Escalação de Privilégios no Container:**
  O container do Vaultwarden é executado com a diretiva de segurança:
  ```yaml
  security_opt:
    - no-new-privileges:true
  ```
  Isso impede que processos internos do container ganhem novos privilégios via binários `setuid` ou `setgid`.
- **Imagens Paginadas por Digest Imutável:**
  A imagem Docker é referenciada estritamente pelo seu digest criptográfico SHA-256:
  ```text
  vaultwarden/server@sha256:4ecafc9049c7d878c7717d1ce4f9059d706758c78b8fa42e5ead21f4b2dfc770
  ```
  Isso protege a infraestrutura contra ataques de envenenamento de tags ou alterações não homologadas da tag `latest`.
- **Permissões dos Arquivos Sensíveis no Host:**
  Os diretórios `/opt/vaultwarden/data` e o arquivo de variáveis `/opt/vaultwarden/.env` possuem permissões restritas no sistema de arquivos local, acessíveis unicamente pelo superusuário (`root`).

---

## 5. Políticas de Aplicação no Vaultwarden

A instância foi endurecida no nível da aplicação para mitigar riscos de abuso:

- **Novos Cadastros Bloqueados (SIGNUPS_ALLOWED=false):**
  O cadastro de novos usuários foi desativado imediatamente após o provisionamento da conta do administrador. Usuários não autorizados não conseguem criar contas na instância pública.
- **Convites Desabilitados (INVITATIONS_ALLOWED=false):**
  O envio e aceitação de convites para ingresso no cofre está desativado.
- **Dicas de Senha Desativadas (PASSWORD_HINTS_ALLOWED=false):**
  Desabilita o envio ou exibição de dicas de senha mestre, evitando enumeração e inferência de senhas por agentes externos.
- **Autenticação em Dois Fatores (TOTP) na Conta Administrativa:**
  O segundo fator via TOTP (*Time-based One-Time Password*) foi configurado e testado com sucesso na conta administrativa. O TOTP não está configurado como política global obrigatória para todos os usuários.
- **Armazenamento Seguro de Códigos de Recuperação:**
  Os códigos de emergência do TOTP e as chaves de recuperação do Vaultwarden não estão salvos na aplicação, no servidor de produção ou no repositório Git, residindo em armazenamento offline físico e seguro.

---

## 6. Proteção e Integridade dos Backups

- **Local Restrito:** Os arquivos de backup gerados pelo script `/usr/local/sbin/vaultwarden-backup` são armazenados em `/var/backups/vaultwarden/`, com permissões restritas a `root`.
- **Validação de Hashes Criptográficos:** Cada backup gera um arquivo de hash correspondente (`.sha256`), o qual é verificado automaticamente no processo de geração e em auditorias manuais (`sha256sum -c`).
- **Segurança no Git:** Backups (`*.tar.gz`), arquivos de banco (`*.sqlite3`, `-wal`, `-shm`) e chaves privadas (`rsa_key.pem`) são expressamente ignorados pelo `.gitignore` e nunca devem ser versionados.
