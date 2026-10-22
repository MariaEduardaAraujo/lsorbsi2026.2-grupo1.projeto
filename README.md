# Relatório de Projeto — Rede Virtualizada com 8 VMs e Gateway NAT (Etapa 1)

> **Disciplina:** LSOR <br>
> **Turma:** bsi-26-2 (2026.2) <br>
> **Professor(a):** Alaelson Jatobá <br>
> **Aluno(a):** Daniel do Carmo, Davi da Silva, João Pedro de Castro Laranjeira e Maria Eduarda de Araújo <br>
> **Grupo:** 1  <br>
> **Datas:** 14/10/2026 <br>

## 1. Ambiente

| Item                | VM1 — Gateway       | VM2-VM8 — Cliente    |
| ------------------- | ------------------- | -------------------- |
| Sistema operacional | Ubuntu Server 26.04 | Ubuntu Server 26.04  |
| Plataforma          | VirtualBox          | VirtualBox           |
| Função              | Gateway             | Cliente              |
| Interface interna   | `enp0s8`            | intnet               |
| Interface externa   | `enp0s3`            | -                    |
| Rede interna        | `172.20.23.67/22`   | `192.168.26.18-25/28`|
| Gateway interno     | `172.20.20.1`       | `192.168.26.17`      |

## 2. Objetivo

O objetivo desta atividade foi montar um ambiente de rede com **8 máquinas virtuais Ubuntu Server (VM1 a VM8)** e **1 VM gateway**, totalizando 9 VMs. O gateway possui duas placas de rede: a **WAN** (`enp0s3`), ligada à rede do laboratório (`172.20.20.1/22`), e a **LAN** (`enp0s8`), ligada à rede do grupo (`192.168.26.16/28`). Toda a saída das VMs para o laboratório e para a internet passa por ele.

Foram realizados:
1. Criação das 9 VMs e configuração de IP estático com Netplan.
2. Habilitação do encaminhamento IPv4 e criação de regras de **NAT/MASQUERADE** com IPTables no gateway, com persistência após reinicialização.
3. Configuração de hostnames (FQDN), arquivo `/etc/hosts` e usuários dos integrantes em todas as VMs.
4. Ativação do SSH e testes de conectividade, resolução de nomes, acesso remoto e rastreamento de rotas.

Este documento reúne os conceitos das Aulas 01, 02, 04, 05, 06, 07 e 08 em um único ambiente e serve também como tutorial para reproduzir o projeto.

## 3. Ambiente
### 3.1 Rede física do laboratório

| Parâmetro | Valor |
|:--|:--|
| Rede do laboratório          | `172.20.20.0/22`        |
| Gateway / DNS do laboratório | `172.20.20.1`           |
| Faixa para IPs estáticos     | `172.20.23.67`          |

### 3.2 Hardware padrão de cada VM

| Item                        | Especificação       |
| --------------------------- | ------------------- |
| Sistema operacional         | Ubuntu Server 26.04 |
| Plataforma de virtualização | VirtualBox          |
| Memória RAM                 | 2 GB                |
| Processadores               | 1 vCPU              |
| Armazenamento               | 32 GB               |
| Rede                        | -                   |
| Computador hospedeiro       | Windows 11          |

### 3.3 Endereçamento do grupo (faixa 16–31)
A turma usa `192.168.26.0/24` dividida em sub-redes `/28`. O Grupo 1 usa `192.168.26.16-31/28`.

| Parâmetro | Valor |
|:--|:--|
| Endereço da rede | `192.168.26.16` |
| Máscara | `255.255.255.240` (`/28`) |
| Primeiro IP útil (gateway, LAN) | `192.168.26.17` |
| VM1 a VM8 | `192.168.26.18` a `192.168.26.25` |
| IPs livres | `192.168.26.26` a `192.168.26.30` |
| Broadcast | `192.168.26.31` |
| Total de hosts úteis | 14 (2⁴ − 2) |

O `/28` tem 16 endereços (32 − 28 = 4 bits de host). O primeiro (.16) é a rede e o último (.31) é o broadcast.

### 3.4 Nomes, interfaces e IPs

Hostname curto: `vmZ`, ou `vm-gateway` para o gateway, sempre em minúsculas. FQDN: hostname + `.grupo1.bsi-26-2.maceio.lab`.

| Host | Placa / modo VirtualBox | Interface | IP / máscara | Gateway padrão | FQDN |
|:--|:--|:--|:--|:--|:--|
| vm-gateway | Adaptador 1 · Bridge (WAN) | enp0s3 | 172.20.23.67/22 | 172.20.20.1 | vm-gateway.grupo1.bsi-26-2.maceio.lab|
| vm-gateway | Adaptador 2 · Bridge (LAN) | enp0s8 | 192.168.26.17/28 | — | — |
| vm1 (VM1) | Adaptador 1 · Bridge | enp0s3 | 192.168.26.18/28 | 192.168.26.17 | vm1.grupo1.bsi-26-2.maceio.lab |
| vm2 (VM2) | Adaptador 1 · Bridge | enp0s3 | 192.168.26.19/28 | 192.168.26.17 | vm2.grupo1.bsi-26-2.maceio.lab |
| vm3 (VM3) | Adaptador 1 · Bridge | enp0s3 | 192.168.26.20/28 | 192.168.26.17 | vm3.grupo1.bsi-26-2.maceio.lab |
| vm4 (VM4) | Adaptador 1 · Bridge | enp0s3 | 192.168.26.21/28 | 192.168.26.17 | vm4.grupo1.bsi-26-2.maceio.lab |
| vm5 (VM5) | Adaptador 1 · Bridge | enp0s3 | 192.168.26.22/28 | 192.168.26.17 | vm5.grupo1.bsi-26-2.maceio.lab |
| vm6 (VM6) | Adaptador 1 · Bridge | enp0s3 | 192.168.26.23/28 | 192.168.26.17 | vm6.grupo1.bsi-26-2.maceio.lab |
| vm7 (VM7) | Adaptador 1 · Bridge | enp0s3 | 192.168.26.24/28 | 192.168.26.17 | vm7.grupo1.bsi-26-2.maceio.lab |
| vm8 (VM8) | Adaptador 1 · Bridge | enp0s3 | 192.168.26.25/28 | 192.168.26.17 | vm8.grupo1.bsi-26-2.maceio.lab |

### 3.5 Topologia

São 4 PCs do laboratório, com 2 VMs em cada um. O gateway é uma VM extra no PC1.

```text
                  Internet
                     |
          Gateway do laboratório
          172.20.20.1  (rede 172.20.20.0/22)
                     |
   ======= switch físico do laboratório =======
      |         |          |          |
     PC1       PC2        PC3        PC4
   +-------+  +-------+  +-------+  +-------+
   | gt    |  | vm3   |  | vm5   |  | vm7   |
   | vm1   |  | vm4   |  | vm6   |  | vm8   |
   | vm2   |  +-------+  +-------+  +-------+
   +-------+

  Gateway (g1-pc1-gw)
    enp0s3 (WAN) = 172.20.23.67/22   -> rede do laboratório
    enp0s8 (LAN) = 192.168.26.17/28 -> rede do grupo

  Rede do grupo 1: 192.168.26.16/28
    VM1..VM8 = 192.168.26.18 .. 192.168.26.25
    rota padrão de todas as VMs = 192.168.26.17
```

**Figura 1 – Topologia lógica e endereçamento do Grupo 1**

<img width="1800" height="1604" alt="topologia-logica-etapa1" src="https://github.com/user-attachments/assets/a2ba2103-a5af-42c1-bea3-097961819022" />

**Figura 2 – Topologia física (4 PCs, 2 VMs em cada, gateway no PC1)**

<img width="1780" height="830" alt="topologia-fisica-etapa1" src="https://github.com/user-attachments/assets/9461fcd7-5357-4c57-9025-9b3d11313ea0" />

## 4. Procedimento

### 4.1 Criação das VMs

1. Copiar a ISO do Ubuntu Server para `C:\2026\BSI\VM\original`.
2. Criar as VMs em `C:\2026\BSI\VM\<NomeDoAluno>`, com 2048 MB de RAM, 1 vCPU e disco de 32 GB (VDI, dinâmico).
3. Instalar o Ubuntu Server 26.04 com particionamento LVM (`/` 29 GB, `/boot` 1 GB, swap 2 GB) e usuário `administrador`.
4. No VirtualBox, configurar as placas:
   - **Gateway:** Adaptador 1 e Adaptador 2 em *Placa em Ponte* (mesma placa física do host).
   - **VM1 a VM8:** Adaptador 1 em *Placa em Ponte*.
5. Se o teclado estiver desconfigurado, usar `sudo dpkg-reconfigure keyboard-configuration` (ABNT2).

**Figura 3 – Adaptadores de rede do gateway no VirtualBox**

<img width="908" height="540" alt="7  vm gateway - print placa de rede 1" src="https://github.com/user-attachments/assets/74764615-244b-4094-b2c7-34392ba714d4" />

<img width="909" height="547" alt="8  vm gateway - print placa de rede 2" src="https://github.com/user-attachments/assets/ae125e04-eb52-46c6-a9a9-23dd0177029c" />

**Figura 4 – Adaptador de rede de uma VM do grupo no VirtualBox**
<img width="889" height="518" alt="Captura de tela 2026-10-21 195317" src="https://github.com/user-attachments/assets/e5bf0e53-7072-4c68-8e2a-0e6df1083958" />


### 4.2 Gateway — Netplan com duas placas 
Confirmar os nomes das interfaces e escolher um IP livre na rede do laboratório:

```bash
ip -br link
ping -c 2 172.20.23.X     # deve FALHAR: o IP está livre
```

Editar o arquivo de Netplan (`ls /etc/netplan` mostra o nome real; neste projeto usa-se `00-installer-config.yaml`):

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:            # WAN - rede do laboratório
      dhcp4: false
      addresses:
        - 172.20.23.67/22   # IP livre escolhido (Aula 06)
      routes:
        - to: default
          via: 172.20.20.1
      nameservers:
        addresses: [172.20.20.1, 1.1.1.1, 8.8.8.8]
    enp0s8:            # LAN - rede do grupo
      dhcp4: false
      addresses:
        - 192.168.26.17/28
```

```bash
sudo netplan apply
```

A saída deve mostrar `enp0s3` com o IP `172.20.23.67/22`, `enp0s8` com `192.168.26.17/28` e a rota padrão via `172.20.20.1`.

**Figura 5 – Netplan do gateway**

<img width="790" height="425" alt="1  vm gateway - configuracao netplan" src="https://github.com/user-attachments/assets/c68ba1cd-8192-4ea6-98b5-8f706bab4a5a" />

### 4.3 Gateway — Encaminhamento IPv4
Habilitar imediatamente e tornar permanente:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-ipforward.conf
sudo sysctl --system
sudo sysctl net.ipv4.ip_forward
```

Resultado esperado: `net.ipv4.ip_forward = 1`.

### 4.4 Gateway — NAT e regras de FORWARD com IPTables
Limpar regras antigas (opcional) e criar as novas:

```bash
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -X

# Mascaramento na saída pela WAN
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE

# LAN -> WAN (novas conexões e já estabelecidas)
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT

# WAN -> LAN (somente retorno)
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

O `MASQUERADE` troca o IP de origem `192.168.26.18-25` pelo IP da WAN do gateway (`172.20.23.67`), e o laboratório passa a ver apenas esse endereço. A regra de retorno só aceita pacotes de conexões iniciadas pela LAN, então a rede do grupo não fica exposta a conexões novas vindas da WAN.

Verificação:

```bash
sudo iptables -t nat -L -n -v
sudo iptables -L -n -v
```

### 4.5 Gateway — Persistência das regras

```bash
sudo apt update
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
sudo cat /etc/iptables/rules.v4
```

As regras IPv4 ficam em `/etc/iptables/rules.v4`. O `ip_forward` já está persistente pelo arquivo `/etc/sysctl.d/99-ipforward.conf`.

### 4.6 VM1 a VM8 — Netplan com o gateway do grupo
Em cada VM, o arquivo é igual, mudando apenas o IP (ver tabela 3.4):

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.26.<IP-DA-VM>/28    # .18 a .25
      routes:
        - to: default
          via: 192.168.26.17
      nameservers:
        addresses: [172.20.20.1, 1.1.1.1, 8.8.8.8]
```

```bash
sudo netplan apply
```

| VM | Hostname | `<IP-DA-VM>` |
|:--|:--|:--:|
| VM1 | vm1 | .18 |
| VM2 | vm2 | .19 |
| VM3 | vm3 | .20 |
| VM4 | vm4 | .21 |
| VM5 | vm5 | .22 |
| VM6 | vm6 | .23 |
| VM7 | vm7 | .24 |
| VM8 | vm8 | .25 |

**Figura 6 – Netplan de uma VM do grupo**

<img width="954" height="419" alt="oooo" src="https://github.com/user-attachments/assets/8376e8c9-d05d-47bc-886f-ebe4325355b2" />

### 4.7 Hostname com FQDN (Aula 08, seção 0)
Em cada uma das 9 VMs, usa-se o FQDN completo. Exemplo na VM1:

```bash
sudo hostnamectl set-hostname vm1.grupo1.bsi-26-2.maceio.lab
hostname status
```

No gateway:

```bash
sudo hostnamectl set-hostname vm-gateway.grupo1.bsi-26-2.maceio.lab
```

### 4.8 /etc/hosts
O bloco abaixo é igual nas **9 VMs**, inclusive no gateway. Para o gateway, vale o IP da LAN (`enp0s8`), que é o endereço que as VMs alcançam diretamente.

```bash
sudo nano /etc/hosts

# Grupo 1
192.168.26.17    vm-gateway.grupo1.bsi-26-2.maceio.lab  vm-gateway
192.168.26.18    vm1.grupo1.bsi-26-2.maceio.lab    vm1
192.168.26.19    vm2.grupo1.bsi-26-2.maceio.lab    vm2
192.168.26.20    vm3.grupo1.bsi-26-2.maceio.lab    vm3
192.168.26.21    vm4.grupo1.bsi-26-2.maceio.lab    vm4
192.168.26.22    vm5.grupo1.bsi-26-2.maceio.lab    vm5
192.168.26.23    vm6.grupo1.bsi-26-2.maceio.lab    vm6
192.168.26.24    vm7.grupo1.bsi-26-2.maceio.lab    vm7
192.168.26.25    vm8.grupo1.bsi-26-2.maceio.lab    vm8
```

A linha `127.0.1.1` criada na instalação deve apontar para o nome da própria VM (exemplo na VM1):
```bash
127.0.1.1   vm-gateway.grupo1.bsi-26-2.maceio.lab vm-gateway
```

Verificação:

```bash
getent hosts g1-pc1-gw
getent hosts g1-pc4-vm2.grupo1.bsi-26-2.maceio.lab
```

**Figura 7 – Hostname e `/etc/hosts` configurados**
<img width="762" height="357" alt="host" src="https://github.com/user-attachments/assets/7e61acd6-92be-4dda-bbb6-103936fb1dfc" /> <br>

### 4.9 Usuários
Cada VM tem o administrador (`administrador`) e **um usuário por integrante**, no formato `nomesobrenome` (minúsculas, sem acentos), com home em `/home/<usuário>`, shell `/bin/bash` e participação no grupo `sudo`. O conjunto é o mesmo em todas as VMs, inclusive no gateway.

```bash
sudo useradd -m -s /bin/bash mariaeduarda
sudo usermod -aG sudo mariaeduarda

# Verificação
getent passwd mariaeduarda
ls -l /home
id mariaeduarda
```

**Figura 8 – Usuários criados na VM**

<img width="758" height="237" alt="usu" src="https://github.com/user-attachments/assets/61937065-b949-4d2b-9a99-957e9be3bd71" />

---
### 4.10 Acesso remoto SSH
Em todas as VMs:

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
```

O login por senha é aceito por padrão, e os usuários dos integrantes podem acessar as VMs.

---

## 5. Testes e Capturas de Tela
Todos os testes seguem a ordem do roteiro. Cada figura corresponde a uma captura de tela.

### 5.1 Gateway — rede, encaminhamento e regras

**Figura 9 – Interfaces e rotas do gateway**

<img width="515" height="382" alt="5  vm gateway print do hosts" src="https://github.com/user-attachments/assets/d6fdadb4-d119-48a5-ac2f-df9fc6d13819" />

**Figura 10 – Verificação do `ip_forward`**

<img width="459" height="61" alt="4  vm gateway comando sysctl ipv4" src="https://github.com/user-attachments/assets/d847f41d-dfeb-4026-a452-b9843f01cb89" />

**Figura 11 – Regras de NAT**

![Figura 11](INSERIR_LINK_DA_IMAGEM)

**Figura 12 – Regras FORWARD**

<img width="1280" height="800" alt="2  vm gateway - configuracao iptables" src="https://github.com/user-attachments/assets/a242b57e-084b-4974-8b9f-7161c54cd64d" />
**Figura 13 – Persistência das regras**

<img width="801" height="596" alt="3  vm gateway - persistindo iptables" src="https://github.com/user-attachments/assets/2e6b73a6-31a3-4462-ae49-b027b3e3cb6d" />

### 5.2 Conectividade entre as VMs do grupo

**Figura 14 – Ping entre todas as VMs por IP, nome curto e FQDN**

![Figura 14](INSERIR_LINK_DA_IMAGEM)

**Figura 15 – Acesso SSH entre as VMs**

<img width="1280" height="800" alt="6  vm1 ssh pra vm2" src="https://github.com/user-attachments/assets/6a2828b7-9b5a-4af7-9d23-623253b0efeb" />

### 5.3 Saída para a internet pelo gateway

**Figura 16 – Ping para a internet**

<img width="596" height="338" alt="Captura de tela 2026-10-21 203309" src="https://github.com/user-attachments/assets/186e4854-0d1f-4906-b990-1f082106dfe8" />

**Figura 17 – Resolução de nomes**

<img width="859" height="271" alt="Captura de tela 2026-10-21 204026" src="https://github.com/user-attachments/assets/1fd6424d-0e74-4b24-a319-d9f3bbcd75c1" />

<img width="809" height="331" alt="Captura de tela 2026-10-21 203444" src="https://github.com/user-attachments/assets/2a0b0a2b-e2ca-4eb3-94d7-b9448c1a229e" />

**Figura 18 – Rastreamento da rota**

<img width="941" height="262" alt="Captura de tela 2026-10-21 203531" src="https://github.com/user-attachments/assets/113987a1-09c3-4335-b2ce-ea52347607cf" />

---

## 6. Problemas e Soluções
Durante a realização da atividade, não foram identificados problemas que impedissem a configuração do ambiente. Os procedimentos foram realizados conforme as orientações e os testes apresentaram os resultados esperados.

---

## 7. Conclusão

A atividade permitiu integrar, em um único ambiente, o que foi praticado nas aulas anteriores: instalação e particionamento do Ubuntu Server, usuários e permissões, acesso remoto por SSH, endereçamento estático com Netplan e roteamento com NAT.

Foram implementadas as 9 VMs do Grupo 1 (8 VMs e o gateway), na sub-rede `192.168.26.16/28`: o gateway usa o `.17` na LAN, as VMs ocupam do `.18` ao `.25`, e a faixa termina no broadcast `.31`. A VM gateway, com duas placas em Bridge, encaminha pacotes IPv4 e aplica NAT/MASQUERADE com IPTables, de modo que toda a saída do grupo para o laboratório (`172.20.20.0/22`) e para a internet passa por ela. As regras de FORWARD permitem o tráfego da LAN para a WAN e aceitam apenas o retorno de conexões já estabelecidas. A persistência com `iptables-persistent` e `sysctl.d` garante que a configuração sobreviva a reinicializações.

Os testes de `ping` (por IP, nome curto e FQDN), `ssh`, `nslookup` e `traceroute` confirmaram a comunicação entre todas as VMs, a resolução de nomes e a saída para a internet, com o gateway do grupo aparecendo como primeiro salto. Com isso, os objetivos da Etapa 1 foram alcançados, com uma compreensão prática de roteamento, NAT e organização de uma rede em sub-redes `/28`.

---

## Referências

* Projeto de LSOR — Etapa 1 (bsi-26-2): [projeto-etapa1](https://github.com/alaelson/labredes-2026.2/tree/main/projeto-etapa1)
* Roteiros das aulas: Aula 01, 02, 04, 05, 06, 07 e 08 — [alaelson/labredes-2026.2](https://github.com/alaelson/labredes-2026.2)
