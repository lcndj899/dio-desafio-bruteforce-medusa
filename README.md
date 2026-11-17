# 🛡️ Auditoria de Segurança e Simulação de Força Bruta com Medusa em Ambiente Controlado

Projeto prático desenvolvido para o desafio da formação em Cibersegurança da **[DIO (Digital Innovation One)](https://www.dio.me/)**.

O objetivo deste projeto é implementar um ambiente de laboratório isolado, realizar o mapeamento de vulnerabilidades em serviços de rede, simular auditorias de autenticação com a ferramenta **Medusa** e exercitar medidas sólidas de mitigação e defesa em profundidade (*Defense in Depth*).

---

## 📌 Disclaimer Ético e Legal

> **AVISO IMPORTANTE:** Todos os testes documentados neste repositório foram conduzidos estritamente em ambiente de laboratório isolado em rede *Host-Only* (sem conexão ou impacto com a internet), com o propósito exclusivo de aprendizado e auditoria autorizada. O uso de técnicas e ferramentas ofensivas contra alvos sem autorização prévia e formal constitui crime cibernético perante a legislação vigente (Lei 12.737/2012 - Lei Carolina Dieckmann e Marco Civil da Internet).

---

## 🏗️ Topologia e Configuração do Laboratório

O ambiente foi montado no **Oracle VirtualBox** utilizando o modo de rede **Placa de rede exclusiva de hospedeiro (Host-Only)**, isolando completamente as máquinas de qualquer rede externa e garantindo a segurança do hospedeiro.

```
+-------------------------------------------------------------+
|             Rede Host-Only (192.168.56.0/24)                |
|                                                             |
|  +-----------------------+       +-----------------------+  |
|  |      Kali Linux       |       |    Metasploitable 2   |  |
|  |  (Atacante / Auditor) | ----> |   (Alvo Vulnerável)   |  |
|  |     192.168.56.102    |       |     192.168.56.101    |  |
|  +-----------------------+       +-----------------------+  |
+-------------------------------------------------------------+
```

### Detalhes dos Ativos:

| Máquina Virtual | Papel | Sistema Operacional | Interface / IP | Credenciais Padrão |
| :--- | :--- | :--- | :--- | :--- |
| **Kali Linux** | Auditor / Origem dos testes | Kali Linux 2026.3 (64-bit) | `eth1`: `192.168.56.102` | `kali` / `kali` |
| **Metasploitable 2** | Alvo Vulnerável | Ubuntu 8.04 LTS (Hardy) | `eth0`: `192.168.56.101` | `msfadmin` / `msfadmin` |

---

## 🔍 Fase 1: Reconhecimento e Mapeamento de Superfície de Ataque

A primeira etapa de qualquer avaliação de segurança é a enumeração de serviços para identificar portas abertas e banners de software. A partir do terminal do Kali Linux, executou-se a varredura com o **Nmap**:

```bash
nmap -sV -p 21,80,139,445 192.168.56.101 -oN results/nmap_scan.txt
```

### Resultados da Varredura:
```text
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
80/tcp  open  http        Apache httpd 2.2.8 ((Ubuntu) DAV/2)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
```

**Análise dos Vetores Identificados:**
1. **Porta 21 (FTP):** Serviço `vsftpd 2.3.4` ativo sem criptografia TLS, vulnerável a inspeção de tráfego e tentativas ilimitadas de login.
2. **Porta 80 (HTTP):** Servidor Apache hospedando aplicações vulneráveis intencionais, incluindo o **DVWA (Damn Vulnerable Web Application)**.
3. **Portas 139 e 445 (SMB):** Serviços de compartilhamento de arquivos Samba sem restrições rigorosas de controle de sessão.

---

## 🧪 Fase 2: Simulação de Auditorias com Medusa

O **Medusa** é uma ferramenta de auditoria modular de força bruta de credenciais em alta velocidade, que utiliza paralelismo e suporte a múltiplos protocolos de rede.

### Principais Parâmetros Utilizados:
* `-h <IP>`: Endereço IP do host de destino.
* `-U <arquivo>`: Caminho da lista de usuários (*wordlist*).
* `-u <usuario>`: Usuário individual para testes direcionados.
* `-P <arquivo>`: Caminho da lista de senhas (*wordlist*).
* `-p <senha>`: Senha individual (utilizada em técnicas de *Password Spraying*).
* `-M <modulo>`: Módulo do protocolo auditado (`ftp`, `web-form`, `smb`, etc.).
* `-f`: Interrompe a execução assim que encontrar o primeiro par válido de credenciais (*fail-fast*).
* `-t <num>`: Quantidade máxima de conexões simultâneas.

---

### Cenário 1: Auditoria em Serviço FTP
O protocolo FTP tradicional não possui proteção nativa contra adivinhação de senhas, confiando no sistema operacional ou em firewalls para rate limiting.

* **Comando executado:**
  ```bash
  medusa -h 192.168.56.101 -U wordlists/users_test.txt -P wordlists/passwords_test.txt -M ftp -f
  ```
* **Mecânica do Teste:** O Medusa estabelece conexões TCP sequenciais/paralelas enviando os comandos `USER` e `PASS` do protocolo FTP, analisando a resposta de retorno (`230 User logged in` para sucesso ou `530 Login incorrect` para falha).
* **Vulnerabilidade Explorada:** Falta de limites de tentativas por IP e uso de credenciais padrão/fracas (`msfadmin:msfadmin`).

---

### Cenário 2: Automação de Formulário Web no DVWA
A aplicação web DVWA simula um painel corporativo autêntico com formulário de login HTTP POST.

* **Comando executado:**
  ```bash
  medusa -h 192.168.56.101 -u admin -P wordlists/passwords_test.txt -M web-form -m FORM:"dvwa/login.php" -m DENY-KEY:"Login failed" -f
  ```
* **Mecânica do Teste:** O módulo `web-form` envia requisições HTTP do tipo POST contendo os campos de usuário e senha do formulário web. O parâmetro `DENY-KEY` instrui o Medusa a interpretar qualquer resposta contendo a string "Login failed" como tentativa frustrada, validando o login quando essa string não for retornada.
* **Vulnerabilidade Explorada:** Ausência de CAPTCHA, ausência de tokens anti-automação (CSRF com validação rigorosa) e falta de rate-limiting em nível de aplicação.

---

### Cenário 3: Password Spraying em Compartilhamentos SMB
Ao invés de testar centenas de senhas em um único usuário (o que ativaria bloqueios de conta facilmente), a técnica de *Password Spraying* testa uma única senha comum contra uma lista com diversos usuários da organização.

* **Comando executado:**
  ```bash
  medusa -h 192.168.56.101 -U wordlists/users_test.txt -p "msfadmin" -M smb
  ```
* **Mecânica do Teste:** O Medusa negocia o protocolo SMB/CIFS autenticando cada conta individualmente com uma única senha corporativa trivial, permitindo mapear contas com senhas fracas sem exceder o limiar de bloqueio padrão de 3 ou 5 falhas seguidas.
* **Vulnerabilidade Explorada:** Falta de política corporativa para proibição de senhas fracas e ausência de correlação de eventos em logs do Active Directory/Samba.

---

## 🛡️ Medidas de Mitigação e Defesa em Profundidade

Identificar a vulnerabilidade é apenas metade do trabalho de um profissional de segurança; a etapa mais relevante é desenhar a arquitetura defensiva para conter esses ataques:

### 1. Política de Senhas Robustas (Alinhamento NIST SP 800-63B)
* Definir comprimento mínimo de 12 a 16 caracteres.
* Banir senhas padrão de fábrica e palavras listadas em dicionários conhecidos de vazamentos (ex: *rockyou*, *top-10000-passwords*).
* Implementar listas de verificação de senhas comprometidas no ato do cadastro ou alteração de credenciais.

### 2. Bloqueio de Contas e Limites de Tentativas (Account Lockout & Throttling)
* Configurar limite de tentativas consecutivas inválidas (ex: 5 tentativas em uma janela de 15 minutos).
* Aplicar atrasos progressivos (*exponential backoff*): cada tentativa incorreta adiciona um tempo de espera obrigatório antes da próxima submissão.

### 3. Autenticação Multifator (MFA / 2FA)
* Tornar obrigatória a exigência de um segundo fator de autenticação (aplicativo autenticador TOTP ou chave de segurança física FIDO2/WebAuthn) em painéis administrativos, serviços SSH e conexões VPN.
* Mesmo que o atacante adivinhe a senha via força bruta, o acesso é barrado pela ausência do token dinâmico.

### 4. Detecção e Prevenção Ativa com Fail2Ban / IPS
* Instalação de softwares de análise de logs em tempo real (como o **Fail2Ban**). Quando um IP atinge $N$ falhas de autenticação em um curto período, o host cria dinamicamente uma regra no `iptables`/`nftables` bloqueando todo o tráfego daquele IP por um tempo determinado (*bantime*).

```ini
# Exemplo de configuração defensiva no Fail2Ban (/etc/fail2ban/jail.local)
[vsftpd]
enabled  = true
port     = ftp,ftp-data,ftps,ftps-data
logpath  = /var/log/vsftpd.log
maxretry = 3
findtime = 600
bantime  = 3600
```

### 5. Criptografia em Trânsito e Desativação de Serviços Legados
* Descontinuar protocolos que trafegam credenciais em texto claro (FTP puro, Telnet e HTTP sem TLS), substituindo-os por **SFTP**, **SSH** e **HTTPS**.
* Restringir o acesso a portas críticas (como SMB 445 e FTP 21) utilizando regras de firewall corporativo, permitindo acesso apenas a partir de redes de gerenciamento autorizadas.

---

## 📁 Estrutura do Repositório

```text
dio-desafio-bruteforce-medusa/
│
├── README.md                   # Documentação completa e detalhada do desafio
│
├── wordlists/                  # Dicionários de teste didáticos
│   ├── users_test.txt          # Lista de nomes de usuários comuns testados
│   └── passwords_test.txt      # Lista de senhas fracas testadas
│
├── configs/                    # Modelos de configurações para hardening e defesa
│   ├── fail2ban-jail.local     # Regras de bloqueio automático por IP para Fail2Ban
│   └── vsftpd-hardening.conf   # Configuração recomendada para proteção do FTP
│
├── results/                    # Evidências e relatórios gerados no laboratório
│   └── nmap_scan.txt           # Saída real da varredura de portas e serviços
│
└── images/                     # Capturas de tela do laboratório
    └── .gitkeep
```

---

## 🚀 Como Reproduzir o Laboratório

1. Instale o **Oracle VirtualBox** em seu sistema operacional hospedeiro.
2. Baixe e importe as máquinas virtuais **Kali Linux** e **Metasploitable 2**.
3. Configure a interface de rede de ambas as máquinas para o modo **Host-Only Network** (*VirtualBox Host-Only Ethernet Adapter*).
4. Inicie as máquinas e verifique a conectividade:
   ```bash
   ping 192.168.56.101
   ```
5. Clone este repositório no seu Kali Linux:
   ```bash
   git clone https://github.com/lcndj899/dio-desafio-bruteforce-medusa.git
   cd dio-desafio-bruteforce-medusa
   ```
6. Execute as auditorias demonstrativas conforme descrito na seção de Cenários.

---

## 🎓 Conclusão

A realização deste projeto reforçou a compreensão prática sobre a facilidade com que serviços mal configurados e credenciais triviais podem ser auditados e comprometidos. A segurança de uma infraestrutura moderna não pode depender exclusivamente da premissa de que os usuários escolherão senhas seguras; é imprescindível a adoção de uma arquitetura defensiva multicamadas que englobe controle de acesso por rede, políticas automáticas de bloqueio, monitoramento ativo de logs e autenticação multifator.

---

## 👤 Autor

* **Desenvolvido por:** [Seu Nome Aqui]
* **Formação:** Cibersegurança - DIO (Digital Innovation One)
* **LinkedIn:** [https://linkedin.com/in/seuperfil](https://linkedin.com)
* **GitHub:** [https://github.com/lcndj899](https://github.com/lcndj899)
