Aqui está o documento atualizado com as tecnologias ativas do laboratório (**Suricata IPS**, **ModSecurity v3 / OWASP CRS WAF no Nginx/Ubuntu Server**, **SIEM Wazuh** e o IP real do atacante Kali), mantendo exatamente a estrutura limpa e copiável sem a sintaxe de linguagem em blocos de código:

# Topologia do Laboratório

## Visão geral

Este laboratório foi criado em VirtualBox para praticar conceitos de redes, defesa de perímetro e segurança de aplicações web, integrando prevenção de intrusão autônoma (IPS), Web Application Firewall (WAF) e monitoramento centralizado de eventos (SIEM).

O pfSense atua como firewall de borda e IPS central com Suricata, interceptando e bloqueando ameaças de rede, enquanto a aplicação web na sub-rede interna é protegida pelo ModSecurity v3 no Nginx e monitorada via agentes do Wazuh.

## Diagrama lógico

```text
                        Internet / Attacker Host
                                   |
                            VirtualBox NAT
                           Gateway: 10.0.2.2
                           DNS: 10.0.2.3
                                   |
                            WAN do pfSense
                         10.0.2.15/24 (Attacker Kali: 10.0.2.4)
                                   |
                         +-------------------+
                         |  pfSense CE +     |
                         |  Suricata (IPS)   |
                         +-------------------+
                                   |
                          LAN 192.168.1.0/24
                                   |
                         pfSense: 192.168.1.1
                                   |
                  +---------------------------------+
                  |  Ubuntu Server (192.168.1.103)  |
                  |  - Nginx + ModSecurity v3 (WAF) |
                  |  - OWASP Core Rule Set (CRS)    |
                  |  - Wazuh Agent / Manager        |
                  +---------------------------------+

```

## Componentes

| Componente | Interface/Rede | Endereço | Função |
| --- | --- | --- | --- |
| pfSense | WAN | 10.0.2.15/24 | Acesso externo, controle de borda e Suricata IPS (Legacy Mode)

 |
| pfSense | LAN | 192.168.1.1/24 | Gateway, firewall e inspeção da rede interna |
| Kali Linux | WAN / Attacker | 10.0.2.4 | Estação de testes ofensivos (Nmap, cURL, payloads)

 |
| Ubuntu Server | LAN | 192.168.1.103/24 | Servidor web (Nginx/Apache), WAF (ModSecurity v3) e SIEM (Wazuh)

 |
| VirtualBox NAT | Gateway | 10.0.2.2 | Saída da WAN para a Internet |
| VirtualBox NAT | DNS | 10.0.2.3 | Resolução DNS upstream |

## Redes utilizadas

### WAN (Perímetro / Origem de Testes)

```text
Rede: 10.0.2.0/24
Gateway: 10.0.2.2
DNS upstream: 10.0.2.3
pfSense WAN: 10.0.2.15
Kali Linux (Attacker): 10.0.2.4

```

A interface WAN utiliza a rede NAT do VirtualBox para tráfego externo e simulação de ataques de origem não confiável oriundos da máquina Kali Linux.

### LAN (Aplicação & Defesa)

```text
Rede: 192.168.1.0/24
Gateway: 192.168.1.1
Ubuntu Server (WAF/SIEM): 192.168.1.103

```

A LAN abriga o servidor de aplicações (Ubuntu Server) protegido localmente pelo WAF ModSecurity e integrado ao SIEM Wazuh para análise de logs em tempo real.

## Camadas de Segurança Configuradas

### 1. Suricata IPS (pfSense - Camada 3/4 e Protocolos)

* **Modo de Operação:** Legacy Mode com `Block Offenders` ativo.


* **Ação:** Bloqueio automático de IP na tabela de firewall ao disparar assinaturas (ex.: scans ICMP/Nmap).


* **IP Pass List:** Definida como `none` para garantir que ataques vindos da sub-rede local/WAN sejam bloqueados imediatamente sem exceção por whitelist automática.



### 2. ModSecurity v3 WAF (Ubuntu Server - Camada 7)

* **Engine:** ModSecurity v3 integrado ao Nginx.


* **Conjunto de Regras:** OWASP Core Rule Set (CRS).
* **Modo de Operação:** `SecRuleEngine On` (Prevenção e Bloqueio Ativo).


* **Ação:** Intercepção de requisições maliciosas enviando resposta `HTTP 403 Forbidden` antes de atingir a aplicação backend.



### 3. Wazuh SIEM (Correlação & Regras Personalizadas)

* **Arquivo de Regras:** `local_rules.xml` em `/var/ossec/etc/rules/`.


* **Regras Customizadas:**
* **Rule ID 100201 (Nível 10):** Detecção genérica de eventos do ModSecurity WAF.


* **Rule ID 100202 (Nível 12):** Alerta crítico para ataques de SQL Injection confirmados pelo WAF.





## Fluxo de Tráfego e Prevenção

### Ataque de Rede (ex.: Scan de Portas / Probing)

```text
Kali Linux (10.0.2.4)
  -> Requisição/Scan para pfSense WAN (10.0.2.15)
  -> Suricata inspeciona pacotes via PCAP Engine
  -> Regra disparada (ex.: GID:SID 1:2200025)
  -> IP 10.0.2.4 inserido em "Last Hosts Blocked"
  -> Tráfego subsequente bloqueado na tabela do Firewall

```

### Ataque de Aplicação Web (SQLi, XSS, Path Traversal)

```text
Atacante (Kali / Terminal)
  -> Requisição HTTP com payload malicioso para http://192.168.1.103/
  -> Nginx recebe o tráfego e repassa ao ModSecurity v3
  -> Inspecção contra regras OWASP CRS
  -> Correspondência de regra de ataque
  -> Resposta imediata: HTTP 403 Forbidden
  -> Registro do log de auditoria em modsec_audit.log
  -> Agente Wazuh lê o log -> Dispara Regra 100201 / 100202 no SIEM

```

## Matriz de Testes & Prevenção

| Vetor de Ataque | Comando de Teste | Componente de Defesa | Ação / Resultado |
| --- | --- | --- | --- |
| **Port Scanning / ICMP** | `nmap -p 80,443 -A 10.0.2.15` | Suricata IPS (pfSense) | IP `10.0.2.4` Bloqueado

 |
| **SQL Injection (SQLi)** | `curl -i "[http://192.168.1.103/?id=1%27%20OR%20%271%27=%271](http://192.168.1.103/?id=1%27%20OR%20%271%27=%271)"` | ModSecurity WAF / OWASP CRS | HTTP 403 Forbidden + Alerta Wazuh 100202

 |
| **Path Traversal** | `curl -i "[http://192.168.1.103/?page=%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd](http://192.168.1.103/?page=%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd)"` | ModSecurity WAF / OWASP CRS | HTTP 403 Forbidden

 |
| **Cross-Site Scripting (XSS)** | `curl -i "[http://192.168.1.103/?q=%3Cscript%3Ealert(1)%3C/script%3E](http://192.168.1.103/?q=%3Cscript%3Ealert(1)%3C/script%3E)"` | ModSecurity WAF / OWASP CRS | HTTP 403 Forbidden

 |

## Evidências Registradas

### Módulo IPS & Firewall (pfSense)

* `legacy-mode-suricata.png`: Configuração das opções `Block Offenders` e `IP Pass List = none` no Suricata.


* `last-alerts-suricata.png`: Histórico de alertas de tráfego ICMP e regras ativadas no Suricata.


* `last-blocks-suricata.png`: Tabela do pfSense confirmando o bloqueio ativo do IP `10.0.2.4` em tempo real.



### Módulo WAF & Aplicação (Ubuntu Server)

* `waf-sqli-block-403.png`: Bloqueio HTTP 403 Forbidden para requisição SQL Injection.


* `waf-path-traversal-403.png`: Bloqueio HTTP 403 Forbidden para tentativa de Directory Traversal (`%2e%2e%2f`).


* `waf-xss-block.png`: Bloqueio HTTP 403 Forbidden para tentativa de XSS.



### Módulo SIEM & Regras (Wazuh Manager)

* `wazuh-local-rules.png`: Configuração XML do grupo `modsecurity_custom` com as regras `100201` e `100202`.



## Estado Atual do Ambiente

* **pfSense WAN:** `10.0.2.15` operacional.


* **Ubuntu Server (WAF):** `192.168.1.103` ativo com Nginx, ModSecurity v3 e OWASP CRS.


* **Suricata IPS:** Habilitado em Legacy Mode com bloqueio automático funcional e `IP Pass List` ajustada para `none`.


* **Proteção L7:** Testes de SQLi, XSS e Path Traversal validados com retorno `HTTP 403 Forbidden`.


* **SIEM Wazuh:** Regras personalizadas integradas para correlacionar eventos de bloqueio do WAF.
