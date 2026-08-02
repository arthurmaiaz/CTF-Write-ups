# TryHackMe — OSINT (Hacker Holidays: The Byte Lotus)

**Plataforma:** TryHackMe
**Evento:** Hacker Holidays The Byte Lotus Hotel
**Dificuldade:** Easy
**Categoria:** OSINT / Email Intelligence

## Briefing

A room gira em torno de uma conversa de Slack entre dois hóspedes do resort, "Ponzi" e "Lambo!". Durante a conversa, Lambo menciona que usava uma ferramenta gratuita (cujo nome começa com a letra `G`) para hospedar seu perfil e vincular outras contas de mídia social mas apagou tudo depois. Ele também deixa seu e-mail de contato: `lambobytelotushotel@gmail.com`.

**Objetivo:** usar esse e-mail como ponto de partida para localizar informações adicionais espalhadas pela internet, seguindo o rastro até encontrar a flag.

## Reconhecimento (Recon)

Comecei testando o e-mail com a ferramenta **Mr. Holmes**, que verifica validade de e-mails e presença em redes sociais/sites vinculados. Confirmou que o e-mail era válido, mas não trouxe pistas suficientes para avançar sozinho.

A pista central da conversa um serviço que começa com "G", permite subir foto de perfil e linkar outras contas apontava para o **Gravatar** (Globally Recognized Avatar), um serviço que associa um avatar e informações de perfil a um endereço de e-mail através de um hash.

## Descoberta

Usei o **Sherlock** para localizar o perfil do Gravatar associado ao e-mail. O Gravatar identifica perfis a partir de um hash do endereço de e-mail, o que explica por que o serviço "segue" a pessoa mesmo depois dela tentar apagar o perfil o hash é gerado deterministicamente a partir do e-mail em si.

O perfil encontrado trazia na bio uma mensagem deixada propositalmente para quem chegasse até ali:

> "Funny thing about email hashes, they follow you places you didn't expect. Glad you found the right corner of the internet! Here is your prize: `VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9`"

## Exploração

A string final tinha o formato característico de **Base64** (apenas letras, números, sem espaços ou símbolos incomuns). Decodifiquei diretamente:

```
echo "VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9" | base64 -d
```

## Flag / Resultado

THM{REDACTED}


## Aprendizados

- **Gravatar é uma fonte de OSINT frequentemente esquecida**: como o perfil é vinculado a um hash determinístico do e-mail (não ao e-mail em texto puro), é possível localizar um perfil mesmo sem saber que ele existe só de ter o e-mail em mãos.
- Perfis "apagados" em serviços online nem sempre desaparecem de fato; caches, snapshots e a própria natureza determinística de certos identificadores (como o hash do Gravatar) podem manter rastros acessíveis.
- Reforcei a diferença entre **codificação** (Base64, reversível por qualquer um, sem chave) e **criptografia** (precisa de uma chave para reverter) útil para reconhecer rapidamente qual ferramenta usar diante de uma string desconhecida.
- Ferramentas de automação como Sherlock aceleram bastante a etapa de correlacionar um dado (e-mail, username) com perfis espalhados por múltiplas plataformas.
