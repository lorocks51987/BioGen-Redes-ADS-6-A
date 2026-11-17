# RELATÓRIO TÉCNICO DE ARQUITETURA DE REDES CORPORATIVAS — P2

> Rascunho para copiar no modelo oficial. Itens entre ⚠️ precisam ser completados pelo grupo (RAs, turma, prints).

**Cenário sorteado & empresa fictícia:** Cenário A10 · Laboratório BioGen (Pesquisa Genética & Salas Limpas)
**Turma:** ⚠️ ( ) Turma A ( ) Turma B
**Grupo 4 — Desafio 10**

| Nº | Nome completo | R.A. |
|---|---|---|
| 1 | ⚠️ [Líder da Equipe]: Luiz Costa (confirmar quem é o líder) | ⚠️ |
| 2 | Vinícius Gualtieri | ⚠️ |
| 3 | Leonardo Munhoz | ⚠️ |
| 4 | Gabriel de Lima | ⚠️ |
| 5 | Matheus Thierry | ⚠️ |
| 6 | Artur Taroco | ⚠️ |

⚠️ O modelo prevê **até 5 alunos** e o grupo tem 6 — confirmar com o professor e acrescentar uma linha na tabela.

---

## 1. SUMÁRIO EXECUTIVO & CONTEXTO DE NEGÓCIO

**Descrição do Desafio Corporativo Sorteado:**
O Laboratório BioGen realiza pesquisa genética e opera salas limpas com sequenciadores de DNA. Esses equipamentos produzem dados genômicos sensíveis e, por executarem sistemas embarcados raramente atualizados, não podem ter qualquer comunicação de saída livre: a política exigida é *Zero-Trust outbound* na Camada 4 — o sequenciador só pode enviar seus arquivos ao servidor de armazenamento (LIMS) e todo o restante (Internet, outras VLANs, ICMP) deve ser bloqueado, tanto em IPv4 quanto em IPv6. Ao mesmo tempo, o laboratório precisa publicar resultados e notícias científicas na Internet, o que exige uma DMZ isolada da rede interna. Os setores envolvidos são Pesquisa/Bioinformática, Salas Limpas (Sequenciadores), Administrativo, Servidores internos, DMZ e Gerência de TI. O bloco disponibilizado é 10.99.0.0/21 com IPv6 em Dual-Stack.

**Objetivos Técnicos do Projeto:**
Implementar segmentação L2 com 6 VLANs de serviço (10, 20, 30, 40, 99 e DMZ 50) mais VLAN nativa 999 e VLAN 998 para portas sem uso; Port-Security em todas as portas de acesso; Rapid-PVST+ com Root Bridge definida e enlace redundante; roteamento Inter-VLAN via Router-on-a-Stick (Cisco 2911); sub-redes IPv4 em VLSM a partir de 10.99.0.0/21 com 75% do bloco reservado para expansão; IPv6 Dual-Stack (2001:db8:b10::/48) com SLAAC/EUI-64 e endereços estáticos; ACLs estendidas IPv4 e IPv6 implementando o Zero-Trust dos sequenciadores, a publicação controlada da DMZ e a proteção da borda; SSH restrito à VLAN de gerência; e serviços DHCP, DNS, HTTP/HTTPS, FTP, NTP e Syslog.

---

## 2. DIAGRAMA DA TOPOLOGIA LÓGICA

⚠️ **[PRINT DA TOPOLOGIA COMPLETA DO PACKET TRACER]** — com notas de VLANs, links e blocos IP.

Equipamentos: 2 roteadores 2911 (R-BORDA e ISP), 5 switches 2960 (SW-CORE, SW-A1, SW-A2, SW-DMZ, SW-INTERNET), 4 servidores e 7 PCs.

---

## 3. CAMADA 1 E CAMADA 2

### 3.1 Escolha de Mídias Físicas e Conexões (L1)
Foram usados cabos de par trançado **direto** (straight-through) entre dispositivos de tipos diferentes — PCs/servidores ↔ switches e roteadores ↔ switches (R-BORDA G0/0 ↔ SW-CORE G0/1, R-BORDA G0/1 ↔ SW-DMZ G0/1 e ISP G0/1 ↔ SW-INTERNET) — pois um lado é MDI e o outro MDI-X. Entre equipamentos do mesmo tipo foi usado cabo **cruzado** (crossover): SW-CORE ↔ SW-A1, SW-CORE ↔ SW-A2, o enlace redundante SW-A1 ↔ SW-A2 e o link WAN R-BORDA G0/2 ↔ ISP G0/0, que simula um enlace Ethernet/fibra metropolitana entregue pelo provedor. Os enlaces entre switches de acesso usam portas Gigabit para reduzir o custo STP e suportar a transferência de grandes arquivos genômicos (FASTQ/BAM). Em ambiente real, a ligação entre o core e as salas limpas seria em fibra óptica, pela distância e pela imunidade a interferência eletromagnética de equipamentos laboratoriais.

### 3.2 Tabela de Segmentação de VLANs e Port-Security (L2)

| VLAN ID | Nome | Sub-rede | Portas (Access) | Port-Security | Violação |
|---|---|---|---|---|---|
| 10 | PESQUISA | 10.99.0.0/24 | SW-A1 Fa0/1–12 | Sim (Max 2 MACs, sticky) | Restrict |
| 20 | SEQUENCIADORES | 10.99.1.128/26 | SW-A2 Fa0/1–10 | Sim (Max 1 MAC, sticky) | Shutdown |
| 30 | ADMINISTRATIVO | 10.99.1.0/25 | SW-A1 Fa0/13–20 | Sim (Max 2 MACs, sticky) | Restrict |
| 40 | SERVIDORES | 10.99.1.192/27 | SW-CORE Fa0/1–5 | Sim (Max 1 MAC, sticky) | Restrict |
| 50 | DMZ | 10.99.1.224/28 | SW-DMZ Fa0/1–2 | Sim (Max 1 MAC, sticky) | Shutdown |
| 99 | GERENCIA | 10.99.1.240/28 | SW-CORE Fa0/6 | Sim (Max 1 MAC) | Shutdown |
| 999 | NATIVA | — | Trunks 802.1Q (SW-CORE G0/1, Fa0/23–24; SW-A1/A2 G0/1–2) | — | — |
| 998 | NAO_UTILIZADAS | — | Demais portas (shutdown) | — | — |

Os sequenciadores usam violação **Shutdown** com 1 MAC: são equipamentos fixos, então qualquer outro dispositivo conectado na sala limpa é tratado como incidente. A VLAN nativa foi movida para a 999 (sem hosts) para evitar ataques de VLAN hopping, e os trunks usam `switchport nonegotiate` (sem DTP). Todas as portas de acesso têm PortFast + BPDU Guard.

### 3.3 Prevenção de Loops com STP
Foi usado **Rapid-PVST+**. O **SW-CORE** foi eleito **Root Bridge** em todas as VLANs com `spanning-tree vlan … root primary` (prioridade 24576). O SW-A1 recebeu prioridade 28672 e o SW-A2 ficou com o padrão 32768. O triângulo SW-CORE — SW-A1 — SW-A2 forma um loop; no enlace SW-A1 ↔ SW-A2 os dois switches anunciam o mesmo custo até a raiz (19), e o desempate pelo menor Bridge ID dá a porta designada ao SW-A1. Com isso, a porta **GigabitEthernet0/2 do SW-A2** entra no estado **Alternate/Blocking**, e é ativada automaticamente em caso de queda do enlace SW-CORE ↔ SW-A2.

⚠️ [Print de `show spanning-tree vlan 20` no SW-A2 mostrando `Gi0/2 Altn BLK` e no SW-CORE mostrando "This bridge is the root"]

---

## 4. CAMADA 3: ENDEREÇAMENTO IPv4 (VLSM) E IPv6

### 4.1 Tabela de Sub-redes IPv4 em VLSM — bloco 10.99.0.0/21

| Sub-rede / Setor | Hosts Req. | CIDR | Máscara | Rede | 1º IP (Gateway) | Último IP | Broadcast |
|---|---|---|---|---|---|---|---|
| VLAN 10 Pesquisa | 200 | /24 | 255.255.255.0 | 10.99.0.0 | 10.99.0.1 | 10.99.0.254 | 10.99.0.255 |
| VLAN 30 Administrativo | 90 | /25 | 255.255.255.128 | 10.99.1.0 | 10.99.1.1 | 10.99.1.126 | 10.99.1.127 |
| VLAN 20 Sequenciadores | 40 | /26 | 255.255.255.192 | 10.99.1.128 | 10.99.1.129 | 10.99.1.190 | 10.99.1.191 |
| VLAN 40 Servidores | 20 | /27 | 255.255.255.224 | 10.99.1.192 | 10.99.1.193 | 10.99.1.222 | 10.99.1.223 |
| VLAN 50 DMZ | 10 | /28 | 255.255.255.240 | 10.99.1.224 | 10.99.1.225 | 10.99.1.238 | 10.99.1.239 |
| VLAN 99 Gerência | 10 | /28 | 255.255.255.240 | 10.99.1.240 | 10.99.1.241 | 10.99.1.254 | 10.99.1.255 |
| Link WAN PTP | 2 | /30 | 255.255.255.252 | 10.99.2.0 | 10.99.2.1 | 10.99.2.2 | 10.99.2.3 |

As sub-redes foram alocadas da maior para a menor, de forma contígua, ocupando apenas 10.99.0.0 – 10.99.2.3. A faixa **10.99.2.4 – 10.99.7.255 (≈1.532 endereços, 75% do /21)** fica reservada para expansão.

### 4.2 Endereçamento IPv6 (Dual-Stack)
Prefixo global: **2001:db8:b10::/48**. Cada VLAN recebe um /64 cujo 4º hexteto repete o ID da VLAN: 2001:db8:b10:**10**::/64 (Pesquisa), :**20**:: (Sequenciadores), :**30**:: (Administrativo), :**40**:: (Servidores), :**50**:: (DMZ), :**99**:: (Gerência) e :**ff**:: para o link WAN. O gateway de cada rede é o `::1` do prefixo (ex.: 2001:db8:b10:10::1/64) e todas as interfaces do roteador usam o link-local fixo `fe80::1`, com `ipv6 unicast-routing` habilitado.
Os PCs de Pesquisa e Administrativo obtêm IPv6 por **SLAAC com EUI-64** (o roteador anuncia o prefixo via Router Advertisement e o host monta o identificador de interface a partir do MAC). Servidores, DMZ, gerência e **sequenciadores usam endereços estáticos** (ex.: SEQ-01 = 2001:db8:b10:20::10) — nos sequenciadores isso é necessário para que a ACL IPv6 de Zero-Trust trate exatamente esses hosts. A política de bloqueio foi replicada em IPv6 (`ACL6_SEQ_ZEROTRUST`), pois em Dual-Stack uma ACL só IPv4 deixaria o IPv6 como caminho de fuga.

⚠️ [Print opcional do `ipconfig /all` do PC-PESQ-01 mostrando IPv4 via DHCP e IPv6 EUI-64]

---

## 5. ROTEAMENTO INTER-VLAN E SEGURANÇA NA CAMADA 4

### 5.1 Método de Roteamento Inter-VLAN Adotado
Foi utilizado **Router-on-a-Stick** no R-BORDA (Cisco 2911): a interface G0/0 é ligada a um trunk 802.1Q do SW-CORE e dividida em subinterfaces G0/0.10, .20, .30, .40 e .99, cada uma com `encapsulation dot1Q <vlan>` e o endereço de gateway IPv4/IPv6 da respectiva VLAN. A DMZ não usa subinterface: fica em uma **interface física dedicada (G0/1)**, de modo que o tráfego público nunca passa pelo trunk interno. A G0/2 conecta ao ISP, com rotas padrão `ip route 0.0.0.0 0.0.0.0 10.99.2.2` e `ipv6 route ::/0 2001:db8:b10:ff::2`. O R-BORDA também atua como servidor DHCP das VLANs 10 e 30.

### 5.2 Regra de ACL Estendida (Filtro por Portas TCP/UDP)

```
! Zero-Trust outbound dos sequenciadores - aplicada em G0/0.20 (in)
ip access-list extended ACL_SEQ_ZEROTRUST
 remark Upload de dados de sequenciamento para o LIMS
 permit tcp 10.99.1.128 0.0.0.63 host 10.99.1.195 eq 21
 permit tcp 10.99.1.128 0.0.0.63 host 10.99.1.195 eq 20
 permit tcp 10.99.1.128 0.0.0.63 host 10.99.1.195 eq 80
 remark Sincronismo de horario e envio de logs
 permit udp 10.99.1.128 0.0.0.63 host 10.99.1.194 eq 123
 permit udp 10.99.1.128 0.0.0.63 host 10.99.1.194 eq 514
 deny ip any any
!
ipv6 access-list ACL6_SEQ_ZEROTRUST
 permit tcp 2001:db8:b10:20::/64 host 2001:db8:b10:40::11 eq 21
 permit tcp 2001:db8:b10:20::/64 host 2001:db8:b10:40::11 eq 20
 permit tcp 2001:db8:b10:20::/64 host 2001:db8:b10:40::11 eq 80
 permit icmp any any nd-ns
 permit icmp any any nd-na
 deny ipv6 any any
!
interface GigabitEthernet0/0.20
 ip access-group ACL_SEQ_ZEROTRUST in
 ipv6 traffic-filter ACL6_SEQ_ZEROTRUST in
!
! Borda: Internet so acessa o site da DMZ - aplicada em G0/2 (in)
ip access-list extended ACL_WAN_IN
 permit tcp any host 10.99.1.226 eq 80
 permit tcp any host 10.99.1.226 eq 443
 permit tcp any 10.99.0.0 0.0.7.255 established
 permit icmp any 10.99.0.0 0.0.7.255 echo-reply
 permit icmp any 10.99.0.0 0.0.7.255 unreachable
 deny ip any any
!
! DMZ isolada: nao inicia conexoes para a rede interna - aplicada em G0/1 (in)
ip access-list extended ACL_DMZ_IN
 permit tcp host 10.99.1.226 eq 80 any established
 permit tcp host 10.99.1.226 eq 443 any established
 permit icmp 10.99.1.224 0.0.0.15 any echo-reply
 deny ip 10.99.1.224 0.0.0.15 10.99.0.0 0.0.7.255
 permit ip 10.99.1.224 0.0.0.15 any
```

A ACL de Zero-Trust foi colocada na subinterface G0/0.20, sentido **in**, ou seja, o mais próximo possível da origem (boa prática para ACL estendida): o pacote proibido é descartado assim que chega ao roteador e nunca atravessa a rede. Como a ACL filtra por porta TCP/UDP, o sequenciador consegue transferir arquivos por FTP e acessar o LIMS por HTTP, mas qualquer outro protocolo — inclusive ICMP — é negado.

### 5.3 Evidência do Bloqueio da ACL
⚠️ **[PRINT: SEQ-01 → `ping 10.99.1.195` = "Destination host unreachable" + Web Browser do SEQ-01 abrindo `http://10.99.1.195` (página do LIMS) com sucesso]**

---

## 6. CAMADA 7: SERVIÇOS DE REDE

| Serviço | IP do Servidor | Porta / Protocolo | Parâmetros Configurados | Status |
|---|---|---|---|---|
| DHCP Server | R-BORDA (10.99.0.1 / 10.99.1.1) | UDP 67/68 | Pools: POOL_PESQUISA 10.99.0.0/24 e POOL_ADMIN 10.99.1.0/25 · Exclusões: 10.99.0.1–10 e 10.99.1.1–10 · DNS 10.99.1.194 | OK (IP dinâmico) |
| DNS Server | 10.99.1.194 (SRV-INTERNO) | UDP 53 | Domínio: biogen.local · A: www → 10.99.1.226, lims → 10.99.1.195 | OK (nslookup) |
| HTTP / Web | 10.99.1.226 (SRV-WEB-DMZ) e 10.99.1.195 (SRV-LIMS) | TCP 80 / 443 | Página de divulgação científica (DMZ) e portal LIMS (interno) | OK (Browser) |
| FTP / NTP / Syslog | 10.99.1.195 (FTP) · 10.99.1.194 (NTP/Syslog) | TCP 20/21 · UDP 123 · UDP 514 | FTP: usuário `sequenciador` p/ upload de dados · roteador com `ntp server` e `logging host` | OK |

⚠️ **[PRINT: PC-PESQ-01 com `nslookup www.biogen.local` e Web Browser abrindo `http://www.biogen.local`]**

---

## 7. VISÃO CRÍTICA DE ENGENHARIA

### 7.1 Pontos Únicos de Falha (SPOF)
- **R-BORDA:** é ao mesmo tempo gateway de todas as VLANs, servidor DHCP, firewall (ACLs) e roteador de borda. Sua falha derruba todo o roteamento inter-VLAN, o acesso à DMZ e à Internet.
- **Interface G0/0 / trunk R-BORDA ↔ SW-CORE:** um único cabo transporta todo o tráfego entre VLANs; se cair, nenhuma VLAN fala com outra.
- **SW-CORE:** concentra os servidores e os uplinks dos switches de acesso. O enlace redundante SW-A1 ↔ SW-A2 protege apenas contra a queda de um uplink, não contra a falha do próprio core.
- **Link WAN único** com um só provedor, sem rota de contingência — perda total da Internet e do site público.
- **SRV-INTERNO e SRV-LIMS** sem réplica: perda do DNS afeta a navegação por nome; perda do LIMS interrompe a gravação dos dados de sequenciamento (corridas que podem durar dias).
- **SW-DMZ** sem redundância e equipamentos sem fontes de alimentação redundantes/UPS — crítico para salas limpas, onde um sequenciamento interrompido perde reagentes caros.

### 7.2 Proposta de Upgrade de Arquitetura & Redundância
1. **Núcleo em camada 3:** substituir o Router-on-a-Stick por dois switches multicamada (Catalyst 3650/9300) com SVIs e `ip routing`, eliminando o gargalo do trunk único.
2. **Redundância de gateway:** HSRP (ou VRRP) entre os dois cores/roteadores, com IP virtual por VLAN (ex.: 10.99.0.1) — falha transparente para os hosts.
3. **EtherChannel LACP** nos uplinks core ↔ acesso (2×1 Gbps ou 10 Gbps em fibra) para banda e redundância, essencial para os arquivos de sequenciamento de dezenas de GB.
4. **Firewall de próxima geração dedicado** (FortiGate ou pfSense) entre Internet, DMZ e rede interna: inspeção *stateful* (substitui o `established`, que não protege UDP), IPS e inspeção L7. Isso também permitiria filtrar o tráfego *de entrada* para os sequenciadores sem quebrar as respostas.
5. **Microssegmentação/NAC:** autenticação 802.1X nas portas das salas limpas e Private VLANs para que um sequenciador não converse com outro.
6. **WAN dupla** (dois provedores) com rota estática flutuante ou BGP, e **NAT/PAT** na borda para tradução para IP público.
7. **Servidores replicados** (DNS secundário, LIMS em cluster/storage com RAID e backup off-site) e **UPS + fontes redundantes**.
8. **Anel com Rapid-PVST+** ligando todos os switches de acesso aos dois cores, com root primária/secundária definidas.

### 7.3 Planejamento de Escalabilidade & Expansão Futura
O plano VLSM usa apenas 10.99.0.0 – 10.99.2.3; a faixa **10.99.2.4 – 10.99.7.255 (≈75% do /21)** permanece livre, o que permite, sem renumerar nada: dobrar a Pesquisa (um novo /24, ex. 10.99.3.0/24), criar novas salas limpas (10.99.2.64/26, 10.99.2.128/26), uma VLAN de Wi-Fi para visitantes (10.99.4.0/24) e links WAN adicionais (/30). No IPv6, o /48 oferece 65.536 sub-redes /64 — o esquema "4º hexteto = ID da VLAN" permite criar a VLAN 60 como 2001:db8:b10:60::/64 imediatamente. Novos switches são incluídos com trunks 802.1Q para o core (VLANs já padronizadas, nativa 999), herdando a topologia STP sem alteração de raiz; novos departamentos exigem apenas uma nova VLAN, uma subinterface/SVI e um pool DHCP, sem parada da produção. Para as mudanças, recomenda-se janela de manutenção e uso do enlace redundante (STP) para trocar equipamentos sem interrupção.

### 7.4 Considerações Finais & Conclusão Técnica
O projeto mostrou na prática como as camadas do modelo OSI se apoiam: a escolha correta de cabos (L1) é pré-requisito para os trunks e o STP (L2); a segmentação em VLANs só tem utilidade com roteamento inter-VLAN e endereçamento bem planejado (L3); e é na L4 que se implementa a política de negócio — no caso, o Zero-Trust dos sequenciadores, que filtra por porta TCP/UDP e não apenas por endereço. A principal lição foi que, em Dual-Stack, toda política de segurança precisa ser duplicada para IPv6; caso contrário, o bloqueio IPv4 é contornado. Os maiores desafios de depuração foram a posição e o sentido das ACLs (interface/subinterface correta, `in` vs `out`, e o `deny` implícito ao final) e entender que ACLs sem estado exigem o `established` para permitir respostas sem abrir novas conexões. A validação final com DHCP, `nslookup`, navegador, FTP e os testes de ping bloqueado confirmou o funcionamento dos serviços L7 sobre toda a pilha.
