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
14. **Conclusão:** Emite mensagem de conclusão bem-sucedida.

---

## 3. Comandos Reais de Validação

Para auditar manualmente um backup existente em `/var/backups/vaultwarden/`:

### 3.1. Listar Backups Existentes
```bash
ls -lh /var/backups/vaultwarden/
```

### 3.2. Validar Checksum SHA-256
```bash
cd /var/backups/vaultwarden/
sha256sum -c vaultwarden_20260922_155527.tar.gz.sha256
```
*Resultado esperado:* `vaultwarden_20260922_155527.tar.gz: OK`

### 3.3. Inspecionar Conteúdo do TAR sem Extrair
```bash
tar -tzvf /var/backups/vaultwarden/vaultwarden_20260922_155527.tar.gz
```
*Arquivos esperados no arquivo:*
- `db.sqlite3`
- `rsa_key.pem`
- `icon_cache/`

---

## 4. O que foi EFETIVAMENTE TESTADO vs O que é PLANEJADO

### ✅ Estado Atual Testado e Validado em Produção
- [x] Execução manual bem-sucedida do script `/usr/local/sbin/vaultwarden-backup`.
- [x] Criação de arquivo `.tar.gz` consistente com dados persistentes.
- [x] Validação estrutural do arquivo via `tar -tzf`.
- [x] Geração e verificação íntegra do hash via `sha256sum -c`.
- [x] Teste prático de restauração dos dados em container isolado (ver [`RESTORE.md`](file:///home/juniorrufo/projetos/vaultwarden/docs/RESTORE.md)).
- [x] Backup de baseline da VM no Proxmox VE / PBS validado com sucesso.

### ⚠️ Itens Pendentes (Não documentar como implementados)
- [ ] **Automação via Systemd:** Criação e ativação do `vaultwarden-backup.service` e `vaultwarden-backup.timer` para execução diária automática.
- [ ] **Política de Retenção Local:** Automação de descarte de backups antigos (meta: retenção de 14 dias diários).
- [ ] **Monitoramento via Zabbix:** Coleta e alertas do status de sucesso/falha da rotina de backup.
- [ ] **Backup Off-site em Nuvem (OCI):** Criação de bucket privado no Oracle Cloud Infrastructure, configuração do Restic com encriptação client-side e chave dedicada com privilégios mínimos.
- [ ] **Teste de Restore a partir da Nuvem:** Validação prática de recuperação direta do OCI Object Storage.
