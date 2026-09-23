# Infraestrutura Vaultwarden — Contexto do Projeto

## 1. Finalidade

Este repositório documenta e armazena a configuração reproduzível para uma
implantação auto-hospedada (self-hosted) do Vaultwarden em execução em uma
máquina virtual dedicada dentro de uma rede doméstica privada.

Os objetivos principais são:

- Gerenciamento seguro de senhas auto-hospedado.
- Superfície de ataque mínima.
- Infraestrutura reproduzível.
- Procedimentos operacionais documentados.
- Backups locais e off-site confiáveis.
- Restauração de desastres (disaster recovery) testada.
- Configuração explícita em vez de alterações manuais/ocultas.
- Capacidade de reconstruir o ambiente a partir da documentação + configuração
  versionada + backups externos.

Esta é uma implantação de gerenciador de senhas pessoal/familiar, e não um
serviço multi-inquilino (multi-tenant) de grande escala.

---

# 2. Arquitetura Atual

## Topologia de alto nível

```text
                            INTERNET
                                |
                                v
                       +----------------+
                       |   Cloudflare   |
                       | DNS / TLS      |
                       +-------+--------+
                               |
                                v
                     Cloudflare Tunnel
                               |
                                v
                     192.168.15.253
                      Cloudflared LXC
                               |
                          HTTP :8080
                               |
                                v
                     192.168.15.200
                      Vaultwarden VM
                               |
                         Docker Engine
                               |
                                v
                     Vaultwarden :80
                               |
                                v
                            SQLite
```

## Virtualização

Hypervisor:

- Proxmox VE

Guest:

- VM dedicada para o Vaultwarden

A aplicação Vaultwarden é intencionalmente isolada em sua própria VM.

---

# 3. Informações da VM

Hostname:

```text
vaultwarden
```

Sistema Operacional:

```text
Debian GNU/Linux 13 (trixie)
```

Arquitetura:

```text
x86-64 / amd64
```

Virtualização:

```text
KVM
```

Kernel atual no deployment:

```text
Linux 6.12.107+deb13-amd64
```

CPU:

```text
2 vCPU
```

Memória:

```text
~2 GB
```

Disco:

```text
~32 GB
```

Interface de rede principal:

```text
ens18
```

IPv4:

```text
192.168.15.200/24
```

Gateway padrão:

```text
192.168.15.1
```

IPv6 está habilitado e funcional.

A VM possui atualmente um endereço IPv6 globalmente roteável atribuído pela
rede. O acesso SSH via IPv6 está intencionalmente bloqueado pelo firewall do host.

---

# 4. Design de Rede

LAN:

```text
192.168.15.0/24
```

Hosts importantes:

```text
Vaultwarden VM:
192.168.15.200

Cloudflare Tunnel LXC:
192.168.15.253

Nginx Proxy Manager:
192.168.15.251
```

O Nginx Proxy Manager existe em outro local da rede, mas NÃO é utilizado pelo
Vaultwarden.

## Por que o NPM não é utilizado

O deployment do Vaultwarden utiliza:

```text
Cloudflare
    |
Cloudflare Tunnel
    |
Vaultwarden
```

em vez de:

```text
Cloudflare
    |
Cloudflare Tunnel
    |
NPM
    |
Vaultwarden
```

Motivos:

- Evitar uma dependência adicional e desnecessária de proxy reverso.
- Reduzir a complexidade operacional.
- Reduzir a superfície de ataque.
- Evitar tornar o Vaultwarden dependente da disponibilidade do NPM.
- A Cloudflare já provê o endpoint HTTPS público.
- O Cloudflare Tunnel pode encaminhar requisições diretamente para a VM do Vaultwarden.

O NPM permanece disponível para outros serviços na rede.

---

# 5. URL Pública

URL de produção:

```text
https://vault.rufonex.com.br
```

O endpoint HTTPS público é gerenciado pela Cloudflare.

Origem:

```text
http://192.168.15.200:8080
```

O Cloudflare Tunnel conecta-se ao endpoint HTTP interno.

Intencionalmente, não há redirecionamento direto de portas da Internet para a
VM do Vaultwarden.

---

# 6. Cloudflare Tunnel

O Cloudflare Tunnel roda em um LXC dedicado.

Host do Cloudflared:

```text
192.168.15.253
```

O túnel é gerenciado remotamente pela Cloudflare.

Modelo atual de configuração local:

```text
/etc/cloudflared/token
```

Nenhum `cert.pem` local é necessário para a configuração em tempo de execução.

Atualmente não existe:

```text
/etc/cloudflared/config.yml
```

O gerenciamento do túnel é realizado a partir do Dashboard da Cloudflare.

IMPORTANTE:

Nunca comite o token do túnel ou qualquer segredo da Cloudflare no Git.

A rota pública configurada na Cloudflare é:

```text
vault.rufonex.com.br
    ->
http://192.168.15.200:8080
```

---

# 7. Arquitetura de Firewall

O host NÃO utiliza um conjunto de regras nftables gerenciado diretamente.

Motivo:

O Docker integra-se com o iptables e espera que o firewall do host coopere
com as cadeias iptables do Docker.

Implementação atual do firewall:

```text
iptables-nft
ip6tables-nft
```

Versões no deployment:

```text
iptables v1.8.11 (nf_tables)
ip6tables v1.8.11 (nf_tables)
```

O serviço Debian `nftables.service` foi intencionalmente desabilitado e mascarado.

Estado atual:

```text
nftables.service
    masked
    inactive
```

Firewall persistente:

```text
netfilter-persistent
```

está habilitado.

---

# 8. Política de Firewall Base do Host

IPv4:

```text
INPUT   DROP
FORWARD DROP
OUTPUT  ACCEPT
```

IPv6:

```text
INPUT   DROP
FORWARD DROP
OUTPUT  ACCEPT
```

IPv4 SSH:

```text
192.168.15.0/24 -> TCP/22 -> ALLOW
```

SSH a partir de IPv6 é intencionalmente não permitido.

Tráfego básico permitido inclui:

- loopback
- conexões estabelecidas/relacionadas (established/related)
- ICMP IPv4
- funcionalidade ICMPv6 necessária
- SSH a partir da LAN de gerenciamento

A configuração do firewall sobrevive a reinicializações da VM.

Um teste de reboot foi realizado com sucesso.

---

# 9. Política de Firewall do Docker

O Docker cria suas cadeias padrão, incluindo:

```text
DOCKER
DOCKER-BRIDGE
DOCKER-CT
DOCKER-FORWARD
DOCKER-INTERNAL
DOCKER-USER
```

A política administrativa de firewall é implementada através de:

```text
DOCKER-USER
    |
    v
VW-DOCKER
```

A cadeia dedicada é:

```text
VW-DOCKER
```

Política atual:

```text
ESTABLISHED,RELATED
    -> ACCEPT

192.168.15.253
    + destino original 192.168.15.200:8080
    -> ACCEPT

outras origens
    + destino original 192.168.15.200:8080
    -> DROP

todo o resto
    -> RETURN
```

A filtragem utiliza correspondência de destino original via conntrack porque o Docker executa
DNAT antes que o tráfego alcance o ponto de filtragem administrativo.

O conjunto efetivo de regras é:

```text
-A DOCKER-USER -j VW-DOCKER

-A VW-DOCKER -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

-A VW-DOCKER -s 192.168.15.253/32 \
    -p tcp \
    -m conntrack \
    --ctorigdst 192.168.15.200 \
    --ctorigdstport 8080 \
    -j ACCEPT

-A VW-DOCKER \
    -p tcp \
    -m conntrack \
    --ctorigdst 192.168.15.200 \
    --ctorigdstport 8080 \
    -j DROP

-A VW-DOCKER -j RETURN
```

Isso significa:

```text
Cloudflared LXC
192.168.15.253
        |
        | TCP/8080
        v
Vaultwarden
192.168.15.200
        |
        v
ALLOW
```

enquanto outros hosts não conseguem acessar diretamente a porta publicada do Vaultwarden.

Isso foi testado com sucesso.

Teste a partir do Cloudflared:

```text
curl http://192.168.15.200:8080/alive
```

Resultado:

```text
HTTP 200
```

Teste a partir de outra origem na LAN:

```text
curl http://192.168.15.200:8080/alive
```

Resultado:

```text
connection timeout
```

A política de firewall é implementada por:

```text
/usr/local/sbin/vaultwarden-docker-firewall
```

e:

```text
/etc/systemd/system/vaultwarden-docker-firewall.service
```

O serviço systemd está habilitado e reaplica a política específica do Docker após
o Docker ficar disponível.

A política foi testada após:

- reinicialização do Docker
- reboot completo da VM

e permaneceu funcional.

---

# 10. Hardening de SSH

Usuário administrativo:

```text
junior
```

A autenticação SSH é exclusivamente por chave pública.

Configuração SSH efetiva atual:

```text
UsePAM yes
MaxAuthTries 3
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
AllowUsers junior
```

A configuração SSH foi validada com:

```bash
sudo sshd -t
```

e a configuração efetiva foi inspecionada com:

```bash
sudo sshd -T
```

Duas chaves públicas estão autorizadas para a conta `junior`:

```text
chave WSL
chave Windows PowerShell
```

Chaves privadas NÃO DEVEM ser armazenadas no Git.

As chaves privadas existentes possuem backup separado em um local seguro e offline.

---

# 11. Acesso SSH

O gerenciamento ocorre normalmente a partir de:

```text
WSL
```

e:

```text
Windows PowerShell
```

Exemplo:

```bash
ssh -o IdentitiesOnly=yes \
    -i ~/.ssh/id_ed25519 \
    junior@192.168.15.200
```

O equivalente em PowerShell utiliza a chave privada do Windows.

---

# 12. Proxmox QEMU Guest Agent

O QEMU Guest Agent está instalado e operacional.

Canal de comunicação do guest:

```text
/dev/virtio-ports/org.qemu.guest_agent.0
```

O serviço é:

```text
qemu-guest-agent.service
```

e está intencionalmente instalado como um serviço systemd estático de acordo com o
modelo de empacotamento do Debian.

Status verificado:

```text
active (running)
```

A VM no Proxmox possui o QEMU Guest Agent habilitado.

---

# 13. Sincronização de Horário (Time Synchronization)

Fuso horário:

```text
America/Sao_Paulo
```

NTP:

```text
enabled
```

Relógio do sistema:

```text
synchronized
```

A VM utiliza sincronização de horário via rede.

O horário correto é essencial para:

- TOTP
- TLS
- logs
- validação de tokens
- backups agendados

---

# 14. Instalação do Docker

O Docker foi instalado a partir do repositório oficial do Docker para Debian.

Componentes atuais no deployment:

```text
Docker Engine: 29.8.1
Docker Compose: 5.5.1
containerd: 2.3.5
runc: 1.5.1
Buildx: 0.37.1
```

O Docker utiliza:

```text
Storage Driver:
overlayfs

Cgroup Driver:
systemd

Cgroup Version:
2

Firewall Backend:
iptables
```

Diretório raiz do Docker:

```text
/var/lib/docker
```

Serviço do Docker:

```text
enabled
active
```

---

# 15. Configuração do Daemon do Docker

Arquivo:

```text
/etc/docker/daemon.json
```

Configuração atual:

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

Finalidade:

- utilizar logging local controlado no Docker
- prevenir crescimento ilimitado dos logs dos containers
- reter um número limitado de arquivos de log
- manter containers suportados em execução durante certos reinícios do daemon do Docker

A configuração do Docker foi validada utilizando:

```bash
dockerd --validate --config-file=/etc/docker/daemon.json
```

---

# 16. Modelo de Privilégios do Docker

O usuário Linux administrativo `junior` NÃO é membro do grupo `docker`.

Comandos do Docker são intencionalmente executados com:

```bash
sudo docker ...
```

Motivo:

Ser membro do grupo Docker efetivamente concede altos privilégios sobre o
daemon do Docker e, portanto, sobre o host.

Não adicione o usuário ao grupo Docker sem um motivo arquitetural específico.

---

# 17. Container do Vaultwarden

Aplicação:

```text
Vaultwarden
```

Versão atual do servidor:

```text
1.37.3
```

Container:

```text
vaultwarden
```

A imagem é fixada por digest imutável.

Digest atual da imagem:

```text
sha256:4ecafc9049c7d878c7717d1ce4f9059d706758c78b8fa42e5ead21f4b2dfc770
```

Plataforma:

```text
linux/amd64
```

Não substitua a referência da imagem por:

```text
latest
```

sem um processo de atualização planejado.

Atualizações de imagem devem ser intencionais e documentadas.

---

# 18. Docker Compose do Vaultwarden

Arquivo Compose atual:

```text
/opt/vaultwarden/docker-compose.yaml
```

O deployment utiliza intencionalmente um único container de aplicação.

NÃO há:

- PostgreSQL
- Redis
- Nginx
- Nginx Proxy Manager
- Portainer
- Watchtower
- container separado de Cloudflared

A arquitetura intencionalmente evita componentes desnecessários.

---

# 19. Rede Docker do Vaultwarden

O Compose cria:

```text
vaultwarden_net
```

Tipo de rede:

```text
bridge
```

A rede é declarada no Compose em vez de criada manualmente.

Design atual:

```text
vaultwarden_net
    |
    +-- vaultwarden
```

Isso mantém o deployment via Compose reproduzível.

---

# 20. Publicação de Portas do Vaultwarden

Binding de produção atual:

```text
192.168.15.200:8080 -> container:80
```

O serviço NÃO é publicado em:

```text
0.0.0.0:8080
```

A política de firewall do Docker restringe o acesso a essa porta publicada de modo que apenas
o LXC do Cloudflare Tunnel consiga alcançá-la.

---

# 21. Configurações de Segurança do Container Vaultwarden

A configuração atual do container inclui:

```yaml
restart: unless-stopped
```

e:

```yaml
security_opt:
  - no-new-privileges:true
```

O container possui:

```yaml
stop_grace_period: 30s
```

O healthcheck fornecido pela imagem é utilizado em vez de um healthcheck personalizado.

O estado atual de saúde foi verificado como:

```text
healthy
```

O endpoint de saúde da aplicação é:

```text
/alive
```

Exemplo:

```bash
curl http://192.168.15.200:8080/alive
```

Resposta esperada de sucesso:

```text
HTTP 200
```

---

# 22. Dados Persistentes do Vaultwarden

Diretório no host:

```text
/opt/vaultwarden/data
```

Montagem no container:

```text
/opt/vaultwarden/data:/data
```

O diretório de dados persistentes contém atualmente itens como:

```text
db.sqlite3
rsa_key.pem
icon_cache/
tmp/
```

Diretórios/arquivos adicionais podem surgir conforme funcionalidades do Vaultwarden forem utilizadas.

O diretório `/data` é crítico para a recuperação de desastres.

Não o exclua nem o recrie sem compreender o impacto na recuperação.

---

# 23. Banco de Dados

Engine de banco de dados:

```text
SQLite
```

Banco de dados primário:

```text
/opt/vaultwarden/data/db.sqlite3
```

Motivo da escolha do SQLite:

- deployment pessoal
- baixa complexidade operacional
- baixo consumo de recursos
- fácil recuperação
- nenhum serviço de banco de dados adicional
- adequado para a carga de trabalho atual

NÃO introduza o PostgreSQL apenas com o objetivo de fazer a arquitetura parecer mais "profissional".

A arquitetura atual valoriza intencionalmente a redução da complexidade operacional.

---

# 24. Conta do Vaultwarden

A conta de usuário primária foi criada com sucesso.

As seguintes operações foram testadas:

```text
criar conta
    ->
login
    ->
logout
    ->
login novamente
```

O Web Vault funciona através de:

```text
https://vault.rufonex.com.br
```

A extensão oficial do navegador Bitwarden também foi configurada para utilizar o
ambiente auto-hospedado em vez da nuvem do Bitwarden (Bitwarden Cloud).

A extensão do navegador foi testada com sucesso.

---

# 25. Configuração do Cliente Bitwarden Auto-hospedado

A extensão do navegador deve apontar para:

```text
https://vault.rufonex.com.br
```

Ambiente:

```text
Self-hosted (Auto-hospedado)
```

NÃO deve permanecer apontada para:

```text
bitwarden.com
```

Se a extensão relatar que a senha mestre é inválida enquanto o Web
Vault funciona normalmente, verifique primeiro o ambiente de servidor selecionado.

---

# 26. Autenticação de Dois Fatores (2FA)

Segundo fator atual:

```text
TOTP
```

O TOTP foi configurado e testado com sucesso na conta administrativa.

O código de recuperação está armazenado fora do Vaultwarden.

O código de recuperação NÃO DEVE ser:

- comitado no Git
- armazenado no próprio cofre do Vaultwarden
- inserido no `.env` do Docker
- armazenado em local público/compartilhado

O deployment atual utiliza intencionalmente apenas TOTP.

WebAuthn não é obrigatório, a menos que seja intencionalmente adicionado no futuro.

---

# 27. Política de Registro do Vaultwarden

Política atual:

```text
SIGNUPS_ALLOWED=false
INVITATIONS_ALLOWED=false
```

A primeira conta foi criada durante o bootstrap inicial enquanto o cadastro estava
temporariamente habilitado.

Após a criação da conta, o cadastro foi desabilitado.

Isso impede que usuários não autorizados se registrem no endpoint público.

---

# 28. Ambiente do Vaultwarden (Environment)

Arquivo de ambiente atual:

```text
/opt/vaultwarden/.env
```

Este arquivo é protegido e NÃO deve ser comitado se contiver
segredos de implantação ou valores sensíveis específicos do ambiente.

No mínimo, contém configurações relacionadas a:

```text
DOMAIN
SIGNUPS_ALLOWED
BIND_ADDRESS
```

O domínio de produção é:

```text
https://vault.rufonex.com.br
```

O arquivo de ambiente deve ser revisado cuidadosamente antes de ser adicionado ao Git.

Utilize um arquivo de exemplo como:

```text
.env.example
```

para controle de versão.

---

# 29. Estratégia de Backup

Arquitetura de backup atual:

```text
Vaultwarden
    |
    +--> backup nativo do SQLite
    |
    +--> arquivo completo de /data
    |
    +--> checksum SHA-256
    |
    +--> armazenamento local de backup
```

Diretório de backup local atual:

```text
/var/backups/vaultwarden
```

Diretório temporário de preparação (staging):

```text
/var/lib/vaultwarden-backup
```

Script de backup:

```text
/usr/local/sbin/vaultwarden-backup
```

---

# 30. Backup Nativo do SQLite no Vaultwarden

O binário do Vaultwarden fornece um comando nativo de backup:

```bash
docker exec vaultwarden /vaultwarden backup
```

Isso gera um backup SQLite com timestamp dentro de `/data`.

Exemplo:

```text
db_20260922_180023.sqlite3
```

O mecanismo de backup nativo é preferível em vez de copiar cegamente o
banco SQLite em execução enquanto ele está sendo modificado.

---

# 31. Arquivo de Backup Completo

O script de backup cria um arquivo compactado completo contendo o estado
persistente importante do Vaultwarden.

O arquivo atualmente preserva:

```text
db.sqlite3
rsa_key.pem
icon_cache/
```

e também preservará arquivos/diretórios persistentes adicionais que possam surgir
em `/data` no futuro, sujeitos às regras de exclusão do script.

O arquivo compactado exclui intencionalmente:

```text
db.sqlite3-wal
db.sqlite3-shm
db_*.sqlite3
tmp/
```

O banco de dados incluído no arquivo final é a cópia consistente do SQLite
gerada pelo mecanismo de backup nativo do Vaultwarden.

---

# 32. Integridade do Backup

Cada backup produz:

```text
vaultwarden_YYYYMMDD_HHMMSS.tar.gz
```

e:

```text
vaultwarden_YYYYMMDD_HHMMSS.tar.gz.sha256
```

Exemplo:

```text
vaultwarden_20260922_155527.tar.gz
vaultwarden_20260922_155527.tar.gz.sha256
```

O arquivo é validado utilizando:

```bash
tar -tzf archive.tar.gz
```

e:

```bash
sha256sum -c archive.tar.gz.sha256
```

Um teste real de backup produziu:

```text
TAR OK
checksum OK
```

---

# 33. Teste de Restauração de Backup

Um teste real de restauração já foi realizado.

Processo:

```text
backup de produção
        |
        v
diretório de restore separado
        |
        v
container temporário do Vaultwarden
        |
        v
Vaultwarden iniciado
        |
        v
Web Vault acessível
        |
        v
dados da conta recuperados
```

A instância de restauração inicializou com sucesso e apresentou a tela de login
do Vaultwarden.

Isso comprova que o backup não é meramente válido sintaticamente; ele é capaz de
reconstruir uma instância funcional do Vaultwarden.

A instância do teste de restauração foi removida em seguida.

Recursos temporários de restauração não devem permanecer no servidor de produção.

---

# 34. Comportamento do Script de Backup

O script de backup executa aproximadamente o seguinte fluxo:

```text
1. Verifica se o container do Vaultwarden existe.
2. Verifica se o container está em execução.
3. Verifica se a aplicação está saudável (healthy).
4. Executa o backup nativo do SQLite no Vaultwarden.
5. Para o Vaultwarden de forma limpa.
6. Copia os dados persistentes necessários não voláteis.
7. Insere o backup consistente do SQLite como db.sqlite3.
8. Cria o arquivo compactado.
9. Gera o checksum SHA-256.
10. Valida o conteúdo do arquivo compactado.
11. Remove arquivos temporários de backup do banco de dados.
12. Inicia o Vaultwarden.
13. Aguarda o Vaultwarden retornar ao estado saudável (healthy).
14. Valida o checksum.
15. Remove backups antigos com mais de RETENTION_DAYS=10 e seus arquivos .sha256 correspondentes.
16. Reporta sucesso.
```

O script utiliza um lock para prevenir backups concorrentes.

A lógica de espera pelo healthcheck foi intencionalmente aumentada porque o healthcheck
da imagem Docker do Vaultwarden não roda necessariamente de forma imediata após a inicialização
do container.

O período atual de espera de saúde foi ajustado para tolerar o intervalo de
healthcheck da imagem.

---

# 35. Lições Aprendidas sobre Backup

Importante:

NÃO assuma que:

```text
docker container running
```

significa:

```text
backup concluído corretamente
```

Também NÃO assuma que:

```text
.tar.gz existe
```

significa:

```text
backup é restaurável
```

A cadeia correta de validação é:

```text
backup
    ->
arquivo compactado
    ->
checksum
    ->
conteúdo do arquivo
    ->
restore
    ->
inicialização da aplicação
    ->
teste de login
```

---

# 36. Retenção do Backup Local

Backups locais são armazenados em:

```text
/var/backups/vaultwarden
```

### Política de Retenção:

- Janela de retenção configurada: **10 dias** (`RETENTION_DAYS=10`).
- O expurgo é executado estritamente **após** o novo backup ter sido criado e verificado (`tar -tzf` e `sha256sum -c`).
- Ele varre `/var/backups/vaultwarden` procurando arquivos com mais de 10 dias (`vaultwarden_*.tar.gz`).
- Para cada arquivo expirado, tanto o arquivo compactado quanto o seu arquivo de checksum `.sha256` correspondente são removidos.

### Teste de Validação Controlado:

- Um teste controlado foi realizado criando um par de backup **FICTÍCIO** (`vaultwarden_*.tar.gz` e `.sha256`) com idade simulada de 15 dias, criado exclusivamente para fins de validação.
- Uma execução real do `vaultwarden-backup.service` foi disparada via `systemctl start vaultwarden-backup.service`.
- O script criou o novo backup com sucesso e validou seu checksum SHA-256.
- O par de backup fictício expirado foi removido com sucesso pela função `prune_old_backups`.
- Todos os backups reais genuínos permaneceram intactos.
- O Vaultwarden concluiu a execução no estado `running/healthy`.

---

# 37. Automação do Backup

A execução do backup é automatizada através de unidades de timer e serviço do systemd.

Arquitetura:

```text
vaultwarden-backup.timer
        ↓
vaultwarden-backup.service
        ↓
/usr/local/sbin/vaultwarden-backup
        ↓
backup nativo SQLite
        ↓
arquivo compactado + SHA-256
        ↓
/var/backups/vaultwarden/
```

### Componentes:

- Service Unit: `/etc/systemd/system/vaultwarden-backup.service`
  - `Type=oneshot`
  - `Requires=docker.service`
  - `After=docker.service`
  - `ExecStart=/usr/local/sbin/vaultwarden-backup`
  - `UMask=0077`
  - `NoNewPrivileges=true`
  - `TimeoutStartSec=20min`

- Timer Unit: `/etc/systemd/system/vaultwarden-backup.timer`
  - Agendamento: `OnCalendar=*-*-* 03:00:00` (diariamente às 03:00)
  - `Persistent=true`
  - `WantedBy=timers.target`
  - Estado: habilitado e ativo

### Status de Validação:

- O script de backup foi validado anteriormente.
- `vaultwarden-backup.service` foi criado como `Type=oneshot`, dependente de `docker.service`.
- `vaultwarden-backup.timer` foi criado, habilitado e está ativo aguardando a próxima execução.
- A execução manual do serviço foi realizada com sucesso (`systemctl start vaultwarden-backup.service`).
- O backup `vaultwarden_20260922_173509.tar.gz` foi criado com sucesso.
- O arquivo de checksum `.sha256` correspondente foi criado e validado.
- O Vaultwarden finalizou o processo no estado `running/healthy`.
- A execução diária do backup via systemd às 03:00 foi observada em execução real: arquivo de backup 'vaultwarden_20260923_030040.tar.gz'. Portanto, o timer diário do systemd está efetivamente funcionando em produção.
- A retenção local permanece em 10 dias (`RETENTION_DAYS=10`).

Importante:

Atualmente, NÃO existe servidor de monitoramento centralizado (como Zabbix) implantado em produção para este ambiente.
A verificação operacional apoia-se diretamente nos logs do serviço systemd (`journalctl -u vaultwarden-backup.service`), inspeção do timer (`systemctl list-timers`) e na verificação dos arquivos gerados e checksums em `/var/backups/vaultwarden/`.
A ausência de alertas centralizados proativos é uma limitação operacional conhecida, e o monitoramento centralizado de backup via Zabbix está classificado como uma melhoria futura / evolução, e não como um pré-requisito para o backup local.

---

# 38. Backup Off-site — OCI via Restic (Implementado e Automatizado)

O backup off-site para a Oracle Cloud Infrastructure (OCI Object Storage) via Restic está implementado, automatizado via systemd e validado com restauração ponta a ponta em ambiente temporário isolado.

### Arquitetura:

```text
Vaultwarden
    |
    v
arquivo de backup local (/var/backups/vaultwarden) [retenção local de 10 dias]
    |
    v (vaultwarden-offsite-backup.service via timer 03:30)
Restic (criptografia client-side)
    |
    v
OCI Object Storage (bucket privado vaultwarden-offsite) [retenção remota de 10 dias]
```

### Configuração do Ambiente OCI:

- Região: `sa-saopaulo-1`
- Namespace: `groqo9fbzuaz`
- Compartment: `Backups`
- Bucket: `vaultwarden-offsite` (estritamente privado, sem acesso público)
- API: API compatível com S3
- ID do Repositório Restic: `7bbbe221`
- Segurança e Segredos: Credenciais em `/etc/vaultwarden-backup/oci.env` e chave de criptografia do repositório Restic são armazenadas com segurança no host fora do Git e NUNCA são comitadas no repositório.

### Automação Off-site e Implementação no Systemd:

A automação é composta por três componentes versionados em `backup/`:
- **Script:** `/usr/local/sbin/vaultwarden-offsite-backup`
  - Obtém lock exclusivo em `/run/lock/vaultwarden-offsite-backup.lock`.
  - Carrega variáveis a partir de `/etc/vaultwarden-backup/oci.env`.
  - Varre `/var/backups/vaultwarden` em busca de backup local recente criado nos últimos 120 minutos (`-mmin -120`), realizando até 60 tentativas (intervalo de 30s) para garantir que o backup local das 03:00 tenha concluído.
  - Valida o checksum SHA-256 do backup local (`sha256sum -c "${LATEST}.sha256"`).
  - Envia backups para o OCI: `restic -o s3.bucket-lookup=path backup /var/backups/vaultwarden`.
  - Aplica retenção remota de 10 dias: `restic -o s3.bucket-lookup=path forget --keep-within 10d --prune`.
  - Verifica a integridade do repositório: `restic -o s3.bucket-lookup=path check`.
- **Service Unit:** `/etc/systemd/system/vaultwarden-offsite-backup.service`
  - `Type=oneshot`, `After=vaultwarden-backup.service`
  - `ExecStart=/usr/local/sbin/vaultwarden-offsite-backup`
  - `Environment=HOME=/root` (corrige explicitamente o aviso de diretório de cache do Restic durante a execução sob o systemd)
  - `UMask=0077`, `NoNewPrivileges=true`, `TimeoutStartSec=60min`
- **Timer Unit:** `/etc/systemd/system/vaultwarden-offsite-backup.timer`
  - `OnCalendar=*-*-* 03:30:00` (agendado diariamente às 03:30, 30 minutos após o backup local)
  - `Persistent=true`
  - `WantedBy=timers.target`

### Ciclo de Validação do Restic e Upload Real:

1. **Inicialização do Repositório:** Inicializado com sucesso (`restic init`).
2. **Verificação Inicial:** `restic check` executou de forma limpa sem erros.
3. **Snapshot de Teste e Restore:** Backup de teste enviado, verificado e expurgado com `forget --prune`.
4. **Execução Manual do Serviço Validada:** `systemctl start vaultwarden-offsite-backup.service` executou com sucesso.
   - Identificou o backup local recente (`vaultwarden_20260923_181123.tar.gz`).
   - Validou o checksum SHA-256 do arquivo local.
   - Snapshot Restic real criado (`d5f61547`).
   - `restic check` concluiu com: `no errors were found`.
   - Retenção remota `forget --keep-within 10d --prune` executada com sucesso.
   - Aviso de cache do Restic eliminado via `Environment=HOME=/root`.

### Validação de Restauração Real a partir do OCI:

- Restauração do snapshot `d5f61547` realizada para um ambiente temporário isolado: `/tmp/restic-vaultwarden-restore` (anteriormente à automação).
- 17 arquivos/diretórios restaurados com sucesso.
- Arquivo restaurado: `vaultwarden_20260923_181123.tar.gz`.
- SHA-256 restaurado: `fc40c0ae6da319aa89283632fb0aea58ab4f2ce286e578beb6dc167631b1ce40` (coincidiu estritamente com o arquivo `.sha256` armazenado).
- Arquivo TAR restaurado confirmado como contendo `./db.sqlite3` e `./rsa_key.pem`.
- O diretório temporário de restauração foi removido após a validação.
- A instância de produção do Vaultwarden NÃO foi substituída nem alterada durante o teste de restore.

### Distinções de Status Operacional:

- **Backup Local (03:00):** A execução automática pelo timer do systemd foi observada e validada em produção real (`vaultwarden_20260923_030040.tar.gz`).
- **Backup Off-site (03:30):** O serviço está implementado e validado manualmente. A execução automática agendada do `vaultwarden-offsite-backup.timer` às 03:30 ainda NÃO foi observada em produção. Documentar como "configurado e validado manualmente", mas NÃO como "execução automática validada".

---

# 39. Requisitos do Backup OCI e Postura de Segurança

- O bucket `vaultwarden-offsite` permanece estritamente privado, sem acesso público a objetos.
- Credencial dedicada com permissões mínimas necessárias.
- Segredos do OCI e senha do repositório Restic NUNCA são armazenados no Git ou em documentos públicos.
- Criptografia client-side é aplicada pelo Restic antes que qualquer dado saia do host.
- A restauração a partir do OCI foi testada e validada em ambiente isolado sem afetar a produção.

---

# 40. Objetivo de Recuperação de Desastres (Disaster Recovery)

O fluxo pretendido de recuperação de desastres é:

```text
VM destruída
    |
    v
Criar nova VM Debian
    |
    v
Configurar hostname/rede
    |
    v
Instalar Docker
    |
    v
Restaurar configuração versionada
    |
    v
Instalar/iniciar Vaultwarden
    |
    v
Restaurar backup do OCI
    |
    v
Restaurar dados persistentes
    |
    v
Configurar Cloudflare Tunnel
    |
    v
https://vault.rufonex.com.br
    |
    v
Login + TOTP
```

O sistema NÃO deve depender da sobrevivência da VM original.

---

# 41. Estratégia de Backup do Proxmox

Um backup Proxmox/PBS da VM foi realizado antes do deployment de produção
do Docker/Vaultwarden.

Esse backup representa o estado de baseline da VM após:

- instalação do Debian
- atualizações do sistema
- configuração de rede
- hardening de firewall
- hardening de SSH
- QEMU Guest Agent

O backup Proxmox/PBS foi validado com sucesso.

Isso fornece uma segunda camada de recuperação:

```text
Backup da aplicação
    +
Backup da VM
```

Eles são complementares.

---

# 42. Camadas de Recuperação

A hierarquia pretendida de recuperação é:

### Camada 1 — Recuperação da Aplicação

Restaurar:

```text
Vaultwarden /data
```

a partir do backup da aplicação.

### Camada 2 — Recuperação da VM

Restaurar a VM completa a partir do Proxmox/PBS.

### Camada 3 — Desastre de Site

Reconstruir a VM em outro local e restaurar o backup off-site criptografado.

---

# 43. Princípios de Segurança

Este deployment segue estes princípios:

1. Sem redirecionamento direto de portas da Internet para o Vaultwarden.
2. Cloudflare Tunnel é o mecanismo de ingresso público.
3. Vaultwarden possui uma VM dedicada.
4. Firewall do host tem DROP como padrão para tráfego de entrada (inbound).
5. SSH utiliza autenticação exclusivamente por chave pública.
6. Login root via SSH está desabilitado.
7. Autenticação por senha via SSH está desabilitada.
8. Apenas a LAN de gerenciamento pode acessar o SSH.
9. Apenas o Cloudflared pode acessar a porta publicada do Vaultwarden.
10. Cadastro público (signups) está desabilitado.
11. Convites estão desabilitados.
12. TOTP está habilitado.
13. Docker utiliza `no-new-privileges`.
14. Logs do Docker possuem rotação controlada.
15. Imagem do Vaultwarden é fixada por digest.
16. Dados persistentes da aplicação são armazenados fora do filesystem do container.
17. Backups são validados.
18. A restauração foi testada.
19. Segredos não são armazenados no Git.

---

# 44. Regras de Segurança para o Git

NUNCA comite:

```text
.env
```

quando contiver segredos ou valores sensíveis específicos do ambiente.

NUNCA comite:

```text
*.pem
*.key
*.secret
```

quando contiverem chaves privadas ou credenciais.

NUNCA comite:

```text
db.sqlite3
*.sqlite3
*.sqlite3-wal
*.sqlite3-shm
```

NUNCA comite:

```text
*.tar.gz
```

quando forem backups do Vaultwarden.

NUNCA comite:

- tokens do Cloudflare Tunnel
- chaves de acesso OCI (access keys)
- chaves secretas OCI (secret keys)
- segredos TOTP
- códigos de recuperação TOTP
- códigos de recuperação do Vaultwarden
- chaves privadas SSH
- credenciais reais de usuários

Modelos públicos de configuração (templates) são aceitáveis.

---

# 45. Estrutura Recomendada para o Git

Estrutura alvo do repositório:

```text
vaultwarden-infra/
|
├── README.md
├── PROJECT_CONTEXT.md
├── ARCHITECTURE.md
├── DECISIONS.md
├── SECURITY.md
|
├── docker/
│   ├── docker-compose.yaml
│   ├── daemon.json
│   └── .env.example
|
├── firewall/
│   ├── vaultwarden-docker-firewall
│   └── vaultwarden-docker-firewall.service
|
├── backup/
│   ├── vaultwarden-backup
│   ├── vaultwarden-backup.service
│   └── vaultwarden-backup.timer
|
├── docs/
│   ├── 01-vm-provisioning.md
│   ├── 02-network.md
│   ├── 03-firewall.md
│   ├── 04-ssh-hardening.md
│   ├── 05-docker.md
│   ├── 06-vaultwarden.md
│   ├── 07-cloudflare.md
│   ├── 08-backup.md
│   ├── 09-restore.md
│   ├── 10-maintenance.md
│   └── 11-disaster-recovery.md
|
└── diagrams/
    └── architecture.mmd
```

A estrutura exata pode evoluir, mas segredos e dados de produção devem permanecer
fora do Git.

---

# 46. Estado Atual em Produção

## Concluído

```text
Debian 13 VM                                                  CONCLUÍDO
IP estático/reservado                                         CONCLUÍDO
Acesso SSH                                                    CONCLUÍDO
Autenticação SSH por chave pública                            CONCLUÍDO
Hardening de SSH                                              CONCLUÍDO
Root SSH desabilitado                                         CONCLUÍDO
Firewall base                                                 CONCLUÍDO
Persistência de firewall                                      CONCLUÍDO
Teste de reboot do firewall                                   CONCLUÍDO
QEMU Guest Agent                                              CONCLUÍDO
Sincronização de horário do sistema                           CONCLUÍDO
Docker Engine                                                 CONCLUÍDO
Docker Compose                                                CONCLUÍDO
Hardening do daemon do Docker                                 CONCLUÍDO
Integração do firewall com Docker                             CONCLUÍDO
Container do Vaultwarden                                      CONCLUÍDO
SQLite                                                        CONCLUÍDO
/data persistente                                             CONCLUÍDO
Pinning da imagem por digest                                  CONCLUÍDO
Cloudflare Tunnel                                             CONCLUÍDO
Domínio HTTPS público                                         CONCLUÍDO
Conta do Vaultwarden                                          CONCLUÍDO
Teste de login/logout                                         CONCLUÍDO
Extensão do navegador Bitwarden                               CONCLUÍDO
TOTP                                                          CONCLUÍDO
Cadastro público desabilitado                                 CONCLUÍDO
Script de backup                                              CONCLUÍDO
Teste de integridade do backup                                CONCLUÍDO
Teste de restauração de backup                                CONCLUÍDO
Timer de backup via systemd                                   CONCLUÍDO
Automação de retenção de backup local                         CONCLUÍDO
Backup de baseline no Proxmox/PBS                             CONCLUÍDO
Repositório Restic no OCI                                     CONCLUÍDO
Upload real para OCI                                          CONCLUÍDO
restic check                                                  CONCLUÍDO
Restore de backup real a partir do OCI                        CONCLUÍDO
Validação do SHA-256 do backup recuperado                     CONCLUÍDO
Script de backup off-site (vaultwarden-offsite-backup)        CONCLUÍDO
Service unit de backup off-site                               CONCLUÍDO
Timer unit de backup off-site (03:30)                         CONCLUÍDO
Retenção remota de 10 dias no OCI (forget --prune)            CONCLUÍDO
Correção de cache do Restic (Environment=HOME=/root)          CONCLUÍDO
```

---

# 47. Trabalho Restante / Melhorias Futuras

O backup local está totalmente implementado, agendado via systemd (execução diária às 03:00 observada), validado, com retenção de 10 dias e restore testado.
O backup off-site para o OCI via Restic está automatizado via systemd (`vaultwarden-offsite-backup.service`/`timer` 03:30), validado manualmente, com retenção remota de 10 dias e restore verificado a partir do OCI em ambiente isolado.

Melhorias futuras / evoluções:

```text
Observação da execução automática do timer off-site (vaultwarden-offsite-backup.timer às 03:30)
Monitoramento centralizado (ex.: Zabbix; sem servidor Zabbix no ambiente atualmente)
Teste completo de disaster recovery (reconstrução ponta a ponta em outro hypervisor)
Verificação periódica de restore
Hardening de credenciais OCI
Procedimento regular de atualização do Vaultwarden
Runbook formal de manutenção
```

Não documente estes itens como concluídos até que tenham sido efetivamente implementados e testados.

---

# 48. Regras Operacionais para Mudanças Futuras

Antes de alterar a produção:

```text
1. Ler este PROJECT_CONTEXT.md.
2. Verificar o estado atual de produção.
3. Criar um backup quando apropriado.
4. Fazer a menor alteração necessária.
5. Validar a configuração antes de reiniciar serviços.
6. Testar a funcionalidade.
7. Verificar logs.
8. Verificar status de saúde (health).
9. Atualizar a documentação.
10. Comitar alterações de configuração no Git.
```

Nunca faça alterações manuais não documentadas que não possam ser reproduzidas.

---

# 49. Política de Atualização de Versões

Atualizações do Vaultwarden devem ser intencionais.

NÃO execute cegamente:

```bash
docker compose pull
```

contra uma imagem não fixada (`latest`).

Processo de atualização:

```text
1. Revisar as notas de versão do Vaultwarden.
2. Verificar se a versão contém correções de segurança.
3. Criar um backup atual da aplicação.
4. Confirmar a integridade do backup.
5. Atualizar o digest da imagem intencionalmente.
6. Baixar a nova imagem (pull).
7. Recriar o container.
8. Aguardar o healthcheck.
9. Verificar o Web Vault.
10. Verificar a extensão do navegador.
11. Verificar login/TOTP.
12. Checar logs.
13. Documentar a mudança de versão.
```

O rollback deve permanecer possível através do digest da imagem e do backup anteriores funcionais conhecidos.

---

# 50. Princípios de Troubleshooting

Ao diagnosticar o Vaultwarden:

Verificar nesta ordem:

```text
Rede
    ->
Cloudflare Tunnel
    ->
porta do host
    ->
DOCKER-USER
    ->
mapeamento de portas do Docker
    ->
container
    ->
saúde da aplicação
    ->
SQLite
```

Comandos úteis:

```bash
sudo docker compose -f /opt/vaultwarden/docker-compose.yaml ps
```

```bash
sudo docker logs --tail=100 vaultwarden
```

```bash
sudo docker inspect vaultwarden \
  --format 'Status={{.State.Status}} Health={{.State.Health.Status}}'
```

```bash
sudo ss -lntp
```

```bash
sudo iptables -S
```

```bash
sudo iptables -S DOCKER-USER
```

```bash
sudo iptables -S VW-DOCKER
```

```bash
curl -i http://192.168.15.200:8080/alive
```

---

# 51. Distinção Operacional Importante

Estes são conceitos distintos:

```text
Disponibilidade da VM
Disponibilidade do Docker
Disponibilidade do container
Saúde da aplicação
Alcance externo
Disponibilidade de autenticação
Disponibilidade de backup
Disponibilidade de restauração
```

Um healthcheck bem-sucedido não prova que a rota externa da Cloudflare funciona.

Uma requisição bem-sucedida da Cloudflare não prova que o backup funciona.

Uma criação bem-sucedida de backup não prova que a restauração funciona.

Cada camada deve ser testada de forma independente.

---

# 52. Modelo Mental Atual

O modelo mental mais importante para esta infraestrutura é:

```text
                       ACESSO PÚBLICO
                             |
                       Cloudflare TLS
                             |
                      Cloudflare Tunnel
                             |
                        .253 LXC
                             |
                        HTTP :8080
                             |
                        .200 VM
                             |
                         Docker
                             |
                      DOCKER-USER
                             |
                      VW-DOCKER
                             |
                       Vaultwarden
                             |
                           SQLite
                             |
                         /data
                             |
                   +---------+---------+
                   |                   |
             backup local        backup off-site
                                       |
                                      OCI
```

A fronteira de segurança existe em várias camadas:

```text
Cloudflare
    +
rede
    +
firewall do host
    +
firewall do Docker
    +
container
    +
autenticação do Vaultwarden
    +
TOTP
    +
cofre criptografado
```

---

# 53. Instruções para o OpenCode

Ao trabalhar neste repositório, o OpenCode deve seguir estas regras:

- Tratar `PROJECT_CONTEXT.md` como o contexto autoritativo do projeto.
- Não inventar infraestrutura que não esteja documentada.
- Não substituir a arquitetura existente sem justificativa explícita.
- Não introduzir serviços adicionais meramente por convenção ou aparência.
- Não remover controles de segurança sem documentar o motivo.
- Não armazenar segredos no repositório.
- Nunca gerar ou comitar chaves privadas.
- Nunca gerar ou comitar arquivos `.env` reais contendo segredos.
- Nunca gerar ou comitar arquivos reais de banco de dados do Vaultwarden.
- Nunca gerar ou comitar backups reais.
- Diferenciar claramente o que é configuração ATUAL, PLANEJADA e OBSOLETA.
- Ao modificar a configuração de produção, primeiro validar a alteração.
- Preferir configuração reproduzível a procedimentos manuais.
- Manter a configuração de produção e a documentação sincronizadas.
- Atualizar este contexto de projeto sempre que uma decisão arquitetural for alterada.
- Nunca declarar que um backup, teste de restauração, controle de segurança ou sistema de monitoramento está implementado a menos que tenha sido efetivamente testado.
- Quando em dúvida sobre o estado atual, inspecionar o host/configuração real em vez de adivinhar.
