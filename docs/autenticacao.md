# Autenticacao na API BDesk

## O que voce vai aprender

Como autenticar na API BDesk e gerenciar seu token de acesso.

---

## Pre-requisitos

- Credenciais BDesk validas (usuario e senha) fornecidas pelo administrador do sistema
- Acesso a URL base da sua instalacao BDesk (exemplo: `https://sua-empresa.bdesk.com.br/askrest`)

---

## Passo a Passo

### Passo 1: Obter o Token de Acesso

Envie uma requisicao `POST` para o endpoint de login com suas credenciais:

**Endpoint:** `POST /v1/login/entrar`

**Cabecalho:**
```
Content-Type: application/json
```

**Corpo da requisicao:**
```json
{
  "Login": "seu-usuario",
  "Senha": "sua-senha"
}
```

O campo `Dominio` e opcional e deve ficar vazio ou ausente: a API autentica usuarios com login nativo do BDesk.

**Resposta da API:**
```json
{
  "Dados": "{\"access_token\":\"eyJhbGciOi...\",\"token_type\":\"bearer\",\"expires_in\":\"1799999999\",\"refresh_token\":null,\"scope\":\"admin\",\"error\":null}",
  "LogAmigavel": [],
  "MensagensErro": [],
  "Versao": null
}
```

> **Atencao:** O campo `Dados` na resposta contem uma **string JSON escapada**, nao um objeto direto. Voce precisa fazer um parse adicional para extrair o token.

Apos fazer o parse do campo `Dados`, voce obtera o objeto de autenticacao:

```json
{
  "access_token": "eyJhbGciOi...",
  "token_type": "bearer",
  "expires_in": "1799999999",
  "refresh_token": null,
  "scope": "admin",
  "error": null
}
```

Use o valor de `access_token` nas chamadas subsequentes.

#### O login responde 200 mesmo quando falha

Se o usuario ou a senha estiverem incorretos, ou se o login nao for permitido, a API **continua respondendo HTTP 200**. A falha aparece na propria resposta:

```json
{
  "Dados": null,
  "LogAmigavel": [],
  "MensagensErro": ["Usuário ou senha inválidos"],
  "Versao": null
}
```

Por isso, o seu codigo deve verificar `Dados` (se for `null`, nao ha token) e `MensagensErro` (se tiver itens, o login falhou) **antes** de tentar o parse. Nao use o status HTTP para decidir se o login funcionou.

---

### Passo 2: Usar o Token nas Requisicoes

Inclua o token em todas as chamadas autenticadas usando o cabecalho `Authorization`:

```
Authorization: Bearer <access_token>
```

Exemplo com cURL:

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

---

### Passo 3: Validade do Token e Re-login

O servidor **nao expira** o token por tempo: ele continua valido enquanto o usuario estiver ativo no BDesk. O campo `expires_in` da resposta de login e apenas informativo e nao e aplicado pelo servidor; nao use esse valor para calcular quando renovar o token. Nao existe refresh token.

O token deixa de funcionar quando:

- o usuario e **desativado** no BDesk; ou
- o token enviado e invalido (digitado errado, truncado ou de outro ambiente).

Nesses casos a API responde **HTTP 401**, sem corpo. Repita o Passo 1 para obter um novo token.

**Estrategia recomendada:**

1. Armazene o token obtido no login e reutilize-o em todas as chamadas (nao faca login a cada requisicao)
2. Guarde o token em local seguro, como qualquer senha
3. Se receber HTTP 401, faca login novamente para obter um novo token
4. Reenvie a requisicao original com o novo token

---

## Exemplos Completos

### cURL

```bash
# Passo 1: Login e obtencao do token
RESPONSE=$(curl -s -X POST "https://sua-empresa.bdesk.com.br/askrest/v1/login/entrar" \
  -H "Content-Type: application/json" \
  -d '{"Login": "seu-usuario", "Senha": "sua-senha"}')

# Passo 2: Extrair o token (usando python3 pois jq pode nao estar instalado)
# O login responde 200 mesmo em falha: confira Dados e MensagensErro
TOKEN=$(echo "$RESPONSE" | python3 -c "
import sys, json
d = json.load(sys.stdin)
if d.get('Dados') is None or d.get('MensagensErro'):
    print('Falha no login:', d.get('MensagensErro'), file=sys.stderr)
    sys.exit(1)
print(json.loads(d['Dados'])['access_token'])
")

# Passo 3: Usar o token em uma requisicao autenticada
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes" \
  -H "Authorization: Bearer $TOKEN"
```

---

### Python

```python
import requests
import json

BASE_URL = "https://sua-empresa.bdesk.com.br/askrest"

# Passo 1: Login
resp = requests.post(
    f"{BASE_URL}/v1/login/entrar",
    json={"Login": "seu-usuario", "Senha": "sua-senha"}
)
resp.raise_for_status()
corpo = resp.json()

# O login responde 200 mesmo em falha: confira Dados e MensagensErro
if corpo.get("Dados") is None or corpo.get("MensagensErro"):
    raise SystemExit(f"Falha no login: {corpo.get('MensagensErro')}")

# Passo 2: Parse duplo — o campo Dados e uma string JSON dentro do JSON
dados = json.loads(corpo["Dados"])
token = dados["access_token"]

# Passo 3: Usar o token
headers = {"Authorization": f"Bearer {token}"}
resp = requests.get(f"{BASE_URL}/v1/requisicoes", headers=headers)
print(resp.json())
```

---

### PowerShell

```powershell
$BaseUrl = "https://sua-empresa.bdesk.com.br/askrest"

# Passo 1: Login
$body = @{ Login = "seu-usuario"; Senha = "sua-senha" } | ConvertTo-Json
$resp = Invoke-RestMethod -Uri "$BaseUrl/v1/login/entrar" `
    -Method Post `
    -Body $body `
    -ContentType "application/json"

# O login responde 200 mesmo em falha: confira Dados e MensagensErro
if (-not $resp.Dados -or $resp.MensagensErro) {
    throw "Falha no login: $($resp.MensagensErro -join '; ')"
}

# Passo 2: Parse duplo — o campo Dados e uma string JSON dentro do JSON
$dados = $resp.Dados | ConvertFrom-Json
$token = $dados.access_token

# Passo 3: Usar o token
$headers = @{ Authorization = "Bearer $token" }
Invoke-RestMethod -Uri "$BaseUrl/v1/requisicoes" -Headers $headers
```

---

## Erros Comuns

| Sintoma | Causa | Solucao |
|---------|-------|---------|
| HTTP 200 com `Dados` nulo e `MensagensErro` preenchido | Credenciais incorretas ou login nao permitido | Leia `MensagensErro`; verifique usuario e senha com o administrador |
| HTTP 401 em chamadas subsequentes | Token invalido, ausente no cabecalho ou usuario desativado | Confirme o cabecalho `Authorization: Bearer <token>`; se estiver correto, faca login novamente ou procure o administrador |
| Token parece invalido ou nao funciona | Nao foi feito o parse do campo `Dados` | Aplique `JSON.parse(response.Dados)` antes de extrair o `access_token` |
| Erro de conexao ou timeout | URL base incorreta | Confirme que a URL usa o dominio correto (ex.: `.bdesk.com.br`, nao `.com`) |

Veja tambem [Tratamento de Erros](referencia/erros.md).

---

## Proximos Passos

Com o token em maos, voce esta pronto para usar a API BDesk:

- [Criar Requisicoes](guias/criar-requisicoes.md)
- [Consultar Requisicoes](guias/consultar-requisicoes.md)
- [Tratamento de Erros](referencia/erros.md)
