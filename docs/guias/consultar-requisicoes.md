# Consultar Requisicoes

## O que voce vai aprender

Como consultar, filtrar e obter detalhes de requisicoes via API — incluindo listar requisicoes abertas
e encerradas com filtros, buscar por ID, visualizar o historico de acoes e usar a busca avancada.

## Pre-requisitos

- Token de autenticacao valido. Consulte o guia de [Autenticacao](../autenticacao.md) para obter o
  seu token antes de continuar.

---

## Passo a Passo

### Passo 1: Obter o Dashboard (Visao Geral)

Use `GET /v1/requisicoes` para obter os totais de requisicoes — quantas estao abertas,
quantas foram encerradas, e os links para acessar cada lista.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta** (objeto direto, sem `_metadata` nem `records`):

```json
{
  "Abertas": 42,
  "Encerradas": 1580,
  "UrlRequisicoesAbertas": "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abertas",
  "UrlRequisicoesEncerradas": "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/encerradas"
}
```

`Abertas` e `Encerradas` sao os totais do ambiente inteiro, e nao apenas das requisicoes do usuario
autenticado. Use-os para monitorar o volume de demandas, ou como ponto de partida antes de chamar
as listagens.

---

### Passo 2: Listar Requisicoes Abertas

Use `GET /v1/requisicoes/abertas` para obter a lista de requisicoes em andamento a que o usuario
autenticado tem acesso.

**Esta rota nao tem paginacao.** Voce nao recebe "paginas": a resposta traz ate um limite de
registros, que voce controla com `LimiteRequisicoes` (padrao: 500). Para trazer menos resultados,
reduza o limite ou use os filtros abaixo.

**Filtros (parametros de query, todos opcionais):**

| Parametro | Tipo | Descricao |
|-----------|------|-----------|
| `RequisicaoId` | inteiro | Numero de uma requisicao especifica |
| `Assunto` | texto | Assunto igual ao informado (comparacao exata) |
| `NomeFormulario` | texto | Nome do formulario contendo o texto |
| `NomeAtividade` | texto | Nome da atividade contendo o texto |
| `IdsFormularios` | lista de inteiros | Ids de formularios (repita o parametro para varios valores) |
| `AtividadesIds` | lista de inteiros | Ids de atividades |
| `DescricoesStatus` | lista de textos | Descricoes de status (ex.: `Aberta`) |
| `GruposSolucionadores` | lista de inteiros | Ids dos grupos solucionadores atuais |
| `AbertoEntre.Inicio` / `AbertoEntre.Fim` | data e hora | Intervalo da data de abertura |
| `EncerradoEntre.Inicio` / `EncerradoEntre.Fim` | data e hora | Intervalo da data de encerramento |
| `LimiteRequisicoes` | inteiro | Maximo de registros devolvidos (padrao 500) |

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abertas?LimiteRequisicoes=50&NomeFormulario=Incidente" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

Se voce precisar de filtros complexos (por exemplo, por dados adicionais do formulario), envie os
mesmos campos como JSON no corpo de `POST /v1/requisicoes/abertas`. Um corpo `{}` e valido e
devolve ate o limite padrao de registros.

**Resposta:**

```json
{
  "_metadata": {
    "MensagensErro": [],
    "LogAmigavel": []
  },
  "records": [
    {
      "RequisicaoId": 35174,
      "Assunto": "Falha no servidor de email",
      "FormularioId": 12,
      "FormularioStatusId": 3,
      "Status": "Em Atendimento",
      "DescricaoAtividade": "Incidente de infraestrutura",
      "DataAbertura": "2026-03-15T10:30:00",
      "DataFimPrevisto": "2026-03-18T10:30:00",
      "GrupoAtualId": 7,
      "UrlDetalhesRequisicao": "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/35174",
      "UrlHistoricoRequisicao": "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/35174/historico"
    }
  ]
}
```

**Campos principais:**

| Campo          | Descricao                                      |
|----------------|------------------------------------------------|
| `RequisicaoId` | Identificador unico da requisicao              |
| `Assunto`      | Titulo descritivo da requisicao                |
| `Status`       | Situacao atual (ex.: "Em Atendimento")         |
| `DataAbertura` | Data e hora de abertura (formato ISO 8601)     |
| `DataFimPrevisto` | Data prevista de conclusao                  |
| `UltimaAcao`, `UltimaAcaoQuando`, `UltimaAcaoQuem` | Ultima acao executada, quando e por quem |
| `Solicitante`, `Solicitado`, `Responsavel` | Participantes principais       |
| `UrlDetalhesRequisicao` | URL para obter os detalhes (Passo 4)   |

Se nenhuma requisicao atender aos filtros, a resposta e HTTP 200 com `"records": []`.

---

### Passo 3: Listar Requisicoes Encerradas

Use `GET /v1/requisicoes/encerradas` para obter a lista de requisicoes ja concluidas. Os filtros e o
formato da resposta sao os mesmos de requisicoes abertas (incluindo `LimiteRequisicoes`, sem
paginacao). O filtro `EncerradoEntre` faz sentido principalmente aqui.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/encerradas?LimiteRequisicoes=100&EncerradoEntre.Inicio=2026-01-01&EncerradoEntre.Fim=2026-01-31" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

Tambem existe `POST /v1/requisicoes/encerradas`, que aceita os filtros como JSON no corpo.

---

### Passo 4: Buscar Detalhes por ID

Use `GET /v1/requisicoes/{id}` para obter todas as informacoes de uma requisicao especifica,
incluindo campos de formulario, participantes e links de navegacao.

Substitua `{id}` pelo valor de `RequisicaoId` obtido na listagem.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/35174" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

A resposta e um objeto direto (sem `_metadata`/`records`) com os campos agrupados em `Conjuntos`,
as URLs de navegacao (`UrlDadosAdicionais`, `UrlAcoes`, `UrlHistorico`, `UrlRequisicoesVinculadas`),
a lista `MensagensErro` e os tempos de atendimento em `Tempos`.

**Atencao:** O campo `Conjuntos` na resposta e um dicionario (objeto chave-valor), nao uma lista.
Cada chave representa um conjunto de dados do formulario, e o valor e outro objeto com os campos
preenchidos. Itere sobre as chaves ao processar esse campo. Os conjuntos "Detalhes Do Pedido",
"Agendamento do Pedido" e "Participantes" estao sempre presentes; os demais dependem do formulario.

Exemplo de acesso em Python:
```python
conjuntos = req_detail["Conjuntos"]
for nome_conjunto, dados in conjuntos.items():
    print(f"Conjunto: {nome_conjunto}")
```

**Requisicao inexistente ou sem acesso:** a API responde HTTP **200** (nao 404), com `Conjuntos`
nulo e a mensagem em `MensagensErro`:

```json
{
  "Conjuntos": null,
  "MensagensErro": ["Requisição inexistente ou usuário sem acesso"]
}
```

Por isso, depois de cada consulta por ID, verifique se `MensagensErro` esta vazio (ou se
`Conjuntos` e nulo) antes de usar os dados.

---

### Passo 5: Ver Historico de Acoes

Use `GET /v1/requisicoes/{id}/historico` para obter a linha do tempo de todas as acoes realizadas
na requisicao — direcionamentos, comentarios, encerramento, etc.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/35174/historico" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta (exemplo de entrada no historico):**

```json
{
  "_metadata": { "MensagensErro": [], "LogAmigavel": [] },
  "records": [
    {
      "Id": 987654,
      "RequisicaoId": 35174,
      "Data": "2026-03-15 11:45",
      "Acao": "Direcionar",
      "CdAcao": "DIR",
      "Descricao": "Requisicao direcionada para o grupo Infraestrutura.",
      "Usuario": "Ana Silva",
      "Grupo": "Infraestrutura"
    }
  ]
}
```

Cada item traz ainda outros campos, como `Solicitado`, `Status`, `Motivo` e `TipoRegistro`. Para
filtrar por tipo de acao, envie um `POST /v1/requisicoes/{id}/historico` com um corpo opcional
como `{ "CdAcao": ["ENC", "DIR"] }`.

Use o historico para auditar o ciclo de vida da requisicao ou integrar com sistemas externos de
rastreamento.

---

### Busca Avancada

Use `POST /v1/requisicoes/busca` para localizar requisicoes combinando varios filtros no corpo
JSON. A busca retorna um resumo de cada requisicao encontrada, abertas e encerradas, limitado por
situacao.

O limite de resultados por situacao e o parametro de **query** `maximoPorSituacao` (padrao 100),
e **nao** um campo do corpo.

```bash
curl -s -X POST "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/busca?maximoPorSituacao=50" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -H "Content-Type: application/json" \
  -d '{
    "Assunto": "email",
    "AbertasDesde": "2026-01-01",
    "AbertasAte": "2026-03-31"
  }'
```

**Campos do corpo (todos opcionais):**

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `Requisicao` | inteiro | Numero da requisicao |
| `Area` | inteiro | Id da area |
| `Formulario` | objeto | `{ "Id": <id do formulario>, "Dados": [...] }` |
| `DadosAdicionais` | lista | `[ { "DadoAdicionalId": 10, "Valor": [...], "ValorParcial": [...] } ]` |
| `Categoria`, `Subcategoria`, `Tarefa`, `Localizacao` | inteiro | Filtros de classificacao |
| `Assunto` | texto | Assunto da requisicao |
| `AbertasDesde` / `AbertasAte` | data | Intervalo de abertura |
| `EncerradasDesde` / `EncerradasAte` | data | Intervalo de encerramento |
| `UltimaAcaoExecutada_Id` | texto | Ultima acao executada, com o codigo entre colchetes (ex.: `Encerrar [ENC]`) |
| `UltimaAcaoExecutada_Desde` / `UltimaAcaoExecutada_Ate` | data | Intervalo da ultima acao |
| `Solicitantes`, `Solicitados`, `Responsaveis`, `Beneficiados`, `Registradores`, `ICs` | lista | Itens `{ "Participante": <id>, "TipoPapel": <id do papel> }` |

**Nao existe um campo de busca por texto livre** (como `Texto`): campos desconhecidos no corpo sao
ignorados sem erro. Para dados adicionais, use `Valor` para dados do tipo dominio e `ValorParcial`
para dados do tipo texto.

**Resposta:** um **array JSON direto** (sem `_metadata` nem `records`):

```json
[
  {
    "Id": 35174,
    "FormularioId": 12,
    "Status": "Em Atendimento",
    "Assunto": "Falha no servidor de email",
    "Atividade": "Incidente de infraestrutura",
    "Categoria": "Infraestrutura",
    "SubCategoria": "Email",
    "Descricao": "O servidor de email nao responde.",
    "Responsavel": "Ana Silva"
  }
]
```

**Erros:** os erros de validacao (por exemplo, acao nao encontrada em `UltimaAcaoExecutada_Id` ou
dado adicional nao encontrado) voltam como HTTP 406 com a mensagem em texto puro.

---

## Exemplos Completos

### cURL — Listar requisicoes abertas

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abertas?LimiteRequisicoes=100" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

---

### Python — Listar abertas e obter detalhes

```python
import requests
import json

BASE_URL = "https://sua-empresa.bdesk.com.br/askrest"
headers = {"Authorization": "Bearer SEU_TOKEN_AQUI"}

# Listar requisicoes abertas (sem paginacao: ate LimiteRequisicoes registros)
resp = requests.get(
    f"{BASE_URL}/v1/requisicoes/abertas",
    params={"LimiteRequisicoes": 100},
    headers=headers
)
resp.raise_for_status()
dados = resp.json()

print(f"Requisicoes abertas retornadas: {len(dados['records'])}")

for req in dados["records"]:
    print(f"  #{req['RequisicaoId']} - {req['Assunto']} [{req['Status']}]")

# Obter detalhes da primeira requisicao
if dados["records"]:
    req_id = dados["records"][0]["RequisicaoId"]
    resp = requests.get(f"{BASE_URL}/v1/requisicoes/{req_id}", headers=headers)
    resp.raise_for_status()
    detalhe = resp.json()
    # Requisicao inexistente ou sem acesso volta com HTTP 200 e MensagensErro preenchido
    if detalhe["MensagensErro"]:
        print("Erro:", detalhe["MensagensErro"])
    else:
        print(json.dumps(detalhe, indent=2, ensure_ascii=False))
```

---

### PowerShell — Listar abertas e obter detalhes

```powershell
$BaseUrl = "https://sua-empresa.bdesk.com.br/askrest"
$headers = @{ Authorization = "Bearer SEU_TOKEN_AQUI" }

# Listar requisicoes abertas (sem paginacao: ate LimiteRequisicoes registros)
$resp = Invoke-RestMethod -Uri "$BaseUrl/v1/requisicoes/abertas?LimiteRequisicoes=100" -Headers $headers
Write-Host "Requisicoes abertas retornadas: $($resp.records.Count)"

foreach ($req in $resp.records) {
    Write-Host "  #$($req.RequisicaoId) - $($req.Assunto) [$($req.Status)]"
}

# Obter detalhes da primeira requisicao
if ($resp.records.Count -gt 0) {
    $reqId = $resp.records[0].RequisicaoId
    $detalhe = Invoke-RestMethod -Uri "$BaseUrl/v1/requisicoes/$reqId" -Headers $headers
    if ($detalhe.MensagensErro.Count -gt 0) {
        Write-Warning ($detalhe.MensagensErro -join "; ")
    } else {
        $detalhe | ConvertTo-Json -Depth 10
    }
}
```

---

## Erros Comuns

| Sintoma                              | Causa provavel                                    | Solucao                                                   |
|--------------------------------------|---------------------------------------------------|-----------------------------------------------------------|
| HTTP 401 Unauthorized                | Token ausente, invalido ou usuario desativado     | Faca login novamente e use o novo token                   |
| Lista vazia (`"records": []`)        | Nenhuma requisicao atende aos filtros ou o usuario nao tem acesso | Revise os filtros e as permissoes do usuario |
| HTTP 200 com `Conjuntos` nulo e `MensagensErro` preenchido | Requisicao nao existe ou usuario sem acesso (consulta por ID) | Confirme o ID e as permissoes do usuario; sempre teste `MensagensErro` |
| HTTP 406 com mensagem em texto puro  | JSON do corpo invalido ou filtro da busca avancada invalido | Leia o corpo da resposta (texto puro, nao JSON)         |
| HTTP 500 em `POST` de listagem       | Corpo `null`                                      | Envie ao menos `{}` como corpo (em `POST /v1/requisicoes/busca`, corpo `null` da 406, nao 500) |
| Resposta maior que o esperado        | Nao ha paginacao; o padrao e ate 500 registros    | Use `LimiteRequisicoes` e os filtros para reduzir         |

---

## Proximos Passos

- [Acoes de Workflow](acoes-workflow.md) — como executar acoes (direcionar, encerrar, comentar) via API
- [Criar Requisicoes](criar-requisicoes.md) — como abrir novas requisicoes programaticamente
- [Limites e Volumes](../referencia/limites.md) — referencia sobre limites de resposta
