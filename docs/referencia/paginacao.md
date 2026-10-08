# Paginacao e Limites de Listagem

A API BDesk **nao pagina** os resultados. Nao existem parametros para escolher pagina ou tamanho de pagina, e as respostas nao trazem informacoes de pagina (total de paginas, pagina atual). Cada listagem devolve **uma unica resposta**, limitada por um teto de registros proprio de cada endpoint.

Para controlar o volume de dados, use os limites e filtros descritos abaixo.

---

## Limites por endpoint

| Endpoint | Como limitar | Padrao |
|----------|--------------|--------|
| `GET` e `POST /v1/requisicoes/abertas` | Parametro `LimiteRequisicoes` (query no GET; campo do corpo JSON no POST) | 500 |
| `GET` e `POST /v1/requisicoes/encerradas` | Parametro `LimiteRequisicoes` (igual ao acima) | 500 |
| `POST /v1/requisicoes/busca` | Parametro de query `maximoPorSituacao`: maximo de requisicoes **por situacao** (aberta e encerrada) | 100 |
| `POST /v1/requisicoes/porcondicao/{condicao}/{quantidade}` | Segmento de rota `{quantidade}`: maximo de linhas | Sem padrao (obrigatorio na rota) |
| `GET /v1/ics` | Sem limite: devolve **todos** os itens de configuracao | Todos |

Outras listagens (catalogo de servicos, acoes de uma requisicao, anexos, historico, participantes) tambem devolvem o resultado completo em uma unica resposta. Na pesquisa de participantes, a resposta traz no maximo 100 itens.

> **Atencao:** quando a lista tem mais registros que o limite, a API devolve apenas os primeiros, **sem aviso** na resposta. Se o seu volume pode ultrapassar o limite, divida a consulta com filtros (veja abaixo) em vez de supor que recebeu tudo.

---

## Como reduzir o volume com filtros

As listagens `abertas` e `encerradas` aceitam filtros, que sao combinados entre si (todos precisam ser atendidos). No `GET` eles vao na query string; no `POST` vao no corpo JSON.

| Filtro | Tipo | O que faz |
|--------|------|-----------|
| `RequisicaoId` | inteiro | Uma requisicao especifica |
| `Assunto` | texto | Assunto exatamente igual ao informado |
| `NomeFormulario` / `NomeAtividade` | texto | Nome do formulario ou da atividade contendo o texto |
| `IdsFormularios` / `AtividadesIds` | lista de inteiros | Formularios ou atividades especificos |
| `DescricoesStatus` | lista de textos | Descricao do status (ex.: `Aberta`) |
| `GruposSolucionadores` | lista de inteiros | Grupo atualmente responsavel |
| `AbertoEntre.Inicio` / `AbertoEntre.Fim` | data e hora | Intervalo da data de abertura |
| `EncerradoEntre.Inicio` / `EncerradoEntre.Fim` | data e hora | Intervalo da data de encerramento |
| `LimiteRequisicoes` | inteiro | Maximo de requisicoes devolvidas |

Uma estrategia segura para extrair um volume grande e dividir por **periodo de abertura**: consulte um intervalo curto por vez (por exemplo, um mes) e some os resultados no cliente.

---

## Exemplos

### cURL

```bash
BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"

# Requisicoes abertas, com limite explicito de 50 registros
curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE_URL/v1/requisicoes/abertas?LimiteRequisicoes=50"

# Mesmo filtro por POST, com intervalo de abertura
curl -s -X POST "$BASE_URL/v1/requisicoes/abertas" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"LimiteRequisicoes":50,"AbertoEntre":{"Inicio":"2026-01-01T00:00:00","Fim":"2026-01-31T23:59:59"}}'
```

### Python

```python
import requests

BASE_URL = "https://sua-empresa.bdesk.com.br/askrest"
headers = {"Authorization": "Bearer SEU_TOKEN_AQUI"}

# Um mes por vez, para nao esbarrar no limite de registros
meses = [
    ("2026-01-01T00:00:00", "2026-01-31T23:59:59"),
    ("2026-02-01T00:00:00", "2026-02-28T23:59:59"),
]

todas = []
for inicio, fim in meses:
    resp = requests.post(
        f"{BASE_URL}/v1/requisicoes/abertas",
        json={"LimiteRequisicoes": 500, "AbertoEntre": {"Inicio": inicio, "Fim": fim}},
        headers=headers,
    )
    resp.raise_for_status()
    registros = resp.json()["records"]
    if len(registros) >= 500:
        print(f"Atencao: {inicio[:7]} atingiu o limite; divida o periodo em intervalos menores.")
    todas.extend(registros)

for req in todas:
    print(f"#{req['RequisicaoId']} - {req['Assunto']}")
```

### PowerShell

```powershell
$BaseUrl = "https://sua-empresa.bdesk.com.br/askrest"
$Headers = @{ Authorization = "Bearer SEU_TOKEN_AQUI" }

$corpo = @{
    LimiteRequisicoes = 500
    AbertoEntre = @{ Inicio = "2026-01-01T00:00:00"; Fim = "2026-01-31T23:59:59" }
} | ConvertTo-Json

$resp = Invoke-RestMethod -Uri "$BaseUrl/v1/requisicoes/abertas" -Method Post `
    -Headers $Headers -ContentType "application/json" -Body $corpo

if ($resp.records.Count -ge 500) {
    Write-Warning "O limite foi atingido; divida o periodo em intervalos menores."
}
foreach ($req in $resp.records) {
    Write-Host "#$($req.RequisicaoId) - $($req.Assunto)"
}
```

---

## Dicas

- **Defina o limite explicitamente** em `LimiteRequisicoes` para que o comportamento nao dependa do padrao do servidor.
- **Lista vazia nao e erro**: quando nenhum registro atende ao filtro, a resposta e 200 com `records` vazio.
- **Nao procure por `_metadata` de paginacao**: o objeto `_metadata` traz apenas `MensagensErro` e `LogAmigavel`.
- **Itens de configuracao**: `GET /v1/ics` devolve todos os itens de uma vez. Para ler um IC especifico, use `GET /v1/ics/{id}` (veja [Gestao de ICs](../guias/gestao-ics.md)).
- **Consulta com filtros avancados**: para pesquisar por dados adicionais, participantes ou ultima acao, use `POST /v1/requisicoes/busca` (veja [Consultar Requisicoes](../guias/consultar-requisicoes.md)).
