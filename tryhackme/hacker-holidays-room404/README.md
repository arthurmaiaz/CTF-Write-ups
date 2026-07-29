# TryHackMe — Room 404 (Hacker Holidays: The Byte Lotus)

**Plataforma:** TryHackMe
**Evento:** Hacker Holidays — The Byte Lotus Hotel
**Dificuldade:** Easy
**Categoria:** Web / Information Disclosure
**Link:** https://tryhackme.com/room/hh-room404-804573bf

## Briefing

> "He booked the quiet room. It's not on the floor plan, not in the brochure, not on any door. But port 8080 is wide open, and the rooms it never lists are the ones worth finding."

A pista central já entrega o caminho: existe algo rodando na porta 8080 que não está referenciado em nenhum lugar visível do site. Isso indicou que o desafio seria de **enumeração de conteúdo escondido**, não de exploração de uma falha óbvia na tela inicial.

## Reconhecimento (Recon)

Comecei com um scan de diretórios usando o `dirb`, priorizando velocidade com a wordlist `common.txt` já que estava na VM do TryHackMe:

```
dirb http://10.65.181.65:8080/ /usr/share/dirb/wordlists/common.txt
```

Resultado:
http://10.65.181.65:8080/.git/HEAD (CODE:200|SIZE:21)

De 4612 palavras testadas, apenas esse caminho retornou algo relevante.

## Descoberta da vulnerabilidade

O `.git/HEAD` respondendo com **200** confirma que a pasta `.git` da aplicação está publicamente acessível via HTTP. Isso é um problema clássico de **information disclosure**: quando o deploy é feito sem remover ou bloquear o acesso ao `.git`, qualquer pessoa pode reconstruir o repositório inteiro remotamente — código-fonte, histórico de commits e até arquivos que já foram deletados em versões posteriores.

Confirmação manual:

```
Utilizei o comando "curl" para ver o conteudo bruto e confirmar que era um arquivo GIT legitimo

curl http://10.65.181.65:8080/.git/HEAD
# ref: refs/heads/main
```

## Exploração

Usei o [git-dumper](https://github.com/arthaud/git-dumper) para reconstruir o repositório localmente a partir dos arquivos expostos:

```
pip3 install git-dumper --break-system-packages
git-dumper http://10.65.181.65:8080/.git/ ./room404-dump
```

A ferramenta baixou os metadados (`HEAD`, `config`, `index`, `objects/`, etc.) e reconstruiu o repositório automaticamente, incluindo o checkout dos arquivos:

cd room404-dump
ls -a

.git README.md app.js index.html

A partir daí, analisei o histórico de commits em busca de informações sensíveis:

```
git log --all --oneline
```

## Flag / Resultado

A flag foi encontrada no histórico de commits do repositório reconstruído.

THM{REDACTED}


## Aprendizados / Conclusão

- Nunca fazer deploy de uma aplicação com a pasta `.git` acessível publicamente — isso permite que qualquer atacante reconstrua o código-fonte e o histórico completo do projeto.
- Aprendi a usar o **git-dumper** para automatizar a reconstrução de repositórios git expostos via HTTP.
- Reforcei a importância de sempre revisar o histórico de commits (`git log -p --all`), já que arquivos "deletados" continuam recuperáveis se não houver reescrita de histórico (`git filter-branch`/`BFG`).

------------------------------------------------------------------------------ENGLISH VERSION------------------------------------------------------------------------------------
# TryHackMe — Room 404 (Hacker Holidays: The Byte Lotus)

**Platform:** TryHackMe
**Event:** Hacker Holidays — The Byte Lotus Hotel
**Difficulty:** Easy
**Category:** Web / Information Disclosure
**Link:** https://tryhackme.com/room/hh-room404-804573bf

## Briefing

> "He booked the quiet room. It's not on the floor plan, not in the brochure, not on any door. But port 8080 is wide open, and the rooms it never lists are the ones worth finding."

The core hint already gives away the path: something is running on port 8080 that isn't referenced anywhere visible on the site. This indicated the challenge would be about **hidden content enumeration**, rather than exploiting an obvious flaw on the landing page.

## Reconnaissance

I started with a directory scan using `dirb`, prioritizing speed with the `common.txt` wordlist since it was already available on the TryHackMe VM:

```
dirb http://10.65.181.65:8080/ /usr/share/dirb/wordlists/common.txt
```

Result:

http://10.65.181.65:8080/.git/HEAD (CODE:200|SIZE:21)


Out of 4612 words tested, only this path returned anything relevant.

## Vulnerability Discovery

The `.git/HEAD` responding with **200** confirms that the application's `.git` folder is publicly accessible via HTTP. This is a classic **information disclosure** issue: when a deployment is done without removing or blocking access to `.git`, anyone can remotely reconstruct the entire repository — source code, commit history, and even files that were later deleted in subsequent versions.

Manual confirmation:

I used the "curl" command to view the raw content and confirm it was a legitimate GIT file

curl http://10.65.181.65:8080/.git/HEAD

ref: refs/heads/main

## Exploitation

I used [git-dumper](https://github.com/arthaud/git-dumper) to locally reconstruct the repository from the exposed files:

pip3 install git-dumper --break-system-packages
git-dumper http://10.65.181.65:8080/.git/ ./room404-dump


The tool downloaded the metadata (`HEAD`, `config`, `index`, `objects/`, etc.) and automatically reconstructed the repository, including checking out the files:

cd room404-dump
ls -a

.git README.md app.js index.html


From there, I analyzed the commit history looking for sensitive information:

git log --all --oneline


## Flag / Result

The flag was found in the commit history of the reconstructed repository.

THM{REDACTED}


## Lessons Learned / Conclusion

- Never deploy an application with the `.git` folder publicly accessible — this allows any attacker to reconstruct the full source code and commit history of the project.
- Learned how to use **git-dumper** to automate the reconstruction of git repositories exposed via HTTP.
- Reinforced the importance of always reviewing commit history (`git log -p --all`), since "deleted" files remain recoverable if the history hasn't been rewritten (`git filter-branch`/`BFG`).