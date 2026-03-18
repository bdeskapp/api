# Consultar Requisicoes

## O que voce vai aprender

Como consultar, filtrar e obter detalhes de requisicoes via API — incluindo listar requisicoes abertas
e encerradas, buscar por ID, visualizar o historico de acoes e usar a busca avancada por texto.

## Pre-requisitos

- Token de autenticacao valido. Consulte o guia de [Autenticacao](../autenticacao.md) para obter o
  seu token antes de continuar.

---

## Passo a Passo

### Passo 1: Obter o Dashboard (Visao Geral)

Use `GET /v1/requisicoes` para obter um resumo das suas requisicoes — quantas estao abertas,
quantas foram encerradas, e os links para acessar cada lista.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta:**

```json
{
  "Abertas": 42,
  "Encerradas": 1580,
  "UrlRequisicoesAbertas": "/v1/requisicoes/abertas",
  "UrlRequisicoesEncerradas": "/v1/requisicoes/encerradas"
}
```

Use os valores de `Abertas` e `Encerradas` para monitorar o volume de demandas, ou como ponto de
partida antes de chamar os endpoints de listagem.

---

### Passo 2: Listar Requisicoes Abertas

Use `GET /v1/requisicoes/abertas` para obter a lista paginada de requisicoes em andamento.

**Parametros de paginacao:**

| Parametro    | Padrao | Maximo | Descricao                        |
|--------------|--------|--------|----------------------------------|
| `pageSize`   | 20     | 100    | Quantidade de registros por pagina |
| `pageNumber` | 1      | —      | Numero da pagina (comeca em 1)   |

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abertas?pageSize=20&pageNumber=1" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta:**

```json
{
  "_metadata": {
    "Release": "9.8.0",
    "MensagensErro": [],
    "LogAmigavel": [],
    "Pagination": {
      "TotalRecords": 42,
      "TotalPages": 3,
      "CurrentPage": 1,
      "PageSize": 20
    }
  },
  "records": [
    {
      "RequisicaoId": 35174,
      "Assunto": "Falha no servidor de email",
      "DataAbertura": "2026-03-15T10:30:00",
      "Status": "Em Atendimento",
      "Prioridade": "Alta"
    }
  ]
}
```

**Campos principais:**

| Campo          | Descricao                                      |
|----------------|------------------------------------------------|
| `RequisicaoId` | Identificador unico da requisicao              |
| `Assunto`      | Titulo descritivo da requisicao                |
| `DataAbertura` | Data e hora de abertura (formato ISO 8601)     |
| `Status`       | Situacao atual (ex.: "Em Atendimento")         |
| `Prioridade`   | Nivel de urgencia (ex.: "Alta", "Normal")      |

Para navegar entre paginas, incremente `pageNumber` ate que `CurrentPage` seja igual a `TotalPages`.

---

### Passo 3: Listar Requisicoes Encerradas

Use `GET /v1/requisicoes/encerradas` para obter a lista de requisicoes ja concluidas. O formato da
resposta e identico ao de requisicoes abertas, incluindo paginacao.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/encerradas?pageSize=20&pageNumber=1" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

A resposta segue a mesma estrutura do Passo 2 — use `_metadata.Pagination` para controlar a
navegacao entre paginas.

---

### Passo 4: Buscar Detalhes por ID

Use `GET /v1/requisicoes/{id}` para obter todas as informacoes de uma requisicao especifica,
incluindo campos de formulario, participantes e links de navegacao.

Substitua `{id}` pelo valor de `RequisicaoId` obtido na listagem.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/35174" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

A resposta inclui campos detalhados da requisicao e URLs para acoes relacionadas (historico, acoes
de workflow, etc.).

**Atencao:** O campo `Conjuntos` na resposta e um dicionario (objeto chave-valor), nao uma lista.
Cada chave representa um conjunto de dados do formulario, e o valor e outro objeto com os campos
preenchidos. Itere sobre as chaves ao processar esse campo.

Exemplo de acesso em Python:
```python
conjuntos = req_detail["Conjuntos"]
for nome_conjunto, dados in conjuntos.items():
    print(f"Conjunto: {nome_conjunto}")
```

---

### Passo 5: Ver Historico de Acoes

Use `GET /v1/requisicoes/{id}/historico` para obter a linha do tempo de todas as acoes realizadas
na requisicao — direccionamentos, comentarios, encerramento, etc.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/35174/historico" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta (exemplo de entrada no historico):**

```json
{
  "_metadata": { ... },
  "records": [
    {
      "DataAcao": "2026-03-15T11:45:00",
      "Acao": "Direcionar",
      "Descricao": "Requisicao direcionada para o grupo Infraestrutura.",
      "Responsavel": "Ana Silva"
    }
  ]
}
```

Use o historico para auditar o ciclo de vida da requisicao ou integrar com sistemas externos de
rastreamento.

---

### Busca Avancada

Use `POST /v1/requisicoes/busca` para localizar requisicoes por texto livre, retornando resultados
agrupados por situacao.

```bash
curl -s -X POST "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/busca" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -H "Content-Type: application/json" \
  -d '{
    "Texto": "servidor de email",
    "maximoPorSituacao": 50
  }'
```

**Parametros do corpo:**

| Campo               | Tipo    | Descricao                                         |
|---------------------|---------|---------------------------------------------------|
| `Texto`             | string  | Termo a buscar no assunto ou descricao            |
| `maximoPorSituacao` | inteiro | Numero maximo de resultados por situacao (status) |

Para opcoes completas de filtro, consulte a referencia de endpoints.

---

## Exemplos Completos

### cURL — Listar todas as requisicoes abertas

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abertas?pageSize=100&pageNumber=1" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

---

### Python — Listar abertas e obter detalhes

```python
import requests
import json

BASE_URL = "https://sua-empresa.bdesk.com.br/askrest"
headers = {"Authorization": "Bearer SEU_TOKEN_AQUI"}

# Listar requisicoes abertas
resp = requests.get(
    f"{BASE_URL}/v1/requisicoes/abertas",
    params={"pageSize": 100, "pageNumber": 1},
    headers=headers
)
resp.raise_for_status()
dados = resp.json()

total = dados["_metadata"]["Pagination"]["TotalRecords"]
print(f"Total de requisicoes abertas: {total}")

for req in dados["records"]:
    print(f"  #{req['RequisicaoId']} - {req['Assunto']} [{req['Status']}]")

# Obter detalhes da primeira requisicao
if dados["records"]:
    req_id = dados["records"][0]["RequisicaoId"]
    resp = requests.get(f"{BASE_URL}/v1/requisicoes/{req_id}", headers=headers)
    resp.raise_for_status()
    print(json.dumps(resp.json(), indent=2, ensure_ascii=False))
```

---

### PowerShell — Listar abertas e obter detalhes

```powershell
$BaseUrl = "https://sua-empresa.bdesk.com.br/askrest"
$headers = @{ Authorization = "Bearer SEU_TOKEN_AQUI" }

# Listar requisicoes abertas
$resp = Invoke-RestMethod -Uri "$BaseUrl/v1/requisicoes/abertas?pageSize=100" -Headers $headers
$total = $resp._metadata.Pagination.TotalRecords
Write-Host "Total de requisicoes abertas: $total"

foreach ($req in $resp.records) {
    Write-Host "  #$($req.RequisicaoId) - $($req.Assunto) [$($req.Status)]"
}

# Obter detalhes da primeira requisicao
if ($resp.records.Count -gt 0) {
    $reqId = $resp.records[0].RequisicaoId
    $detalhe = Invoke-RestMethod -Uri "$BaseUrl/v1/requisicoes/$reqId" -Headers $headers
    $detalhe | ConvertTo-Json -Depth 10
}
```

---

## Erros Comuns

| Sintoma                              | Causa provavel                                    | Solucao                                                   |
|--------------------------------------|---------------------------------------------------|-----------------------------------------------------------|
| HTTP 401 Unauthorized                | Token expirado ou ausente no cabecalho            | Faca login novamente e use o novo token                   |
| Lista vazia (`"records": []`)        | Sem requisicoes no contexto do usuario logado     | Verifique se o usuario tem permissao para ver requisicoes |
| HTTP 404 ao buscar por ID            | Requisicao nao existe ou usuario sem permissao    | Confirme o ID e as permissoes do usuario na plataforma    |
| HTTP 406 com `MensagensErro`         | Filtro invalido ou parametro fora do intervalo    | Leia a lista `_metadata.MensagensErro` na resposta        |
| Paginacao retorna mesmos registros   | `pageNumber` nao foi incrementado corretamente    | Verifique se `pageNumber` comeca em 1 e aumenta a cada pagina |

---

## Proximos Passos

- [Acoes de Workflow](acoes-workflow.md) — como executar acoes (direcionar, encerrar, comentar) via API
- [Criar Requisicoes](criar-requisicoes.md) — como abrir novas requisicoes programaticamente
- [Paginacao](../referencia/paginacao.md) — referencia completa sobre controle de paginas e limites
