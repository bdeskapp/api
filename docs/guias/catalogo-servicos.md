# Catalogo de Servicos

## O que voce vai aprender

Como navegar o catalogo de servicos BDesk via API: listar areas e formularios disponiveis,
obter os campos de um formulario especifico, carregar listas suspensas (dropdowns) e descobrir
as atividades associadas a um servico — tudo que voce precisa antes de abrir uma requisicao.

---

## Pre-requisitos

- **Token de autenticacao valido** — veja o guia de [Autenticacao](../autenticacao.md).

Nenhum outro dado e necessario para comecar a explorar o catalogo.

---

## Passo a Passo

### Passo 1: Listar as areas do catalogo

Use `GET /v1/cardapio` para obter todas as areas disponiveis com seus formularios aninhados.
Este e o ponto de entrada principal do catalogo.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/cardapio" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta:**

```json
{
  "_metadata": {
    "Release": "9.8.0",
    "MensagensErro": null,
    "LogAmigavel": null
  },
  "records": [
    {
      "Id": 1,
      "Nome": "Suporte de TI",
      "Formularios": [
        {
          "Id": 101,
          "Nome": "Solicitacao de Suporte de PC",
          "Versao": "2.1",
          "PrazoMinimoDeAtendimento": 4,
          "Origens": [
            { "Id": 1, "Nome": "Web" },
            { "Id": 2, "Nome": "Email" }
          ],
          "Url": "https://sua-empresa.bdesk.com.br/askrest/v1/cardapio/formularios/101"
        }
      ]
    }
  ]
}
```

> **Nota:** `MensagensErro` e `LogAmigavel` podem ser `null` (nao apenas array vazio) neste
> endpoint. O campo `Url` de cada formulario e uma URL absoluta pronta para uso.

Use o campo `Formularios[].Id` (o `FormularioId`) para:
- Chamar `GET /v1/cardapio/formularios/{formularioId}` para obter os campos do formulario.
- Incluir no payload ao abrir uma requisicao.

---

### Passo 2: Listar categorias (para renderizacao dinamica de formularios)

Use `GET /v1/cardapio/categorias` para obter a arvore de categorias com seus itens. Este
endpoint e indicado quando voce precisa do campo `Apresentacao` dos campos — informacao
necessaria para renderizar formularios dinamicamente (ex: `TextBox`, `ComboBox`, `DatePicker`).

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/cardapio/categorias" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta:**

```json
[
  {
    "Id": 18,
    "Nome": "Acompanhamentos",
    "Imagem": "iconcardapio-apple",
    "Itens": [
      {
        "Id": 30401,
        "Nome": "Acompanhamento de Requisicoes",
        "URL": "https://sua-empresa.bdesk.com.br/askrest/v1/cardapio/30401"
      }
    ]
  }
]
```

> **Terminologia de IDs:**
> - **`ItemCardapioId`** — campo `Itens[].Id` retornado por este endpoint. Use para chamar
>   `GET /v1/cardapio/{id}`.
> - **`FormularioId`** — campo `Formularios[].Id` retornado por `GET /v1/cardapio`. Use para
>   chamar `GET /v1/cardapio/formularios/{formularioId}` e no payload de abertura.

---

### Passo 3: Obter detalhes de um item do catalogo

Use `GET /v1/cardapio/{id}` com o `ItemCardapioId` (obtido no Passo 2) para ver os campos
detalhados do formulario, incluindo o tipo de apresentacao de cada campo, quais extensoes de
arquivo nao sao permitidas e o texto de instrucao para upload de anexos.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/cardapio/30401" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta:**

```json
{
  "LogAmigavel": ["..."],
  "MensagensErro": [],
  "ExtensoesNaoPermitidas": "asp,aspx,js,exe,vbs,bat",
  "DescricaoParaAnexarDocumentos": "Anexe documentos relevantes",
  "Descricao_Conjunto_Anexos": "Anexos",
  "MensagemFinal": "Sua solicitacao foi registrada com sucesso.",
  "Conjuntos": [
    {
      "Chave": "DadosBasicos",
      "Nome": "Detalhes do Pedido",
      "Multiplo": false,
      "PermitirUpload": false,
      "Campos": [
        {
          "Nome": "Assunto",
          "Chave": "Assunto",
          "Apresentacao": "TextBox",
          "TipoDeDado": "String",
          "Obrigatoriedade": true,
          "Tamanho": "100"
        },
        {
          "Nome": "Categoria",
          "Chave": "Categoria",
          "Apresentacao": "ComboBox",
          "TipoDeDado": "Integer",
          "Obrigatoriedade": true,
          "Tamanho": ""
        }
      ],
      "Condicoes": null,
      "Posicao": 0,
      "Visivel": false,
      "Dados_Adicionais_Excel": false
    }
  ],
  "CondicoesDadosAdicionais": []
}
```

> **Atencao:** Este endpoint retorna HTTP 200 mesmo quando ocorre um erro. Verifique sempre o
> campo `MensagensErro` no corpo da resposta. Se estiver preenchido, o item nao foi encontrado
> ou o usuario nao tem acesso.

**Campos uteis da resposta:**

| Campo | Descricao |
|-------|-----------|
| `ExtensoesNaoPermitidas` | Extensoes de arquivo bloqueadas para upload neste formulario (separadas por virgula) |
| `Conjuntos[].Campos[].Apresentacao` | Tipo de controle visual: `TextBox`, `ComboBox`, `DatePicker`, etc. |
| `Conjuntos[].Campos[].Obrigatoriedade` | Se o campo e obrigatorio ao abrir a requisicao |
| `CondicoesDadosAdicionais` | Regras de visibilidade condicional entre campos |
| `MensagemFinal` | Texto exibido ao usuario apos a abertura da requisicao |

---

### Passo 4: Obter o formulario e seus campos

Use `GET /v1/cardapio/formularios/{formularioId}` com o `FormularioId` (obtido no Passo 1)
para ver a estrutura completa do formulario com URLs de atividades e abertura.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/cardapio/formularios/101" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta:**

```json
{
  "Id": 101,
  "Nome": "Solicitacao de Suporte de PC",
  "Versao": "2.1",
  "PrazoMinimoDeAtendimento": 4,
  "UrlAtividades": "/v1/cardapio/formularios/101/atividades",
  "AbrirRequisicao": "/v1/requisicoes/abrir",
  "DesdobrarRequisicao": "/v1/requisicoes/desdobrar",
  "Conjuntos": [
    {
      "Id": null,
      "Nome": "Detalhes do Pedido",
      "Chave": "DadosBasicos",
      "Multiplo": false,
      "PermitirUpload": false,
      "Campos": [
        {
          "Nome": "Assunto",
          "Chave": "Assunto",
          "TipoDeDado": "String",
          "Obrigatoriedade": true,
          "Tamanho": "100"
        }
      ]
    }
  ]
}
```

> **Nota:** Este endpoint retorna um objeto direto — sem envelope `_metadata`/`records`.

> **Atencao:** O campo `Apresentacao` e removido automaticamente neste endpoint. Para obter
> o tipo de controle visual de cada campo (necessario para renderizacao dinamica), use
> `GET /v1/cardapio/{id}` (Passo 3) em vez deste endpoint.

---

### Passo 5: Listar as atividades de um formulario

Use `GET /v1/cardapio/formularios/{formularioId}/atividades` para obter as atividades
disponiveis — necessario quando o formulario exige selecionar uma atividade ao abrir
a requisicao.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/cardapio/formularios/101/atividades" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta:**

```json
{
  "_metadata": {
    "Release": "9.8.0",
    "MensagensErro": [],
    "LogAmigavel": []
  },
  "records": [
    { "Id": 201, "Nome": "Instalar Software" },
    { "Id": 202, "Nome": "Resolver Problema de Hardware" }
  ]
}
```

Use o `Id` da atividade desejada no campo `AtividadeId` ao abrir a requisicao.

---

### Passo 6: Carregar opcoes de campos de selecao (dropdowns)

Campos com `Apresentacao = "ComboBox"` ou similar precisam de uma lista de opcoes
carregada via API. Use o `idDad` (ID do dado adicional) do campo para buscar os valores.

#### Dropdown simples

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/itensselecao/42" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta:**

```json
{
  "_metadata": {
    "Release": "9.8.0",
    "MensagensErro": [],
    "LogAmigavel": []
  },
  "records": [
    { "Id": "37", "Texto": "Sao Paulo",  "Legenda": "Escritorio SP", "IdDominioPai": "" },
    { "Id": "36", "Texto": "Rio de Janeiro", "Legenda": "", "IdDominioPai": "" },
    { "Id": "35", "Texto": "Belo Horizonte", "Legenda": "", "IdDominioPai": "" }
  ]
}
```

> **Atencao:** O campo de exibicao e `Texto` (nao `Nome`). O campo `Id` e sempre do tipo
> **string**, mesmo que o valor seja numerico. Nunca trate esses IDs como inteiros.

#### Dropdown em cascata

Quando a selecao de um campo depende do valor escolhido em outro campo (cascata), use o
segundo endpoint passando o ID da opcao pai:

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/itensselecao/55/37" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

Neste exemplo, `55` e o `idDad` do campo filho e `37` e o `Id` (string) da opcao
selecionada no campo pai. A resposta segue o mesmo formato do dropdown simples.

---

## Exemplos Completos

### Fluxo completo: descobrir um formulario e suas opcoes (Python)

```python
import requests

BASE = "https://sua-empresa.bdesk.com.br/askrest"
TOKEN = "SEU_TOKEN_AQUI"
HEADERS = {"Authorization": f"Bearer {TOKEN}"}

# 1. Listar areas
areas = requests.get(f"{BASE}/v1/cardapio", headers=HEADERS).json()
primeira_area = areas["records"][0]
print(f"Area: {primeira_area['Nome']}")

# 2. Pegar o primeiro formulario da area
formulario = primeira_area["Formularios"][0]
formulario_id = formulario["Id"]
print(f"Formulario: {formulario['Nome']} (Id={formulario_id})")

# 3. Obter detalhes via /cardapio/{id} — inclui Apresentacao
# (precisamos do ItemCardapioId — buscamos via /categorias)
cats = requests.get(f"{BASE}/v1/cardapio/categorias", headers=HEADERS).json()
item_id = cats[0]["Itens"][0]["Id"]
detalhes = requests.get(f"{BASE}/v1/cardapio/{item_id}", headers=HEADERS).json()
if detalhes.get("MensagensErro"):
    print("Erro:", detalhes["MensagensErro"])
else:
    for conjunto in detalhes.get("Conjuntos", []):
        for campo in conjunto.get("Campos", []):
            print(f"  Campo: {campo['Nome']} | Tipo: {campo['Apresentacao']}")

# 4. Carregar opcoes de um campo ComboBox (idDad = 42)
opcoes = requests.get(f"{BASE}/v1/itensselecao/42", headers=HEADERS).json()
for opcao in opcoes["records"]:
    print(f"  Opcao: {opcao['Texto']} (Id={opcao['Id']})")
```

---

### Fluxo completo: descobrir formularios (PowerShell)

```powershell
$Base    = "https://sua-empresa.bdesk.com.br/askrest"
$Token   = "SEU_TOKEN_AQUI"
$Headers = @{ Authorization = "Bearer $Token" }

# 1. Listar areas
$areas = Invoke-RestMethod -Uri "$Base/v1/cardapio" -Headers $Headers
$primeiraArea = $areas.records[0]
Write-Host "Area: $($primeiraArea.Nome)"

# 2. Primeiro formulario
$formulario = $primeiraArea.Formularios[0]
Write-Host "Formulario: $($formulario.Nome) | Id: $($formulario.Id)"

# 3. Atividades do formulario
$atividades = Invoke-RestMethod `
  -Uri "$Base/v1/cardapio/formularios/$($formulario.Id)/atividades" `
  -Headers $Headers
$atividades.records | ForEach-Object { Write-Host "  Atividade: $($_.Nome) (Id=$($_.Id))" }

# 4. Opcoes de dropdown (idDad = 42)
$opcoes = Invoke-RestMethod -Uri "$Base/v1/itensselecao/42" -Headers $Headers
$opcoes.records | ForEach-Object { Write-Host "  $($_.Texto) [Id=$($_.Id)]" }
```

---

## Erros Comuns

| Codigo HTTP | Situacao | Causa provavel | Como resolver |
|-------------|----------|----------------|---------------|
| 200 com `MensagensErro` preenchido | `GET /v1/cardapio/{id}` retorna erro em HTTP 200 | ID invalido ou usuario sem acesso ao item | Verifique o `ItemCardapioId` e as permissoes do usuario |
| 401 | Token invalido ou ausente | Header `Authorization` ausente ou token expirado | Renove o token via `POST /v1/login/entrar` |
| 406 | Erro de negocio | Validacao falhou no servidor | Leia `_metadata.MensagensErro` para detalhes |
| Opcoes de dropdown vazias | `records` retorna array vazio | `idDad` incorreto ou campo nao tem dominio cadastrado | Confirme o `idDad` no campo `Campos[].idDad` do formulario |

---

## Proximos Passos

- [Criar Requisicoes](criar-requisicoes.md) — use os IDs descobertos aqui para abrir uma requisicao
