# Guia de Configuração Individual das VMs — Grupo 1

Passo a passo para configurar **cada VM** do grupo, uma por vez. Ordem recomendada: **gateway primeiro**, depois `vm1` a `vm8`.

## 1. Resumo

| VM | PC | Interface | IP | Gateway padrão | FQDN |
|:--|:--:|:--|:--|:--|:--|
| vm-gateway | PC1 | enp0s3 (WAN) | 172.20.23.67/22 | 172.20.20.1 | vm-gateway.grupo1.bsi-26-2.maceio.lab |
| vm-gateway | PC1 | enp0s8 (LAN) | 192.168.26.17/28 | — | — |
| vm1 | PC1 | enp0s3 | 192.168.26.18/28 | 192.168.26.17 | vm1.grupo1.bsi-26-2.maceio.lab |
| vm2 | PC1 | enp0s3 | 192.168.26.19/28 | 192.168.26.17 | vm2.grupo1.bsi-26-2.maceio.lab |
| vm3 | PC2 | enp0s3 | 192.168.26.20/28 | 192.168.26.17 | vm3.grupo1.bsi-26-2.maceio.lab |
| vm4 | PC2 | enp0s3 | 192.168.26.21/28 | 192.168.26.17 | vm4.grupo1.bsi-26-2.maceio.lab |
| vm5 | PC3 | enp0s3 | 192.168.26.22/28 | 192.168.26.17 | vm5.grupo1.bsi-26-2.maceio.lab |
| vm6 | PC3 | enp0s3 | 192.168.26.23/28 | 192.168.26.17 | vm6.grupo1.bsi-26-2.maceio.lab |
| vm7 | PC4 | enp0s3 | 192.168.26.24/28 | 192.168.26.17 | vm7.grupo1.bsi-26-2.maceio.lab |
| vm8 | PC4 | enp0s3 | 192.168.26.25/28 | 192.168.26.17 | vm8.grupo1.bsi-26-2.maceio.lab |

- Rede do grupo: `192.168.26.16/28` (broadcast `.31`).
- Rede do laboratório: `172.20.20.0/22`, gateway/DNS `172.20.20.1`.
- Todas as placas de rede no VirtualBox ficam em **Placa em Ponte (Bridge)**.

## 2. Passos comuns a todas as VMs
Execute em **cada** uma das 9 VMs (troque `<nome>` pelo nome da VM).

### 2.1 Hostname
```bash
sudo hostnamectl set-hostname <nome>.grupo1.bsi-26-2.maceio.lab
hostname
```

### 2.2 /etc/hosts
```bash
sudo nano /etc/hosts
```

Acrescente o bloco abaixo, igual para as 9 VMs:

```text
# Grupo 1 - bsi-26-2 - grupo1.bsi-26-2.maceio.lab
192.168.26.17    vm-gateway.grupo1.bsi-26-2.maceio.lab    vm-gateway
192.168.26.18    vm1.grupo1.bsi-26-2.maceio.lab    vm1
192.168.26.19    vm2.grupo1.bsi-26-2.maceio.lab    vm2
192.168.26.20    vm3.grupo1.bsi-26-2.maceio.lab    vm3
192.168.26.21    vm4.grupo1.bsi-26-2.maceio.lab    vm4
192.168.26.22    vm5.grupo1.bsi-26-2.maceio.lab    vm5
192.168.26.23    vm6.grupo1.bsi-26-2.maceio.lab    vm6
192.168.26.24    vm7.grupo1.bsi-26-2.maceio.lab    vm7
192.168.26.25    vm8.grupo1.bsi-26-2.maceio.lab    vm8
```

Ajuste a linha `127.0.1.1` para o nome da própria VM:

```text
127.0.1.1    <nome>.grupo1.bsi-26-2.maceio.lab    <nome>
```

### 2.3 Usuários

Substitua pelos usuários dos integrantes (lista da planilha de acompanhamento):

```bash
for u in usuario1 usuario2 usuario3 usuario4; do
  sudo useradd -m -s /bin/bash "$u"
  sudo usermod -aG sudo "$u"
  sudo passwd "$u"
done
getent passwd usuario1 usuario2 usuario3 usuario4
```

### 2.4 SSH

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
```

## 3. VM Gateway (`vm-gateway`)

Duas placas em Bridge: Adaptador 1 (WAN) e Adaptador 2 (LAN).

### 3.1 Netplan

```bash
ip -br link
ping -c 2 172.20.23.67     # deve falhar: IP livre
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 172.20.23.67/22
      routes:
        - to: default
          via: 172.20.20.1
      nameservers:
        addresses: [172.20.20.1, 1.1.1.1, 8.8.8.8]
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.26.17/28
```

```bash
sudo netplan apply
ip -br a
ip route
```

Esperado: `enp0s3` com `172.20.23.67/22`, `enp0s8` com `192.168.26.17/28` e `default via 172.20.20.1`.

### 3.2 Encaminhamento IPv4

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-ipforward.conf
sudo sysctl --system
sudo sysctl net.ipv4.ip_forward
```

Esperado: `net.ipv4.ip_forward = 1`.

### 3.3 NAT e FORWARD

```bash
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -t nat -L -n -v
sudo iptables -L -n -v
```

### 3.4 Persistência

```bash
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
sudo cat /etc/iptables/rules.v4
```

Depois dos passos da seção 2 (hostname `vm-gateway`, hosts, usuários, SSH), o gateway está pronto.

## 4. VMs clientes (`vm1` a `vm8`)

Uma placa em Bridge (Adaptador 1). O arquivo Netplan é igual em todas; muda só o IP.

Modelo:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.26.<IP>/28
      routes:
        - to: default
          via: 192.168.26.17
      nameservers:
        addresses: [172.20.20.1, 1.1.1.1, 8.8.8.8]
```

Aplicar e testar (em qualquer cliente):

```bash
sudo netplan apply
ip -br a
ip route
ping -c 2 192.168.26.17
ping -c 2 1.1.1.1
```

### 4.1 vm1 (PC1) — 192.168.26.18

```bash
sudo hostnamectl set-hostname vm1.grupo1.bsi-26-2.maceio.lab
sudo nano /etc/netplan/00-installer-config.yaml   # addresses: 192.168.26.18/28
sudo netplan apply
ip -br a
ping -c 2 192.168.26.17
```

Esperado: `enp0s3` com `192.168.26.18/28`. Em seguida, `/etc/hosts` (linha `127.0.1.1` = `vm1`), usuários e SSH (seção 2).

### 4.2 vm2 (PC1) — 192.168.26.19

```bash
sudo hostnamectl set-hostname vm2.grupo1.bsi-26-2.maceio.lab
sudo nano /etc/netplan/00-installer-config.yaml   # addresses: 192.168.26.19/28
sudo netplan apply
ip -br a
ping -c 2 192.168.26.17
```

Esperado: `enp0s3` com `192.168.26.19/28`. Em seguida, `/etc/hosts` (linha `127.0.1.1` = `vm2`), usuários e SSH (seção 2).

### 4.3 vm3 (PC2) — 192.168.26.20

```bash
sudo hostnamectl set-hostname vm3.grupo1.bsi-26-2.maceio.lab
sudo nano /etc/netplan/00-installer-config.yaml   # addresses: 192.168.26.20/28
sudo netplan apply
ip -br a
ping -c 2 192.168.26.17
```

Esperado: `enp0s3` com `192.168.26.20/28`. Em seguida, `/etc/hosts` (linha `127.0.1.1` = `vm3`), usuários e SSH (seção 2).

### 4.4 vm4 (PC2) — 192.168.26.21

```bash
sudo hostnamectl set-hostname vm4.grupo1.bsi-26-2.maceio.lab
sudo nano /etc/netplan/00-installer-config.yaml   # addresses: 192.168.26.21/28
sudo netplan apply
ip -br a
ping -c 2 192.168.26.17
```

Esperado: `enp0s3` com `192.168.26.21/28`. Em seguida, `/etc/hosts` (linha `127.0.1.1` = `vm4`), usuários e SSH (seção 2).

### 4.5 vm5 (PC3) — 192.168.26.22

```bash
sudo hostnamectl set-hostname vm5.grupo1.bsi-26-2.maceio.lab
sudo nano /etc/netplan/00-installer-config.yaml   # addresses: 192.168.26.22/28
sudo netplan apply
ip -br a
ping -c 2 192.168.26.17
```

Esperado: `enp0s3` com `192.168.26.22/28`. Em seguida, `/etc/hosts` (linha `127.0.1.1` = `vm5`), usuários e SSH (seção 2).

### 4.6 vm6 (PC3) — 192.168.26.23

```bash
sudo hostnamectl set-hostname vm6.grupo1.bsi-26-2.maceio.lab
sudo nano /etc/netplan/00-installer-config.yaml   # addresses: 192.168.26.23/28
sudo netplan apply
ip -br a
ping -c 2 192.168.26.17
```

Esperado: `enp0s3` com `192.168.26.23/28`. Em seguida, `/etc/hosts` (linha `127.0.1.1` = `vm6`), usuários e SSH (seção 2).

### 4.7 vm7 (PC4) — 192.168.26.24

```bash
sudo hostnamectl set-hostname vm7.grupo1.bsi-26-2.maceio.lab
sudo nano /etc/netplan/00-installer-config.yaml   # addresses: 192.168.26.24/28
sudo netplan apply
ip -br a
ping -c 2 192.168.26.17
```

Esperado: `enp0s3` com `192.168.26.24/28`. Em seguida, `/etc/hosts` (linha `127.0.1.1` = `vm7`), usuários e SSH (seção 2).

### 4.8 vm8 (PC4) — 192.168.26.25

```bash
sudo hostnamectl set-hostname vm8.grupo1.bsi-26-2.maceio.lab
sudo nano /etc/netplan/00-installer-config.yaml   # addresses: 192.168.26.25/28
sudo netplan apply
ip -br a
ping -c 2 192.168.26.17
```

Esperado: `enp0s3` com `192.168.26.25/28`. Em seguida, `/etc/hosts` (linha `127.0.1.1` = `vm8`), usuários e SSH (seção 2).

## 5. Testes finais (em qualquer VM cliente)

```bash
for h in vm-gateway vm1 vm2 vm3 vm4 vm5 vm6 vm7 vm8; do ping -c 2 $h; done
ping -c 2 vm1.grupo1.bsi-26-2.maceio.lab
ping -c 4 1.1.1.1
ping -c 4 google.com
sudo apt install -y traceroute
traceroute google.com
ssh <usuario>@vm3
```

- O primeiro salto do `traceroute` deve ser `192.168.26.17`.
- No gateway: `sysctl net.ipv4.ip_forward`, `sudo iptables -t nat -L -n -v`, `sudo iptables -L -n -v`.

## 6. Problemas comuns

| Sintoma | Verificação |
|:--|:--|
| Cliente não alcança o gateway | `ip -br a`: IP e máscara `/28` corretos? Placa em Bridge no VirtualBox? |
| Alcança o gateway, mas não a internet | No gateway: `ip_forward = 1` e regra `MASQUERADE` na `enp0s3` |
| Ping por IP funciona, por nome não | Conferir `/etc/hosts` e `getent hosts <nome>` |
| Erro ao aplicar Netplan | Indentação do YAML: só espaços, nunca tab |
| Regras somem após reiniciar | `sudo netfilter-persistent save` |
| `ssh` recusa conexão | `sudo systemctl status ssh` |
