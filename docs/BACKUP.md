# Procedimentos de Backup — Vaultwarden

Este documento descreve detalhadamente a estratégia, o fluxo de execução, a integridade e os procedimentos de validação dos backups do Vaultwarden.

---

## 1. Visão Geral da Estratégia de Backup

A proteção dos dados do Vaultwarden é composta por múltiplas camadas complementares:

1. **Backup da Aplicação (Local):** Arquivamento consistente do banco SQLite e dos artefatos criptográficos persistentes em `/opt/vaultwarden/data`.
2. **Backup da VM (Infraestrutura):** Snapshot completo do estado baseline da VM no Proxmox VE / PBS (*Proxmox Backup Server*).
3. **Backup Off-site (Nuvem - OCI via Restic):** Envio criptografado client-side para bucket privado no Oracle Cloud Infrastructure via Restic, com integridade e restauração real validadas a partir da nuvem.

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
- [x] O agendamento diário e o disparo automático do timer no horário programado (03:00) foram observados e validados em regime real de produção na madrugada de 23/09/2026, com a criação bem-sucedida do backup `vaultwarden_20260923_030040.tar.gz`.

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

### 3.6. Monitoramento e Limitações Conhecidas

- **Ausência de Servidor de Monitoramento:** Atualmente, **não** existe servidor Zabbix em produção para este ambiente. Por essa razão, nenhum template, script ou integração com Zabbix foi criada nesta etapa.
- **Forma de Verificação Atual:** O acompanhamento operacional é realizado localmente através:
  1. Dos logs do serviço systemd (`journalctl -u vaultwarden-backup.service`).
  2. Do estado e agendamento do timer (`systemctl status vaultwarden-backup.timer` e `systemctl list-timers`).
  3. Da validação dos arquivos e hashes gerados em `/var/backups/vaultwarden/`.
- **Limitação Conhecida e Evolução:** A ausência de alertas ativos e centralizados em caso de falha é uma limitação conhecida do cenário atual. A implementação de monitoramento centralizado via Zabbix é classificada como uma melhoria futura / evolução e não é requisito para o funcionamento do backup local já concluído.

---

## 4. Backup Off-site em Nuvem (OCI Object Storage via Restic)

O backup off-site provê salvaguarda independente fora da infraestrutura física local, garantindo proteção contra desastres físicos, perda do Proxmox ou falha ampla no ambiente local.

### 4.1. Configuração do Repositório OCI

- **Região:** `sa-saopaulo-1`
- **Namespace:** `groqo9fbzuaz`
- **Compartment:** `Backups`
- **Bucket:** `vaultwarden-offsite`
- **Visibilidade:** Bucket estritamente privado (sem acesso público)
- **Interface de Acesso:** API compatível com S3
- **ID do Repositório Restic:** `7bbbe221`
- **Criptografia Client-side:** Todos os dados são criptografados pelo Restic antes do envio pela rede, impedindo qualquer acesso não autorizado ao conteúdo do cofre em nuvem.
- **Sigilo de Credenciais:** As credenciais OCI (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) e a senha mestra do repositório (`RESTIC_PASSWORD`) são mantidas estritamente isoladas no arquivo `/etc/vaultwarden-backup/oci.env` e **NUNCA** são expostas no Git.

### 4.2. Script de Automação Off-site (`vaultwarden-offsite-backup`)

O script `/usr/local/sbin/vaultwarden-offsite-backup` (versionado no repositório em `backup/vaultwarden-offsite-backup`) executa as seguintes etapas:
1. **Controle de Concorrência (Lock):** Adquire lock exclusivo em `/run/lock/vaultwarden-offsite-backup.lock`.
2. **Carregamento de Ambiente:** Carrega variáveis a partir de `/etc/vaultwarden-backup/oci.env`.
3. **Verificação do Backup Local Recente:** Busca em `/var/backups/vaultwarden/` por backups gerados nos últimos 120 minutos (`find -mmin -120`), realizando até 60 tentativas (intervalo de 30s) para garantir sincronia com a finalização do backup local.
4. **Validação de Checksum Local:** Executa `sha256sum -c "${LATEST}.sha256"` antes de iniciar o upload.
5. **Upload via Restic:** Executa `restic -o s3.bucket-lookup=path backup /var/backups/vaultwarden`.
6. **Aplicação da Retenção Remota (10 Dias):** Aplica a política de retenção remota através de:
   ```bash
   restic -o s3.bucket-lookup=path forget --keep-within 10d --prune
   ```
7. **Auditoria de Integridade:** Valida a consistência criptográfica com:
   ```bash
   restic -o s3.bucket-lookup=path check
   ```

### 4.3. Unidades Systemd do Backup Off-site

As definições estão versionadas no repositório em `backup/` e instaladas em `/etc/systemd/system/`:

- **Service Unit (`/etc/systemd/system/vaultwarden-offsite-backup.service`):**
  - `Type=oneshot`
  - `After=vaultwarden-backup.service`
  - `ExecStart=/usr/local/sbin/vaultwarden-offsite-backup`
  - `Environment=HOME=/root`: Garante definição de diretório home para o Restic, eliminando o aviso de diretório de cache inexistente durante execuções pelo systemd.
  - `UMask=0077`, `NoNewPrivileges=true`, `TimeoutStartSec=60min`.

- **Timer Unit (`/etc/systemd/system/vaultwarden-offsite-backup.timer`):**
  - `OnCalendar=*-*-* 03:30:00`: Programado diariamente às 03:30 (30 minutos após o backup local das 03:00).
  - `Persistent=true`: Disparo garantido caso o host esteja indisponível no horário agendado.
  - `WantedBy=timers.target`: Habilitado na inicialização do sistema.

### 4.4. Estado Validado da Automação Off-site

- [x] O repositório Restic foi inicializado com sucesso e o check inicial executou sem erros.
- [x] Teste inicial de upload, restore e expurgo (`forget --prune`) concluído sem erros.
- [x] A execução MANUAL do serviço `vaultwarden-offsite-backup.service` foi validada com sucesso via `sudo systemctl start vaultwarden-offsite-backup.service`.
- [x] O serviço localizou o backup local recente em `/var/backups/vaultwarden/`.
- [x] O checksum SHA-256 do backup local foi validado com sucesso.
- [x] Snapshot Restic real criado no bucket privado do OCI.
- [x] `restic check` concluído com sucesso (`no errors were found`).
- [x] Retenção remota de 10 dias (`forget --keep-within 10d --prune`) executada com sucesso.
- [x] Aviso de cache do Restic corrigido com a inclusão de `Environment=HOME=/root` na service unit.
- [x] A restauração de dados reais a partir do OCI já havia sido validada previamente em ambiente temporário isolado (`/tmp/restic-vaultwarden-restore`), com validação do SHA-256 e arquivos `db.sqlite3` e `rsa_key.pem`.
- [ ] *Ressalva importante sobre o timer off-site:* A execução automática do timer `vaultwarden-offsite-backup.timer` às 03:30 ainda **NÃO** foi observada em regime de produção. Portanto, o timer está documentado como "configurado e validado manualmente", mas **NÃO** como "execução automática validada". A execução automática do backup local das 03:00 já foi observada e permanece validada.

---

## 5. Comandos Reais de Validação

Para auditar os backups locais existentes, o agendador e o repositório OCI:

### 5.1. Listar Backups Locais Existentes
```bash
ls -lh /var/backups/vaultwarden/
```

### 5.2. Validar Checksum SHA-256 Local
```bash
cd /var/backups/vaultwarden/
sha256sum -c vaultwarden_20260923_181123.tar.gz.sha256
```
*Resultado esperado:* `vaultwarden_20260923_181123.tar.gz: OK`

### 5.3. Inspecionar Conteúdo do TAR sem Extrair
```bash
tar -tzvf /var/backups/vaultwarden/vaultwarden_20260923_181123.tar.gz
```
*Arquivos esperados no arquivo:*
- `./db.sqlite3`
- `./rsa_key.pem`
- `./icon_cache/`

### 5.4. Verificar Status do Timer Local e Próximo Disparo
```bash
sudo systemctl status vaultwarden-backup.timer
sudo systemctl list-timers | grep vaultwarden-backup
```

### 5.5. Consultar Logs da Última Execução do Serviço Local
```bash
sudo journalctl -u vaultwarden-backup.service --no-pager -n 50
```

### 5.6. Verificar Status do Timer Off-site e Logs do Serviço
```bash
sudo systemctl status vaultwarden-offsite-backup.timer
sudo systemctl list-timers | grep vaultwarden-offsite
sudo journalctl -u vaultwarden-offsite-backup.service --no-pager -n 50
```

### 5.7. Disparo Manual do Serviço Off-site
```bash
sudo systemctl start vaultwarden-offsite-backup.service
```

### 5.8. Consultar Snapshots e Integridade no OCI (Restic)
```bash
# Listar snapshots no bucket OCI
restic snapshots

# Verificar integridade estrutural e criptográfica do repositório OCI
restic check
```

---

## 6. O que foi EFETIVAMENTE TESTADO vs O que é PLANEJADO

### ✅ Estado Atual Testado e Validado em Produção
- [x] Backup local diário via systemd (execução noturna observada em 23/09/2026 com `vaultwarden_20260923_030040.tar.gz`).
- [x] Retenção local de 10 dias (`RETENTION_DAYS=10`, expurgo automático de pares `.tar.gz` e `.sha256` pós-backup).
- [x] Integridade SHA-256 local (`sha256sum -c` e `tar -tzf`).
- [x] Restore local (validação funcional em ambiente temporário isolado sem impacto na produção).
- [x] Snapshot de baseline da VM no Proxmox VE / PBS validado com sucesso.
- [x] Restic repository OCI (inicializado com sucesso em bucket privado `vaultwarden-offsite`, ID `7bbbe221`).
- [x] Upload real para OCI via Restic (`vaultwarden-offsite-backup` com validação de checksum prévia).
- [x] `restic check` (verificações concluídas com `no errors were found`).
- [x] Retenção remota de 10 dias no OCI (`restic forget --keep-within 10d --prune` executado e validado com sucesso).
- [x] Automação off-site via systemd (`vaultwarden-offsite-backup.service` e `vaultwarden-offsite-backup.timer` diariamente às 03:30, execução manual do service validada com sucesso; cache corrigido com `Environment=HOME=/root`).
- [x] Restore de backup real a partir do OCI (recuperação do snapshot `d5f61547` para `/tmp/restic-vaultwarden-restore` com 17 itens, sem substituir ou alterar a produção).
- [x] Validação do SHA-256 do backup recuperado (`fc40c0ae6da319aa89283632fb0aea58ab4f2ce286e578beb6dc167631b1ce40` idêntico ao `.sha256` armazenado).

### ⚠️ Melhorias Futuras / Evolução (Planejado)
- [ ] **Observação da execução automática do timer off-site:** Registro da primeira execução real noturna disparada automaticamente pelo timer às 03:30 (timer configurado e com execução manual validada).
- [ ] **Monitoramento centralizado:** Configuração de monitoramento centralizado e alertas dos timers de backup e métricas de integridade (melhoria futura; atualmente não existe servidor Zabbix no ambiente).
- [ ] **Teste completo de disaster recovery:** Simulação ponta a ponta de perda total da VM e reconstrução em outro hypervisor.
- [ ] **Verificação periódica de restore:** Formalização e execução de cronograma de rotinas regulares de testes de recuperação.
