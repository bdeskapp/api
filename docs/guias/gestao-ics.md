# Gestao de Itens de Configuracao (ICs)

## O que voce vai aprender

Como buscar, consultar e gerenciar Itens de Configuracao (ICs) via API BDesk — incluindo listar todos
os ICs, buscar por ID, explorar relacionamentos e gerenciar listas de papeis.

---

## Pre-requisitos

- **Token de autenticacao valido** — veja [Autenticacao](../autenticacao.md) para obter o seu token.

---

## O que sao Itens de Configuracao?

Itens de Configuracao (ICs) sao os ativos registrados no CMDB (Configuration Management Database)
do BDesk. Representam qualquer elemento da infraestrutura ou catalogo que pode ser associado a
requisicoes de servico:

- Servidores fisicos e virtuais
- Estacoes de trabalho e notebooks
- Softwares e licencas
- Equipamentos de rede (switches, roteadores, firewalls)
- Servicos e aplicacoes de negocio

Cada IC possui relacionamentos com outros ICs, usuarios responsaveis e listas de papeis que definem
quem pode visualiza-lo, edita-lo ou associa-lo a requisicoes.

---

## Passo a Passo

### Passo 1: Listar Todos os ICs

Use `GET /v1/ics` para obter a lista paginada de todos os Itens de Configuracao disponiveis.

**Parametros de paginacao:**

| Parametro | Padrao | Maximo | Descricao |
|-----------|--------|--------|-----------|
| `pageSize` | 20 | 100 | Quantidade de registros por pagina |
| `pageNumber` | 1 | — | Numero da pagina (comeca em 1) |

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/ics?pageSize=20&pageNumber=1" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta:**

```json
{
  "_metadata": {
    "Release": "9.8.0",
    "MensagensErro": [],
    "Pagination": {
      "TotalRecords": 340,
      "TotalPages": 17,
      "CurrentPage": 1,
      "PageSize": 20
    }
  },
  "records": [
    {
      "Id": 101,
      "Nome": "SRV-APP-01",
      "NomeCompleto": "Servidor de Aplicacao 01",
      "Tipo": "Servidor",
      "Classe": "Infraestrutura",
      "Numero": "IC-00101",
      "Status": "Ativo",
      "Ativo": true
    }
  ]
}
```

---

### Passo 2: Buscar um IC por ID

Use `GET /v1/ics/{id}` para obter os detalhes completos de um IC especifico, incluindo URLs para
acessar seus sub-recursos.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/ics/101" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Campos retornados:**

| Campo | Descricao |
|-------|-----------|
| `Id` | Identificador numerico unico do IC no BDesk |
| `Nome` | Nome curto do IC (ex.: `SRV-APP-01`) |
| `NomeCompleto` | Nome descritivo completo |
| `Tipo` | Tipo do IC (ex.: Servidor, Workstation, Software) |
| `Classe` | Classe de classificacao (ex.: Infraestrutura, Negocio) |
| `Numero` | Codigo de inventario (ex.: `IC-00101`) |
| `Status` | Estado atual (ex.: Ativo, Em Manutencao, Aposentado) |
| `Ativo` | `true` se o IC esta ativo no CMDB |

---

### Passo 3: Explorar Relacionamentos

Os ICs podem ter componentes, associacoes com outros ICs, usuarios responsaveis e relacionamentos em
arvore de dependencia.

#### Componentes

```bash
GET /v1/ics/{id}/componentes
```

Lista os componentes filhos do IC — por exemplo, os discos e interfaces de um servidor.

#### Associacoes

```bash
GET /v1/ics/{id}/associacoes
```

Retorna os ICs associados horizontalmente ao IC consultado — por exemplo, aplicacoes que rodam em
um servidor.

#### Usuarios Responsaveis

```bash
GET /v1/ics/{id}/usuarios
```

Lista os usuarios associados ao IC e seus papeis de responsabilidade.

#### Mapa de Relacionamentos por Nivel

```bash
GET /v1/ics/{id}/relacionamentos?nivel=3
```

Retorna o grafo de dependencias do IC ate o nivel especificado. Use `nivel=1` para dependencias
diretas, `nivel=3` para uma visao mais ampla da cadeia de impacto.

---

### Passo 4: Gerenciar Listas de Papeis

Listas de papeis definem grupos de usuarios com permissoes especificas sobre um IC.

#### Listar as Listas de Papeis do IC

```bash
GET /v1/ics/{id}/listaspapeis
```

**Exemplo:**

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/ics/101/listaspapeis" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta:**

```json
{
  "_metadata": { "Release": "9.8.0", "MensagensErro": [] },
  "records": [
    { "Id": 1, "Nome": "Responsaveis" },
    { "Id": 2, "Nome": "Usuarios Autorizados" }
  ]
}
```

#### Adicionar ou Remover Membros de uma Lista

```bash
POST /v1/ics/{id}/listaspapeis/{lista}/membros
```

Substitua `{lista}` pelo ID da lista retornado no passo anterior.

**Payload para adicionar um membro:**

```json
{
  "UsuarioId": 42,
  "Operacao": "Adicionar"
}
```

**Payload para remover um membro:**

```json
{
  "UsuarioId": 42,
  "Operacao": "Remover"
}
```

---

## Exemplos Completos

### cURL

```bash
BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"

# Listar ICs (primeira pagina)
curl -s "$BASE_URL/v1/ics?pageSize=20&pageNumber=1" \
  -H "Authorization: Bearer $TOKEN"

# Buscar IC especifico
curl -s "$BASE_URL/v1/ics/101" \
  -H "Authorization: Bearer $TOKEN"

# Listar componentes
curl -s "$BASE_URL/v1/ics/101/componentes" \
  -H "Authorization: Bearer $TOKEN"

# Mapa de relacionamentos ate nivel 2
curl -s "$BASE_URL/v1/ics/101/relacionamentos?nivel=2" \
  -H "Authorization: Bearer $TOKEN"

# Listar listas de papeis
curl -s "$BASE_URL/v1/ics/101/listaspapeis" \
  -H "Authorization: Bearer $TOKEN"

# Adicionar usuario a uma lista de papeis
curl -s -X POST "$BASE_URL/v1/ics/101/listaspapeis/1/membros" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "UsuarioId": 42, "Operacao": "Adicionar" }'
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

# Listar todos os ICs (com paginacao)
pagina  = 1
todos   = []
while True:
    resp = requests.get(
        f"{BASE_URL}/v1/ics",
        params={"pageSize": 100, "pageNumber": pagina},
        headers=headers
    )
    resp.raise_for_status()
    dados  = resp.json()
    todos += dados["records"]
    paginacao = dados["_metadata"].get("Pagination", {})
    if pagina >= paginacao.get("TotalPages", 1):
        break
    pagina += 1

print(f"Total de ICs: {len(todos)}")

# Buscar IC por ID e seus relacionamentos
ic_id = 101
resp = requests.get(f"{BASE_URL}/v1/ics/{ic_id}", headers=headers)
resp.raise_for_status()
ic = resp.json()
print(f"IC: {ic['Nome']} — Status: {ic['Status']}")

# Listar componentes
resp = requests.get(f"{BASE_URL}/v1/ics/{ic_id}/componentes", headers=headers)
resp.raise_for_status()
for comp in resp.json()["records"]:
    print(f"  Componente: {comp['Nome']}")

# Adicionar usuario a lista de papeis
payload = {"UsuarioId": 42, "Operacao": "Adicionar"}
resp = requests.post(
    f"{BASE_URL}/v1/ics/{ic_id}/listaspapeis/1/membros",
    json=payload,
    headers=headers
)
resp.raise_for_status()
print("Membro adicionado com sucesso.")
```

---

### PowerShell

```powershell
$BaseUrl = "https://sua-empresa.bdesk.com.br/askrest"
$Headers = @{
    Authorization  = "Bearer SEU_TOKEN_AQUI"
    "Content-Type" = "application/json"
}

# Listar ICs
$Resp = Invoke-RestMethod -Uri "$BaseUrl/v1/ics?pageSize=20&pageNumber=1" -Headers $Headers
Write-Host "Total de ICs: $($Resp._metadata.Pagination.TotalRecords)"
foreach ($ic in $Resp.records) {
    Write-Host "  [$($ic.Id)] $($ic.Nome) — $($ic.Status)"
}

# Buscar IC por ID
$IcId = 101
$Ic = Invoke-RestMethod -Uri "$BaseUrl/v1/ics/$IcId" -Headers $Headers
Write-Host "IC: $($Ic.NomeCompleto) | Tipo: $($Ic.Tipo)"

# Mapa de relacionamentos
$Rels = Invoke-RestMethod -Uri "$BaseUrl/v1/ics/$IcId/relacionamentos?nivel=2" -Headers $Headers
Write-Host "Relacionamentos encontrados: $($Rels.records.Count)"

# Adicionar membro a lista de papeis
$Payload = @{ UsuarioId = 42; Operacao = "Adicionar" } | ConvertTo-Json
Invoke-RestMethod `
    -Uri "$BaseUrl/v1/ics/$IcId/listaspapeis/1/membros" `
    -Method Post `
    -Headers $Headers `
    -Body $Payload `
    -ContentType "application/json" | Out-Null
Write-Host "Membro adicionado com sucesso."
```

---

## Proximos Passos

Com os ICs mapeados, voce pode:

- [Criar Requisicoes](criar-requisicoes.md) — associar um IC a uma requisicao ao abri-la
