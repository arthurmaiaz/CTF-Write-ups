# Write-up: Investigação de Logs

Relatório do desafio de análise de logs e decodificação, feito como parte de um processo seletivo de cibersegurança.

## Sobre o desafio

O desafio partia de um arquivo `.zip` com dois arquivos:

- `access.log`: registro de acessos de um servidor.
- `encodedflag.txt`: arquivo com um texto codificado.

O objetivo era descobrir quem tentou obter uma informação escondida, revelar as mensagens ocultas e avaliar a reputação de dois endereços de IP suspeitos.

## Resumo das respostas

| # | Pergunta | Resposta |
|---|----------|----------|
| 1 | IP do atacante que tentou extrair a flag Base64 no log | `212.14.17.145` |
| 2 | Frase secreta revelada com o From Base64 no CyberChef | `THM{CYBERCHEF_WIZARD}` |
| 3 | País e empresa do IP `54.36.115.221` que varreu o arquivo `.env` | França, OVH (OVHcloud) |
| 4 | Mensagem secreta revelada a partir do `encodedflag.txt` | Endereço MAC `08-2E-9A-4B-7F-61` |

## Como cheguei às respostas

### 1. Preparação

Baixei o `.zip` e criei uma pasta específica para a análise, chamada `projeto-analise`. Ao descompactar, obtive o `access.log` e o `encodedflag.txt`.

### 2. O IP e a primeira frase secreta

O registro de acessos é muito extenso, então busquei apenas as linhas com `==`, sinal comum no final de textos em Base64:

```bash
grep "==" access.log
```

Uma única linha foi retornada:

```bash
212.14.17.145 - - [27/Sep/2023:07:00:53 +0000] "GET /VEhNe0NZQkVSQ0hFRl9XSVpBUkR9== HTTP/1.1" 401 5196 ...
```

Ela mostra que, em 27 de setembro de 2023, às 7h (horário de Greenwich), o IP `212.14.17.145` pediu ao servidor o endereço `/VEhNe0NZQkVSQ0hFRl9XSVpBUkR9==`. O servidor respondeu com o código 401 (sem autorização) e negou o acesso.

A sequência no final do endereço era a mensagem escondida. No CyberChef, colei-a na entrada e usei **From Base64**, o que resultou em `THM{CYBERCHEF_WIZARD}`.

### 3. O arquivo `encodedflag.txt`

Copiei todo o conteúdo do arquivo para o CyberChef. Na primeira tentativa usei o **From Hex** antes das demais ferramentas e a saída veio vazia. Seguindo a orientação da organização, comecei pelo **From Base64**, que revelou um texto longo, formado por pequenos blocos de números e letras separados por hífen, sem significado aparente.

Em seguida, usei a ferramenta **Regular expression (Regex)** com a opção pronta **MAC address** e a saída em lista. Ela funciona como um filtro que mantém apenas o que tem o formato de um endereço MAC. Restou um único resultado: `08-2E-9A-4B-7F-61`.

### 4. Consulta dos IPs no VirusTotal

Pesquisei os dois IPs no VirusTotal, analisando as abas *Detection* e *Details*:

| IP | País | Resultado |
|----|------|-----------|
| `212.14.17.145` | Polônia | Sem suspeita nem indício de ataque. Ainda assim, foi o IP que solicitou a frase escondida no log, o que justifica manter atenção. |
| `54.36.115.221` | França | 1 classificação como malicioso e 1 como suspeito, além de indícios de brute force na seção *Crowdsourced context*. Isso é coerente com a tentativa de acesso ao arquivo `.env`, que costuma guardar informações sensíveis do servidor. |

## Conclusão

A análise permitiu responder às quatro perguntas. O exercício mostrou como informações importantes podem estar escondidas em grandes volumes de texto e como a escolha e a ordem das ferramentas fazem diferença no resultado.

