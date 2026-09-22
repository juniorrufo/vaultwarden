# Integração de Firewall e Docker (DOCKER-USER) — Vaultwarden

Este documento explica tecnicamente a arquitetura de firewall implementada no host Debian 13 (`192.168.15.200`), detalhando como o tráfego destinado ao container Docker do Vaultwarden é estritamente filtrado.

---

## 1. O Problema: O Desvio de Firewall Nativo do Docker

Por padrão, quando o Docker publica uma porta em um container (ex.: `-p 192.168.15.200:8080:80`):

1. O Docker adiciona regras na tabela `nat` (cadeia `PREROUTING`) que fazem DNAT (*Destination Network Address Translation*), reescrevendo o destino do pacote para o IP interno da bridge (`vaultwarden_net`).
2. O tráfego não passa pela cadeia `INPUT` do Linux, pois é considerado tráfego roteado/encaminhado.
3. O pacote é enviado diretamente para a cadeia `FORWARD` e processado pelas cadeias internas do Docker (`DOCKER`, `DOCKER-FORWARD`).

**Consequência:** Regras convencionais de firewall configuradas na cadeia `INPUT` do host (como uma política `INPUT DROP`) são completamente ignoradas para qualquer porta publicada pelo Docker. Se nenhuma regra adicional for criada, qualquer dispositivo na LAN conseguiria alcançar a porta `:8080`.

---

## 2. A Solução Arquitetural: DOCKER-USER e VW-DOCKER

Para permitir que administradores controlem o acesso a containers sem terem suas regras sobrescritas pelo daemon do Docker, o Docker fornece a cadeia reservada:
```text
DOCKER-USER
```

Todas as regras na cadeia `DOCKER-USER` são processadas antes de qualquer regra interna do Docker na tabela `filter` (tabela de encaminhamento).

### 2.1. Criação da Cadeia Dedicada VW-DOCKER
Para manter a organização e permitir manutenção isolada da política do Vaultwarden, a regra raiz delega o tráfego para uma cadeia especializada:

```bash
-A DOCKER-USER -j VW-DOCKER
```

### 2.2. O Desafio do Conntrack e Destino Original (--ctorigdst)
Quando o pacote atinge a cadeia `FORWARD` / `DOCKER-USER`, o Docker **já realizou o DNAT** no `PREROUTING`. O endereço IP de destino do pacote não é mais `192.168.15.200`, e sim o IP da interface virtual do container (ex.: `172.x.x.x`).

Por essa razão, regras que filtram com base em `-d 192.168.15.200` falhariam. A solução correta e robusta é utilizar o módulo `conntrack` do kernel Linux com `--ctorigdst` (*connection tracking original destination*) e `--ctorigdstport`:

```text
-m conntrack --ctorigdst 192.168.15.200 --ctorigdstport 8080
```

---

## 3. Conjunto Efetivo de Regras

As regras ativas na cadeia `VW-DOCKER` são:

```text
# 1. Permite tráfego de conexões já estabelecidas ou relacionadas
-A VW-DOCKER -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# 2. Permite requisições vindas EXCLUSIVAMENTE do Cloudflared LXC (192.168.15.253) para a porta 8080
-A VW-DOCKER -s 192.168.15.253/32 -p tcp -m conntrack --ctorigdst 192.168.15.200 --ctorigdstport 8080 -j ACCEPT

# 3. Descarta silenciosamente qualquer outra origem na LAN tentando acessar a porta 8080
-A VW-DOCKER -p tcp -m conntrack --ctorigdst 192.168.15.200 --ctorigdstport 8080 -j DROP

# 4. Retorna o controle para as demais cadeias do Docker para outros tráfegos não relacionados
-A VW-DOCKER -j RETURN
```

---

## 4. Persistência e Sobrevivência a Reinicializações

O daemon do Docker recria suas cadeias padrão durante sua inicialização, o que poderia apagar ou bagunçar regras manuais se não houvesse persistência configurada.

A persistência no ambiente foi resolvida através de duas camadas:

1. **Script de Aplicação das Regras:**
   ```text
   /usr/local/sbin/vaultwarden-docker-firewall
   ```
2. **Serviço Systemd Dedicado:**
   ```text
   /etc/systemd/system/vaultwarden-docker-firewall.service
   ```
   Configurado com dependência para iniciar logo após o serviço `docker.service`.

### Testes de Sobrevivência Realizados:
- **Reinício do Docker:** As regras foram reaplicadas com sucesso após `systemctl restart docker`.
- **Reboot da VM:** Após o reinício completo do Debian 13, o serviço subiu e as regras foram validadas no kernel.

---

## 5. Validação Prática em Produção

O isolamento foi rigorosamente testado em ambiente de produção com os seguintes resultados documentados:

### Teste 1: Acesso a partir do LXC Cloudflared (Origem Autorizada)
Executado a partir de `192.168.15.253`:
```bash
curl -i http://192.168.15.200:8080/alive
```
*Resultado:* **HTTP/1.1 200 OK** (Tráfego aceito pela regra 2).

### Teste 2: Acesso a partir de outra máquina na LAN (Origem Não Autorizada)
Executado a partir de outro host na rede `192.168.15.0/24`:
```bash
curl -i http://192.168.15.200:8080/alive
```
*Resultado:* **Connection timed out** (Tráfego descartado silenciosamente pela regra 3).

---

## 6. Por que iptables-nft e não nftables direto?

- O Debian 13 utiliza por padrão o backend `nftables` no kernel.
- O Docker Engine moderno interage nativamente com a interface de compatibilidade `iptables-nft` (v1.8.11).
- Tentar gerenciar regras diretamente através de um arquivo `/etc/nftables.conf` enquanto o Docker cria dinamicamente tabelas iptables gera tabelas paralelas, conflitos de precedência de chains e comportamentos imprevisíveis.
- Portanto, o serviço `nftables.service` do Debian foi intencionalmente desabilitado e mascarado (`systemctl mask nftables.service`), padronizando o gerenciamento no `iptables-nft` mantido pelo `netfilter-persistent` e o serviço customizado.
