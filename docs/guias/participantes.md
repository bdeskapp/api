# Participantes

## O que voce vai aprender

Como buscar participantes (usuarios, grupos e analistas) e entender os papeis que cada um pode
exercer dentro de uma requisicao no BDesk.

---

## Pre-requisitos

- **Token de autenticacao valido** — veja [Autenticacao](../autenticacao.md) para obter o seu token.

---

## Passo a Passo

### Buscar Participantes por Nome

Use este endpoint para pesquisar usuarios ou grupos disponiveis para um determinado papel em um
formulario. E o mesmo endpoint usado pela interface do BDesk no campo de selecionador de participante.

**Endpoint:** `GET /v1/participantes/{formularioPapelId}/pesquisar/{termo}`

| Parametro | Tipo | Descricao |
|-----------|------|-----------|
| `formularioPapelId` | inteiro | ID do par formulario + papel configurado no BDesk |
| `termo` | texto | Texto para busca (nome parcial ou completo) |

**Exemplo:**

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/participantes/7/pesquisar/joao" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta** (um **array direto**, sem envelope `_metadata`/`records`; ate 100 itens):

```json
[
  { "Id": "{\"IdParticipante\":1803,\"IdTipoPapel\":1}", "Texto": "Joao Silva", "Legenda": null, "IdDominioPai": null },
  { "Id": "{\"IdParticipante\":1784,\"IdTipoPapel\":1}", "Texto": "Joao Pereira - TI", "Legenda": null, "IdDominioPai": null }
]
```

> **Atencao:** O campo `Id` e uma **string que contem JSON** (com `IdParticipante` e `IdTipoPapel`),
> e nao um numero nem um objeto. Ao usar esse valor em outros endpoints (ex.: indicar o participante
> como participante do formulario na abertura da requisicao), envie-o como string exatamente como foi
> retornado, sem converter. `Legenda` e `IdDominioPai` vem sempre `null` nesta busca.

---

### Buscar Participantes (Alternativa via Query String)

Este endpoint oferece a mesma funcionalidade com parametros passados via query string, util para
integracao com ferramentas que nao suportam segmentos de URL dinamicos.

**Endpoint:** `GET /v1/participantes/ObterParticipantes2?formularioPapelId={X}&termo={Y}`

**Exemplo:**

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/participantes/ObterParticipantes2?formularioPapelId=7&termo=joao" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

Os itens (`Id`, `Texto`) sao os mesmos do endpoint anterior, mas aqui a resposta **tem
envelope**: os itens ficam em `records`, dentro de `{ "_metadata": {...}, "records": [...] }`.
Os dois parametros sao obrigatorios; se algum faltar, a API tende a responder 404.

---

## Papeis no BDesk

Cada participante de uma requisicao exerce um papel especifico. A tabela abaixo lista os papeis
padrao do sistema:

| ID | Papel | Descricao |
|----|-------|-----------|
| 1 | Registrador | Quem registrou a requisicao no sistema |
| 2 | Solicitante | Quem solicita o atendimento (beneficiario) |
| 3 | Solicitado | Analista ou grupo responsavel pelo atendimento |
| 9 | Copiado | Recebe copias das notificacoes da requisicao |
| 12 | Sistema | Acoes automatizadas realizadas pelo proprio sistema |
| 14 | Administrador do Processo | Gerencia e administra o fluxo da requisicao |
| 21 | E-Mail | Participante que interage via e-mail |

Esses IDs sao usados internamente pelo BDesk e podem ser referenciados em configuracoes de formularios,
regras de fluxo e relatorios.

---

## Exemplos Completos

### cURL

```bash
BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"

# Buscar participantes pelo nome (formato path)
curl -s "$BASE_URL/v1/participantes/7/pesquisar/maria" \
  -H "Authorization: Bearer $TOKEN"

# Buscar participantes pelo nome (formato query string)
curl -s "$BASE_URL/v1/participantes/ObterParticipantes2?formularioPapelId=7&termo=maria" \
  -H "Authorization: Bearer $TOKEN"
```

---

### Python

```python
import requests

BASE_URL = "https://sua-empresa.bdesk.com.br/askrest"
headers  = {
    "Authorization": "Bearer SEU_TOKEN_AQUI",
    "Content-Type": "application/json"
}

formulario_papel_id = 7
termo               = "maria"

# Buscar participantes (esta rota devolve um array direto, sem "records")
resp = requests.get(
    f"{BASE_URL}/v1/participantes/{formulario_papel_id}/pesquisar/{termo}",
    headers=headers
)
resp.raise_for_status()

participantes = resp.json()
print(f"Participantes encontrados: {len(participantes)}")
for p in participantes:
    # Atencao: p["Id"] e uma string que contem JSON
    print(f"  Id={p['Id']} — {p['Texto']}")

# Usar o Id do primeiro resultado como veio, sem converter (exemplo)
if participantes:
    id_participante = participantes[0]["Id"]  # ex.: '{"IdParticipante":123,"IdTipoPapel":2}'
    print(f"Enviar o texto '{id_participante}' como valor do participante, sem alterar")
```

---

### PowerShell

```powershell
$BaseUrl          = "https://sua-empresa.bdesk.com.br/askrest"
$Headers          = @{ Authorization = "Bearer SEU_TOKEN_AQUI" }
$FormularioPapelId = 7
$Termo            = "maria"

# Buscar participantes (formato path)
$Resp = Invoke-RestMethod `
    -Uri "$BaseUrl/v1/participantes/$FormularioPapelId/pesquisar/$Termo" `
    -Headers $Headers

# Esta rota devolve um array direto, sem "records"
Write-Host "Participantes encontrados: $(@($Resp).Count)"
foreach ($p in $Resp) {
    # Atencao: $p.Id e uma string que contem JSON
    Write-Host "  Id=$($p.Id) — $($p.Texto)"
}

# Buscar via query string (alternativa)
$Resp2 = Invoke-RestMethod `
    -Uri "$BaseUrl/v1/participantes/ObterParticipantes2?formularioPapelId=$FormularioPapelId&termo=$Termo" `
    -Headers $Headers

# Esta rota (ObterParticipantes2) tem envelope: os itens ficam em "records"
Write-Host "Resultado alternativo: $($Resp2.records.Count) participantes"
```

---

## Proximos Passos

Com os participantes identificados, voce pode:

- [Criar Requisicoes](criar-requisicoes.md) — usar o `Id` retornado ao definir participantes na abertura
- [Acoes de Workflow](acoes-workflow.md) — para direcionar uma requisicao, use o `Id` retornado por `GET /v1/requisicoes/{id}/acoes/DIR/grupos`, e nao o desta busca
