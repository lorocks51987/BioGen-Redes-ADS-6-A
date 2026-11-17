# Cenário A10 · Laboratório BioGen — Plano de Rede

**Desafio:** os sequenciadores de DNA (salas limpas) só podem *enviar* dados ao servidor LIMS interno — nenhuma outra saída é permitida, em IPv4 **e** IPv6 (Zero-Trust outbound, filtro na Camada 4). Uma DMZ isolada hospeda o site de divulgação científica.
**Bloco:** IPv4 `10.99.0.0/21` (10.99.0.0 – 10.99.7.255) + IPv6 Dual-Stack `2001:db8:b10::/48`.

---

## 1. Topologia

```
                         [SRV-INTERNET 203.0.113.100]   [PC-INTERNET 203.0.113.10]
                                         \                 /
                                          [SW-INTERNET (sem config)]
                                                   | Fa0/1
                                                   | G0/1 203.0.113.1
                                              ( ISP - 2911 )
                                                   | G0/0 10.99.2.2/30
                                                   |   (cabo cruzado - link WAN)
                                                   | G0/2 10.99.2.1/30
   [SRV-WEB-DMZ]--Fa0/1-[SW-DMZ]-G0/1-----G0/1-( R-BORDA - 2911 )
     10.99.1.226        VLAN 50     10.99.1.225/28 | G0/0  (Router-on-a-Stick: .10 .20 .30 .40 .99)
                                                   | trunk 802.1Q (cabo direto)
                                                   | G0/1
                                          [ SW-CORE - 2960 ]  <- Root Bridge (STP)
                          Fa0/1 SRV-INTERNO / Fa0/2 SRV-LIMS / Fa0/6 PC-GERENCIA
                                 Fa0/23 /                \ Fa0/24     (cabos cruzados)
                                 G0/1  /                  \ G0/1
                          [ SW-A1 - 2960 ]--G0/2------G0/2--[ SW-A2 - 2960 ]   <- G0/2 de SW-A2 fica BLOQUEADA
                       Pesquisa + Administrativo              Sala Limpa (Sequenciadores)
                     Fa0/1 PC-PESQ-01  Fa0/2 PC-PESQ-02       Fa0/1 SEQ-01  Fa0/2 SEQ-02
                     Fa0/13 PC-ADM-01
```

## 2. VLANs

| VLAN | Nome | Uso | Portas de acesso |
|---|---|---|---|
| 10 | PESQUISA | Pesquisadores / bioinformática | SW-A1 Fa0/1–12 |
| 20 | SEQUENCIADORES | Sequenciadores de DNA (salas limpas) | SW-A2 Fa0/1–10 |
| 30 | ADMINISTRATIVO | Administrativo / financeiro / RH | SW-A1 Fa0/13–20 |
| 40 | SERVIDORES | DNS/NTP/Syslog + LIMS (armazenamento genômico) | SW-CORE Fa0/1–5 |
| 50 | DMZ | Site público de divulgação científica (interface física separada no roteador) | SW-DMZ Fa0/1–2 |
| 99 | GERENCIA | SVIs dos switches + PC da equipe de TI | SW-CORE Fa0/6 |
| 998 | NAO_UTILIZADAS | Portas sem uso (desligadas) | todas as demais |
| 999 | NATIVA | VLAN nativa dos trunks (sem hosts) | — |

## 3. VLSM IPv4 — `10.99.0.0/21` (da maior para a menor rede)

| Setor | Hosts req. | CIDR | Máscara | Rede | 1º IP (Gateway) | Último IP | Broadcast |
|---|---|---|---|---|---|---|---|
| VLAN 10 Pesquisa | 200 | /24 | 255.255.255.0 | 10.99.0.0 | 10.99.0.1 | 10.99.0.254 | 10.99.0.255 |
| VLAN 30 Administrativo | 90 | /25 | 255.255.255.128 | 10.99.1.0 | 10.99.1.1 | 10.99.1.126 | 10.99.1.127 |
| VLAN 20 Sequenciadores | 40 | /26 | 255.255.255.192 | 10.99.1.128 | 10.99.1.129 | 10.99.1.190 | 10.99.1.191 |
| VLAN 40 Servidores | 20 | /27 | 255.255.255.224 | 10.99.1.192 | 10.99.1.193 | 10.99.1.222 | 10.99.1.223 |
| VLAN 50 DMZ | 10 | /28 | 255.255.255.240 | 10.99.1.224 | 10.99.1.225 | 10.99.1.238 | 10.99.1.239 |
| VLAN 99 Gerência | 10 | /28 | 255.255.255.240 | 10.99.1.240 | 10.99.1.241 | 10.99.1.254 | 10.99.1.255 |
| Link WAN PTP | 2 | /30 | 255.255.255.252 | 10.99.2.0 | 10.99.2.1 | 10.99.2.2 | 10.99.2.3 |
| **Reserva (crescimento)** | — | — | — | **10.99.2.4 – 10.99.7.255** | | | ≈ 1.532 endereços (75% do bloco) |

> Rede externa simulada (Internet): `203.0.113.0/24` (faixa de documentação TEST-NET-3).

## 4. IPv6 Dual-Stack — prefixo `2001:db8:b10::/48`

O 4º hexteto = nº da VLAN (fácil de ler e de expandir). Todas as interfaces do roteador usam link-local `fe80::1`.

| Rede | Prefixo /64 | Gateway | Como os hosts obtêm IPv6 |
|---|---|---|---|
| VLAN 10 Pesquisa | 2001:db8:b10:10::/64 | 2001:db8:b10:10::1 | SLAAC (EUI-64) |
| VLAN 20 Sequenciadores | 2001:db8:b10:20::/64 | 2001:db8:b10:20::1 | Estático (::10, ::11) — necessário p/ ACL |
| VLAN 30 Administrativo | 2001:db8:b10:30::/64 | 2001:db8:b10:30::1 | SLAAC (EUI-64) |
| VLAN 40 Servidores | 2001:db8:b10:40::/64 | 2001:db8:b10:40::1 | Estático |
| VLAN 50 DMZ | 2001:db8:b10:50::/64 | 2001:db8:b10:50::1 | Estático |
| VLAN 99 Gerência | 2001:db8:b10:99::/64 | 2001:db8:b10:99::1 | Estático |
| WAN | 2001:db8:b10:ff::/64 | R-BORDA ::1 / ISP ::2 | Estático |
| Internet (ISP) | 2001:db8:ffff::/64 | 2001:db8:ffff::1 | Estático |

## 5. Endereços dos equipamentos e hosts

| Dispositivo | Porta | IPv4 | Gateway | IPv6 | Obs. |
|---|---|---|---|---|---|
| PC-PESQ-01 | SW-A1 Fa0/1 | DHCP (10.99.0.11+) | 10.99.0.1 | SLAAC | |
| PC-PESQ-02 | SW-A1 Fa0/2 | DHCP | 10.99.0.1 | SLAAC | |
| PC-ADM-01 | SW-A1 Fa0/13 | DHCP (10.99.1.11+) | 10.99.1.1 | SLAAC | |
| SEQ-01 (PC-PT) | SW-A2 Fa0/1 | 10.99.1.130/26 | 10.99.1.129 | 2001:db8:b10:20::10/64 | DNS em branco |
| SEQ-02 (PC-PT) | SW-A2 Fa0/2 | 10.99.1.131/26 | 10.99.1.129 | 2001:db8:b10:20::11/64 | |
| SRV-INTERNO | SW-CORE Fa0/1 | 10.99.1.194/27 | 10.99.1.193 | 2001:db8:b10:40::10/64 | DNS, NTP, Syslog |
| SRV-LIMS | SW-CORE Fa0/2 | 10.99.1.195/27 | 10.99.1.193 | 2001:db8:b10:40::11/64 | HTTP, FTP |
| PC-GERENCIA | SW-CORE Fa0/6 | 10.99.1.250/28 | 10.99.1.241 | 2001:db8:b10:99::50/64 | único que acessa SSH |
| SW-CORE (VLAN 99) | — | 10.99.1.242/28 | 10.99.1.241 | — | |
| SW-A1 (VLAN 99) | — | 10.99.1.243/28 | 10.99.1.241 | — | |
| SW-A2 (VLAN 99) | — | 10.99.1.244/28 | 10.99.1.241 | — | |
| SRV-WEB-DMZ | SW-DMZ Fa0/1 | 10.99.1.226/28 | 10.99.1.225 | 2001:db8:b10:50::10/64 | HTTP/HTTPS |
| PC-INTERNET | SW-INTERNET | 203.0.113.10/24 | 203.0.113.1 | 2001:db8:ffff::10/64 | DNS 10.99.1.194 opcional |
| SRV-INTERNET | SW-INTERNET | 203.0.113.100/24 | 203.0.113.1 | 2001:db8:ffff::100/64 | HTTP (simula site externo) |

## 6. Cabeamento (Camada 1)

| Enlace | Cabo | Motivo |
|---|---|---|
| PC/Servidor ↔ Switch | Par trançado **direto** | dispositivos diferentes (MDI ↔ MDI-X) |
| R-BORDA G0/0 ↔ SW-CORE G0/1 | **Direto** | roteador ↔ switch |
| R-BORDA G0/1 ↔ SW-DMZ G0/1 | **Direto** | roteador ↔ switch |
| ISP G0/1 ↔ SW-INTERNET Fa0/1 | **Direto** | roteador ↔ switch |
| SW-CORE Fa0/23 ↔ SW-A1 G0/1 | **Cruzado** | switch ↔ switch (mesmo tipo) |
| SW-CORE Fa0/24 ↔ SW-A2 G0/1 | **Cruzado** | switch ↔ switch |
| SW-A1 G0/2 ↔ SW-A2 G0/2 | **Cruzado** | enlace redundante (gera o loop que o STP resolve) |
| R-BORDA G0/2 ↔ ISP G0/0 | **Cruzado** | roteador ↔ roteador (simula WAN Ethernet/fibra) |

## 7. Matriz de segurança (o que cada ACL faz)

| Origem → Destino | Permitido? | Onde é aplicado |
|---|---|---|
| Sequenciadores → SRV-LIMS FTP (20/21) e HTTP (80) | ✅ | `ACL_SEQ_ZEROTRUST` (in G0/0.20) |
| Sequenciadores → SRV-INTERNO NTP (123) e Syslog (514) | ✅ | idem |
| Sequenciadores → qualquer outra coisa (ping, Internet, outras VLANs, DNS…) | ❌ | idem + `ACL6_SEQ_ZEROTRUST` no IPv6 |
| Internet → SRV-WEB-DMZ HTTP/HTTPS | ✅ | `ACL_WAN_IN` (in G0/2) |
| Internet → rede interna (novas conexões) | ❌ (só respostas `established`/echo-reply) | `ACL_WAN_IN` |
| DMZ → rede interna (novas conexões) | ❌ (só respostas do HTTP) | `ACL_DMZ_IN` (in G0/1) |
| Interno → DMZ (site) / Internet | ✅ | — |
| SSH nos equipamentos | ✅ só da VLAN 99 | `access-list 10` nas linhas VTY |
