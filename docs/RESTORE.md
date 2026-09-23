# Procedimentos de Restauração (Restore) — Vaultwarden

Este documento descreve o procedimento de restauração de desastres, o teste prático de restore que já foi realizado e validado no ambiente, e as diretrizes para recuperação de dados.

---

## 1. Princípio Fundamental de Recuperação

Um arquivo `.tar.gz` existente em disco **não** garante que o backup seja recuperável. A cadeia de validação completa deve ser:

```text
Backup
  ↓
Arquivo compactado (.tar.gz)
  ↓
Checksum SHA-256 válido
  ↓
Leitura da estrutura TAR íntegra
  ↓
Extração dos dados em ambiente isolado
  ↓
Inicialização do container Vaultwarden
  ↓
Teste de autenticação e integridade do cofre
```

---

## 2. Teste Real de Restauração Já Executado

Teste real de restauração realizado em ambiente temporário isolado, utilizando uma cópia do backup. O Vaultwarden foi iniciado separadamente e o acesso ao Web Vault foi validado com sucesso. A instância produtiva não foi substituída nem alterada durante o teste.

### 2.1. Metodologia do Teste Executado
1. **Seleção do Artefato:** Foi selecionado um arquivo de backup recém-gerado em `/var/backups/vaultwarden/`.
2. **Validação Prévia do Hash:**
   ```bash
   sha256sum -c vaultwarden_20260922_155527.tar.gz.sha256
   ```
3. **Criação de Diretório Temporário Isolado:**
   Criou-se uma pasta de teste separada (fora de `/opt/vaultwarden/data`) para não interferir nos dados da instância produtiva:
   ```bash
   mkdir -p /tmp/vw-restore-test/data
   ```
4. **Extração dos Dados:**
   ```bash
   tar -xzf /var/backups/vaultwarden/vaultwarden_20260922_155527.tar.gz -C /tmp/vw-restore-test/data
   ```
5. **Verificação dos Arquivos Extraídos:**
   Confirmada a presença de:
   - `db.sqlite3`
   - `rsa_key.pem`
   - `icon_cache/`
6. **Execução de Container de Teste Isolado:**
   Foi lançado um container temporário do Vaultwarden vinculado à pasta `/tmp/vw-restore-test/data`, utilizando uma porta temporária não conflitante (`127.0.0.1:8888`), garantindo que a instância principal não sofresse qualquer interferência:
   ```bash
   docker run -d --name vw-restore-test \
     -v /tmp/vw-restore-test/data:/data \
     -p 127.0.0.1:8888:80 \
     vaultwarden/server@sha256:4ecafc9049c7d878c7717d1ce4f9059d706758c78b8fa42e5ead21f4b2dfc770
   ```
7. **Validação Funcional:**
   - O container inicializou com sucesso e os logs não apresentaram erros de integridade na base SQLite.
   - A interface web respondeu via HTTP na porta de teste.
   - A tela de login foi exibida e os dados do usuário foram reconhecidos pela base restaurada.
8. **Desmontagem e Limpeza Completa:**
   Após a validação, o container de teste e o diretório temporário foram imediatamente removidos para não deixar resíduos no sistema:
   ```bash
   docker stop vw-restore-test && docker rm vw-restore-test
   rm -rf /tmp/vw-restore-test
   ```

*Conclusão do Teste:* O procedimento comprovou a capacidade de recuperação a partir dos arquivos de backup locais gerados pelo script, realizado de forma segura em ambiente temporário isolado sem qualquer alteração na instância produtiva.

---

## 3. Teste Real de Restauração a partir do OCI (Restic) Já Executado

Teste real de restauração a partir do repositório em nuvem no OCI executado em ambiente temporário isolado. A integridade dos dados e o hash SHA-256 foram validados com sucesso. A instância produtiva não foi substituída nem alterada durante o teste.

### 3.1. Metodologia do Teste a partir do OCI
1. **Identificação do Snapshot Alvo:**
   Identificado o snapshot de produção enviado ao repositório OCI:
   - Snapshot ID: `d5f61547`
   - Repositório Restic ID: `7bbbe221` (Bucket privado `vaultwarden-offsite`, namespace `groqo9fbzuaz`, região `sa-saopaulo-1`).
2. **Ambiente Temporário Isolado:**
   Criado diretório de restauração exclusivo para o teste:
   ```bash
   mkdir -p /tmp/restic-vaultwarden-restore
   ```
3. **Execução da Restauração via Restic:**
   ```bash
   restic restore d5f61547 --target /tmp/restic-vaultwarden-restore
   ```
4. **Verificação dos Artefatos Recuperados:**
   - 17 arquivos/diretórios foram extraídos com sucesso.
   - O arquivo compactado `vaultwarden_20260923_181123.tar.gz` foi recuperado integralmente.
5. **Conferência Criptográfica do SHA-256:**
   - Hash SHA-256 calculado no arquivo restaurado:
     `fc40c0ae6da319aa89283632fb0aea58ab4f2ce286e578beb6dc167631b1ce40`
   - O valor coincidiu com precisão absoluta com o arquivo `.sha256` armazenado.
6. **Inspeção Estrutural do Conteúdo TAR:**
   ```bash
   tar -tzvf /tmp/restic-vaultwarden-restore/.../vaultwarden_20260923_181123.tar.gz
   ```
   Confirmada a integridade e presença dos componentes vitais:
   - `./db.sqlite3`
   - `./rsa_key.pem`
7. **Limpeza Completa:**
   O diretório `/tmp/restic-vaultwarden-restore` foi totalmente removido após a validação. A instância produtiva do Vaultwarden não sofreu qualquer parada ou modificação.

*Conclusão do Teste:* Comprovada a capacidade de recuperação de desastres a partir do repositório remoto no OCI, garantindo independência completa da infraestrutura local física.

---

## 4. Procedimento de Restauração em Caso de Desastre na VM (Backup Local)

Se o container ou o banco de produção forem corrompidos ou excluídos:

### Passo 1: Parar o Serviço de Produção
```bash
sudo docker compose -f /opt/vaultwarden/docker-compose.yaml down
```

### Passo 2: Fazer Cópia de Segurança do Estado Atual (se existir)
```bash
sudo mv /opt/vaultwarden/data /opt/vaultwarden/data_corrupted_$(date +%Y%m%d_%H%M%S)
sudo mkdir -p /opt/vaultwarden/data
```

### Passo 3: Escolher e Validar o Backup Alvo
```bash
cd /var/backups/vaultwarden/
sha256sum -c vaultwarden_YYYYMMDD_HHMMSS.tar.gz.sha256
```

### Passo 4: Extrair os Dados no Diretório de Produção
```bash
sudo tar -xzf /var/backups/vaultwarden/vaultwarden_YYYYMMDD_HHMMSS.tar.gz -C /opt/vaultwarden/data
```

### Passo 5: Ajustar Permissões (se necessário)
Garantir que os arquivos pertençam ao usuário de execução e não estejam com permissões abertas indevidas:
```bash
sudo chown -R root:root /opt/vaultwarden/data
sudo chmod 700 /opt/vaultwarden/data
```

### Passo 6: Iniciar o Vaultwarden
```bash
sudo docker compose -f /opt/vaultwarden/docker-compose.yaml up -d
```

### Passo 7: Validar Inicialização e Acesso
```bash
sudo docker inspect vaultwarden --format 'Status={{.State.Status}} Health={{.State.Health.Status}}'
sudo docker logs --tail=50 vaultwarden
```
Acessar `https://vault.rufonex.com.br`, realizar login com TOTP e sincronizar os clientes.

---

## 5. Procedimento de Restauração Off-site (Nuvem OCI) em Caso de Perda Total

Em cenário de desastre catastrófico com perda física do hypervisor Proxmox ou destruição da VM e de seus backups locais:

### Passo 1: Provisionar Novo Host Debian e Instalar Ferramentas
Provisionar novo host Debian, instalar Docker Engine, Compose e Restic.

### Passo 2: Configurar Acesso Seguro ao Repositório OCI
Exportar em sessão shell segura as credenciais de acesso ao bucket OCI compatível com S3 e a senha do Restic:
```bash
export AWS_ACCESS_KEY_ID="<oci-access-key-id>"
export AWS_SECRET_ACCESS_KEY="<oci-secret-access-key>"
export RESTIC_REPOSITORY="s3:https://groqo9fbzuaz.compat.objectstorage.sa-saopaulo-1.oraclecloud.com/vaultwarden-offsite"
export RESTIC_PASSWORD="<senha-do-repositorio-restic>"
```
*(Atenção: NUNCA salvar credenciais em arquivos versionados pelo Git).*

### Passo 3: Localizar Snapshots e Restaurar o Arquivo Alvo
```bash
# Listar snapshots disponíveis
restic snapshots

# Restaurar o snapshot desejado em diretório de staging
mkdir -p /tmp/oci-restore
restic restore <snapshot_id> --target /tmp/oci-restore
```

### Passo 4: Validar Integridade SHA-256 e Extrair Dados
```bash
cd /tmp/oci-restore/var/backups/vaultwarden/
sha256sum -c vaultwarden_YYYYMMDD_HHMMSS.tar.gz.sha256

# Extrair sobre o diretório de dados oficial
sudo mkdir -p /opt/vaultwarden/data
sudo tar -xzf vaultwarden_YYYYMMDD_HHMMSS.tar.gz -C /opt/vaultwarden/data
sudo chown -R root:root /opt/vaultwarden/data
sudo chmod 700 /opt/vaultwarden/data
```

### Passo 5: Inicializar o Container e Reestabelecer o Túnel
Recriar a stack Compose `/opt/vaultwarden/docker-compose.yaml` com o digest imutável, iniciar a aplicação e reconectar o Cloudflare Tunnel.

---

## 6. Camadas de Restauração de Desastres

| Camada | Cenário de Incidente | Procedimento de Recuperação | Status |
| :--- | :--- | :--- | :--- |
| **Camada 1: Dados da Aplicação** | Corrupção de banco SQLite ou deleção acidental de `/data` | Restaurar arquivo `.tar.gz` local sobre `/opt/vaultwarden/data` | ✅ Testado e Validado |
| **Camada 2: VM Inteira** | Falha de SO Debian, kernel panic ou crash do disco virtual | Restaurar snapshot completo da VM no Proxmox VE via PBS | ✅ Testado e Validado |
| **Camada 3: Perda do Site / Hypervisor** | Destruição física do Proxmox / perda total da residência | Reconstruir host Debian e restaurar backup criptografado do OCI via Restic | ✅ Restore do OCI testado e validado em ambiente isolado (Automação do envio implementada e validada manualmente; DR ponta a ponta planejado) |
