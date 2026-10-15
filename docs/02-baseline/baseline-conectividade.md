# Baseline de Conectividade e Segurança

**Data:** 14/10/2026

**Ambiente:** Laboratório de Redes, IPS e WAF em VirtualBox

**Status:** Validado e Operacional

---

## Objetivo

Registrar o estado funcional validado da rede, do perímetro e das camadas de aplicação após a implementação do **pfSense CE + Suricata (IPS)**, do **Ubuntu Server com Nginx + ModSecurity v3 (WAF)** e da integração de logs com o **Wazuh SIEM**.

---

## Topologia Atual

```text
Atacante (Kali Linux)
  | 10.0.2.4
  |
WAN / External
  | 10.0.2.0/24 (NAT Gateway: 10.0.2.2 | DNS: 10.0.2.3)
  |
pfSense CE + Suricata IPS
  | WAN: 10.0.2.15
  | LAN: 192.168.1.1
  |
Rede Interna / LAN (192.168.1.0/24)
  |
Ubuntu Server (192.168.1.103)
  - Nginx + ModSecurity v3 (WAF)
  - OWASP Core Rule Set (CRS)
  - Agente/Manager Wazuh SIEM

```

---
## Endereçamento e Componentes

| Dispositivo | Interface / Rede | Endereço | Função |
|---|---|---|---|
| pfSense | WAN | 10.0.2.15/24 | Borda de rede, Firewall e Suricata IPS (Legacy Mode) |
| pfSense | LAN | 192.168.1.1/24 | Gateway e inspeção da rede interna |
| Kali Linux | WAN / Attacker | 10.0.2.4/24 | Origem das simulações de ataque e testes de carga |
| Ubuntu Server | LAN | 192.168.1.103/24 | Servidor Web, ModSecurity v3 WAF e análise de logs com Wazuh SIEM |
| VirtualBox NAT | Gateway | 10.0.2.2 | Gateway upstream |
| VirtualBox NAT | DNS | 10.0.2.3 | DNS upstream |

---

## Serviços e Defesas Validadas

* **Navegação e Resolução DNS:** O tráfego de saída utiliza NAT pelo pfSense e o DNS Resolver responde na porta 53.
* **Prevenção de Intrusão em Borda (IPS):** O Suricata em *Legacy Mode* monitora a interface do pfSense, identificando e inserindo o IP do atacante (`10.0.2.4`) na lista `Last Hosts Blocked`.


* **Proteção de Aplicação Web (WAF):** O ModSecurity v3 com OWASP CRS intercepta requisições HTTP maliciosas (SQLi, XSS e Path Traversal), retornando o status `HTTP/1.1 403 Forbidden` no Nginx.


* **Correlação de Eventos (SIEM):** Regras personalizadas no `local_rules.xml` do Wazuh Manager categorizam e geram alertas críticos (`Rule ID 100201` e `100202`) para eventos interceptados pelo WAF.

---

## Evidências de Testes

### 1. Teste de Resolução DNS e Conectividade Externa

Comando executado no Kali Linux:

```text
nslookup example.com 192.168.1.1

```

Resultado:

```text
Server: 192.168.1.1
Address: 192.168.1.1#53

Name: example.com
Address: 93.184.215.14

```

---

### 2. Validação do Suricata IPS (pfSense)

Comando executado no Kali Linux (`10.0.2.4`):

```text
nmap -p 80,443 -A --script http-enum 10.0.2.15

```

Resultado:
O Suricata dispara o alerta `SURICATA ICMPv4 unknown code` (SID `1:2200025`), inserindo o IP `10.0.2.4` na tabela de bloqueio e cortando todo o tráfego subsequente.

---

### 3. Validação do ModSecurity WAF (Camada 7)

Comando executado via CMD/Terminal apontado para a aplicação (`192.168.1.103`):

* **SQL Injection (SQLi):**
```text
curl -i "[http://192.168.1.103/?id=1%27%20OR%20%271%27=%271](http://192.168.1.103/?id=1%27%20OR%20%271%27=%271)"

```


* **Cross-Site Scripting (XSS):**
```text
curl -i "[http://192.168.1.103/?q=%3Cscript%3Ealert(1)%3C/script%3E](http://192.168.1.103/?q=%3Cscript%3Ealert(1)%3C/script%3E)"

```


* **Path Traversal:**
```text
curl -i "[http://192.168.1.103/?page=%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd](http://192.168.1.103/?page=%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd)"

```



Resultado retornado pelo servidor em todas as simulações:

```text
HTTP/1.1 403 Forbidden
Server: nginx/1.20.2
Date: Thu, 08 Oct 2026 13:39:55 GMT
Content-Type: text/html
Content-Length: 153
Connection: keep-alive

<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.20.2</center>
</body>
</html>

```

---

## Critério de Sucesso

O baseline do ambiente foi considerado aprovado porque:

1. A resolução DNS externa e a conectividade de saída estão operacionais através do pfSense.
2. O **Suricata IPS** identifica tentativas de varredura e aplica bloqueios automáticos no nível de firewall.


3. O **ModSecurity v3 WAF** no Nginx barra com sucesso requisições OWASP Top 10 emitindo o código `HTTP 403 Forbidden`.


4. Os logs do WAF são ingeridos e correlacionados corretamente via regras no **Wazuh SIEM**.
