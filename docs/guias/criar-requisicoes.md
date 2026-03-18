# Criar Requisicoes

## O que voce vai aprender

Como criar requisicoes na API BDesk de diferentes formas: abertura simples (retorna apenas o ID), abertura com dados adicionais de formulario, abertura completa (retorna o objeto inteiro) e abertura via template para ferramentas de monitoramento.

---

## Pre-requisitos

Antes de comecar, voce precisa ter:

- **Token de autenticacao valido** — veja [Autenticacao](../autenticacao.md) para obter o seu token.
- **ID do formulario** que deseja usar — veja [Catalogo de Servicos](catalogo-servicos.md) para descobrir os IDs disponiveis.

---

## Passo a Passo

### Passo 1: Escolha o metodo de abertura adequado

A API oferece quatro formas de abrir uma requisicao. Escolha de acordo com a sua necessidade:

| Endpoint | Quando usar |
|---|---|
| `POST /v1/requisicoes/abrir` | Integracao simples — voce so precisa do ID da requisicao criada |
| `POST /v1/requisicoes/abrirRequisicao` | Quando voce precisa do objeto completo da requisicao apos a criacao |
| `POST /v1/requisicoes/abrirFormatoZabbix` | Integracao com Zabbix e outras ferramentas de monitoramento |

---

### Passo 2: Monte o payload basico

Todo payload de abertura de requisicao requer ao menos tres campos:

| Campo | Tipo | Descricao |
|---|---|---|
| `Formulario` | inteiro | ID do formulario (tipo de servico) |
| `Assunto` | texto | Titulo curto da requisicao |
| `Descricao` | texto | Descricao detalhada do problema ou solicitacao |

**Exemplo de payload minimo:**

```json
{
  "Formulario": 123,
  "Assunto": "Titulo da requisicao",
  "Descricao": "Descricao detalhada do problema"
}
```

---

### Passo 3: Envie a requisicao

Adicione o header de autenticacao `Authorization: Bearer SEU_TOKEN_AQUI` e faça o POST para o endpoint escolhido.

A resposta de `/v1/requisicoes/abrir` e o ID da requisicao criada diretamente no corpo da resposta como texto, por exemplo:

```
"RQ-20240317-001"
```

---

### Passo 4: (Opcional) Inclua dados adicionais do formulario

Alguns formularios possuem campos extras alem do assunto e descricao. Esses campos ficam organizados em grupos chamados **Conjuntos**.

O campo `Conjuntos` e um **dicionario**: a chave e o nome do conjunto e o valor e outro objeto com os campos daquele conjunto.

**Exemplo com dados adicionais:**

```json
{
  "Formulario": 123,
  "Assunto": "Titulo",
  "Descricao": "Descricao",
  "Conjuntos": {
    "NomeDoConjunto": {
      "Campo1": "valor1",
      "Campo2": "valor2"
    }
  }
}
```

> Para descobrir os nomes dos conjuntos e campos disponiveis para cada formulario, veja [Catalogo de Servicos](catalogo-servicos.md).

---

### Passo 5: (Opcional) Adicione anexos apos a abertura

Apos criar a requisicao, voce pode anexar arquivos usando o endpoint:

```
POST /v1/requisicoes/{id}/anexo
```

A requisicao deve ser enviada no formato `multipart/form-data`.

> Veja [Gerenciar Anexos](anexos.md) para detalhes completos sobre upload de arquivos.

---

## Metodo 1: Abertura Simples

**Endpoint:** `POST /v1/requisicoes/abrir`

Use este metodo quando voce so precisa do ID da requisicao criada, sem informacoes adicionais. E o metodo mais direto para integracao com sistemas externos.

**Payload:**

```json
{
  "Formulario": 123,
  "Assunto": "Falha no servidor de email",
  "Descricao": "O servidor SMTP parou de responder desde as 10h."
}
```

**Resposta de sucesso (HTTP 200):**

```
"RQ-20240317-001"
```

A resposta e o identificador da requisicao aberta, retornado como texto simples no corpo da resposta.

**Exemplo rapido com cURL:**

```bash
curl -X POST "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abrir" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -H "Content-Type: application/json" \
  -d '{
    "Formulario": 123,
    "Assunto": "Falha no servidor de email",
    "Descricao": "O servidor SMTP parou de responder desde as 10h."
  }'
```

---

## Metodo 2: Abertura com Dados Adicionais (Formulario)

Formularios podem ter campos extras alem do assunto e descricao. Use o campo `Conjuntos` para preenche-los.

**O que sao Conjuntos?**

Conjuntos sao agrupamentos de campos adicionais definidos no formulario. Por exemplo, um formulario de "Acesso a Sistema" pode ter um conjunto chamado `"DadosDoAcesso"` com campos como `"Sistema"` e `"NivelDeAcesso"`.

**Importante:** `Conjuntos` e um **dicionario** (objeto JSON), nao uma lista. A chave de cada entrada e o nome do conjunto (Chave), e o valor e outro objeto com os campos daquele conjunto.

**Payload:**

```json
{
  "Formulario": 123,
  "Assunto": "Solicitar acesso ao sistema de RH",
  "Descricao": "Preciso de acesso para consultar folha de pagamento.",
  "Conjuntos": {
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

---

## Metodo 3: Abertura Completa (retorna objeto)

**Endpoint:** `POST /v1/requisicoes/abrirRequisicao`

Use este metodo quando alem de abrir a requisicao, voce precisa dos dados completos do objeto criado — como o ID numerico interno, datas, status e outros campos retornados pelo sistema.

**Payload:** mesmo formato do Metodo 1 e Metodo 2.

**Resposta de sucesso (HTTP 200):**

```json
{
  "_metadata": { ... },
  "resultado": {
    "IdRequisicaoAberta": "RQ-20240317-001",
    "Numero": 4872,
    "Assunto": "Falha no servidor de email",
    "Status": "Aberta",
    "DataAbertura": "2024-03-17T10:00:00"
  }
}
```

**Quando usar cada endpoint:**

- Use `/v1/requisicoes/abrir` quando sua integracao so precisa saber qual foi o ID criado.
- Use `/v1/requisicoes/abrirRequisicao` quando voce precisa processar dados do objeto retornado (ex: gravar no seu sistema o numero da requisicao e a data de abertura).

---

## Metodo 4: Abertura via Template (Zabbix/Monitoramento)

**Endpoint:** `POST /v1/requisicoes/abrirFormatoZabbix`

Este endpoint aceita o formato de alerta do Zabbix e de outras ferramentas de monitoramento de infraestrutura, permitindo abrir requisicoes automaticamente a partir de alertas gerados por essas ferramentas.

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
    "Assunto": "Falha no servidor de email",
    "Descricao": "O servidor SMTP parou de responder desde as 10h."
  }'

# Abertura com dados adicionais
curl -X POST "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abrir" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -H "Content-Type: application/json" \
  -d '{
    "Formulario": 123,
    "Assunto": "Solicitar acesso ao sistema de RH",
    "Descricao": "Preciso de acesso para consultar folha de pagamento.",
    "Conjuntos": {
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
    "Assunto": "Falha no servidor de email",
    "Descricao": "O servidor SMTP parou de responder desde as 10h."
}

resp = requests.post(f"{BASE_URL}/v1/requisicoes/abrir", json=payload, headers=headers)
resp.raise_for_status()
print(f"Requisicao criada: {resp.text}")

# Abertura com dados adicionais
payload_completo = {
    "Formulario": 123,
    "Assunto": "Solicitar acesso ao sistema de RH",
    "Descricao": "Preciso de acesso para consultar folha de pagamento.",
    "Conjuntos": {
        "DadosDoAcesso": {
            "Sistema": "SistemaRH",
            "NivelDeAcesso": "Leitura"
        }
    }
}

resp = requests.post(f"{BASE_URL}/v1/requisicoes/abrir", json=payload_completo, headers=headers)
resp.raise_for_status()
print(f"Requisicao criada: {resp.text}")

# Abertura completa (retorna objeto)
resp = requests.post(f"{BASE_URL}/v1/requisicoes/abrirRequisicao", json=payload, headers=headers)
resp.raise_for_status()
dados = resp.json()
id_requisicao = dados["resultado"]["IdRequisicaoAberta"]
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
    Assunto    = "Falha no servidor de email"
    Descricao  = "O servidor SMTP parou de responder desde as 10h."
} | ConvertTo-Json

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
    Assunto    = "Solicitar acesso ao sistema de RH"
    Descricao  = "Preciso de acesso para consultar folha de pagamento."
    Conjuntos  = @{
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

# Abertura completa (retorna objeto)
$resp = Invoke-RestMethod `
    -Uri "$BaseUrl/v1/requisicoes/abrirRequisicao" `
    -Method Post `
    -Headers $headers `
    -Body $payload `
    -ContentType "application/json"

Write-Host "ID da requisicao: $($resp.resultado.IdRequisicaoAberta)"
```

---

## Erros Comuns

| Sintoma | Causa provavel | Solucao |
|---|---|---|
| HTTP 406 — "Formulario nao encontrado" | O ID informado no campo `Formulario` nao existe ou esta inativo | Verifique o ID correto via [Catalogo de Servicos](catalogo-servicos.md) |
| HTTP 406 — "Campo obrigatorio nao preenchido" | Um campo marcado como obrigatorio no formulario nao foi enviado no payload | Consulte os campos obrigatorios do formulario no [Catalogo de Servicos](catalogo-servicos.md) |
| HTTP 406 — "Atividade nao informada" | O formulario exige que uma atividade seja selecionada | Adicione o campo `Atividade` com o valor correspondente ao payload |
| HTTP 401 — Unauthorized | Token expirado ou invalido | Faca login novamente e obtenha um novo token — veja [Autenticacao](../autenticacao.md) |
| HTTP 400 — Bad Request | Payload mal formado (JSON invalido) | Valide o JSON antes de enviar; ferramentas como `jq` ou o Postman ajudam a identificar erros de formatacao |

---

## Proximos Passos

Com a requisicao criada, voce pode:

- [Consultar Requisicoes](consultar-requisicoes.md) — buscar pelo ID ou listar requisicoes abertas
- [Acoes de Workflow](acoes-workflow.md) — direcionar, encerrar, recategorizar e outras acoes
- [Gerenciar Anexos](anexos.md) — adicionar arquivos a uma requisicao existente
- [Catalogo de Servicos](catalogo-servicos.md) — explorar formularios e seus campos disponiveis
