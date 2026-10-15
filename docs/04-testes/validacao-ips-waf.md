# Validação de Segurança, Conectividade, IPS e WAF

## Objetivo

Validar o funcionamento integrado do ecossistema de laboratório:

* Resolução de nomes e saída de rede via pfSense CE.
* Prevenção de Intrusão em tempo real (IPS) na borda com **Suricata** no pfSense.


* Proteção de Aplicação Web em Camada 7 com **ModSecurity v3 / OWASP CRS** no Nginx/Ubuntu Server.


* Ingestão e correlação de eventos de segurança no **Wazuh SIEM**.

---

## Topologia Testada

```text
Atacante (Kali Linux)
10.0.2.4
     |
     | Perímetro / WAN: 10.0.2.0/24
     |
pfSense CE (Firewall & Suricata IPS)
WAN: 10.0.2.15 | LAN: 192.168.1.1
     |
     | Rede Interna / LAN: 192.168.1.0/24
     |
Ubuntu Server (ModSecurity v3 WAF & Wazuh SIEM)
192.168.1.103

```

---

## Configuração Esperada

| Item | Valor Esperado | Função / Mecanismo |
| --- | --- | --- |
| Endereço do Atacante (Kali) | 10.0.2.4 | Origem das simulações ofensivas (Camada 3, 4 e 7)

 |
| IP WAN do pfSense | 10.0.2.15 | Borda de rede com Suricata IPS (Legacy Mode)

 |
| IP LAN do pfSense | 192.168.1.1 | Gateway e inspeção de rede interna |
| IP do Ubuntu Server | 192.168.1.103 | Servidor Web (Nginx/Apache), ModSecurity WAF e Wazuh Manager

 |
| Gateway Upstream / DNS | 10.0.2.2 / 10.0.2.3 | Saída NAT do VirtualBox e resolução DNS |

---

## Testes Realizados & Validações

### 1. Conectividade Base & Resolução DNS

Comando executado no Kali Linux:

```text
nslookup example.com 192.168.1.1

```

Resultado obtido:

```text
Server: 192.168.1.1
Address: 192.168.1.1#53

Name: example.com
Address: 93.184.215.14

```

O pfSense respondeu corretamente às consultas DNS e encaminhou o tráfego via upstream `10.0.2.3`.

---

### 2. Validação do Suricata IPS (Camada 3/4 - pfSense)

Configurações ativas no Suricata:

* `Block Offenders`: Habilitado


* `IPS Mode`: Legacy Mode


* `IP Pass List`: `none` (bloqueia o IP atacante sem exceção)



Comando de varredura executado do Kali (`10.0.2.4`):

```text
nmap -p 80,443 -A --script http-enum 10.0.2.15

```

Resultado obtido:
O Suricata disparou o alerta `1:2200025` (`SURICATA ICMPv4 unknown code`), inserindo automaticamente o IP `10.0.2.4` na lista **Last Hosts Blocked by Suricata** e bloqueando todo o tráfego subsequente no firewall.

---

### 3. Validação de WAF (Camada 7 - ModSecurity v3 / OWASP CRS)

Simulações executadas apontando para o servidor web (`192.168.1.103`):

#### A. SQL Injection (SQLi)

```text
curl -i "http://192.168.1.103/?id=1%27%20OR%20%271%27=%271"

```

#### B. Cross-Site Scripting (XSS)

```text
curl -i "http://192.168.1.103/?q=%3Cscript%3Ealert(1)%3C/script%3E"

```

#### C. Path Traversal

```text
curl -i "http://192.168.1.103/?page=%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd"

```

Resultado retornado pelo WAF para todas as requisições maliciosas:

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

### 4. Correlação de Regras no Wazuh SIEM

Arquivo de regras customizadas `local_rules.xml` no Manager:

```xml
<group name="modsecurity_custom">
  <rule id="100201" level="10">
    <decoded_as>json</decoded_as>
    <match>ModSecurity</match>
    <description>SOC Lab: Evento de ataque detectado pelo ModSecurity WAF</description>
  </rule>

  <rule id="100202" level="12">
    <if_sid>100201</if_sid>
    <match>SQL Injection</match>
    <description>SOC Lab: Ataque Crítico de SQL Injection interceptado pelo WAF</description>
  </rule>
</group>

```

Resultado:
Os logs de auditoria gravados pelo ModSecurity no Nginx/Ubuntu são ingeridos pelo agente do Wazuh e geram alertas críticos automaticamente no Dashboard (Rule ID `100201` e `100202`).

---

## Matriz de Resultados dos Testes

| Teste | Mecanismo de Defesa | Vetor/Payload | Resultado |
| --- | --- | --- | --- |
| **Resolução DNS & HTTPS** | pfSense Unbound | Consultas externas | Aprovado |
| **Bloqueio de Scan / Probe** | Suricata IPS | Nmap / ICMP Probe | Aprovado (IP `10.0.2.4` Bloqueado)

 |
| **Proteção SQLi (Camada 7)** | ModSecurity WAF | `%27%20OR%20%271%27=%271` | Aprovado (HTTP 403 Forbidden)

 |
| **Proteção XSS (Camada 7)** | ModSecurity WAF | `%3Cscript%3E` | Aprovado (HTTP 403 Forbidden)

 |
| **Proteção Path Traversal** | ModSecurity WAF | `%2e%2e%2f` | Aprovado (HTTP 403 Forbidden)

 |
| **Alertas no SIEM** | Wazuh Manager | Regras 100201 / 100202 | Aprovado (Eventos classificados) |

---

## Conclusão

A arquitetura defensiva do laboratório foi totalmente aprovada.

O **pfSense** garante o roteamento de saída e intercepta ameaças de varredura na borda via **Suricata IPS** com bloqueio automático. Em camada de aplicação, o **ModSecurity v3 WAF** barra requisições do OWASP Top 10 com retorno `HTTP 403 Forbidden`, enquanto o **Wazuh SIEM** correlaciona os eventos em tempo real com regras personalizadas no arquivo `local_rules.xml`.

---

## Evidências Registradas

* `legacy-mode-suricata.png`: Habilitação do `Block Offenders` e `IP Pass List = none` no pfSense.


* `last-alerts-suricata.png`: Log de alertas em tempo real do Suricata IPS.


* `last-blocks-suricata.png`: Tabela de bloqueio ativo no pfSense com o IP `10.0.2.4`.


* `waf-sqli-block-403.png`: Intercepção de SQL Injection no cURL.


* `waf-xss-block.png`: Intercepção de XSS no cURL.


* `waf-path-traversal-403.png`: Intercepção de Directory Traversal no cURL.


* `wazuh-local-rules.png`: Configuração do grupo de regras `modsecurity_custom` no Wazuh Manager.
