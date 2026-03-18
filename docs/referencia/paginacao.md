# Paginacao

A API BDesk retorna resultados paginados para listagens que podem conter muitos registros. Isso garante respostas rapidas e evita sobrecarga na rede.

---

## Como Funciona

Ao consultar endpoints de listagem (como requisicoes abertas ou catalogo de servicos), a API retorna uma pagina de resultados por vez. Voce controla quantos itens recebe por pagina e qual pagina deseja acessar usando parametros de query string.

---

## Parametros

| Parametro    | Tipo    | Padrao | Maximo | Descricao                          |
|--------------|---------|--------|--------|------------------------------------|
| `pageSize`   | inteiro | 20     | 100    | Quantidade de itens por pagina     |
| `pageNumber` | inteiro | 1      | —      | Numero da pagina desejada (inicia em 1) |

**Exemplo de URL:**

```
GET https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abertas?pageSize=50&pageNumber=2
```

---

## Formato da Resposta

Toda resposta paginada inclui um objeto `_metadata.Pagination` com informacoes sobre a paginacao atual:

```json
{
  "_metadata": {
    "Pagination": {
      "TotalRecords": 150,
      "TotalPages": 8,
      "CurrentPage": 1,
      "PageSize": 20
    }
  },
  "records": [...]
}
```

| Campo          | Descricao                                  |
|----------------|--------------------------------------------|
| `TotalRecords` | Total de registros encontrados             |
| `TotalPages`   | Total de paginas disponíveis               |
| `CurrentPage`  | Pagina atual retornada                     |
| `PageSize`     | Quantidade de itens retornados nesta pagina |

---

## Exemplo: Navegar Todas as Paginas

### cURL

```bash
BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"
PAGE=1

while true; do
  RESPONSE=$(curl -s -H "Authorization: Bearer $TOKEN" \
    "$BASE_URL/v1/requisicoes/abertas?pageSize=100&pageNumber=$PAGE")

  echo "$RESPONSE" | node -e "
    const d = JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));
    d.records.forEach(r => console.log('#' + r.RequisicaoId + ' - ' + r.Assunto));
  "

  TOTAL_PAGES=$(echo "$RESPONSE" | node -e "
    const d = JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));
    process.stdout.write(String(d._metadata.Pagination.TotalPages));
  ")

  if [ "$PAGE" -ge "$TOTAL_PAGES" ]; then
    break
  fi
  PAGE=$((PAGE + 1))
done
```

### Python

```python
import requests

BASE_URL = "https://sua-empresa.bdesk.com.br/askrest"
headers = {"Authorization": "Bearer SEU_TOKEN_AQUI"}

page = 1
while True:
    resp = requests.get(
        f"{BASE_URL}/v1/requisicoes/abertas",
        params={"pageSize": 100, "pageNumber": page},
        headers=headers
    )
    dados = resp.json()

    for req in dados["records"]:
        print(f"#{req['RequisicaoId']} - {req['Assunto']}")

    if page >= dados["_metadata"]["Pagination"]["TotalPages"]:
        break
    page += 1
```

### PowerShell

```powershell
$BaseUrl = "https://sua-empresa.bdesk.com.br/askrest"
$Token   = "SEU_TOKEN_AQUI"
$Headers = @{ Authorization = "Bearer $Token" }
$Page    = 1

do {
    $Response = Invoke-RestMethod `
        -Uri "$BaseUrl/v1/requisicoes/abertas?pageSize=100&pageNumber=$Page" `
        -Headers $Headers

    foreach ($req in $Response.records) {
        Write-Host "#$($req.RequisicaoId) - $($req.Assunto)"
    }

    $TotalPages = $Response._metadata.Pagination.TotalPages
    $Page++
} while ($Page -le $TotalPages)
```

---

## Dicas

- **Use `pageSize=100`** para reduzir o numero de chamadas necessarias para percorrer todos os registros.
- **Sempre verifique `TotalPages`** antes de iterar: se `TotalPages` for 1, nao ha proxima pagina.
- **Pagina inexistente**: se `pageNumber` for maior que `TotalPages`, a API retorna `records` vazio — trate esse caso no seu codigo.
- **Combinacao com filtros**: os parametros de paginacao funcionam junto com outros filtros de query string suportados pelo endpoint.
