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

**Resposta:**

```json
{
  "_metadata": { "Release": "9.8.0", "MensagensErro": [] },
  "records": [
    { "Id": "{\"IdParticipante\":1803,\"IdTipoPapel\":1}", "Texto": "Joao Silva" },
    { "Id": "{\"IdParticipante\":1784,\"IdTipoPapel\":1}", "Texto": "Joao Pereira - TI" }
  ]
}
```

> **Atencao:** O campo `Id` e retornado como uma **string JSON codificada** contendo `IdParticipante` e `IdTipoPapel`. Ao usar esse valor em outros endpoints (ex.: atribuir analista em uma acao de workflow), envie-o como string exatamente como foi retornado.

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

A estrutura da resposta e identica ao endpoint anterior.

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

# Buscar participantes
resp = requests.get(
    f"{BASE_URL}/v1/participantes/{formulario_papel_id}/pesquisar/{termo}",
    headers=headers
)
resp.raise_for_status()

participantes = resp.json()["records"]
print(f"Participantes encontrados: {len(participantes)}")
for p in participantes:
    # Atencao: p["Id"] e uma string
    print(f"  Id={p['Id']} — {p['Texto']}")

# Usar o Id do primeiro resultado em uma acao de workflow (exemplo)
if participantes:
    id_participante = participantes[0]["Id"]  # string, ex.: "45"
    print(f"Usar Id '{id_participante}' no campo NovoSolicitado ao direcionar a requisicao")
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

Write-Host "Participantes encontrados: $($Resp.records.Count)"
foreach ($p in $Resp.records) {
    # Atencao: $p.Id e uma string
    Write-Host "  Id=$($p.Id) — $($p.Texto)"
}

# Buscar via query string (alternativa)
$Resp2 = Invoke-RestMethod `
    -Uri "$BaseUrl/v1/participantes/ObterParticipantes2?formularioPapelId=$FormularioPapelId&termo=$Termo" `
    -Headers $Headers

Write-Host "Resultado alternativo: $($Resp2.records.Count) participantes"
```

---

## Proximos Passos

Com os participantes identificados, voce pode:

- [Criar Requisicoes](criar-requisicoes.md) — usar o `Id` retornado ao definir participantes na abertura
- [Acoes de Workflow](acoes-workflow.md) — usar o `Id` no campo `NovoSolicitado` ao direcionar ou atribuir uma requisicao
