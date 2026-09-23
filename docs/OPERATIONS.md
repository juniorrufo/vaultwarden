# Guia Operacional — Vaultwarden

Este documento contém os procedimentos operacionais do dia a dia, rotinas de verificação, análise de logs, troubleshooting e política de atualização da infraestrutura do Vaultwarden.

Todos os comandos listados aqui são reais e baseados no ambiente Debian 13 em execução na VM (`192.168.15.200`).

---

## 1. Verificação de Estado e Saúde

Como o usuário administrativo `junior` não faz parte do grupo `docker` por razões de segurança, os comandos Docker devem ser executados com `sudo`.

### 1.1. Status do Container
Verificar se o container está em execução e gerenciado pelo Compose:
```bash
sudo docker compose -f /opt/vaultwarden/docker-compose.yaml ps
```

### 1.2. Status de Saúde da Aplicação (Healthcheck)
Consultar o status de integridade do container:
```bash
sudo docker inspect vaultwarden \
  --format 'Status={{.State.Status}} Health={{.State.Health.Status}}'
```
*Saída esperada:* `Status=running Health=healthy`

### 1.3. Teste do Endpoint de Vida (/alive)
O endpoint de saúde da aplicação pode ser consultado diretamente via HTTP:
```bash
curl -i http://192.168.15.200:8080/alive
```
*Saída esperada:* `HTTP/1.1 200 OK` (quando executado a partir do host autorizado `192.168.15.253`).
*Atenção:* Se executado de outros hosts da rede LAN, o firewall retornará `connection timeout`, conforme projetado.

### 1.4. Portas em Escuta no Host
Verificar portas TCP locais abertas:
```bash
sudo ss -lntp
```
Deve constar a porta `192.168.15.200:8080` (Docker proxy) e a porta `22` (sshd).

---

## 2. Monitoramento de Logs

### 2.1. Logs Recentes do Vaultwarden
Exibir as últimas 100 linhas de log do container:
```bash
sudo docker logs --tail=100 vaultwarden
```

### 2.2. Acompanhamento Contínuo de Logs
```bash
sudo docker logs -f vaultwarden
```

### 2.3. Rotação de Logs
A política de logs foi definida em `/etc/docker/daemon.json` para evitar crescimento descontrolado de disco:
- Driver: `local`
- Tamanho máximo por arquivo: `20m`
- Quantidade máxima de arquivos retidos: `5`

---

## 3. Auditoria de Firewall

O ambiente utiliza `iptables-nft` e a persistência é garantida pelo serviço `vaultwarden-docker-firewall.service`.

### 3.1. Listar Regras Gerais do Host
```bash
sudo iptables -S
sudo ip6tables -S
```

### 3.2. Listar Regras da Cadeia DOCKER-USER
```bash
sudo iptables -S DOCKER-USER
```
*Saída esperada:*
```text
-N DOCKER-USER
-A DOCKER-USER -j VW-DOCKER
```

### 3.3. Listar Regras da Cadeia VW-DOCKER
```bash
sudo iptables -S VW-DOCKER
```
*Saída esperada:*
```text
-N VW-DOCKER
-A VW-DOCKER -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A VW-DOCKER -s 192.168.15.253/32 -p tcp -m conntrack --ctorigdst 192.168.15.200 --ctorigdstport 8080 -j ACCEPT
-A VW-DOCKER -p tcp -m conntrack --ctorigdst 192.168.15.200 --ctorigdstport 8080 -j DROP
-A VW-DOCKER -j RETURN
```

---

## 4. Política de Atualização de Versão

A imagem Docker é propositalmente fixada por **digest SHA-256** para evitar atualizações não testadas ou alterações na tag `latest`.

### Processo Formal de Atualização:
1. **Revisão de Release:** Ler os changelogs e notas da versão do Vaultwarden no repositório oficial, identificando eventuais correções de segurança ou mudanças estruturais.
2. **Backup Preventivo Obrigatório:** Executar o script de backup antes de qualquer alteração:
   ```bash
   sudo /usr/local/sbin/vaultwarden-backup
   ```
3. **Validação do Backup:** Confirmar que o arquivo gerado em `/var/backups/vaultwarden/` possui integridade TAR (`tar -tzf`) e SHA-256 validada.
4. **Obtenção do Novo Digest:** Identificar o digest SHA-256 oficial da nova versão para a arquitetura `linux/amd64`.
5. **Atualização no Compose:** Editar `/opt/vaultwarden/docker-compose.yaml` substituindo o digest antigo pelo novo digest verificado.
6. **Download da Imagem:**
   ```bash
   sudo docker compose -f /opt/vaultwarden/docker-compose.yaml pull
   ```
7. **Recriação do Container:**
   ```bash
   sudo docker compose -f /opt/vaultwarden/docker-compose.yaml up -d
   ```
8. **Acompanhamento de Inicialização:** Aguardar o status `healthy`:
   ```bash
   sudo docker inspect vaultwarden --format 'Status={{.State.Status}} Health={{.State.Health.Status}}'
   ```
9. **Testes Funcionais Finais:**
   - Acesso ao Web Vault em `https://vault.rufonex.com.br`.
   - Conexão e sincronização pela extensão do navegador Bitwarden.
   - Login e validação de segundo fator (TOTP).
   - Verificação de logs em busca de erros: `sudo docker logs --tail=100 vaultwarden`.
10. **Registro da Atualização:** Atualizar a documentação e commits do repositório com o novo digest, mantendo registrado o digest anterior para eventual rollback.

---

## 5. Operação e Monitoramento do Backup Agendado (Systemd)

O backup da aplicação é agendado e executado através das unidades `vaultwarden-backup.timer` e `vaultwarden-backup.service`.

### 5.1. Verificar Estado do Agendador (Timer)
Checar se o timer está ativo e qual o horário do próximo disparo:
```bash
sudo systemctl status vaultwarden-backup.timer
```
*Saída esperada:* `Active: active (waiting)`

Listar todos os timers do sistema e localizar o agendamento:
```bash
sudo systemctl list-timers | grep vaultwarden
```
*Horário padrão:* diariamente às 03:00 da madrugada (`Persistent=true`).

### 5.2. Disparo Manual do Backup via Systemd
Para testar a rotina pelo mesmo mecanismo de serviço que o timer utiliza:
```bash
sudo systemctl start vaultwarden-backup.service
```

### 5.3. Acompanhamento dos Logs do Serviço
Inspecionar a saída da última execução do backup:
```bash
sudo journalctl -u vaultwarden-backup.service --no-pager -n 50
```

### 5.4. Execução Manual Direta pelo Script
Caso seja necessário disparar o backup diretamente pelo shell:
```bash
sudo /usr/local/sbin/vaultwarden-backup
```

### 5.5. Auditoria da Retenção Local de Backups (10 Dias)
O script de backup aplica automaticamente a política de retenção local (`RETENTION_DAYS=10`) ao final de cada execução bem-sucedida, somente após a validação do arquivo `.tar.gz` (`tar -tzf`), do checksum `.sha256` (`sha256sum -c`) e da confirmação de que o container permanece `running/healthy`.

Para listar os backups locais armazenados e seus arquivos de integridade:
```bash
ls -lht /var/backups/vaultwarden/
```

Para inspecionar arquivos com mais de 10 dias (que serão expurgados na próxima execução bem-sucedida):
```bash
sudo find /var/backups/vaultwarden -maxdepth 1 -type f -name 'vaultwarden_*.tar.gz' -mmin +14400
```

### 5.6. Estado do Monitoramento e Limitações Conhecidas
- **Ausência de Monitoramento Centralizado:** Atualmente não existe um servidor Zabbix em produção para este ambiente. Por esta razão, nenhum template, script ou integração com Zabbix foi implementada nesta etapa.
- **Forma de Verificação:** A validação do funcionamento dos backups é exclusivamente local, inspecionando os logs do serviço (`journalctl -u vaultwarden-backup.service`), o agendamento do timer (`systemctl list-timers`) e a integridade dos arquivos em `/var/backups/vaultwarden/`.
- **Evolução Futura:** A ausência de alertas ativos remotos em caso de falha é uma limitação operacional conhecida; a integração com Zabbix é classificada como melhoria futura / evolução e não impacta o funcionamento do backup local implementado.

### 5.7. Operação e Auditoria do Repositório OCI (Restic)
O repositório em nuvem no Oracle Cloud Infrastructure (`sa-saopaulo-1`, bucket privado `vaultwarden-offsite`, ID `7bbbe221`) utiliza criptografia client-side gerenciada pelo Restic e automação systemd configurada para 03:30 diariamente.

- **Status do Timer Off-site:**
  ```bash
  sudo systemctl status vaultwarden-offsite-backup.timer
  ```
- **Disparo Manual do Envio Off-site via Systemd:**
  ```bash
  sudo systemctl start vaultwarden-offsite-backup.service
  ```
- **Acompanhamento dos Logs do Serviço Off-site:**
  ```bash
  sudo journalctl -u vaultwarden-offsite-backup.service --no-pager -n 50
  ```
- **Listar Snapshots no OCI:**
  ```bash
  restic snapshots
  ```
- **Auditoria de Integridade Criptográfica do Repositório:**
  ```bash
  restic check
  ```
- **Política de Retenção Remota (10 Dias):**
  A retenção remota é executada automaticamente pelo script após cada backup bem-sucedido:
  ```bash
  restic forget --keep-within 10d --prune
  ```
- **Restauração de Teste em Diretório Isolado:**
  ```bash
  restic restore <snapshot_id> --target /tmp/restic-vaultwarden-restore
  ```
- **Nota Operacional sobre Cache do Restic no Systemd:**
  O systemd não define `$HOME` para serviços `Type=oneshot` por padrão, o que causava um aviso de cache (`unable to open cache ...`). A unidade de serviço define explicitamente `Environment=HOME=/root`, corrigindo a localização do cache em `/root/.cache/restic`.
- **Atenção Operacional e Status de Validação:**
  - O script `/usr/local/sbin/vaultwarden-offsite-backup` localiza o backup local mais recente (`-mmin -120`), valida o checksum `.sha256`, envia o snapshot via Restic ao OCI, executa `restic check` e aplica a retenção remota de 10 dias (`forget --keep-within 10d --prune`).
  - A execução manual do serviço (`systemctl start vaultwarden-offsite-backup.service`) foi executada e validada com sucesso em produção.
  - A **execução automática** agendada via timer (`03:30:00`) ainda **NÃO foi observada** em produção (a ser confirmada no próximo ciclo agendado). Portanto, o timer está documentado como configurado e validado manualmente, mas não como execução automática validada.
  - As credenciais OCI e senha do Restic residem exclusivamente no host (`/etc/vaultwarden/oci.env` com permissão `0600`) e nunca devem ser versionadas no Git.

---

## 6. Matriz de Troubleshooting Ordenado

Ao investigar qualquer indisponibilidade, siga rigorosamente esta ordem de camadas:

```text
1. Conectividade de Rede (IP e rotas da VM 192.168.15.200)
       ↓
2. Cloudflare Tunnel (Status do LXC 192.168.15.253 e conexão externa)
       ↓
3. Porta de Escuta do Host (ss -lntp verificando 192.168.15.200:8080)
       ↓
4. Regras de Firewall (iptables -S VW-DOCKER e contadores de DROP)
       ↓
5. Mapeamento de Portas e Bridge Docker (vaultwarden_net)
       ↓
6. Container do Vaultwarden (Status 'running' e logs de execução)
       ↓
7. Aplicação Vaultwarden (Endpoint /alive retornando HTTP 200)
       ↓
8. Banco de Dados SQLite (Integridade e permissões de /opt/vaultwarden/data/db.sqlite3)
```

---

## 7. Distinção de Camadas de Disponibilidade

Para evitar diagnósticos incorretos, mantenha em mente que estas situações representam níveis operacionais distintos:

| Camada | O que comprova | O que NÃO comprova |
| :--- | :--- | :--- |
| **VM Ativa (Ping/SSH)** | Host Debian e rede operacional | Docker ou Vaultwarden funcionando |
| **Docker Daemon Ativo** | Motor de containers operacional | Container do Vaultwarden saudável |
| **Container "running"** | Processo do container não caiu | Aplicação respondendo a requisições |
| **Healthcheck "healthy" (/alive)** | Vaultwarden responde localmente | Rota pública Cloudflare ou túnel operando |
| **Acesso Web Vault OK** | Serviço acessível publicamente | Backups estão sendo gerados ou são válidos |
| **Arquivo .tar.gz gerado** | Script de backup local foi executado | Dados são íntegros ou recuperáveis em caso de desastre |
| **Restore Local Testado** | Recuperação local validada em teste prático isolado | Envio para nuvem ou proteção contra desastres físicos |
| **Restore a partir do OCI Testado** | Recuperação externa validada a partir da nuvem | Execução automática do timer OCI às 03:30 observada ou disaster recovery completo |
| **Backup Off-site OCI (Restic)** | Envio à nuvem, integridade SHA-256/Restic e retenção remota de 10 dias validados manualmente | Execução automática agendada às 03:30 ainda não foi observada em produção |
