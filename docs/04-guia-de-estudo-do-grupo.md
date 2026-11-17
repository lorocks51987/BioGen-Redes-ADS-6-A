# Guia de estudo do Grupo 4 — entendendo a rede BioGen do zero

> Este guia é para quem sabe pouco de redes. Lê na ordem: cada parte usa o que foi explicado antes.
> Abra o arquivo `BioGen-A10.pkt` no Packet Tracer ao lado enquanto lê — tudo que está aqui dá para ver e testar nele.

**Sumário**

1. Como usar o Packet Tracer (o básico do básico)
2. O que é o projeto, em uma página
3. O modelo OSI — o "mapa" que o professor usa
4. Camada 1: equipamentos e cabos
5. Camada 2: switches, VLANs, trunk, STP e Port-Security
6. Camada 3: endereço IP, máscara, sub-rede e VLSM (com a conta passo a passo)
7. IPv6 e Dual-Stack
8. Roteamento: Router-on-a-Stick e rotas padrão
9. Camada 4: ACLs — o coração do nosso cenário
10. Camada 7: DHCP, DNS, Web, FTP, NTP e Syslog
11. Segurança extra: SSH, senhas e VLAN nativa
12. Roteiro de testes (o que fazer, o que deve aparecer e por quê)
13. Perguntas que o professor pode fazer (com respostas)
14. Glossário rápido
15. Sugestão de divisão para a apresentação

---

## 1. Como usar o Packet Tracer (o básico do básico)

**Abrir o projeto:** dê dois cliques em `BioGen-A10.pkt` (ou *File → Open*). Abrirá a tela **Logical** com a topologia.

**Ao abrir, espere uns 30–60 segundos.** As bolinhas nos cabos:
- 🟢 **verde** = link funcionando;
- 🟠 **laranja** = switch ainda "pensando" (o STP está verificando se há loop — explicado no item 5.4). Depois de alguns segundos fica verde. Para acelerar, clique no botão **⏩ (Fast Forward Time)** no canto inferior esquerdo algumas vezes.
- 🔴 **vermelho** = link desligado (porta com `shutdown` ou cabo errado).

**Clicar em um equipamento abre uma janela com abas:**

| Aba | Para que serve | Onde aparece |
|---|---|---|
| **Physical** | ver o equipamento "de verdade", ligar/desligar | todos |
| **Config** | configurar por botões (nome, IP, serviços) | todos |
| **CLI** | linha de comando do Cisco IOS (onde se digitam os comandos) | roteadores e switches |
| **Desktop** | "área de trabalho" do PC: IP Configuration, Command Prompt, Web Browser | PCs e servidores |
| **Services** | ligar/desligar DNS, HTTP, FTP etc. | servidores |

**Modos do CLI (muito importante entender):**

```
R-BORDA>                  modo usuário   (só consulta básica)
R-BORDA> enable
R-BORDA#                  modo privilegiado (comandos "show", salvar, reiniciar)
R-BORDA# configure terminal
R-BORDA(config)#          modo de configuração global (mudar o equipamento)
R-BORDA(config)# interface g0/0
R-BORDA(config-if)#       configurando uma interface específica
R-BORDA(config-if)# exit  volta um nível
R-BORDA(config)# end      volta direto para o #
```

A senha do `enable` e do usuário `admin` no projeto é **BioGen@2026**.

**Comandos "show" mais úteis (no modo `#`):**

| Comando | O que mostra |
|---|---|
| `show running-config` | toda a configuração atual |
| `show ip interface brief` | interfaces, IPs e se estão up/down |
| `show vlan brief` | VLANs do switch e quais portas estão em cada uma |
| `show interfaces trunk` | quais portas são trunk e quais VLANs passam por elas |
| `show spanning-tree vlan 20` | quem é a Root Bridge e quais portas estão bloqueadas |
| `show port-security` | portas protegidas e violações |
| `show access-lists` | ACLs e quantos pacotes cada linha pegou ("matches") |
| `show ip route` | tabela de rotas do roteador |
| `show ip dhcp binding` | quais IPs o DHCP já entregou |

> **Passo obrigatório depois de abrir o arquivo:** a chave do SSH não fica salva no `.pkt`. No **R-BORDA, SW-CORE, SW-A1 e SW-A2**, entre no CLI e digite:
> ```
> enable            (senha BioGen@2026)
> configure terminal
> crypto key generate rsa      (quando perguntar o tamanho, digite 1024)
> end
> copy running-config startup-config
> ```
> Depois salve o arquivo do Packet Tracer (*Ctrl+S*). Sem isso, o teste de SSH não funciona — o resto funciona normalmente.

---

## 2. O que é o projeto, em uma página

O **Laboratório BioGen** faz pesquisa genética. Ele tem:

- **Pesquisadores** (VLAN 10) e **Administrativo** (VLAN 30) — usam a rede normalmente, inclusive Internet.
- **Sequenciadores de DNA** nas salas limpas (VLAN 20) — máquinas caras, com sistemas antigos e dados sigilosos. **Regra do cenário:** elas só podem *enviar* os arquivos para o servidor de dados (LIMS). Nada de Internet, nada de falar com outros setores. Isso se chama **Zero-Trust de saída** ("não confie em ninguém, libere só o necessário").
- **Servidores internos** (VLAN 40): SRV-INTERNO (DNS, NTP, Syslog) e SRV-LIMS (onde os dados genéticos são guardados — HTTP e FTP).
- **DMZ** (VLAN 50): o site público de divulgação científica. Fica acessível pela Internet, mas **isolado** da rede interna.
- **Gerência** (VLAN 99): o PC da equipe de TI, o único que pode acessar os equipamentos remotamente (SSH).

O professor nos deu o bloco **10.99.0.0/21** (2.048 endereços IPv4) e pediu **IPv6 ao mesmo tempo (Dual-Stack)**.

**Caminho de um pacote, para fixar a ideia:** o PC-PESQ-01 quer abrir o site `www.biogen.local`:
1. pergunta ao DNS (SRV-INTERNO) qual é o IP → recebe `10.99.1.226`;
2. como esse IP é de outra rede, manda o pacote para o seu **gateway** (o roteador R-BORDA, `10.99.0.1`);
3. o pacote sai do PC, entra no SW-A1, sobe pelo **trunk** até o SW-CORE e chega no R-BORDA;
4. o R-BORDA vê que `10.99.1.226` está na interface da DMZ e encaminha;
5. o servidor responde pelo caminho inverso — e a **ACL da DMZ** deixa passar porque é uma *resposta*.

---

## 3. O modelo OSI — o "mapa" que o professor usa

O relatório é organizado pelas camadas do modelo OSI. Pense nele como "andares" de um prédio: cada andar resolve um problema.

| Camada | Nome | Pergunta que responde | No nosso projeto |
|---|---|---|---|
| 7 | Aplicação | *Qual serviço o usuário usa?* | DHCP, DNS, HTTP, FTP, NTP, Syslog |
| 4 | Transporte | *Para qual programa vai (porta)?* | ACLs filtrando portas TCP/UDP (21, 80, 123…) |
| 3 | Rede | *Para qual endereço/rede vai?* | IPs, VLSM, IPv6, roteamento |
| 2 | Enlace | *Para qual placa de rede da mesma rede local?* | switches, MAC, VLANs, trunk, STP, port-security |
| 1 | Física | *Por qual meio o sinal passa?* | cabos direto/cruzado, portas Fast/Gigabit |

(As camadas 5 e 6 — Sessão e Apresentação — não aparecem no projeto.)

---

## 4. Camada 1: equipamentos e cabos

### 4.1 Os equipamentos

| Nome | Modelo | Função |
|---|---|---|
| **R-BORDA** | Roteador Cisco 2911 | o "cérebro": liga todas as VLANs, a DMZ e a Internet; aplica as ACLs; entrega DHCP |
| **ISP** | Roteador Cisco 2911 | simula o provedor de Internet |
| **SW-CORE** | Switch Cisco 2960 | switch central; liga roteador, servidores e os outros switches |
| **SW-A1** | Switch 2960 | switch de acesso de Pesquisa e Administrativo |
| **SW-A2** | Switch 2960 | switch de acesso da sala limpa (sequenciadores) |
| **SW-DMZ** | Switch 2960 | switch separado só para a DMZ |
| **SW-INTERNET** | Switch 2960 | parte da "Internet" simulada (sem configuração) |
| PCs e servidores | PC-PT / Server-PT | usuários, sequenciadores (simulados como PCs) e servidores |

**Roteador × switch, em uma frase:** o **switch** conecta aparelhos *da mesma rede* (usa endereço MAC); o **roteador** conecta *redes diferentes* (usa endereço IP).

### 4.2 Por que cada cabo

Um cabo de rede tem fios de **transmitir** e de **receber**. Equipamentos de tipos diferentes já "esperam" os fios invertidos; equipamentos do mesmo tipo, não. Por isso:

| Ligação | Cabo | Regra |
|---|---|---|
| PC/servidor ↔ switch | **Direto** (straight-through) | tipos diferentes |
| Roteador ↔ switch | **Direto** | tipos diferentes |
| Switch ↔ switch | **Cruzado** (crossover) | mesmo tipo |
| Roteador ↔ roteador (R-BORDA ↔ ISP) | **Cruzado** | mesmo tipo |

No Packet Tracer, cabo direto é a linha **contínua** e cruzado é a linha **tracejada**.

**Fast × Gigabit:** `FastEthernet` (Fa) = 100 Mbps; `GigabitEthernet` (Gi/G) = 1000 Mbps. Os switches 2960 têm 24 portas Fa e 2 portas Gi; usamos as Gi para ligar switches entre si.

---

## 5. Camada 2: switches, VLANs, trunk, STP e Port-Security

### 5.1 MAC e o que o switch faz

Toda placa de rede tem um **endereço MAC** de fábrica (ex.: `00E0.B100.1234`). O switch aprende "qual MAC está em qual porta" e manda cada quadro só para a porta certa.

### 5.2 VLAN — dividir um switch em vários

**VLAN (Virtual LAN)** é dividir um switch físico em vários switches "virtuais". Quem está na VLAN 10 não enxerga diretamente quem está na VLAN 20, mesmo estando no mesmo switch. Para conversar, precisa passar pelo roteador — e é lá que colocamos as regras de segurança.

Nossas VLANs:

| VLAN | Nome | Quem está |
|---|---|---|
| 10 | PESQUISA | PC-PESQ-01, PC-PESQ-02 (SW-A1 Fa0/1–12) |
| 20 | SEQUENCIADORES | SEQ-01, SEQ-02 (SW-A2 Fa0/1–10) |
| 30 | ADMINISTRATIVO | PC-ADM-01 (SW-A1 Fa0/13–20) |
| 40 | SERVIDORES | SRV-INTERNO, SRV-LIMS (SW-CORE Fa0/1–5) |
| 50 | DMZ | SRV-WEB-DMZ (SW-DMZ) |
| 99 | GERENCIA | PC-GERENCIA + o IP de gerência dos switches |
| 998 | NAO_UTILIZADAS | todas as portas sem uso (desligadas) |
| 999 | NATIVA | VLAN "vazia" usada como nativa nos trunks |

Como uma porta é colocada numa VLAN (exemplo do SW-A1):

```
interface FastEthernet0/1
 switchport mode access          ← porta de "acesso": um computador, uma VLAN
 switchport access vlan 10       ← esta porta é da VLAN 10 (Pesquisa)
```

**Veja no PT:** SW-A1 → CLI → `show vlan brief`.

### 5.3 Trunk — o "cabo que carrega várias VLANs"

Entre switches (e entre o SW-CORE e o roteador) passam dados de *todas* as VLANs pelo mesmo cabo. Para não misturar, cada quadro recebe uma **etiqueta (tag)** com o número da VLAN — isso é o padrão **IEEE 802.1Q**. Porta que faz isso = **porta trunk**.

```
interface GigabitEthernet0/1
 switchport mode trunk                          ← vira trunk
 switchport trunk native vlan 999               ← quadros sem etiqueta vão para a 999 (vazia)
 switchport trunk allowed vlan 10,20,30,40,99,999   ← só essas VLANs passam
 switchport nonegotiate                         ← não negocia trunk automaticamente (DTP)
```

**Por que VLAN nativa 999?** A VLAN nativa é a que passa *sem etiqueta*. Se fosse a VLAN 1 (padrão), um atacante poderia usar um truque chamado *VLAN hopping* (pular de VLAN). Usando uma VLAN sem ninguém, o truque não leva a lugar nenhum.

**Veja no PT:** SW-CORE → `show interfaces trunk`.

### 5.4 STP — evitar loops

Ligamos **SW-CORE ↔ SW-A1**, **SW-CORE ↔ SW-A2** e também **SW-A1 ↔ SW-A2**. Isso forma um **triângulo**. É bom porque, se um cabo cair, existe outro caminho (**redundância**). Mas em camada 2 um triângulo é perigoso: um quadro de broadcast ficaria girando para sempre (**tempestade de broadcast**) e derrubaria a rede.

O **Spanning Tree Protocol (STP)** resolve: os switches conversam, elegem um "chefe" (**Root Bridge**) e **bloqueiam** uma porta para quebrar o loop. Se um cabo cair, a porta bloqueada é liberada sozinha.

Usamos a versão rápida, **Rapid-PVST+** (uma árvore por VLAN, converge em poucos segundos).

**Como forçamos o resultado:**
- Vence a eleição o switch com **menor prioridade**. Padrão = 32768.
- SW-CORE: `spanning-tree vlan ... priority 24576` → **Root Bridge**.
- SW-A1: prioridade 28672 → segundo lugar.
- SW-A2: padrão 32768.
- No cabo entre SW-A1 e SW-A2, os dois estão "à mesma distância" da raiz; o desempate é pela menor prioridade → o SW-A1 ganha. Então **a porta Gi0/2 do SW-A2 fica bloqueada (Alternate/Blocking)**.

**Veja no PT:** SW-A2 → `show spanning-tree vlan 20` → a linha da `Gi0/2` mostra `Altn BLK`. No SW-CORE aparece *"This bridge is the root"*. É exatamente o print da seção 3.3 do relatório.

**PortFast e BPDU Guard** (nas portas de PCs): PortFast faz a porta de um PC ligar na hora (sem esperar o STP). BPDU Guard desliga a porta se alguém plugar um switch nela — evita que alguém crie um loop sem querer.

### 5.5 Port-Security — travar a porta pelo MAC

Impede que alguém tire o cabo de um PC e plugue outro aparelho.

```
switchport port-security                     ← liga a proteção
switchport port-security maximum 1           ← só 1 MAC nesta porta
switchport port-security mac-address sticky  ← "gruda" o primeiro MAC que aparecer
switchport port-security violation shutdown  ← se aparecer outro MAC, desliga a porta
```

Modos de violação:
- **shutdown** — desliga a porta (fica *err-disabled*, luz vermelha). Usamos nos **sequenciadores**, na DMZ e no PC da gerência: são equipamentos fixos, qualquer troca é suspeita.
- **restrict** — descarta o tráfego do intruso e registra, mas a porta continua funcionando. Usamos em Pesquisa, Administrativo (máx. 2 MACs) e Servidores.

**Veja no PT:** SW-A2 → `show port-security`.

---

## 6. Camada 3: endereço IP, máscara, sub-rede e VLSM

### 6.1 Endereço IPv4 e máscara

Um IPv4 tem 32 bits, escritos em 4 números de 0 a 255: `10.99.0.11`.
A **máscara** diz quantos desses bits identificam a **rede** e quantos identificam o **host** (o aparelho). `/24` = os primeiros 24 bits são rede = máscara `255.255.255.0`.

Em toda sub-rede:
- o **primeiro** endereço é o **endereço de rede** (não pode ser usado por aparelho);
- o **último** é o **broadcast** (mensagem para todos; também não pode);
- o resto são **hosts válidos**. Fórmula: **hosts = 2^(bits de host) − 2**.

| CIDR | Máscara | Bits de host | Endereços | Hosts válidos |
|---|---|---|---|---|
| /24 | 255.255.255.0 | 8 | 256 | 254 |
| /25 | 255.255.255.128 | 7 | 128 | 126 |
| /26 | 255.255.255.192 | 6 | 64 | 62 |
| /27 | 255.255.255.224 | 5 | 32 | 30 |
| /28 | 255.255.255.240 | 4 | 16 | 14 |
| /30 | 255.255.255.252 | 2 | 4 | 2 |
| /21 | 255.255.248.0 | 11 | 2048 | 2046 |

### 6.2 O bloco que recebemos: 10.99.0.0/21

`/21` = 2.048 endereços, de **10.99.0.0 até 10.99.7.255** (8 "blocos de /24": 10.99.0.x, 10.99.1.x … 10.99.7.x).

### 6.3 VLSM — cada setor com o tamanho certo

**VLSM** (*Variable Length Subnet Mask*) = usar máscaras de tamanhos diferentes para cada setor, em vez de dar um /24 para todo mundo. Assim não desperdiçamos endereços.

**Regra de ouro:** ordene do **maior** setor para o **menor** e vá encaixando um depois do outro.

Passo a passo (é assim que você explica no quadro):

1. **Pesquisa — 200 hosts.** Qual a menor sub-rede que cabe 200? 2^7−2 = 126 (não cabe); 2^8−2 = 254 (cabe) → 8 bits de host → **/24**.
   Começa no início do bloco: **10.99.0.0/24** → rede `.0`, hosts `10.99.0.1–10.99.0.254`, broadcast `10.99.0.255`.
2. **Administrativo — 90 hosts.** 2^7−2 = 126 ≥ 90 → **/25** (128 endereços).
   Próximo endereço livre: 10.99.1.0 → **10.99.1.0/25** → hosts `.1–.126`, broadcast `.127`.
3. **Sequenciadores — 40 hosts.** 2^6−2 = 62 ≥ 40 → **/26** (64 endereços).
   Próximo livre: 10.99.1.128 → **10.99.1.128/26** → hosts `.129–.190`, broadcast `.191`.
4. **Servidores — 20 hosts.** 2^5−2 = 30 ≥ 20 → **/27** (32).
   Próximo livre: 10.99.1.192 → **10.99.1.192/27** → hosts `.193–.222`, broadcast `.223`.
5. **DMZ — 10 hosts.** 2^4−2 = 14 ≥ 10 → **/28** (16).
   **10.99.1.224/28** → hosts `.225–.238`, broadcast `.239`.
6. **Gerência — 10 hosts.** /28 → **10.99.1.240/28** → hosts `.241–.254`, broadcast `.255`.
7. **Link WAN — 2 hosts** (só os dois roteadores). 2^2−2 = 2 → **/30** (4).
   **10.99.2.0/30** → R-BORDA `.1`, ISP `.2`, broadcast `.3`.

Usamos de `10.99.0.0` até `10.99.2.3`. **Sobra de 10.99.2.4 até 10.99.7.255 ≈ 1.532 endereços = 75% do bloco** para o laboratório crescer.

**Convenção:** o **primeiro IP válido de cada sub-rede é o gateway** (o endereço do roteador naquela rede): 10.99.0.1, 10.99.1.1, 10.99.1.129 …

**Truque para conferir se um IP pertence a uma sub-rede:** o "tamanho do bloco" é 256 − último número da máscara. Para /26 (255.255.255.**192**) → 256−192 = **64** → as redes começam em .0, .64, .128, .192. O IP 10.99.1.130 está no bloco que começa em .128 → rede 10.99.1.128/26 ✔.

---

## 7. IPv6 e Dual-Stack

### 7.1 Por que IPv6

O IPv4 acabou no mundo; o IPv6 tem 128 bits (endereços enormes, escritos em hexadecimal). **Dual-Stack** = cada aparelho tem IPv4 **e** IPv6 ao mesmo tempo.

### 7.2 Como ler um IPv6

`2001:0db8:0b10:0010:0000:0000:0000:0001` pode ser abreviado:
- zeros à esquerda de cada grupo somem: `2001:db8:b10:10:0:0:0:1`;
- uma sequência de grupos de zeros vira `::` (só uma vez): **`2001:db8:b10:10::1`**.

`2001:db8::/32` é a faixa reservada para **documentação/estudo** (como a nossa).

### 7.3 Nosso plano

Recebemos um **/48**: `2001:db8:b10::/48`. Cada VLAN ganha um **/64** (o tamanho padrão de rede local em IPv6). Truque de organização: **o 4º grupo é o número da VLAN**:

| VLAN | Prefixo | Gateway |
|---|---|---|
| 10 | 2001:db8:b10:**10**::/64 | 2001:db8:b10:10::1 |
| 20 | 2001:db8:b10:**20**::/64 | 2001:db8:b10:20::1 |
| 30 | 2001:db8:b10:**30**::/64 | 2001:db8:b10:30::1 |
| 40 | 2001:db8:b10:**40**::/64 | 2001:db8:b10:40::1 |
| 50 (DMZ) | 2001:db8:b10:**50**::/64 | 2001:db8:b10:50::1 |
| 99 | 2001:db8:b10:**99**::/64 | 2001:db8:b10:99::1 |
| WAN | 2001:db8:b10:**ff**::/64 | R-BORDA ::1 / ISP ::2 |

Todas as interfaces do roteador também têm o endereço **link-local** `fe80::1` (endereço que só vale dentro do próprio cabo/VLAN, usado pelo IPv6 para os vizinhos se acharem).

### 7.4 Como os PCs pegam IPv6

- **SLAAC com EUI-64** (Pesquisa e Administrativo): o roteador anuncia "a rede aqui é 2001:db8:b10:10::/64" (*Router Advertisement*). O PC monta sozinho o resto do endereço a partir do seu MAC (EUI-64: divide o MAC no meio, coloca `FFFE` e inverte um bit). Ninguém precisa configurar nada.
- **Estático** (servidores, DMZ, gerência e **sequenciadores**): digitado à mão. Nos sequenciadores é estático para que as regras de segurança saibam exatamente quem são.

Para o roteador fazer IPv6, foi preciso o comando global `ipv6 unicast-routing`.

**Veja no PT:** PC-PESQ-01 → Desktop → Command Prompt → `ipconfig /all`.

---

## 8. Roteamento: Router-on-a-Stick e rotas padrão

### 8.1 Router-on-a-Stick

As VLANs 10, 20, 30, 40 e 99 chegam ao R-BORDA **por um único cabo** (o trunk da G0/0). O roteador divide essa porta física em várias **subinterfaces** virtuais — uma por VLAN — cada uma com o gateway daquela VLAN. Como é um roteador "pendurado" num cabo só, o nome é *router-on-a-stick*.

```
interface GigabitEthernet0/0.10          ← subinterface ".10"
 encapsulation dot1Q 10                  ← "eu cuido dos quadros com etiqueta 10"
 ip address 10.99.0.1 255.255.255.0      ← gateway da VLAN 10
 ipv6 address 2001:db8:b10:10::1/64
```

A **DMZ** não usa subinterface: tem a porta física **G0/1** só para ela. Assim o tráfego público nunca passa pelo trunk interno.

A alternativa seria um **switch de camada 3** (ex.: 3560) com `ip routing` e interfaces `interface vlan 10`. Escolhemos router-on-a-stick por ser o método da matéria e porque o roteador também faz DHCP e ACL.

### 8.2 Rotas padrão

O R-BORDA conhece sozinho as redes ligadas nele. Para todo o resto (Internet), usamos a **rota padrão** — "se não sabe para onde mandar, mande para o ISP":

```
ip route 0.0.0.0 0.0.0.0 10.99.2.2
ipv6 route ::/0 2001:db8:b10:ff::2
```

O ISP, por sua vez, precisa saber voltar: `ip route 10.99.0.0 255.255.248.0 10.99.2.1` (todo o nosso /21 está atrás do R-BORDA).

**Veja no PT:** R-BORDA → `show ip route` (as redes com **C** são diretamente conectadas; a com **S\*** é a rota padrão estática).

---

## 9. Camada 4: ACLs — o coração do nosso cenário

### 9.1 O que é uma ACL

**ACL (Access Control List)** é uma lista de regras que o roteador confere, **de cima para baixo**, em cada pacote. Na **primeira** regra que bater, ele faz o que ela manda (`permit` = deixa passar, `deny` = descarta) e para de ler. **No final de toda ACL existe um `deny` invisível** (*deny implícito*): o que não foi permitido, é bloqueado.

- **ACL padrão** (numeradas 1–99): olha **só a origem**. Usamos uma para o SSH.
- **ACL estendida** (as nossas principais): olha **origem, destino, protocolo e porta**. É por isso que são "Camada 4": filtram por **porta TCP/UDP**.

### 9.2 Portas que aparecem

| Porta | Protocolo | Serviço |
|---|---|---|
| TCP 20/21 | FTP | transferência de arquivos (os sequenciadores enviam os dados por aqui) |
| TCP 80 | HTTP | site |
| TCP 443 | HTTPS | site com criptografia |
| UDP 53 | DNS | traduz nomes em IP |
| UDP 67/68 | DHCP | entrega de IP |
| UDP 123 | NTP | sincroniza o relógio |
| UDP 514 | Syslog | envio de logs |
| TCP 22 | SSH | acesso remoto seguro |

### 9.3 Wildcard — a "máscara ao contrário"

ACLs usam **wildcard** em vez de máscara: é a máscara invertida (255 − cada número).
- /26 → máscara 255.255.255.192 → wildcard **0.0.0.63**
- /28 → 255.255.255.240 → **0.0.0.15**
- /21 → 255.255.248.0 → **0.0.7.255**
- `host 10.99.1.195` = exatamente aquele IP (wildcard 0.0.0.0).
- `any` = qualquer endereço.

Então `10.99.1.128 0.0.0.63` significa "qualquer IP de 10.99.1.128 a 10.99.1.191" = a VLAN dos sequenciadores.

### 9.4 ACL 1 — Zero-Trust dos sequenciadores (a principal)

```
ip access-list extended ACL_SEQ_ZEROTRUST
 permit tcp 10.99.1.128 0.0.0.63 host 10.99.1.195 eq 21    ← FTP para o LIMS
 permit tcp 10.99.1.128 0.0.0.63 host 10.99.1.195 eq 20    ← dados do FTP
 permit tcp 10.99.1.128 0.0.0.63 host 10.99.1.195 eq 80    ← site do LIMS
 permit udp 10.99.1.128 0.0.0.63 host 10.99.1.194 eq 123   ← relógio (NTP)
 permit udp 10.99.1.128 0.0.0.63 host 10.99.1.194 eq 514   ← logs (Syslog)
 deny ip any any                                           ← TODO O RESTO: bloqueado

interface GigabitEthernet0/0.20
 ip access-group ACL_SEQ_ZEROTRUST in
```

Lendo em português: *"Da rede dos sequenciadores, só pode: FTP e HTTP para o LIMS, NTP e Syslog para o SRV-INTERNO. Mais nada."*

**Onde e em que sentido foi aplicada — e por quê:**
- Na **subinterface G0/0.20** (a porta "virtual" da VLAN 20), no sentido **`in`** = pacotes *entrando* no roteador vindos dos sequenciadores.
- Regra da Cisco: **ACL estendida vai o mais perto possível da origem**. Assim o pacote proibido morre logo na chegada e nem atravessa a rede.

**Detalhe importante:** como o `ping` usa **ICMP** (não é TCP 80 nem 21), ele é bloqueado. Por isso, no teste, o sequenciador **não consegue pingar** o LIMS, **mas consegue abrir o site** do LIMS. É exatamente isso que o print da seção 5.3 tem que mostrar.

**A mesma regra em IPv6 (ACL6_SEQ_ZEROTRUST):** se bloqueássemos só o IPv4, o sequenciador poderia "fugir" pelo IPv6. Então a regra foi duplicada:

```
ipv6 access-list ACL6_SEQ_ZEROTRUST
 permit tcp 2001:db8:b10:20::/64 host 2001:db8:b10:40::11 eq 21
 permit tcp 2001:db8:b10:20::/64 host 2001:db8:b10:40::11 eq 20
 permit tcp 2001:db8:b10:20::/64 host 2001:db8:b10:40::11 eq 80
 permit icmp any any nd-ns        ← "vizinho, qual é o seu MAC?" (necessário no IPv6)
 permit icmp any any nd-na        ← resposta do vizinho
 deny ipv6 any any
interface GigabitEthernet0/0.20
 ipv6 traffic-filter ACL6_SEQ_ZEROTRUST in
```

(No IPv6, o equivalente ao ARP é o *Neighbor Discovery* via ICMPv6 — se bloquearmos, nada funciona. Por isso as duas linhas `nd-ns` e `nd-na`.)

### 9.5 ACL 2 — entrada da Internet (ACL_WAN_IN, na G0/2 `in`)

```
 permit tcp any host 10.99.1.226 eq 80          ← qualquer um da Internet pode ver o site da DMZ
 permit tcp any host 10.99.1.226 eq 443
 permit tcp any 10.99.0.0 0.0.7.255 established ← respostas de conexões que NÓS abrimos
 permit icmp any 10.99.0.0 0.0.7.255 echo-reply ← respostas de ping que NÓS fizemos
 permit icmp any 10.99.0.0 0.0.7.255 unreachable
 deny ip any any                                ← o resto da Internet não entra
```

**`established`** é o truque: num pacote TCP de *resposta*, os bits ACK ou RST estão ligados. Essa palavra deixa passar só pacotes com esses bits — ou seja, a Internet **responde**, mas não consegue **iniciar** conversa com a rede interna.

### 9.6 ACL 3 — DMZ isolada (ACL_DMZ_IN, na G0/1 `in`)

```
 permit tcp host 10.99.1.226 eq 80 any established   ← o site pode RESPONDER a quem acessou
 permit tcp host 10.99.1.226 eq 443 any established
 permit icmp 10.99.1.224 0.0.0.15 any echo-reply
 deny ip 10.99.1.224 0.0.0.15 10.99.0.0 0.0.7.255    ← a DMZ nunca INICIA conversa com a rede interna
 permit ip 10.99.1.224 0.0.0.15 any                  ← mas pode sair para a Internet (ex.: atualizações)
```

Ideia: se um hacker invadir o site público, ele **não consegue** partir dele para atacar o laboratório.

### 9.7 ACL do SSH

```
access-list 10 permit 10.99.1.240 0.0.0.15
line vty 0 4
 access-class 10 in
```

Só a VLAN 99 (gerência) pode abrir sessão remota nos equipamentos.

**Veja no PT:** R-BORDA → `show access-lists`. Depois de fazer os testes, cada linha mostra `(N match(es))` — prova de que a regra foi usada.

---

## 10. Camada 7: os serviços

### 10.1 DHCP — entrega IP automático

Configurado **no próprio R-BORDA** para as VLANs 10 e 30:

```
ip dhcp excluded-address 10.99.0.1 10.99.0.10    ← reserva .1 a .10 (gateway e futuros fixos)
ip dhcp pool POOL_PESQUISA
 network 10.99.0.0 255.255.255.0
 default-router 10.99.0.1                        ← gateway que o PC recebe
 dns-server 10.99.1.194                          ← DNS que o PC recebe
 domain-name biogen.local
```

Então o primeiro PC da Pesquisa recebe **10.99.0.11**. Funciona em 4 mensagens (**DORA**): *Discover* (PC grita "alguém me dá um IP?"), *Offer* (roteador oferece), *Request* (PC aceita), *Ack* (roteador confirma). Usa UDP 67/68.

Sequenciadores, servidores e DMZ usam **IP fixo** (servidor e equipamento crítico não podem mudar de IP).

**Veja no PT:** PC-PESQ-01 → Desktop → IP Configuration → marcado **DHCP**. No R-BORDA: `show ip dhcp binding`.

### 10.2 DNS — nome → IP (SRV-INTERNO, 10.99.1.194)

Registros cadastrados (Services → DNS):

| Nome | IP |
|---|---|
| www.biogen.local | 10.99.1.226 (site da DMZ) |
| biogen.local | 10.99.1.226 |
| lims.biogen.local | 10.99.1.195 (LIMS) |

### 10.3 HTTP/HTTPS — os sites

- **SRV-WEB-DMZ (10.99.1.226):** "BioGen – Divulgação Científica" (HTTP e HTTPS ligados).
- **SRV-LIMS (10.99.1.195):** "BioGen LIMS – Repositório de Dados Genômicos" (só interno).
- **SRV-INTERNET (203.0.113.100):** site "externo" para testar a saída para a Internet.

A página pode ser editada em Services → HTTP → `index.html` → Edit.

### 10.4 FTP — envio dos dados de sequenciamento

SRV-LIMS com FTP ligado e usuário **sequenciador / seq123**. É por FTP que um sequenciador manda os arquivos de DNA (no mundo real, arquivos FASTQ de vários GB).

### 10.5 NTP e Syslog

- **NTP** (relógio): todos os equipamentos com o mesmo horário — essencial para comparar logs.
- **Syslog**: o roteador manda seus registros (logs) para o SRV-INTERNO (`logging host 10.99.1.194`).

---

## 11. Segurança extra

| Comando | Para que serve |
|---|---|
| `enable secret BioGen@2026` | senha do modo privilegiado, guardada com criptografia |
| `service password-encryption` | esconde as senhas que aparecem no `show run` |
| `username admin privilege 15 secret ...` | usuário local para login |
| `banner motd # ... #` | mensagem de aviso de "acesso restrito" |
| `transport input ssh` | acesso remoto só por SSH (Telnet manda senha em texto puro) |
| `ip ssh version 2` | versão segura do SSH |
| `no ip domain-lookup` | evita que um comando digitado errado trave o CLI tentando achar um "nome" |
| portas sem uso em `shutdown` + VLAN 998 | ninguém pluga um cabo numa porta livre e entra na rede |

---

## 12. Roteiro de testes

Faça nesta ordem, depois de esperar os links ficarem verdes. Para abrir o terminal de um PC: clique no PC → **Desktop → Command Prompt**. Para o navegador: **Desktop → Web Browser**.

| # | Onde | O que fazer | O que deve aparecer | Por que |
|---|---|---|---|---|
| 1 | PC-PESQ-01 | `ipconfig /all` | IPv4 `10.99.0.11` (ou parecido), gateway `10.99.0.1`, DNS `10.99.1.194` e um IPv6 `2001:DB8:B10:10:...` | DHCP + SLAAC funcionando |
| 2 | PC-PESQ-01 | `ping 10.99.1.11` (o IP do PC-ADM-01 — confira com `ipconfig` nele) | Reply | roteamento entre VLANs. O 1º pacote às vezes falha (ARP) — repita |
| 3 | PC-PESQ-01 | `nslookup www.biogen.local` | Address: 10.99.1.226 | DNS ok — **print da seção 6** |
| 4 | PC-PESQ-01 | navegador: `http://www.biogen.local` | página "BioGen – Divulgação Científica" | Web + DNS + roteamento até a DMZ — **print da seção 6** |
| 5 | SEQ-01 | `ping 10.99.1.195` | `Destination host unreachable` | ACL bloqueia ICMP — **print da seção 5.3** |
| 6 | SEQ-01 | navegador: `http://10.99.1.195` | página do LIMS abre | ACL libera TCP 80 — **print da seção 5.3** |
| 7 | SEQ-01 | `ping 203.0.113.100` | unreachable | sequenciador sem Internet |
| 8 | SEQ-01 | `ftp 10.99.1.195` → usuário `sequenciador`, senha `seq123` | login OK | ACL libera FTP |
| 9 | SEQ-01 | `ping 2001:db8:b10:40::11` | falha | bloqueio também no IPv6 |
| 10 | PC-INTERNET | navegador: `http://10.99.1.226` | site da DMZ | site publicado na Internet |
| 11 | PC-INTERNET | `ping 10.99.0.11` | falha | Internet não entra na rede interna |
| 12 | SRV-WEB-DMZ | `ping 10.99.1.194` | falha | DMZ isolada |
| 13 | PC-PESQ-01 | navegador: `http://203.0.113.100` | site externo | usuários normais têm Internet (`established`) |
| 14 | SW-A2 (CLI) | `show spanning-tree vlan 20` | Gi0/2 `Altn BLK` | STP — **print da seção 3.3** |
| 15 | SW-CORE (CLI) | `show spanning-tree vlan 20` | "This bridge is the root" | Root Bridge |
| 16 | PC-GERENCIA | `ssh -l admin 10.99.1.242` | pede senha e entra | SSH da gerência (precisa ter gerado a chave RSA — item 1) |
| 17 | PC-PESQ-01 | `ssh -l admin 10.99.1.242` | conexão recusada | `access-class` bloqueia quem não é da gerência |
| 18 | R-BORDA (CLI) | `show access-lists` | contadores "matches" nas linhas | prova das ACLs |
| 19 | SW-A2 (CLI) | `show port-security` | Fa0/1–10 MaxSecureAddr 1, Shutdown | port-security |

**Por que o ping falha com "Destination host unreachable" e não com "Request timed out"?** Quando a ACL descarta um pacote, o roteador avisa a origem com uma mensagem ICMP "proibido administrativamente". O Packet Tracer mostra isso como *Destination host unreachable* vindo de `10.99.1.129` (o gateway).

**Teste opcional que impressiona:** arraste um PC novo, ligue no SW-A2 Fa0/3 (que está desligada) — nada acontece, porta em shutdown. Agora troque o SEQ-01 de lugar com outro PC na Fa0/1: a porta vai para *err-disabled* (vermelha) por causa do port-security.

**Como tirar os prints:** use a ferramenta de recorte do Windows (**Win + Shift + S**) e cole no relatório.

---

## 13. Perguntas que o professor pode fazer (com respostas)

**Por que vocês usaram VLSM?**
Para dar a cada setor só o número de endereços que ele precisa. Com /24 para todos, gastaríamos 7 blocos de 256; com VLSM usamos menos de 3 e sobrou 75% do /21.

**Por que a ACL dos sequenciadores está na G0/0.20 e em `in`?**
ACL estendida deve ficar o mais perto da origem. A G0/0.20 é por onde o tráfego dos sequenciadores entra no roteador, então o pacote proibido é descartado na chegada.

**O que acontece se tirar a linha `deny ip any any` da ACL?**
Nada muda: toda ACL tem um *deny implícito* no final. Colocamos explícito para ficar claro (e para ver o contador de matches no `show access-lists`).

**Por que o sequenciador não consegue pingar o servidor, mas abre o site?**
Porque a ACL permite TCP porta 80 (HTTP) e 21/20 (FTP), e o ping é ICMP, que cai no deny.

**Por que fizeram ACL também em IPv6?**
Em Dual-Stack o aparelho tem as duas "estradas". Bloquear só o IPv4 deixaria o IPv6 aberto como rota de fuga.

**O que é a DMZ e por que ela tem uma porta própria?**
É a zona para servidores acessados pela Internet. Fica numa interface física separada (G0/1) e com ACL que impede ela de iniciar conexões para dentro. Se o site for invadido, o atacante não alcança o laboratório.

**Para que serve o `established`?**
Deixa passar só pacotes TCP de resposta (com bit ACK/RST). Assim os usuários internos navegam, mas ninguém da Internet abre conexão com a rede interna.

**Qual switch é a Root Bridge e qual porta está bloqueada? Por quê?**
SW-CORE (prioridade 24576, a menor). Bloqueada: Gi0/2 do SW-A2 — no enlace SW-A1–SW-A2 os dois têm o mesmo custo até a raiz, e o desempate pela prioridade favorece o SW-A1 (28672).

**Qual a diferença entre `shutdown` e `restrict` no port-security?**
`shutdown` desliga a porta inteira; `restrict` só descarta o tráfego do MAC intruso e registra. Usamos shutdown onde o equipamento é fixo (sequenciadores).

**Por que a VLAN nativa é a 999?**
Para evitar VLAN hopping: a nativa passa sem etiqueta; usando uma VLAN sem hosts, o ataque não leva a lugar nenhum.

**Por que cabo cruzado entre os switches?**
Equipamentos do mesmo tipo transmitem e recebem nos mesmos pinos; o cruzado inverte os pares.

**Como os PCs recebem IPv6?**
Por SLAAC: o roteador anuncia o prefixo /64 e o PC completa o endereço com EUI-64 (baseado no MAC).

**Onde está o ponto único de falha (SPOF)?**
No R-BORDA (gateway, DHCP, firewall e borda ao mesmo tempo), no trunk único até ele, no SW-CORE e no link único com o ISP.

**Como vocês melhorariam?**
Segundo roteador com HSRP (gateway virtual), EtherChannel (LACP) entre switches, firewall dedicado (pfSense/FortiGate) com inspeção stateful, link de Internet duplo e servidores replicados.

**Como a rede cresce?**
Sobram 10.99.2.4 a 10.99.7.255. Um novo setor = nova VLAN + nova subinterface + novo pool DHCP + novo /64 IPv6 (ex.: VLAN 60 → 2001:db8:b10:60::/64), sem mexer no que já existe.

---

## 14. Glossário rápido

| Termo | Significado simples |
|---|---|
| **ACL** | lista de regras permit/deny no roteador |
| **Broadcast** | mensagem para todos da rede |
| **CIDR (/24, /26…)** | jeito curto de escrever a máscara |
| **DHCP** | entrega IP automático |
| **DMZ** | área para servidores públicos, separada da rede interna |
| **DNS** | traduz nome em IP |
| **Dual-Stack** | IPv4 e IPv6 ao mesmo tempo |
| **EUI-64** | jeito de montar o IPv6 a partir do MAC |
| **Gateway** | o roteador da sua rede; para onde vai tudo que é de fora |
| **Host** | qualquer aparelho com IP |
| **ISP** | provedor de Internet |
| **LIMS** | sistema de gestão de laboratório (onde ficam os dados genéticos) |
| **MAC** | endereço físico da placa de rede |
| **Port-Security** | trava a porta do switch por MAC |
| **Root Bridge** | switch "chefe" do STP |
| **Router-on-a-Stick** | roteador com várias subinterfaces num único cabo trunk |
| **SLAAC** | IPv6 automático sem servidor |
| **SPOF** | ponto único de falha |
| **STP** | protocolo que impede loops entre switches |
| **Sub-rede** | pedaço de uma rede maior |
| **Trunk / 802.1Q** | cabo que carrega várias VLANs com etiqueta |
| **VLAN** | rede virtual dentro do switch |
| **VLSM** | sub-redes de tamanhos diferentes |
| **Wildcard** | máscara invertida usada nas ACLs |
| **Zero-Trust** | bloquear tudo e liberar só o estritamente necessário |

---

## 15. Sugestão de divisão para a apresentação

| Quem | Parte | Slides / seções do relatório |
|---|---|---|
| **Luiz (líder)** | Abertura, cenário e decisões finais | Slides 1, 2 e 4 · Seções 1 e 7 |
| Vinícius | Camadas 1 e 2: cabos, VLANs, trunk | Seção 3.1 e 3.2 |
| Leonardo | STP e Port-Security | Seção 3.3 |
| Gabriel | VLSM e IPv6 (fazer a conta no quadro) | Seção 4 |
| Matheus | Router-on-a-Stick e ACLs (demonstração do ping bloqueado × site liberado) | Seção 5 |
| Artur | Serviços (DHCP, DNS, Web, FTP) e demonstração no navegador | Seção 6 |

**Dica:** cada um deve conseguir explicar, com suas palavras, a pergunta da seção 13 da sua parte. Treinem fazendo os testes da seção 12 ao vivo no Packet Tracer.
