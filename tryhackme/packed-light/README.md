# TryHackMe — Packed Light (Hacker Holidays: The Byte Lotus)
# Primeiro contato com Descriptografia

**Plataforma:** TryHackMe
**Evento:** Hacker Holidays — The Byte Lotus Hotel
**Dificuldade:** Medium
**Categoria:** Network Forensics / Covert Channel / Malware Reversing
**Link:** https://tryhackme.com/room/hh-packedlight-02e5330c

## Briefing

> "Tiny packets. Odd hours. Suspiciously regular. Someone's smuggling out the data equivalent of a hotel towel every night, folded neatly inside traffic that looks ordinary until you decode it."
>
> "A short capture from the guest network is all VERA could pull before the connection dropped. Somewhere in that traffic, a quiet little errand is running on a loop, and it isn't part of any service the hotel actually offers."

A comunidade já tinha deixado uma pista boa antes de eu nem começar: um post de uma hóspede (@0xMia) reclamando que o notebook dela ficava fazendo ping numa porta `:8080` toda hora, tipo um relógio, e que os headers da requisição "não pareciam de um app de verdade". Isso já direcionou bem o meu foco: tinha alguma coisa rodando escondida naquele tráfego, disfarçada de tráfego normal.

## Reconhecimento (Recon)

Abri o `traffic.pcapng` no Wireshark e, antes de sair filtrando qualquer coisa, dei uma olhada geral em **Statistics → Protocol Hierarchy** só pra entender o que tinha ali (HTTP, DNS, TLS, etc).

Com a pista da porta 8080 em mente, apliquei o filtro:

tcp.port == 8080


E lá estava: um monte de requisições HTTP acontecendo em um intervalo curtíssimo, quase sempre 1 segundo de diferença uma da outra, com um `User-Agent` estranho  `ByteLotusClient/1.1` bem diferente do navegador normal que aparecia no resto da captura.

## Descoberta da vulnerabilidade

Segui uma dessas conversas com **Follow → HTTP Stream**, e uma das primeiras respostas do servidor (uma requisição pra `/temp/updates.py`) devolveu, em texto puro, o próprio **código-fonte do malware** que estava rodando na máquina da vítima. A princípio encontrei facilmente um keylogger em Python inteirinho apenas entrando e inspecionando o codigo do site em questão sem nenhuma ofuscação.

O script fazia basicamente isso:
1. Capturava cada tecla digitada com a biblioteca `pynput`
2. Fazia **XOR** de cada caractere com uma chave fixa
3. Codificava o resultado em **Base64**
4. Mandava esse valor escondido dentro do header **`Cookie`** (`hotel_sess_state=...`) de uma requisição `GET /` pro servidor

Ou seja, aquele beacon que eu já tinha achado no recon não era só um ping bobo cada request que carregava era disfarçado dentro de um cookie de sessão qualquer, acreditava que fosse uma tecla que a vítima tinha digitado.

## Exploração

No Wireshark, filtrei só as requisições que tinham cookie:

http.request and http.cookie


Adicionei uma coluna customizada (Edit → Preferences → Appearance → Columns) com o campo `http.cookie`, pra conseguir ver o valor de cada um direto na lista, sem precisar abrir pacote por pacote. Depois exportei tudo e limpei o prefixo `hotel_sess_state=` de cada linha, deixando só o valor em Base64 puro.

A parte mais interessante foi escrever o script de decodificação. Minha primeira tentativa foi meio ingênua: eu assumi que a chave "andava" a cada caractere, tipo um XOR clássico com chave repetida (`key[i % len(key)]`, incrementando o índice a cada cookie). Rodei e saiu um monte de letras confusas e desordenadas com caracteres que não faziam nenhum sentido. Inclusive tive que pedir a ajuda da Inteligência Artificial(IA) para me ajudar a montar esse código, que não tinha conhecimento suficiente para montar ele. 

Voltei pro código do malware pra entender onde eu tinha errado, e aí percebi o detalhe: a função `sendltr()` manda **uma tecla de cada vez**, numa chamada nova de `xor()` a cada tecla. E como o índice usado dentro do `xor()` vem de um `enumerate()` local, ele nasce e morre dentro de cada chamada nunca "lembra" quantas teclas já foram enviadas antes. Na prática, isso significa que **toda tecla foi cifrada usando sempre o mesmo primeiro caractere da chave**, não uma sequência avançando.

Depois de ajustar isso (usando sempre `key[0]` em vez de um índice que muda), o script finalmente reconstruiu a mensagem inteira:

```python
import base64

cookies = open("cookies.txt").read().splitlines()
key = "H0t3lSt@ff0Nly" + "K3epS3cr3t!"

resultado = ""
for cookie in cookies:
    raw_byte = base64.b64decode(cookie)[0]
    key_char = ord(key[0])
    resultado += chr(raw_byte ^ key_char)

print(resultado)
```

## Flag / Resultado

# Por motivos óbvios não coloquei a saída da flag descriptografada.
```
THM{REDACTED}
```

## Aprendizados / Conclusão

Essa room ensinou bastante coisa na prática:

- Foi meu primeiro contato com descriptografia então tive bastante dificuldade por onde começar, tinha uma noção de que poderia usar a ferramenta wireshark de começo já que o desafio da room packed-light me entregou um arquivo de trafego de rede (.pcapng).  
- Tráfego repetitivo e regular demais pra um host/porta que não faz parte do serviço normal é quase sempre sinal de beaconing  vale sempre isolar isso com filtro de porta e olhar o timing das requisições.
- Dados exfiltrados não precisam estar em lugares óbvios; um header de Cookie comum pode estar carregando informação sensível codificada.
- O erro que cometi decodificando errado no início foi um bom lembrete de que, ao reverter uma criptografia customizada, não basta "entender o algoritmo por cima" o escopo das variáveis importa, e um índice que parece global pode na real resetar a cada chamada de função.
- E um detalhe legal de matemática que usei sem perceber no começo: como XOR é reversível com a mesma chave (`A XOR B = C` e `C XOR B = A`), dá pra usar a própria função de criptografia do malware pra decodificar os dados sem precisar escrever uma versão "inversa" separada.