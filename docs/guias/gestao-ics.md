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

Use `GET /v1/ics` para obter a lista de todos os Itens de Configuracao do ambiente.

**Atencao:** esta rota **nao tem paginacao nem filtros**. Cada chamada devolve todos os ICs de uma
vez, entao a resposta pode ser grande em ambientes com muitos ICs. Parametros de pagina (tamanho ou
numero da pagina) nao existem e sao ignorados. Para filtrar, faca isso no seu lado, depois de receber a lista.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/ics" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta:**

```json
{
  "_metadata": {
    "Release": null,
    "LogAmigavel": null,
    "MensagensErro": null
  },
  "records": [
    {
      "Id": 101,
      "Nome": "SRV-APP-01",
      "NomeCompleto": "Servidor de Aplicacao 01",
      "Tipo": { "Id": 3, "Nome": "Servidor" },
      "Classe": { "Id": 1, "Nome": "Infraestrutura" },
      "Numero": "IC-00101",
      "Status": { "Id": 1, "Nome": "Ativo" },
      "Ativo": true,
      "Relacoes": "https://sua-empresa.bdesk.com.br/askrest/v1/ics/101/relacoes",
      "Detalhes": "https://sua-empresa.bdesk.com.br/askrest/v1/ics/101"
    }
  ]
}
```

Nesta rota `_metadata` vem com valores nulos; nao use esse campo para detectar erros. `Tipo`,
`Classe` e `Status` (e tambem `Localizacao` e `Marca`) sao objetos com `Id` e `Nome`, nao texto.
Cada registro traz tambem as URLs dos sub-recursos (`Relacoes`, `Detalhes`, `Componentes`,
`Associacoes`, `Usuarios`, `Relacionamentos` e `ListasPapeis`).

---

### Passo 2: Buscar um IC por ID

Use `GET /v1/ics/{id}` para obter os detalhes completos de um IC especifico, incluindo URLs para
acessar seus sub-recursos. A resposta e o proprio objeto do IC, sem envelope `_metadata`/`records`.

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
| `Tipo` | Objeto `{ "Id", "Nome" }` com o tipo do IC (ex.: Servidor, Workstation, Software) |
| `Classe` | Objeto `{ "Id", "Nome" }` com a classe de classificacao (ex.: Infraestrutura, Negocio) |
| `Numero` | Codigo de inventario (ex.: `IC-00101`) |
| `Status` | Objeto `{ "Id", "Nome" }` com o estado atual (ex.: Ativo, Em Manutencao, Aposentado) |
| `Ativo` | `true` se o IC esta ativo no CMDB |
| `Localizacao`, `Marca`, `Hierarquia`, `Responsavel`, `GrupoDeAcesso` | Objetos com `Id` e `Nome` (alguns trazem campos extras) |
| `Ralacoes`, `Detalhes`, `Componentes`, `Associacoes`, `Usuarios`, `Relacionamentos`, `ListasPapeis` | URLs dos sub-recursos do IC |

> **Atencao a grafia:** no detalhe do IC (`GET /v1/ics/{id}`), a URL da rota `/relacoes` vem na
> chave **`Ralacoes`** (com "a", e nao "Relacoes"). Na listagem (`GET /v1/ics`) a chave se chama
> `Relacoes`. Ao ler o detalhe, use exatamente `Ralacoes`.

Alem desses campos, o detalhe traz uma chave para cada dado adicional cadastrado no IC. Para um IC
que nao existe, o servidor pode responder com erro (HTTP 500) em vez de 404.

---

### Passo 3: Explorar Relacionamentos

Os ICs podem ter componentes, associacoes com outros ICs, usuarios responsaveis e relacionamentos em
arvore de dependencia.

#### Componentes

```bash
GET /v1/ics/{id}/componentes
```

Lista os componentes do IC — por exemplo, os discos e interfaces de um servidor. Cada item de
`records` traz `Id`, `Tipo`, `Quantidade`, `Descricao`, `Relacionamento` e `Temporario`.

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

Lista os usuarios associados ao IC e seus papeis de responsabilidade. Cada item de `records` traz
`Id`, `Nome`, `TipoAssociacao`, `UsuarioDesde`, `IdRelacao` e `DescricaoRelacao`.

#### Tudo de uma vez (Relacoes)

```bash
GET /v1/ics/{id}/relacoes
```

Devolve numa unica chamada as tres listas, **sem** o envelope `_metadata`/`records`:
`{ "Usuarios": [...], "Componentes": [...], "Associacoes": [...] }`.

#### Mapa de Relacionamentos por Nivel

```bash
GET /v1/ics/{id}/relacionamentos?nivel=3
```

Retorna o grafo de dependencias do IC ate o nivel especificado (padrao 3). Use `nivel=1` para
dependencias diretas, `nivel=3` para uma visao mais ampla da cadeia de impacto. Cada item de
`records` traz `Id`, `Nome`, `IdPai` (para remontar a arvore), `Classe`, `TipoIc`, `Marca`,
`Relacao`, `Nivel` e outros campos.

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
  "_metadata": {},
  "records": [
    {
      "Id": 1,
      "Nome": "Responsaveis",
      "Detalhes": "https://sua-empresa.bdesk.com.br/askrest/v1/ics/101/listaspapeis/1",
      "Membros": "https://sua-empresa.bdesk.com.br/askrest/v1/ics/101/listaspapeis/1/membros"
    },
    {
      "Id": 2,
      "Nome": "Usuarios Autorizados",
      "Detalhes": "https://sua-empresa.bdesk.com.br/askrest/v1/ics/101/listaspapeis/2",
      "Membros": "https://sua-empresa.bdesk.com.br/askrest/v1/ics/101/listaspapeis/2/membros"
    }
  ]
}
```

`Detalhes` e `Membros` sao as URLs para consultar a lista e os usuarios que fazem parte dela. A
consulta de membros (`GET .../membros`) devolve `records` com `Id` e `Nome` de cada usuario.

#### Adicionar ou Remover Membros de uma Lista

```bash
POST /v1/ics/{id}/listaspapeis/{lista}/membros
```

Substitua `{lista}` pelo ID da lista retornado no passo anterior. O corpo (JSON) tem duas listas
opcionais, `Inserir` e `Remover`, e cada uma recebe **nomes de exibicao** dos usuarios (o nome
como aparece no cadastro do usuario, por exemplo `Maria Silva`). **Nao** envie login nem id numerico:
o BDesk localiza o usuario pelo nome.

**Payload para adicionar e remover membros:**

```json
{
  "Inserir": ["Maria Silva"],
  "Remover": ["Joao Souza"]
}
```

Voce pode enviar so `Inserir` ou so `Remover`, e varios nomes de uma vez. Os nomes em `Remover`
sao processados antes dos de `Inserir`.

**Resposta (HTTP 200):**

```json
{
  "Remover": [ { "Nome": "Joao Souza", "Resultado": "Removido" } ],
  "Inserir": [ { "Nome": "Maria Silva", "Resultado": "Inserido" } ]
}
```

`Resultado` pode ser `Inserido`, `Removido` ou `Não encontrado`. Se voce omitir `Inserir` ou
`Remover` no corpo, a chave correspondente volta `null`.

**Cuidados importantes:**

- A resposta e **sempre 200**, mesmo quando o nome nao e encontrado. Confira o `Resultado` de cada
  nome para saber o que aconteceu; nao basta olhar o status HTTP.
- O nome precisa ser igual ao cadastrado. Variacoes de maiusculas/minusculas ou de acentuacao podem
  aparecer como `Não encontrado` para o nome enviado.
- Nomes iguais (homonimos) nao sao distinguidos: confira o resultado e, se houver duvida, verifique
  os membros com `GET .../membros`.
- Inserir um usuario que ja e membro da lista pode duplicar o registro; consulte os membros antes.
- Enviar a requisicao sem corpo pode causar erro do servidor (HTTP 500).

---

## Exemplos Completos

### cURL

```bash
BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"

# Listar todos os ICs (sem paginacao)
curl -s "$BASE_URL/v1/ics" \
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

# Adicionar usuario a uma lista de papeis (pelo nome de exibicao)
curl -s -X POST "$BASE_URL/v1/ics/101/listaspapeis/1/membros" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "Inserir": ["Maria Silva"] }'
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

# Listar todos os ICs (uma unica chamada, sem paginacao)
resp = requests.get(f"{BASE_URL}/v1/ics", headers=headers)
resp.raise_for_status()
todos = resp.json()["records"]

print(f"Total de ICs: {len(todos)}")

# Buscar IC por ID e seus relacionamentos
ic_id = 101
resp = requests.get(f"{BASE_URL}/v1/ics/{ic_id}", headers=headers)
resp.raise_for_status()
ic = resp.json()
print(f"IC: {ic['Nome']} — Status: {ic['Status']['Nome']}")

# Listar componentes
resp = requests.get(f"{BASE_URL}/v1/ics/{ic_id}/componentes", headers=headers)
resp.raise_for_status()
for comp in resp.json()["records"]:
    print(f"  Componente: {comp['Descricao']} (x{comp['Quantidade']})")

# Adicionar usuario a lista de papeis (pelo nome de exibicao)
payload = {"Inserir": ["Maria Silva"]}
resp = requests.post(
    f"{BASE_URL}/v1/ics/{ic_id}/listaspapeis/1/membros",
    json=payload,
    headers=headers
)
resp.raise_for_status()
# A resposta e sempre 200: confira o resultado de cada nome
for item in resp.json()["Inserir"]:
    print(f"{item['Nome']}: {item['Resultado']}")
```

---

### PowerShell

```powershell
$BaseUrl = "https://sua-empresa.bdesk.com.br/askrest"
$Headers = @{
    Authorization  = "Bearer SEU_TOKEN_AQUI"
    "Content-Type" = "application/json"
}

# Listar todos os ICs (sem paginacao)
$Resp = Invoke-RestMethod -Uri "$BaseUrl/v1/ics" -Headers $Headers
Write-Host "Total de ICs: $($Resp.records.Count)"
foreach ($ic in $Resp.records) {
    Write-Host "  [$($ic.Id)] $($ic.Nome) — $($ic.Status.Nome)"
}

# Buscar IC por ID
$IcId = 101
$Ic = Invoke-RestMethod -Uri "$BaseUrl/v1/ics/$IcId" -Headers $Headers
Write-Host "IC: $($Ic.NomeCompleto) | Tipo: $($Ic.Tipo.Nome)"

# Mapa de relacionamentos
$Rels = Invoke-RestMethod -Uri "$BaseUrl/v1/ics/$IcId/relacionamentos?nivel=2" -Headers $Headers
Write-Host "Relacionamentos encontrados: $($Rels.records.Count)"

# Adicionar membro a lista de papeis (pelo nome de exibicao)
$Payload = @{ Inserir = @("Maria Silva") } | ConvertTo-Json
$Res = Invoke-RestMethod `
    -Uri "$BaseUrl/v1/ics/$IcId/listaspapeis/1/membros" `
    -Method Post `
    -Headers $Headers `
    -Body $Payload `
    -ContentType "application/json"
# A resposta e sempre 200: confira o resultado de cada nome
foreach ($item in $Res.Inserir) { Write-Host "$($item.Nome): $($item.Resultado)" }
```

---

## Proximos Passos

Com os ICs mapeados, voce pode:

- [Criar Requisicoes](criar-requisicoes.md) — associar um IC a uma requisicao ao abri-la
