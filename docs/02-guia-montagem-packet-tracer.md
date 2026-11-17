# Guia de montagem no Cisco Packet Tracer — BioGen (A10)

> **A rede já está pronta no arquivo `BioGen-A10.pkt`** (todos os equipamentos, cabos, configurações e serviços). Basta abrir, esperar os links ficarem verdes e gerar a chave do SSH (`crypto key generate rsa` → 1024) no R-BORDA, SW-CORE, SW-A1 e SW-A2. Depois, pule direto para o **Passo 6 (testes)**.
> Os passos 1 a 5 abaixo servem para entender como a rede foi montada ou para remontar do zero.

Endereços e portas estão em [01-plano-de-enderecamento.md](01-plano-de-enderecamento.md); os comandos de cada equipamento estão em [configs/](../configs/).

## Passo 1 — Colocar os equipamentos

| Qtde | Modelo no PT | Nomes (Config → Display Name) |
|---|---|---|
| 2 | Router **2911** | R-BORDA, ISP |
| 5 | Switch **2960-24TT** | SW-CORE, SW-A1, SW-A2, SW-DMZ, SW-INTERNET |
| 4 | Server-PT | SRV-INTERNO, SRV-LIMS, SRV-WEB-DMZ, SRV-INTERNET |
| 7 | PC-PT | PC-PESQ-01, PC-PESQ-02, PC-ADM-01, SEQ-01, SEQ-02, PC-GERENCIA, PC-INTERNET |

Dica: use a ferramenta de **Nota (tecla N)** e **retângulos coloridos** para marcar cada VLAN, as sub-redes e a DMZ — o template exige "anotações visíveis de VLANs, links e blocos IP".

## Passo 2 — Cabos (exatamente nestas portas)

| De | Para | Cabo |
|---|---|---|
| R-BORDA G0/0 | SW-CORE G0/1 | Direto |
| R-BORDA G0/1 | SW-DMZ G0/1 | Direto |
| R-BORDA G0/2 | ISP G0/0 | Cruzado |
| ISP G0/1 | SW-INTERNET Fa0/1 | Direto |
| SW-CORE Fa0/23 | SW-A1 G0/1 | Cruzado |
| SW-CORE Fa0/24 | SW-A2 G0/1 | Cruzado |
| SW-A1 G0/2 | SW-A2 G0/2 | Cruzado |
| SRV-INTERNO / SRV-LIMS | SW-CORE Fa0/1 / Fa0/2 | Direto |
| PC-GERENCIA | SW-CORE Fa0/6 | Direto |
| PC-PESQ-01 / PC-PESQ-02 / PC-ADM-01 | SW-A1 Fa0/1 / Fa0/2 / Fa0/13 | Direto |
| SEQ-01 / SEQ-02 | SW-A2 Fa0/1 / Fa0/2 | Direto |
| SRV-WEB-DMZ | SW-DMZ Fa0/1 | Direto |
| PC-INTERNET / SRV-INTERNET | SW-INTERNET Fa0/2 / Fa0/3 | Direto |

## Passo 3 — Colar as configurações

Em cada equipamento: aba **CLI** → responder `no` ao "initial configuration dialog" → `enable` → colar o conteúdo do arquivo correspondente de `configs/`.

Ordem sugerida: SW-CORE, SW-A1, SW-A2, SW-DMZ → R-BORDA → ISP. O SW-INTERNET não precisa de configuração.

> Se o `crypto key generate rsa general-keys modulus 1024` reclamar, digite apenas `crypto key generate rsa` e responda `1024`.

## Passo 4 — Configurar hosts (aba Desktop → IP Configuration)

| Host | IPv4 | IPv6 |
|---|---|---|
| PC-PESQ-01/02, PC-ADM-01 | **DHCP** | **Automatic** (SLAAC) |
| SEQ-01 | 10.99.1.130 / 255.255.255.192, GW 10.99.1.129, DNS vazio | 2001:db8:b10:20::10/64, GW 2001:db8:b10:20::1 |
| SEQ-02 | 10.99.1.131 / 255.255.255.192, GW 10.99.1.129 | 2001:db8:b10:20::11/64, GW 2001:db8:b10:20::1 |
| SRV-INTERNO | 10.99.1.194 / 255.255.255.224, GW 10.99.1.193, DNS 10.99.1.194 | 2001:db8:b10:40::10/64, GW ::1 do prefixo |
| SRV-LIMS | 10.99.1.195 / 255.255.255.224, GW 10.99.1.193, DNS 10.99.1.194 | 2001:db8:b10:40::11/64 |
| PC-GERENCIA | 10.99.1.250 / 255.255.255.240, GW 10.99.1.241, DNS 10.99.1.194 | 2001:db8:b10:99::50/64 |
| SRV-WEB-DMZ | 10.99.1.226 / 255.255.255.240, GW 10.99.1.225 | 2001:db8:b10:50::10/64 |
| PC-INTERNET | 203.0.113.10 / 255.255.255.0, GW 203.0.113.1 | 2001:db8:ffff::10/64 |
| SRV-INTERNET | 203.0.113.100 / 255.255.255.0, GW 203.0.113.1 | 2001:db8:ffff::100/64 |

## Passo 5 — Serviços nos servidores (aba Services)

Em **todos** os servidores, deixe **DHCP = Off** (quem distribui IP é o R-BORDA).

**SRV-INTERNO**
- **DNS**: On. Registros tipo A:
  - `www.biogen.local` → 10.99.1.226
  - `biogen.local` → 10.99.1.226
  - `lims.biogen.local` → 10.99.1.195
- **NTP**: On. **SYSLOG**: On. HTTP/FTP: pode desligar.

**SRV-LIMS**
- **HTTP**: On. Editar `index.html` → título "BioGen LIMS — Repositório de Dados Genômicos".
- **FTP**: On. Criar usuário `sequenciador` / senha `seq123` com Write, Read, List.

**SRV-WEB-DMZ**
- **HTTP** e **HTTPS**: On. Editar `index.html` → "BioGen — Divulgação Científica" (publicações, notícias). DNS/FTP: Off.

**SRV-INTERNET**
- **HTTP**: On (`index.html` → "Site externo / Internet").

## Passo 6 — Testes e prints para o relatório

Aguarde ~30–60 s após montar (STP convergir / DHCP). Use **Fast Forward Time** se necessário.

| # | Onde | Comando / ação | Resultado esperado | Seção do relatório |
|---|---|---|---|---|
| 1 | PC-PESQ-01 | `ipconfig /all` | IPv4 10.99.0.11 via DHCP + IPv6 2001:db8:b10:10:xxxx (EUI-64) | 4.2 / 6.1 |
| 2 | PC-PESQ-01 | `ping 10.99.1.11` (IP do PC-ADM-01 — confira com `ipconfig` nele) | Sucesso (inter-VLAN) | 5.1 |
| 3 | PC-PESQ-01 | `nslookup www.biogen.local` | 10.99.1.226 | **6.1 (print)** |
| 4 | PC-PESQ-01 | Web Browser → `http://www.biogen.local` | Página da DMZ | **6.1 (print)** |
| 5 | SEQ-01 | `ping 10.99.1.195` | **Destination host unreachable** | **5.3 (print)** |
| 6 | SEQ-01 | `ping 203.0.113.100` | Destination host unreachable (sem saída p/ Internet) | 5.3 |
| 7 | SEQ-01 | Web Browser → `http://10.99.1.195` | Página do LIMS abre | **5.3 (print)** |
| 8 | SEQ-01 | `ftp 10.99.1.195` (user `sequenciador`) | Login OK — upload liberado | 5.3 / 6.1 |
| 9 | SEQ-01 | `ping 2001:db8:b10:40::11` | Falha (zero-trust também em IPv6) | 4.2 / 5.3 |
| 10 | PC-INTERNET | Browser → `http://10.99.1.226` | Site da DMZ abre | 6.1 |
| 11 | PC-INTERNET | `ping 10.99.0.11` | Falha (Internet não entra na rede interna) | 5.2 |
| 12 | SRV-WEB-DMZ | `ping 10.99.1.194` | Falha (DMZ isolada) | 5.2 |
| 13 | PC-PESQ-01 | Browser → `http://203.0.113.100` | Site externo abre (saída liberada para usuários) | 5.2 |
| 14 | SW-A2 | `show spanning-tree vlan 20` | Gi0/2 = **Altn BLK** | **3.3** |
| 15 | SW-CORE | `show spanning-tree vlan 20` | "This bridge is the root" | **3.3** |
| 16 | PC-GERENCIA | `ssh -l admin 10.99.1.242` | Acesso OK | 3.2 / 7 |
| 17 | PC-PESQ-01 | `ssh -l admin 10.99.1.242` | Conexão recusada (access-class) | 3.2 / 7 |
| 18 | R-BORDA | `show access-lists` | Contadores de "matches" nas linhas deny | 5.2 |
| 19 | SW-A2 | `show port-security` | Fa0/1-10 Max 1, Shutdown | 3.2 |
| 20 | R-BORDA | `show ip interface brief` / `show ipv6 interface brief` | Tudo up/up | 4 |

**Teste extra de port-security (opcional, bom para nota):** desconecte o SEQ-01 e conecte outro PC na SW-A2 Fa0/1 → a porta vai para *err-disabled* (luz vermelha). Para reativar: `interface fa0/1` → `shutdown` → `no shutdown` (após remover o MAC sticky com `clear port-security sticky interface fa0/1`).

## Problemas comuns

- **PC não pega DHCP:** verifique se o trunk SW-CORE G0/1 está up e se a VLAN existe em todos os switches (`show vlan brief`, `show interfaces trunk`).
- **Ping entre VLANs falha logo no início:** o primeiro ping costuma perder pacotes por ARP — repita.
- **"Native VLAN mismatch":** todos os trunks entre switches devem ter `native vlan 999`.
- **Navegador do SEQ-01 não abre o LIMS:** confirme que o HTTP do SRV-LIMS está On e que a ACL está na **subinterface G0/0.20, sentido `in`** (`show ip interface g0/0.20`).
- **Hosts IPv6 não recebem SLAAC:** falta `ipv6 unicast-routing` no R-BORDA.
