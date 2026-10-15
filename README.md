# 🛡️ Cybersecurity Home Lab: Perimeter Security, IPS & WAF Automation

Este repositório documenta a implementação de um laboratório de cibersegurança virtualizado. O objetivo principal do ambiente é a simulação de ataques de rede e aplicação web, acompanhados de mecanismos autônomos de detecção e prevenção de ameaças em tempo real (IPS e WAF).

---

## 📐 Arquitetura do Ambiente

O ambiente é virtualizado via **VirtualBox** e composto por três entidades principais:

* **Attacker (Kali Linux):** Origem das simulações de varredura, exploração e requisições maliciosas.
* **Firewall / IPS (pfSense CE + Suricata):** Borda de rede com inspeção profunda de pacotes (DPI) e bloqueio autônomo de IPs atacantes.
* **Web / Application Server (Ubuntu Server + Nginx / ModSecurity WAF):** Servidor de aplicações protegido por WAF em camada de aplicação (Camada 7).

---

## 🚀 Módulo 1: pfSense & Suricata (IPS)

### 1. Visão Geral

O **Suricata** está configurado em **Legacy Mode (IPS)** no pfSense, monitorando o tráfego da interface. Quando uma assinatura de ataque (como varredura de portas via Nmap ou requisições HTTP anômalas) é disparada, o Suricata insere automaticamente o IP atacante na tabela de bloqueio do firewall.

### 2. Configurações Chave no Suricata

* **Interface Settings:** Block Offenders **[X]**
* **IPS Mode:** Legacy Mode
* **Which IP to Block:** SRC (Origem da ameaça)
* **IP Pass List:** none *(Garante que máquinas na mesma sub-rede local/WAN não sejam ignoradas pela whitelist padrão)*

### 3. Validação Prática do Bloqueio

**Ataques Simulados (Kali Linux):**

```
nmap -p 80,443 -A --script http-enum <IP_DO_PFSENSE>

```

```
curl -A "nikto" http://<IP_DO_PFSENSE>/

```

**Resultado:**
O Suricata captura o pacote, dispara a regra `1:2200025` (`SURICATA ICMPv4 unknown code`) ou de User-Agent malicioso, inserindo o IP do Kali Linux na lista **Last Hosts Blocked by Suricata** em tempo real, cortando qualquer tráfego subsequente.

---

## 🧱 Módulo 2: Ubuntu Server & ModSecurity (WAF)

### 1. Visão Geral

Para proteção em Camada 7 (HTTP/HTTPS), o **Ubuntu Server** executa o Nginx integrado com a engine **ModSecurity v3** e o conjunto de regras **OWASP Core Rule Set (CRS)**.

### 2. Passos de Instalação no Ubuntu Server

```
sudo apt update && sudo apt install -y nginx libmodsecurity3 libmodsecurity-dev modsecurity-crs

```

```
sudo sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' /etc/modsecurity/modsecurity.conf

```

```
sudo systemctl restart nginx

```

### 3. Validação Prática do WAF

**Ataques Simulados (SQL Injection / Path Traversal):**

```
curl -i "http://192.168.1.103/?id=1%27%20OR%20%271%27=%271"

```

```
curl -i "http://192.168.1.103/?page=%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd"

```

**Resultado:**
O Nginx/ModSecurity bloqueia a requisição imediatamente com um código de status **HTTP 403 Forbidden**, e os logs do evento são gravados em `/var/log/nginx/error.log` ou `/var/log/modsec_audit.log`.

---

## 📊 Matriz de Testes & Validação

| Vetor de Ataque | Ferramenta | Camada OSI | Componente de Defesa | Ação Tomada |
| --- | --- | --- | --- | --- |
| **Port Scanning** | Nmap | Camada 3/4 | Suricata (pfSense) | IP Bloqueado na tabela de Firewall |
| **User-Agent Scanner** | cURL / Nikto | Camada 7 | Suricata (pfSense) | IP Bloqueado na tabela de Firewall |
| **SQL Injection (SQLi)** | cURL / sqlmap | Camada 7 | ModSecurity WAF | HTTP 403 Forbidden |
| **Directory Traversal** | cURL | Camada 7 | ModSecurity WAF | HTTP 403 Forbidden |

---

## 🛠️ Como Reproduzir este Laboratório

1. Importar as VMs no **VirtualBox** e configurar as placas de rede (Rede Interna / Host-Only).
2. Configurar o **pfSense** e instalar o pacote do Suricata via *Package Manager*.
3. Ajustar a `IP Pass List` do Suricata para `none` e ativar o `Block Offenders`.
4. Subir o **Ubuntu Server** com Nginx + ModSecurity v3.
5. Executar os comandos de teste a partir da máquina **Kali Linux** e monitorar as abas **Services > Suricata > Blocks** no pfSense.
