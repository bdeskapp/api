# Acoes de Workflow

## O que voce vai aprender

Como executar acoes em requisicoes via API BDesk — incluindo encerrar, direcionar para outro grupo,
atribuir a um analista, alterar prioridade e outras acoes de ciclo de vida disponibilizadas pelo
sistema conforme o estado e as permissoes do usuario autenticado.

---

## Pre-requisitos

Antes de comecar, voce precisa ter:

- **Token de autenticacao valido** — veja [Autenticacao](../autenticacao.md) para obter o seu token.
- **ID da requisicao** sobre a qual deseja agir — veja [Consultar Requisicoes](consultar-requisicoes.md)
  para localizar o ID correto.

---

## Conceito: Acoes e Codigos

Cada requisicao possui um conjunto de acoes disponivel que varia conforme o estado atual da requisicao
e as permissoes do usuario autenticado. Nao e possivel executar uma acao que o sistema nao libera —
tentativas resultam em erro HTTP 406.

As acoes sao identificadas pelo formato `"Nome [CODIGO]"`. Por exemplo: `"Encerrar [ENC]"`,
`"Direcionar [DIR]"`. Esse identificador composto e retornado ao listar as acoes e deve ser enviado
exatamente como recebido no campo `Id` ao executar a acao.

### Codigos de Acao

| Codigo | Acao | Descricao |
|--------|------|-----------|
| `DIR` | Direcionar | Encaminha a requisicao para outro grupo ou analista |
| `ENC` | Encerrar | Conclui a requisicao |
| `ATR` | Atribuir | Atribui a requisicao a um analista especifico |
| `ATRR` | Atribuir Responsabilidade | Altera o responsavel pela requisicao |
| `ALTPRI` | Alterar Prioridade | Muda o nivel de prioridade |
| `ALTDES` | Alterar Descricao | Modifica o titulo ou a descricao da requisicao |
| `RECAT` | Recategorizar | Altera a categorizacao (atividade/formulario) |
| `DEVD` | Devolver Direto | Devolve a requisicao ao solicitante sem encerrar |
| `AVAL` | Avaliar | Envia a requisicao para avaliacao ou aprovacao |
| `VINC` | Vincular | Vincula esta requisicao a outra requisicao existente |

---

## Passo a Passo

### Passo 1: Consulte as acoes disponiveis

Antes de executar qualquer acao, liste o que esta disponivel para o usuario autenticado naquela
requisicao especifica. Isso evita erros por tentar executar acoes que o sistema nao permite no
estado atual.

**Endpoint:** `GET /v1/requisicoes/{id}/acoes`

Substitua `{id}` pelo ID numerico da requisicao (campo `RequisicaoId` retornado nas listagens).

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/35174/acoes" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta (HTTP 200):**

```json
{
  "_metadata": {
    "Release": "9.8.0",
    "MensagensErro": [],
    "LogAmigavel": []
  },
  "records": [
    {
      "Nome": "Encerrar",
      "Id": "Encerrar [ENC]",
      "Campos": [
        {
          "Nome": "Descricao",
          "Chave": "Descricao",
          "TipoDeDado": "Text",
          "Obrigatoriedade": true
        }
      ]
    },
    {
      "Nome": "Direcionar",
      "Id": "Direcionar [DIR]",
      "Campos": [
        {
          "Nome": "Grupo",
          "Chave": "GrupoId",
          "TipoDeDado": "Numeric",
          "Obrigatoriedade": true
        },
        {
          "Nome": "Descricao",
          "Chave": "Descricao",
          "TipoDeDado": "Text",
          "Obrigatoriedade": false
        }
      ]
    }
  ]
}
```

**Campos da resposta:**

| Campo | Descricao |
|-------|-----------|
| `Nome` | Nome legivel da acao |
| `Id` | Identificador no formato `"Nome [CODIGO]"` — use este valor exato ao executar |
| `Campos` | Lista de campos esperados no payload de execucao |
| `Campos[].Obrigatoriedade` | `true` = campo obrigatorio para esta acao |

Use o valor do campo `Id` exatamente como retornado — incluindo o codigo entre colchetes.

---

### Passo 2: (Opcional) Consulte os grupos disponiveis para direcionamento

Se voce precisar direcionar a requisicao (`DIR`), consulte quais grupos estao disponiveis antes de
montar o payload. O campo `GrupoId` deve conter um ID valido retornado por este endpoint.

**Endpoint:** `GET /v1/requisicoes/{id}/acoes/DIR/grupos`

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/35174/acoes/DIR/grupos" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta (HTTP 200):**

```json
{
  "_metadata": {
    "Release": "9.8.0",
    "MensagensErro": []
  },
  "records": [
    { "Id": 5, "Nome": "Infraestrutura" },
    { "Id": 12, "Nome": "Suporte N2" },
    { "Id": 18, "Nome": "Seguranca da Informacao" }
  ]
}
```

Use o campo `Id` do grupo desejado como valor de `GrupoId` no payload de execucao.

---

### Passo 3: Execute a acao

Envie um `POST` para o mesmo endpoint de acoes, com o identificador da acao no campo `Id` e os
parametros adicionais conforme exigido por cada acao.

**Endpoint:** `POST /v1/requisicoes/{id}/acoes`

O campo `Id` deve ser o valor exato retornado na listagem do Passo 1.

**Campos do payload:**

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `Id` | string | Sim | Identificador da acao no formato `"Nome [CODIGO]"` |
| `Descricao` | string | Depende | Comentario ou motivo da acao (obrigatorio conforme a acao) |
| `GrupoId` | inteiro | Para `DIR` | ID do grupo de destino |
| `NovoSolicitado` | string | Para `DIR` | ID do analista de destino (alternativo ao `GrupoId`) |
| `prioridade` | inteiro | Para `ALTPRI` | Novo nivel de prioridade |
| `Motivo` | inteiro | Para `ENC` | ID do motivo de encerramento (quando exigido) |
| `IdRequisicaoAVincular` | inteiro | Para `VINC` | ID da requisicao a ser vinculada |
| `AssuntoRequisicao` | string | Para `ALTDES` | Novo titulo da requisicao |
| `DescricaoRequisicao` | string | Para `ALTDES` | Nova descricao da requisicao |
| `AtividadeId` | inteiro | Para `RECAT` | ID da nova atividade/categorizacao |

**Resposta de sucesso (HTTP 200):**

```json
{
  "_metadata": {
    "Release": "9.8.0",
    "MensagensErro": [],
    "LogAmigavel": []
  },
  "records": []
}
```

Uma resposta com `MensagensErro` vazia indica que a acao foi executada com sucesso.

---

### Exemplos de Payload por Acao

**Encerrar:**

```json
{
  "Id": "Encerrar [ENC]",
  "Descricao": "Problema resolvido. Acesso ao sistema liberado para o usuario."
}
```

**Direcionar para outro grupo:**

```json
{
  "Id": "Direcionar [DIR]",
  "GrupoId": 5,
  "Descricao": "Encaminhando para a equipe de Infraestrutura para analise de rede."
}
```

**Alterar Prioridade:**

```json
{
  "Id": "Alterar Prioridade [ALTPRI]",
  "prioridade": 2,
  "Descricao": "Prioridade ajustada por impacto em producao."
}
```

**Atribuir a um analista:**

```json
{
  "Id": "Atribuir [ATR]",
  "NovoSolicitado": "45",
  "Descricao": "Atribuindo ao analista responsavel pela conta."
}
```

**Vincular a outra requisicao:**

```json
{
  "Id": "Vincular [VINC]",
  "IdRequisicaoAVincular": 35100
}
```

---

## Exemplos Completos

### cURL

```bash
BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"
REQ_ID=35174

# Listar acoes disponiveis
curl -s "$BASE_URL/v1/requisicoes/$REQ_ID/acoes" \
  -H "Authorization: Bearer $TOKEN"

# Encerrar a requisicao
curl -s -X POST "$BASE_URL/v1/requisicoes/$REQ_ID/acoes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "Id": "Encerrar [ENC]",
    "Descricao": "Problema resolvido. Acesso ao sistema liberado para o usuario."
  }'

# Direcionar para outro grupo
curl -s -X POST "$BASE_URL/v1/requisicoes/$REQ_ID/acoes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "Id": "Direcionar [DIR]",
    "GrupoId": 5,
    "Descricao": "Encaminhando para a equipe de Infraestrutura."
  }'
```

---

### Python

```python
import requests

BASE_URL = "https://sua-empresa.bdesk.com.br/askrest"
headers = {
    "Authorization": "Bearer SEU_TOKEN_AQUI",
    "Content-Type": "application/json"
}
req_id = 35174

# Listar acoes disponiveis
resp = requests.get(f"{BASE_URL}/v1/requisicoes/{req_id}/acoes", headers=headers)
resp.raise_for_status()
acoes = resp.json()["records"]
print("Acoes disponiveis:")
for acao in acoes:
    print(f"  {acao['Id']}")

# Encerrar a requisicao
payload_encerrar = {
    "Id": "Encerrar [ENC]",
    "Descricao": "Problema resolvido. Acesso ao sistema liberado para o usuario."
}
resp = requests.post(
    f"{BASE_URL}/v1/requisicoes/{req_id}/acoes",
    json=payload_encerrar,
    headers=headers
)
resp.raise_for_status()
erros = resp.json()["_metadata"]["MensagensErro"]
if not erros:
    print("Acao executada com sucesso.")
else:
    print(f"Erros: {erros}")

# Consultar grupos para direcionamento e direcionar
resp = requests.get(
    f"{BASE_URL}/v1/requisicoes/{req_id}/acoes/DIR/grupos",
    headers=headers
)
resp.raise_for_status()
grupos = resp.json()["records"]
print("Grupos disponiveis:")
for grupo in grupos:
    print(f"  [{grupo['Id']}] {grupo['Nome']}")

# Direcionar para o primeiro grupo disponivel
if grupos:
    payload_dir = {
        "Id": "Direcionar [DIR]",
        "GrupoId": grupos[0]["Id"],
        "Descricao": "Encaminhando para a equipe responsavel."
    }
    resp = requests.post(
        f"{BASE_URL}/v1/requisicoes/{req_id}/acoes",
        json=payload_dir,
        headers=headers
    )
    resp.raise_for_status()
    print("Direcionamento realizado com sucesso.")
```

---

### PowerShell

```powershell
$BaseUrl = "https://sua-empresa.bdesk.com.br/askrest"
$headers = @{
    Authorization  = "Bearer SEU_TOKEN_AQUI"
    "Content-Type" = "application/json"
}
$ReqId = 35174

# Listar acoes disponiveis
$resp = Invoke-RestMethod -Uri "$BaseUrl/v1/requisicoes/$ReqId/acoes" -Headers $headers
Write-Host "Acoes disponiveis:"
foreach ($acao in $resp.records) {
    Write-Host "  $($acao.Id)"
}

# Encerrar a requisicao
$payloadEncerrar = @{
    Id        = "Encerrar [ENC]"
    Descricao = "Problema resolvido. Acesso ao sistema liberado para o usuario."
} | ConvertTo-Json

$resp = Invoke-RestMethod `
    -Uri "$BaseUrl/v1/requisicoes/$ReqId/acoes" `
    -Method Post `
    -Headers $headers `
    -Body $payloadEncerrar `
    -ContentType "application/json"

if ($resp._metadata.MensagensErro.Count -eq 0) {
    Write-Host "Acao executada com sucesso."
} else {
    Write-Host "Erros: $($resp._metadata.MensagensErro -join ', ')"
}

# Consultar grupos e direcionar
$grupos = Invoke-RestMethod `
    -Uri "$BaseUrl/v1/requisicoes/$ReqId/acoes/DIR/grupos" `
    -Headers $headers

Write-Host "Grupos disponiveis:"
foreach ($grupo in $grupos.records) {
    Write-Host "  [$($grupo.Id)] $($grupo.Nome)"
}

# Direcionar para o primeiro grupo
if ($grupos.records.Count -gt 0) {
    $payloadDir = @{
        Id        = "Direcionar [DIR]"
        GrupoId   = $grupos.records[0].Id
        Descricao = "Encaminhando para a equipe responsavel."
    } | ConvertTo-Json

    Invoke-RestMethod `
        -Uri "$BaseUrl/v1/requisicoes/$ReqId/acoes" `
        -Method Post `
        -Headers $headers `
        -Body $payloadDir `
        -ContentType "application/json" | Out-Null

    Write-Host "Direcionamento realizado com sucesso."
}
```

---

## Erros Comuns

| Sintoma | Causa provavel | Solucao |
|---------|----------------|---------|
| HTTP 406 — "Acao nao disponivel" | A acao nao esta liberada para o usuario neste estado da requisicao | Consulte primeiro `GET /v1/requisicoes/{id}/acoes` e use apenas acoes listadas |
| HTTP 406 — "Descricao obrigatoria" | O campo `Descricao` e obrigatorio para a acao escolhida e nao foi enviado | Inclua o campo `Descricao` no payload com um texto explicativo |
| HTTP 406 — "Grupo invalido" | O `GrupoId` informado nao existe ou nao esta disponivel para esta requisicao | Consulte `GET /v1/requisicoes/{id}/acoes/DIR/grupos` para obter IDs validos |
| HTTP 406 — com lista de erros em `MensagensErro` | Erro de validacao de negocio (campo faltando, valor invalido, etc.) | Leia cada mensagem em `_metadata.MensagensErro` — elas descrevem o problema exato |
| HTTP 401 — Unauthorized | Token expirado ou ausente no cabecalho | Faca login novamente e obtenha um novo token — veja [Autenticacao](../autenticacao.md) |
| Campo `Id` da acao nao reconhecido | O identificador foi digitado manualmente em vez de copiado da listagem | Use o valor exato retornado pelo `GET /acoes`, incluindo espacos e colchetes |

---

## Proximos Passos

- [Gerenciar Anexos](anexos.md) — adicionar arquivos a uma requisicao existente
- [Consultar Requisicoes](consultar-requisicoes.md) — buscar pelo ID, listar abertas e ver historico de acoes
