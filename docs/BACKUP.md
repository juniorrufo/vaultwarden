# Procedimentos de Backup — Vaultwarden

Este documento descreve detalhadamente a estratégia, o fluxo de execução, a integridade e os procedimentos de validação dos backups do Vaultwarden.

---

## 1. Visão Geral da Estratégia de Backup

A proteção dos dados do Vaultwarden é composta por múltiplas camadas complementares:

1. **Backup da Aplicação (Local):** Arquivamento consistente do banco SQLite e dos artefatos criptográficos persistentes em `/opt/vaultwarden/data`.
2. **Backup da VM (Infraestrutura):** Snapshot completo do estado baseline da VM no Proxmox VE / PBS (*Proxmox Backup Server*).
3. **Backup Off-site (Nuvem - Planejado):** Envio criptografado para OCI Object Storage via Restic.

---

## 2. Processo Atual de Backup Local

O backup local é executado através do script administrativo:
```text
/usr/local/sbin/vaultwarden-backup
```

### 2.1. Diretórios Utilizados
- **Destino final dos backups:** `/var/backups/vaultwarden/`
- **Área de trabalho temporária (staging):** `/var/lib/vaultwarden-backup/`
- **Diretório de dados da aplicação:** `/opt/vaultwarden/data/`

### 2.2. Execução Passo a Passo do Script
O script opera com controle de concorrência (lock) e executa o seguinte fluxo:

1. **Verificação Preliminar:** Confirma se o container `vaultwarden` existe, está em execução (`running`) e saudável (`healthy`).
2. **Backup Nativo do SQLite:** Executa o comando embutido no binário do Vaultwarden:
   ```bash
   docker exec vaultwarden /vaultwarden backup
   ```
   Isso produz um arquivo com timestamp consistente dentro de `/data` (ex.: `db_20260922_180023.sqlite3`), utilizando o mecanismo de backup online do próprio SQLite para evitar corrupção por escrita concorrente.
3. **Parada Limpa do Container:** Interrompe graciosamente o container do Vaultwarden para garantir que nenhum processo acesse os arquivos durante a extração:
   ```bash
   sudo docker compose -f /opt/vaultwarden/docker-compose.yaml stop
   ```
4. **Cópia de Arquivos Persistentes:** Copia os dados essenciais para a área de staging em `/var/lib/vaultwarden-backup/`.
5. **Substituição da Base:** Substitui o arquivo da base pelo snapshot limpo gerado no passo 2 como `db.sqlite3`.
6. **Empacotamento Compactado:** Cria o arquivo final compactado:
   ```text
   vaultwarden_YYYYMMDD_HHMMSS.tar.gz
   ```
7. **Exclusões Deliberadas:** São expressamente excluídos do arquivo final:
   - `db.sqlite3-wal` (arquivo WAL volátil)
   - `db.sqlite3-shm` (memória compartilhada SQLite)
   - `db_*.sqlite3` (artefatos temporários intermediários do comando backup)
   - `tmp/` (arquivos temporários)
8. **Geração de Checksum:** Gera o arquivo de hash SHA-256 no destino:
   ```text
   vaultwarden_YYYYMMDD_HHMMSS.tar.gz.sha256
   ```
9. **Validação do Arquivo TAR:** Testa a integridade estrutural do arquivo compactado:
   ```bash
   tar -tzf vaultwarden_YYYYMMDD_HHMMSS.tar.gz
   ```
10. **Limpeza Temporária:** Remove os arquivos transitórios da base SQLite de `/opt/vaultwarden/data/`.
11. **Reinicialização do Container:** Inicia novamente o container do Vaultwarden:
    ```bash
    sudo docker compose -f /opt/vaultwarden/docker-compose.yaml start
    ```
12. **Aguardar Recuperação da Aplicação:** Aguarda o container atingir o estado `healthy` (respeitando o intervalo do healthcheck interno).
13. **Validação Final do Hash:** Verifica se o checksum SHA-256 coincide rigorosamente:
    ```bash
    sha256sum -c vaultwarden_YYYYMMDD_HHMMSS.tar.gz.sha256
    ```
14. **Aplicação da Retenção Local:** Executa `prune_old_backups` para remover arquivos `vaultwarden_*.tar.gz` e respectivos `.sha256` com mais de 10 dias (`RETENTION_DAYS=10`), somente após a criação e validação bem-sucedida do backup atual.
15. **Conclusão:** Emite mensagem de conclusão bem-sucedida.

---

## 3. Automação do Backup via Systemd (Timer e Service)

A execução do backup é automatizada por meio de unidades dedicadas do systemd integradas ao ciclo de vida do host e do Docker.

### 3.1. Arquitetura da Automação

```text
vaultwarden-backup.timer
        ↓
vaultwarden-backup.service
        ↓
/usr/local/sbin/vaultwarden-backup
        ↓
native SQLite backup
        ↓
archive + SHA-256
        ↓
/var/backups/vaultwarden/
```

### 3.2. Unidades Systemd Implementadas

As definições estão versionadas no repositório em `backup/` e instaladas em `/etc/systemd/system/`:

- **Service Unit (`/etc/systemd/system/vaultwarden-backup.service`):**
  - `Type=oneshot`: Executa o script até sua conclusão e encerra.
  - `Requires=docker.service` e `After=docker.service`: Garante que o serviço de backup só inicie com o Docker operacional.
  - `ExecStart=/usr/local/sbin/vaultwarden-backup`: Aciona o script principal.
  - `UMask=0077`: Garante que arquivos gerados sejam lidos apenas pelo superusuário.
  - `NoNewPrivileges=true`: Previne escalonamento de privilégios.
  - `TimeoutStartSec=20min`: Janela de tolerância para conclusão de cópias e healthcheck.

- **Timer Unit (`/etc/systemd/system/vaultwarden-backup.timer`):**
  - `OnCalendar=*-*-* 03:00:00`: Disparo diário programado para as 03:00 da madrugada.
  - `Persistent=true`: Caso a VM esteja desligada no horário programado, o systemd dispara o backup imediatamente após a inicialização.
  - `WantedBy=timers.target`: Ativado automaticamente no boot do sistema operacional.

### 3.3. Estado Validado da Automação

- [x] O script de backup `/usr/local/sbin/vaultwarden-backup` já existia e foi validado previamente.
- [x] O serviço `vaultwarden-backup.service` foi criado como `Type=oneshot` com dependência direta de `docker.service`.
- [x] O timer `vaultwarden-backup.timer` foi criado, habilitado e colocado em estado ativo (`active (waiting)`).
- [x] A execução MANUAL do serviço foi disparada via `sudo systemctl start vaultwarden-backup.service` e concluída com sucesso.
- [x] O arquivo de backup `vaultwarden_20260922_173509.tar.gz` foi criado com sucesso em `/var/backups/vaultwarden/`.
- [x] O arquivo de checksum correspondente (`.sha256`) foi gerado e conferido com integridade.
- [x] O container do Vaultwarden reiniciou e concluiu o ciclo em estado `running/healthy`.
- [x] O agendamento da próxima execução foi registrado pelo systemd para 23/09/2026 às 03:00.
- [ ] *Ressalva importante:* A execução automática no horário agendado (madrugada) ainda **não** foi observada em produção; portanto, não deve ser documentada como teste automático concluído.

### 3.4. Política de Retenção Local

- **Janela de Retenção:** 10 dias (`RETENTION_DAYS=10`).
- **Comportamento Seguro:** A rotina de expurgo (`prune_old_backups`) só é executada estritamente após a conclusão e validação bem-sucedida do novo backup (`tar -tzf` e `sha256sum -c`) e confirmação do container em estado `healthy`.
- **Escopo do Expurgo:** Remove arquivos `vaultwarden_*.tar.gz` criados há mais de 10 dias em `/var/backups/vaultwarden/` e apaga simultaneamente o respectivo arquivo de verificação `.sha256`.

### 3.5. Teste Controlado de Retenção Validado

- **Ambiente de Teste:** Para validar a lógica de exclusão sem colocar em risco os backups reais existentes, foi criado previamente em `/var/backups/vaultwarden/` um par de arquivos **FICTÍCIO** (`vaultwarden_*.tar.gz` e `.sha256`) com data de modificação simulada de 15 dias atrás, criado exclusivamente para validação.
- **Execução Real:** O serviço `vaultwarden-backup.service` foi executado via `sudo systemctl start vaultwarden-backup.service`.
- **Resultados Validados:**
  - O par de arquivos fictício de 15 dias foi identificado e removido com sucesso.
  - Todos os backups reais legítimos permaneceram intactos no diretório.
  - O backup da própria execução foi gerado normalmente.
  - O checksum SHA-256 do novo backup foi validado com sucesso.
  - O container Vaultwarden encerrou o ciclo em estado `running/healthy`.

---

## 4. Comandos Reais de Validação

Para auditar os backups existentes e o estado do agendador:

### 4.1. Listar Backups Existentes
```bash
ls -lh /var/backups/vaultwarden/
```

### 4.2. Validar Checksum SHA-256
```bash
cd /var/backups/vaultwarden/
sha256sum -c vaultwarden_20260922_173509.tar.gz.sha256
```
*Resultado esperado:* `vaultwarden_20260922_173509.tar.gz: OK`

### 4.3. Inspecionar Conteúdo do TAR sem Extrair
```bash
tar -tzvf /var/backups/vaultwarden/vaultwarden_20260922_173509.tar.gz
```
*Arquivos esperados no arquivo:*
- `./db.sqlite3`
- `./rsa_key.pem`
- `./icon_cache/`

### 4.4. Verificar Status do Timer e Próximo Disparo
```bash
sudo systemctl status vaultwarden-backup.timer
sudo systemctl list-timers | grep vaultwarden
```

### 4.5. Consultar Logs da Última Execução do Serviço
```bash
sudo journalctl -u vaultwarden-backup.service --no-pager -n 50
```

---

## 5. O que foi EFETIVAMENTE TESTADO vs O que é PLANEJADO

### ✅ Estado Atual Testado e Validado em Produção
- [x] Backup diário via systemd (`vaultwarden-backup.service` e `vaultwarden-backup.timer`).
- [x] Integridade SHA-256 (validação automatizada de integridade estrutural `tar -tzf` e hash `sha256sum -c`).
- [x] Restore testado (validação funcional realizada em ambiente temporário isolado, ver [`RESTORE.md`](file:///home/juniorrufo/projetos/vaultwarden/docs/RESTORE.md)).
- [x] Retenção de 10 dias (`RETENTION_DAYS=10`, expurgo automático pós-validação de pares `.tar.gz` e `.sha256` testado com par fictício de 15 dias).
- [x] Backup de baseline da VM no Proxmox VE / PBS validado com sucesso.

### ⚠️ Itens Pendentes (Não documentar como implementados)
- [ ] **Observação de Disparo Automático Agendado:** Registro da primeira execução real noturna disparada automaticamente pelo timer às 03:00.
- [ ] **Monitoramento via Zabbix:** Coleta e alertas do status de sucesso/falha da rotina de backup.
- [ ] **OCI Object Storage:** Criação e configuração de bucket privado no Oracle Cloud Infrastructure.
- [ ] **Backup off-site criptografado:** Configuração de repositório criptografado via Restic em nuvem.
- [ ] **Teste de restore a partir do OCI:** Validação prática de recuperação direta do armazenamento em nuvem.
- [ ] **Disaster recovery completo:** Simulação ponta a ponta de perda total da VM e reconstrução em outro hypervisor.
