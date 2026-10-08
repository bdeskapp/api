# Criar Requisicoes

## O que voce vai aprender

Como criar requisicoes na API BDesk de diferentes formas: abertura simples (retorna apenas o ID), abertura com dados adicionais de formulario, abertura com o ID dentro do envelope padrao da API e abertura via template para ferramentas de monitoramento.

---

## Pre-requisitos

Antes de comecar, voce precisa ter:

- **Token de autenticacao valido** — veja [Autenticacao](../autenticacao.md) para obter o seu token.
- **ID do formulario** que deseja usar — veja [Catalogo de Servicos](catalogo-servicos.md) para descobrir os IDs disponiveis.

---

## Passo a Passo

### Passo 1: Escolha o metodo de abertura adequado

A API oferece tres endpoints de abertura (os Metodos 1 e 2 abaixo usam o mesmo). Escolha de acordo com a sua necessidade:

| Endpoint | Quando usar |
|---|---|
| `POST /v1/requisicoes/abrir` | Integracao simples — a resposta e apenas o numero da requisicao criada |
| `POST /v1/requisicoes/abrirRequisicao` | Mesmo corpo de `abrir`, mas o numero vem dentro do envelope padrao da API (`records`) |
| `POST /v1/requisicoes/abrirFormatoZabbix/{variante}` | Integracao com Zabbix e outras ferramentas de monitoramento (a `{variante}` e opcional) |

Os dois primeiros endpoints recebem exatamente o mesmo corpo. O de monitoramento tem
guia proprio (veja o Metodo 4 abaixo).

---

### Passo 2: Monte o payload basico

Todo payload de abertura de requisicao tem o formulario e um dicionario `Conjuntos`. Os dados
principais (assunto e descricao) ficam dentro do conjunto `DadosBasicos`:

| Campo | Tipo | Descricao |
|---|---|---|
| `Formulario` | inteiro | ID do formulario (tipo de servico) |
| `Conjuntos.DadosBasicos.Assunto` | texto | Titulo curto da requisicao |
| `Conjuntos.DadosBasicos.Descricao` | texto | Descricao detalhada do problema ou solicitacao |

**Exemplo de payload minimo:**

```json
{
  "Formulario": 123,
  "Conjuntos": {
    "DadosBasicos": {
      "Assunto": "Titulo da requisicao",
      "Descricao": "Descricao detalhada do problema"
    }
  }
}
```

O `DadosBasicos` pode ter outros campos conforme o formulario (por exemplo, `Atividade`). Consulte
os campos de cada formulario em `GET /v1/cardapio/{id}` (veja o [Catalogo de Servicos](catalogo-servicos.md)).

---

### Passo 3: Envie a requisicao

Adicione o header de autenticacao `Authorization: Bearer SEU_TOKEN_AQUI` e faca o POST para o endpoint escolhido.

A resposta de `/v1/requisicoes/abrir` e o numero da requisicao criada, diretamente no corpo da resposta como um texto JSON, por exemplo:

```
"222886"
```

---

### Passo 4: (Opcional) Inclua dados adicionais do formulario

Alguns formularios possuem campos extras alem do assunto e descricao. Esses campos ficam organizados em grupos chamados **Conjuntos**, ao lado de `DadosBasicos`.

O campo `Conjuntos` e um **dicionario**: a chave e o nome do conjunto e o valor e outro objeto com os campos daquele conjunto (ou uma lista de objetos, quando o conjunto aceita varios itens).

**Exemplo com dados adicionais:**

```json
{
  "Formulario": 123,
  "Conjuntos": {
    "DadosBasicos": {
      "Assunto": "Titulo",
      "Descricao": "Descricao"
    },
    "NomeDoConjunto": {
      "Campo1": "valor1",
      "Campo2": "valor2"
    }
  }
}
```

> Para descobrir os nomes dos conjuntos e campos disponiveis para cada formulario, veja [Catalogo de Servicos](catalogo-servicos.md).

Um conjunto que nao seja um objeto nem uma lista faz a API responder com erro.

---

### Passo 5: (Opcional) Adicione anexos apos a abertura

Apos criar a requisicao, voce pode anexar arquivos. O envio e feito em duas chamadas:

1. `POST /v1/requisicoes/{id}/anexo` (formato `multipart/form-data`) envia o arquivo e devolve um identificador temporario. Nesta etapa o arquivo **ainda nao** esta anexado.
2. `POST /v1/requisicoes/{id}/anexos/submeter` (JSON) vincula o arquivo a requisicao.

> Veja [Gerenciar Anexos](anexos.md) para detalhes completos, exemplos e erros dos dois passos.

---

## Metodo 1: Abertura Simples

**Endpoint:** `POST /v1/requisicoes/abrir`

Use este metodo quando voce so precisa do numero da requisicao criada, sem informacoes adicionais. E o metodo mais direto para integracao com sistemas externos.

**Payload:**

```json
{
  "Formulario": 123,
  "Conjuntos": {
    "DadosBasicos": {
      "Assunto": "Falha no servidor de email",
      "Descricao": "O servidor SMTP parou de responder desde as 10h."
    }
  }
}
```

**Resposta de sucesso (HTTP 200):**

```
"222886"
```

A resposta e o numero da requisicao aberta, retornado como um texto JSON simples no corpo da resposta (sem envelope).

**Exemplo rapido com cURL:**

```bash
curl -X POST "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abrir" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -H "Content-Type: application/json" \
  -d '{
    "Formulario": 123,
    "Conjuntos": {
      "DadosBasicos": {
        "Assunto": "Falha no servidor de email",
        "Descricao": "O servidor SMTP parou de responder desde as 10h."
      }
    }
  }'
```

---

## Metodo 2: Abertura com Dados Adicionais (Formulario)

Formularios podem ter campos extras alem do assunto e descricao. Use o campo `Conjuntos` para preenche-los.

**O que sao Conjuntos?**

Conjuntos sao agrupamentos de campos adicionais definidos no formulario. Por exemplo, um formulario de "Acesso a Sistema" pode ter um conjunto chamado `"DadosDoAcesso"` com campos como `"Sistema"` e `"NivelDeAcesso"`.

**Importante:** `Conjuntos` e um **dicionario** (objeto JSON), nao uma lista. A chave de cada entrada e o nome do conjunto (Chave), e o valor e outro objeto com os campos daquele conjunto (ou uma lista de objetos, para conjuntos com varios itens).

**Payload:**

```json
{
  "Formulario": 123,
  "Conjuntos": {
    "DadosBasicos": {
      "Assunto": "Solicitar acesso ao sistema de RH",
      "Descricao": "Preciso de acesso para consultar folha de pagamento."
    },
    "DadosDoAcesso": {
      "Sistema": "SistemaRH",
      "NivelDeAcesso": "Leitura"
    },
    "Justificativa": {
      "Motivo": "Novo colaborador na equipe de financas",
      "Aprovador": "Joao Silva"
    }
  }
}
```

> Para descobrir os nomes dos conjuntos e campos de cada formulario, consulte [Catalogo de Servicos](catalogo-servicos.md).

Participantes (como solicitante ou responsavel) entram no conjunto `Papeis`, com o `Id` obtido em
[Participantes](participantes.md) para o papel correspondente.

---

## Metodo 3: Abertura com o numero no envelope

**Endpoint:** `POST /v1/requisicoes/abrirRequisicao`

Use este metodo quando preferir receber o numero da requisicao dentro do envelope padrao da API, junto com `_metadata`. O corpo enviado e o mesmo dos Metodos 1 e 2.

**Payload:** mesmo formato do Metodo 1 e Metodo 2.

**Resposta de sucesso (HTTP 200):**

```json
{
  "_metadata": {
    "Release": "9.8.0",
    "LogAmigavel": [],
    "MensagensErro": []
  },
  "records": [
    { "IdRequisicaoAberta": "222886" }
  ]
}
```

O numero da requisicao esta em `records[0].IdRequisicaoAberta` (como texto). Se a abertura nao
devolver o numero, `records` vem vazio. A resposta **nao** traz assunto, status nem data de
abertura: para obter os dados completos da requisicao criada, consulte
`GET /v1/requisicoes/{id}` (veja [Consultar Requisicoes](consultar-requisicoes.md)).

**Quando usar cada endpoint:**

- Use `/v1/requisicoes/abrir` quando sua integracao so precisa do numero, direto no corpo.
- Use `/v1/requisicoes/abrirRequisicao` quando seu cliente HTTP espera sempre o envelope com `_metadata` e `records`.

---

## Metodo 4: Abertura via Template (Zabbix/Monitoramento)

**Endpoint:** `POST /v1/requisicoes/abrirFormatoZabbix/{variante}`

Este endpoint aceita o formato de alerta do Zabbix e de outras ferramentas de monitoramento de infraestrutura, permitindo abrir requisicoes automaticamente a partir de alertas gerados por essas ferramentas. A `{variante}` identifica a configuracao do tipo de incidente e e opcional.

> Para detalhes sobre integracao com Zabbix e outras ferramentas de monitoramento, veja [Integracoes de Monitoramento](integracoes-monitoramento.md).

---

## Exemplos Completos

### cURL

```bash
# Abertura simples
curl -X POST "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abrir" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -H "Content-Type: application/json" \
  -d '{
    "Formulario": 123,
    "Conjuntos": {
      "DadosBasicos": {
        "Assunto": "Falha no servidor de email",
        "Descricao": "O servidor SMTP parou de responder desde as 10h."
      }
    }
  }'

# Abertura com dados adicionais
curl -X POST "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abrir" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -H "Content-Type: application/json" \
  -d '{
    "Formulario": 123,
    "Conjuntos": {
      "DadosBasicos": {
        "Assunto": "Solicitar acesso ao sistema de RH",
        "Descricao": "Preciso de acesso para consultar folha de pagamento."
      },
      "DadosDoAcesso": {
        "Sistema": "SistemaRH",
        "NivelDeAcesso": "Leitura"
      }
    }
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

# Abertura simples
payload = {
    "Formulario": 123,
    "Conjuntos": {
        "DadosBasicos": {
            "Assunto": "Falha no servidor de email",
            "Descricao": "O servidor SMTP parou de responder desde as 10h."
        }
    }
}

resp = requests.post(f"{BASE_URL}/v1/requisicoes/abrir", json=payload, headers=headers)
if resp.status_code != 200:
    # 406: a mensagem vem em texto puro
    raise SystemExit(f"Falha na abertura ({resp.status_code}): {resp.text}")
print(f"Requisicao criada: {resp.json()}")  # texto JSON, ex.: "222886"

# Abertura com dados adicionais
payload_completo = {
    "Formulario": 123,
    "Conjuntos": {
        "DadosBasicos": {
            "Assunto": "Solicitar acesso ao sistema de RH",
            "Descricao": "Preciso de acesso para consultar folha de pagamento."
        },
        "DadosDoAcesso": {
            "Sistema": "SistemaRH",
            "NivelDeAcesso": "Leitura"
        }
    }
}

resp = requests.post(f"{BASE_URL}/v1/requisicoes/abrir", json=payload_completo, headers=headers)
if resp.status_code != 200:
    raise SystemExit(f"Falha na abertura ({resp.status_code}): {resp.text}")
print(f"Requisicao criada: {resp.json()}")

# Abertura com o numero no envelope
resp = requests.post(f"{BASE_URL}/v1/requisicoes/abrirRequisicao", json=payload, headers=headers)
if resp.status_code != 200:
    raise SystemExit(f"Falha na abertura ({resp.status_code}): {resp.text}")
dados = resp.json()
id_requisicao = dados["records"][0]["IdRequisicaoAberta"]
print(f"ID da requisicao: {id_requisicao}")
```

---

### PowerShell

```powershell
$BaseUrl = "https://sua-empresa.bdesk.com.br/askrest"
$headers = @{
    Authorization  = "Bearer SEU_TOKEN_AQUI"
    "Content-Type" = "application/json"
}

# Abertura simples
$payload = @{
    Formulario = 123
    Conjuntos  = @{
        DadosBasicos = @{
            Assunto   = "Falha no servidor de email"
            Descricao = "O servidor SMTP parou de responder desde as 10h."
        }
    }
} | ConvertTo-Json -Depth 5

$resp = Invoke-RestMethod `
    -Uri "$BaseUrl/v1/requisicoes/abrir" `
    -Method Post `
    -Headers $headers `
    -Body $payload `
    -ContentType "application/json"

Write-Host "Requisicao criada: $resp"

# Abertura com dados adicionais
$payloadCompleto = @{
    Formulario = 123
    Conjuntos  = @{
        DadosBasicos = @{
            Assunto   = "Solicitar acesso ao sistema de RH"
            Descricao = "Preciso de acesso para consultar folha de pagamento."
        }
        DadosDoAcesso = @{
            Sistema       = "SistemaRH"
            NivelDeAcesso = "Leitura"
        }
    }
} | ConvertTo-Json -Depth 5

$resp = Invoke-RestMethod `
    -Uri "$BaseUrl/v1/requisicoes/abrir" `
    -Method Post `
    -Headers $headers `
    -Body $payloadCompleto `
    -ContentType "application/json"

Write-Host "Requisicao criada: $resp"

# Abertura com o numero no envelope
$resp = Invoke-RestMethod `
    -Uri "$BaseUrl/v1/requisicoes/abrirRequisicao" `
    -Method Post `
    -Headers $headers `
    -Body $payload `
    -ContentType "application/json"

Write-Host "ID da requisicao: $($resp.records[0].IdRequisicaoAberta)"
```

---

## Erros Comuns

Os erros de abertura voltam como HTTP 406 com a mensagem em **texto puro** (nao em JSON).

| Sintoma | Causa provavel | Solucao |
|---|---|---|
| HTTP 406 — formulario inexistente | O ID informado no campo `Formulario` nao existe | Verifique o ID correto via [Catalogo de Servicos](catalogo-servicos.md) |
| HTTP 406 — "O campo 'Conjunto.Campo' recebeu a seguinte mensagem de validação: ..." | Um campo do formulario nao passou na validacao (por exemplo, obrigatorio nao preenchido) | Corrija o campo indicado na mensagem; consulte os campos do formulario no [Catalogo de Servicos](catalogo-servicos.md) |
| HTTP 406 — "Os dados para abertura não foram completamente informados..." | O corpo esta vazio ou e `null` | Envie o JSON completo, com `Formulario` e `Conjuntos` |
| HTTP 406 — "Erro no JSON ..." | O corpo nao e um JSON valido | Valide o JSON antes de enviar; ferramentas como o Postman ajudam a identificar erros de formatacao |
| HTTP 406 — "Conjunto '...' não é objeto JSON ou array" | Um conjunto foi enviado como texto ou numero | Envie cada conjunto como objeto (ou lista de objetos) |
| HTTP 401 — Unauthorized | Token ausente, invalido ou usuario desativado | Faca login novamente e obtenha um novo token — veja [Autenticacao](../autenticacao.md) |

> **Atencao:** em alguns casos o erro 406 e devolvido **depois** de a requisicao ter sido criada
> (uma validacao posterior acumulou mensagens). Antes de repetir a chamada depois de um 406, confira
> em [Consultar Requisicoes](consultar-requisicoes.md) se a requisicao ja foi aberta, para nao criar
> duplicadas.

---

## Proximos Passos

Com a requisicao criada, voce pode:

- [Consultar Requisicoes](consultar-requisicoes.md) — buscar pelo ID ou listar requisicoes abertas
- [Acoes de Workflow](acoes-workflow.md) — direcionar, encerrar, recategorizar e outras acoes
- [Gerenciar Anexos](anexos.md) — adicionar arquivos a uma requisicao existente
- [Catalogo de Servicos](catalogo-servicos.md) — explorar formularios e seus campos disponiveis
