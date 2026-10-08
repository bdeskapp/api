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
e as permissoes do usuario autenticado. Nao e possivel executar uma acao que o sistema nao libera.

**Atencao a como os erros chegam:** na maioria das falhas ao executar uma acao a API responde
**HTTP 200**, e a mensagem de erro vem em `_metadata.MensagensErro`. Nao basta checar o status HTTP:
leia sempre `MensagensErro` depois de executar uma acao (veja "Erros Comuns").

As acoes sao identificadas pelo formato `"Nome [CODIGO]"`. Por exemplo: `"Encerrar [ENC]"`,
`"Direcionar [DIR]"`. Esse identificador composto e retornado ao listar as acoes e deve ser enviado
exatamente como recebido no campo `Id` ao executar a acao. O codigo entre colchetes e obrigatorio:
`"Encerrar [ENC]"` e `"[ENC]"` funcionam, mas `"ENC"` sozinho nao e reconhecido (a API responde 200
com a mensagem "Ação não encontrada").

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
  "_metadata": {},
  "records": [
    {
      "Nome": "Encerrar",
      "Id": "Encerrar [ENC]",
      "CodigoAcao": "ENC",
      "Campos": [
        {
          "Nome": "Descricao",
          "Chave": "Descricao",
          "TipoDeDado": "Text",
          "Obrigatoriedade": true
        },
        {
          "Nome": "Avaliacao",
          "Chave": "tipoAvaliacao",
          "TipoDeDado": "Integer",
          "Obrigatoriedade": true
        }
      ]
    },
    {
      "Nome": "Direcionar",
      "Id": "Direcionar [DIR]",
      "CodigoAcao": "DIR",
      "Campos": [
        {
          "Nome": "Novo solicitado",
          "Chave": "NovoSolicitado",
          "TipoDeDado": "Integer",
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

Cada item traz ainda outros membros, como `FormularioId`, `FormularioAcaoId` e `Icone`. Se o usuario
nao tem nenhuma acao disponivel naquele status, a resposta e 200 com `"records": []`.

**Campos da resposta:**

| Campo | Descricao |
|-------|-----------|
| `Nome` | Nome legivel da acao |
| `Id` | Identificador no formato `"Nome [CODIGO]"` — use este valor exato ao executar |
| `CodigoAcao` | Codigo da acao (ex.: `ENC`) |
| `Campos` | Lista de campos esperados no payload de execucao |
| `Campos[].Chave` | Nome do campo a ser usado no corpo da execucao |
| `Campos[].Obrigatoriedade` | `true` = campo obrigatorio para esta acao |

Use o valor do campo `Id` exatamente como retornado — incluindo o codigo entre colchetes.

---

### Passo 2: (Opcional) Consulte os destinos disponiveis para direcionamento

Se voce precisar direcionar a requisicao (`DIR`), consulte quais grupos e usuarios podem receber
a requisicao antes de montar o payload. O valor de destino deve ser um `Id` retornado por este endpoint.

**Endpoint:** `GET /v1/requisicoes/{id}/acoes/DIR/grupos`

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/35174/acoes/DIR/grupos" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta (HTTP 200):**

```json
{
  "_metadata": {},
  "records": [
    { "Id": "{ IdParticipante : 5, IdTipoPapel : 2 } ", "Texto": "Infraestrutura" },
    { "Id": "{ IdParticipante : 12, IdTipoPapel : 2 } ", "Texto": "Suporte N2" },
    { "Id": "{ IdParticipante : 18, IdTipoPapel : 2 } ", "Texto": "Seguranca da Informacao" }
  ]
}
```

O `Id` de cada item e um **texto pronto**, que deve ser copiado exatamente como veio (inclusive o
espaco final) para o campo `NovoSolicitado` do payload de execucao. `Texto` e o nome para exibicao.

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
| `NovoSolicitado` | string | Para `DIR` | O `Id` do destino, copiado de `GET /v1/requisicoes/{id}/acoes/DIR/grupos` |
| `prioridade` | inteiro | Para `ALTPRI` | Nova prioridade: 1 = Alta, 2 = Media, 3 = Baixa |
| `tipoAvaliacao` | inteiro | Para `ENC` e `AVAL` | Avaliacao: 1 = acima do esperado, 2 = conforme o esperado, 3 = abaixo do esperado |
| `usuResponsavelId` | inteiro | Para `ATR` e `ATRR` | ID do usuario que assumira a requisicao |
| `Motivo` | inteiro | Quando a acao tem motivos | ID do motivo (quando exigido pela acao) |
| `IdRequisicaoAVincular` | inteiro | Para `VINC` | ID da requisicao a ser vinculada |
| `AssuntoRequisicao` | string | Para `ALTDES` | Novo titulo da requisicao |
| `DescricaoRequisicao` | string | Para `ALTDES` | Nova descricao da requisicao |
| `AtividadeId` | inteiro | Para `RECAT` | ID da nova atividade/categorizacao |

Os campos exigidos variam por acao e por formulario: a lista `Campos` retornada por
`GET /v1/requisicoes/{id}/acoes` (Passo 1) e a fonte confiavel. Para anexar um arquivo **nao** use
esta rota: veja o guia [Anexos](anexos.md).

**Resposta (HTTP 200):**

```json
{
  "_metadata": {
    "Release": "9.8.0",
    "LogAmigavel": [],
    "MensagensErro": []
  },
  "records": null
}
```

`MensagensErro` vazia indica que a acao foi executada com sucesso. Quando a acao devolve uma
mensagem de texto, a resposta pode ser, em vez do envelope acima, esse texto como um JSON simples
(por exemplo `"Requisicao encerrada"`). Seu codigo deve tratar as duas formas.

**Erros chegam com HTTP 200:** se alguma validacao falhar (acao nao encontrada, usuario sem
permissao nesse status, campo obrigatorio ausente), o status continua 200 e a mensagem aparece em
`_metadata.MensagensErro`. Sempre verifique essa lista.

---

### Exemplos de Payload por Acao

**Encerrar:**

```json
{
  "Id": "Encerrar [ENC]",
  "Descricao": "Problema resolvido. Acesso ao sistema liberado para o usuario.",
  "tipoAvaliacao": 2
}
```

**Direcionar para outro grupo:**

```json
{
  "Id": "Direcionar [DIR]",
  "NovoSolicitado": "{ IdParticipante : 5, IdTipoPapel : 2 } ",
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
  "usuResponsavelId": 45,
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
    "Descricao": "Problema resolvido. Acesso ao sistema liberado para o usuario.",
    "tipoAvaliacao": 2
  }'

# Direcionar para outro grupo (NovoSolicitado = Id obtido em acoes/DIR/grupos)
curl -s -X POST "$BASE_URL/v1/requisicoes/$REQ_ID/acoes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "Id": "Direcionar [DIR]",
    "NovoSolicitado": "{ IdParticipante : 5, IdTipoPapel : 2 } ",
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
    "Descricao": "Problema resolvido. Acesso ao sistema liberado para o usuario.",
    "tipoAvaliacao": 2
}
resp = requests.post(
    f"{BASE_URL}/v1/requisicoes/{req_id}/acoes",
    json=payload_encerrar,
    headers=headers
)
resp.raise_for_status()
corpo = resp.json()
# Erros de negocio chegam com HTTP 200, em _metadata.MensagensErro.
# Algumas acoes respondem com um texto simples em vez do envelope.
erros = corpo["_metadata"]["MensagensErro"] if isinstance(corpo, dict) else []
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
    print(f"  {grupo['Texto']} -> {grupo['Id']}")

# Direcionar para o primeiro destino disponivel
if grupos:
    payload_dir = {
        "Id": "Direcionar [DIR]",
        "NovoSolicitado": grupos[0]["Id"],  # copie o valor exatamente como veio
        "Descricao": "Encaminhando para a equipe responsavel."
    }
    resp = requests.post(
        f"{BASE_URL}/v1/requisicoes/{req_id}/acoes",
        json=payload_dir,
        headers=headers
    )
    resp.raise_for_status()
    corpo = resp.json()
    erros = corpo["_metadata"]["MensagensErro"] if isinstance(corpo, dict) else []
    if erros:
        print(f"Direcionamento falhou: {erros}")
    else:
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
    Id            = "Encerrar [ENC]"
    Descricao     = "Problema resolvido. Acesso ao sistema liberado para o usuario."
    tipoAvaliacao = 2
} | ConvertTo-Json

$resp = Invoke-RestMethod `
    -Uri "$BaseUrl/v1/requisicoes/$ReqId/acoes" `
    -Method Post `
    -Headers $headers `
    -Body $payloadEncerrar `
    -ContentType "application/json"

# Erros de negocio chegam com HTTP 200, em _metadata.MensagensErro.
# Algumas acoes respondem com um texto simples em vez do envelope.
if ($resp -is [string] -or $resp._metadata.MensagensErro.Count -eq 0) {
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
    Write-Host "  $($grupo.Texto) -> $($grupo.Id)"
}

# Direcionar para o primeiro destino (NovoSolicitado = Id exatamente como veio)
if ($grupos.records.Count -gt 0) {
    $payloadDir = @{
        Id             = "Direcionar [DIR]"
        NovoSolicitado = $grupos.records[0].Id
        Descricao      = "Encaminhando para a equipe responsavel."
    } | ConvertTo-Json

    $resp = Invoke-RestMethod `
        -Uri "$BaseUrl/v1/requisicoes/$ReqId/acoes" `
        -Method Post `
        -Headers $headers `
        -Body $payloadDir `
        -ContentType "application/json"

    if ($resp -is [string] -or $resp._metadata.MensagensErro.Count -eq 0) {
        Write-Host "Direcionamento realizado com sucesso."
    } else {
        Write-Host "Direcionamento falhou: $($resp._metadata.MensagensErro -join ', ')"
    }
}
```

---

## Erros Comuns

Quase todos os erros ao executar uma acao voltam como **HTTP 200** com a mensagem em
`_metadata.MensagensErro`. A unica excecao e a falta de acesso a requisicao, que volta como
HTTP 406 com a mensagem em texto puro.

| Sintoma | Causa provavel | Solucao |
|---------|----------------|---------|
| HTTP 200 com `MensagensErro` contendo "Ação não encontrada" | O `Id` nao tem o codigo entre colchetes (ex.: `"ENC"`) ou nao existe para este formulario | Use o valor exato retornado por `GET /v1/requisicoes/{id}/acoes` (ex.: `"Encerrar [ENC]"`) |
| HTTP 200 com `MensagensErro` sobre permissao | A acao nao esta liberada para o usuario neste estado da requisicao | Consulte primeiro `GET /v1/requisicoes/{id}/acoes` e use apenas acoes listadas |
| HTTP 200 com `MensagensErro` pedindo o `Id` | O campo `Id` nao foi enviado ou veio vazio | Envie `Id` no payload |
| HTTP 200 com `MensagensErro` sobre campo obrigatorio | Falta `Descricao`, `tipoAvaliacao`, `NovoSolicitado` ou outro campo exigido pela acao | Veja `Campos` em `GET /v1/requisicoes/{id}/acoes` e envie os campos com `Obrigatoriedade: true` |
| HTTP 200 com `MensagensErro` sobre data | O campo `Data` nao e uma data valida | Envie a data em formato ISO (ex.: `2026-03-15T10:00:00`) |
| HTTP 406 em texto puro | Requisicao inexistente ou usuario sem acesso, ou corpo ausente | Confirme o ID da requisicao e envie o corpo JSON |
| HTTP 401 — Unauthorized | Token ausente, invalido ou usuario desativado | Faca login novamente e obtenha um novo token — veja [Autenticacao](../autenticacao.md) |
| `NovoSolicitado` nao aceito no `DIR` | O valor foi montado manualmente em vez de copiado da listagem | Copie o `Id` de `GET /v1/requisicoes/{id}/acoes/DIR/grupos` sem alteracoes |

---

## Proximos Passos

- [Gerenciar Anexos](anexos.md) — adicionar arquivos a uma requisicao existente
- [Consultar Requisicoes](consultar-requisicoes.md) — buscar pelo ID, listar abertas e ver historico de acoes
