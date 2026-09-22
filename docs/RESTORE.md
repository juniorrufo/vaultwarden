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

## 3. Procedimento de Restauração em Caso de Desastre na VM

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

## 4. Camadas de Restauração de Desastres

| Camada | Cenário de Incidente | Procedimento de Recuperação | Status |
| :--- | :--- | :--- | :--- |
| **Camada 1: Dados da Aplicação** | Corrupção de banco SQLite ou deleção acidental de `/data` | Restaurar arquivo `.tar.gz` local sobre `/opt/vaultwarden/data` | ✅ Testado e Validado |
| **Camada 2: VM Inteira** | Falha de SO Debian, kernel panic ou crash do disco virtual | Restaurar snapshot completo da VM no Proxmox VE via PBS | ✅ Testado e Validado |
| **Camada 3: Perda do Site / Hypervisor** | Destruição física do Proxmox / perda total da residência | Reconstruir VM Debian em outro ambiente e restaurar cópia criptografada da OCI | ⚠️ Planejado (pendente implementação OCI) |
