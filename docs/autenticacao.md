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

**Resposta da API:**
```json
{
  "Dados": "{\"token_type\":\"Bearer\",\"access_token\":\"eyJhbGciOi...\",\"expires_in\":3600}",
  "LogAmigavel": [],
  "MensagensErro": [],
  "Versao": "9.8.0"
}
```

> **Atencao:** O campo `Dados` na resposta contem uma **string JSON escapada**, nao um objeto direto. Voce precisa fazer um parse adicional para extrair o token.

Apos fazer o parse do campo `Dados`, voce obtera o objeto de autenticacao:

```json
{
  "token_type": "Bearer",
  "access_token": "eyJhbGciOi...",
  "expires_in": 3600
}
```

Use o valor de `access_token` nas chamadas subsequentes.

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

### Passo 3: Renovar o Token quando Necessario

O token expira apos o tempo indicado em `expires_in` (geralmente 3600 segundos = 1 hora). Nao existe mecanismo de refresh token — quando o token expirar, repita o Passo 1 para obter um novo.

**Estrategia recomendada:**

1. Armazene o token obtido no login
2. Use-o em todas as chamadas
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
TOKEN=$(echo "$RESPONSE" | python3 -c "import sys,json; d=json.load(sys.stdin); t=json.loads(d['Dados']); print(t['access_token'])")

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

# Passo 2: Parse duplo — o campo Dados e uma string JSON dentro do JSON
dados = json.loads(resp.json()["Dados"])
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
| HTTP 406 com "Usuario ou senha invalidos" | Credenciais incorretas | Verifique o usuario e a senha com o administrador |
| HTTP 401 em chamadas subsequentes | Token expirado ou ausente no cabecalho | Faca login novamente e inclua o token no cabecalho `Authorization` |
| Token parece invalido ou nao funciona | Nao foi feito o parse do campo `Dados` | Aplique `JSON.parse(response.Dados)` antes de extrair o `access_token` |
| Erro de conexao ou timeout | URL base incorreta | Confirme que a URL usa o dominio correto (ex.: `.bdesk.com.br`, nao `.com`) |

---

## Proximos Passos

Com o token em maos, voce esta pronto para usar a API BDesk:

- [Criar Requisicoes](guias/criar-requisicoes.md)
- [Consultar Requisicoes](guias/consultar-requisicoes.md)
- [Tratamento de Erros](referencia/erros.md)
