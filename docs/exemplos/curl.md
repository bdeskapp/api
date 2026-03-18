# Exemplos cURL — API BDesk

Scripts completos e prontos para execução em shell (bash/zsh). Copie, ajuste as variáveis de configuração e execute.

> **Pré-requisitos:** `curl` e `python3` instalados. Não é necessário `jq`.

---

## Configuração

Defina as variáveis abaixo antes de executar qualquer exemplo. Elas são referenciadas em todos os scripts desta página.

```bash
# Configuracao
BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
LOGIN="seu-usuario"
SENHA="sua-senha"
```

---

## 1. Login e Obter Token

O endpoint de login retorna o token dentro do campo `Dados`, que é uma **string JSON escapada** — não um objeto direto. É necessário fazer dois níveis de parse para extrair o `access_token`.

```bash
#!/usr/bin/env bash
# login.sh — Autentica e exibe o token obtido

BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
LOGIN="seu-usuario"
SENHA="sua-senha"

RESPOSTA=$(curl -s -X POST "$BASE_URL/v1/login/entrar" \
  -H "Content-Type: application/json" \
  -d "{\"Login\": \"$LOGIN\", \"Senha\": \"$SENHA\"}")

# Dados e uma string JSON escapada — requer dois levels de parse
TOKEN=$(echo "$RESPOSTA" | python3 -c "
import sys, json
resp = json.load(sys.stdin)
if resp.get('MensagensErro'):
    print('ERRO:', resp['MensagensErro'], file=sys.stderr)
    sys.exit(1)
dados = json.loads(resp['Dados'])
print(dados['access_token'])
")

if [ $? -ne 0 ]; then
  echo "Falha ao autenticar. Verifique login/senha."
  exit 1
fi

echo "Token obtido: ${TOKEN:0:20}..."
echo "Use: -H \"Authorization: Bearer $TOKEN\""
```

**Estrutura da resposta do login:**

```json
{
  "Dados": "{\"token_type\":\"Bearer\",\"access_token\":\"eyJhbGci...\",\"expires_in\":3600}",
  "LogAmigavel": [],
  "MensagensErro": [],
  "Versao": "9.8.0"
}
```

> **Atenção:** `Dados` e uma string (não um objeto). Execute `JSON.parse(resp.Dados)` para extrair o token. Erros de autenticacao retornam HTTP 406 com `MensagensErro` preenchido.

---

## 2. Listar Requisicoes Abertas

Lista as requisicoes abertas do usuario autenticado com suporte a paginacao.

```bash
#!/usr/bin/env bash
# listar-abertas.sh — Lista requisicoes abertas com paginacao

BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"

PAGE_SIZE=20
PAGE_NUMBER=1

curl -s -X GET \
  "$BASE_URL/v1/requisicoes/abertas?pageSize=$PAGE_SIZE&pageNumber=$PAGE_NUMBER" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "
import sys, json
resp = json.load(sys.stdin)
meta = resp.get('_metadata', {})
paginacao = meta.get('Pagination', {})

print(f\"Total: {paginacao.get('TotalRecords', '?')} requisicoes\")
print(f\"Pagina {paginacao.get('CurrentPage', '?')} de {paginacao.get('TotalPages', '?')}\")
print()
for req in resp.get('records', []):
    print(f\"#{req['RequisicaoId']} - {req['Assunto']}\")
    print(f\"  Status: {req['Status']} | Responsavel: {req.get('Responsavel', '-')}\")
    print(f\"  Abertura: {req['DataAbertura'][:10]}\")
    print()
"
```

**Estrutura da resposta:**

```json
{
  "_metadata": {
    "Release": "9.8.0",
    "LogAmigavel": [],
    "MensagensErro": [],
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
      "Assunto": "Problema com impressora",
      "Status": "Em Andamento",
      "Responsavel": "Joao Silva",
      "DataAbertura": "2025-09-30T10:00:00"
    }
  ]
}
```

> **Nota:** O campo e `RequisicaoId` (nao `Id`). Paginacao: `pageSize` (padrao 20, maximo 100), `pageNumber` (base 1).

---

## 3. Buscar Requisicao por ID

Retorna os detalhes completos de uma requisicao especifica. A resposta usa `Conjuntos` como dicionario (diferente do catalogo, onde e um array).

```bash
#!/usr/bin/env bash
# buscar-requisicao.sh — Exibe detalhes de uma requisicao

BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"
REQUISICAO_ID=35174

curl -s -X GET \
  "$BASE_URL/v1/requisicoes/$REQUISICAO_ID" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "
import sys, json
resp = json.load(sys.stdin)

if resp.get('MensagensErro'):
    print('Erro:', resp['MensagensErro'])
    sys.exit(1)

conjuntos = resp.get('Conjuntos', {})

# Detalhes basicos
if 'Detalhes Do Pedido' in conjuntos:
    det = conjuntos['Detalhes Do Pedido']
    print(f\"Assunto : {det.get('Assunto', '-')}\")
    print(f\"Status  : {det.get('Status', '-')}\")
    print(f\"Descricao: {det.get('Descricao', '-')}\")

# Participantes
if 'Participantes' in conjuntos:
    part = conjuntos['Participantes']
    print()
    print('Participantes:')
    for papel, nome in part.items():
        print(f'  {papel}: {nome}')

print()
print('URLs disponiveis:')
for chave in ['UrlAcoes', 'UrlHistorico', 'UrlDadosAdicionais']:
    print(f'  {chave}: {resp.get(chave, \"N/A\")}')
"
```

**Estrutura da resposta:**

```json
{
  "Conjuntos": {
    "Detalhes Do Pedido": {
      "Assunto": "Problema com impressora",
      "Descricao": "Impressora nao liga desde ontem",
      "Status": "Em Andamento"
    },
    "Participantes": {
      "Solicitante": "Maria Santos",
      "Responsavel": "Joao Silva"
    }
  },
  "UrlAcoes": "/v1/requisicoes/35174/acoes",
  "UrlHistorico": "/v1/requisicoes/35174/historico",
  "MensagensErro": []
}
```

---

## 4. Criar Requisicao

Abre uma nova requisicao. O campo `Formulario` recebe o ID do formulario (obtido via `GET /v1/cardapio`). Os dados sao organizados em `Conjuntos` — um dicionario onde cada chave e a `Chave` do conjunto no formulario.

```bash
#!/usr/bin/env bash
# criar-requisicao.sh — Abre uma nova requisicao

BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"

# ID do formulario (obtido via GET /v1/cardapio)
FORMULARIO_ID=101

PAYLOAD=$(cat <<'ENDJSON'
{
  "Formulario": 101,
  "Conjuntos": {
    "DadosBasicos": {
      "Assunto": "Impressora nao liga",
      "Descricao": "A impressora do setor financeiro nao liga desde esta manha."
    }
  }
}
ENDJSON
)

RESPOSTA=$(curl -s -X POST "$BASE_URL/v1/requisicoes/abrir" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD")

echo "Resposta: $RESPOSTA"

# A resposta e apenas o ID como string (ex: "12345")
REQUISICAO_ID=$(echo "$RESPOSTA" | python3 -c "
import sys, json
val = json.load(sys.stdin)
# /abrir retorna string simples com o ID
print(val)
")

echo "Requisicao criada com ID: $REQUISICAO_ID"
```

> **Dica:** Use `POST /v1/requisicoes/abrirRequisicao` para obter um envelope completo com `IdRequisicaoAberta` e `_metadata`. O endpoint `/abrir` retorna apenas o ID como string simples.

**Exemplo de corpo com campos adicionais:**

```bash
PAYLOAD=$(cat <<'ENDJSON'
{
  "Formulario": 318,
  "Conjuntos": {
    "DadosBasicos": {
      "Assunto": "Compra de equipamentos",
      "Descricao": "Solicitacao de compra para o departamento."
    },
    "Itens": {
      "Produto": "Teclado",
      "Quantidade": "2"
    }
  }
}
ENDJSON
)
```

---

## 5. Executar Acao (Encerrar)

Executa uma acao de workflow em uma requisicao existente. O campo `Id` usa o formato `"Nome [CODIGO]"`.

```bash
#!/usr/bin/env bash
# encerrar-requisicao.sh — Encerra uma requisicao

BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"
REQUISICAO_ID=35174

# Primeiro, liste as acoes disponiveis
echo "=== Acoes disponiveis ==="
curl -s -X GET \
  "$BASE_URL/v1/requisicoes/$REQUISICAO_ID/acoes" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "
import sys, json
resp = json.load(sys.stdin)
for acao in resp.get('records', []):
    print(f\"  {acao['Id']}\")
"

echo
echo "=== Executando acao Encerrar ==="

PAYLOAD=$(cat <<'ENDJSON'
{
  "Id": "Encerrar [ENC]",
  "Descricao": "Problema resolvido. Impressora substituida e testada.",
  "Motivo": 1
}
ENDJSON
)

RESPOSTA=$(curl -s -w "\n%{http_code}" -X POST \
  "$BASE_URL/v1/requisicoes/$REQUISICAO_ID/acoes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD")

HTTP_CODE=$(echo "$RESPOSTA" | tail -1)
CORPO=$(echo "$RESPOSTA" | head -1)

if [ "$HTTP_CODE" = "200" ]; then
  echo "Acao executada com sucesso."
elif [ "$HTTP_CODE" = "406" ]; then
  echo "Erro de negocio (HTTP 406):"
  echo "$CORPO" | python3 -c "
import sys, json
resp = json.load(sys.stdin)
for msg in resp.get('_metadata', {}).get('MensagensErro', []):
    print(f'  - {msg}')
"
else
  echo "Erro HTTP $HTTP_CODE: $CORPO"
fi
```

**Outros exemplos de acoes:**

```bash
# Direcionar para outro grupo
curl -s -X POST "$BASE_URL/v1/requisicoes/$REQUISICAO_ID/acoes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"Id": "Direcionar [DIR]", "Descricao": "Direcionando para infra.", "GrupoId": 10}'

# Alterar prioridade
curl -s -X POST "$BASE_URL/v1/requisicoes/$REQUISICAO_ID/acoes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"Id": "Alterar Prioridade [ALTPRI]", "Descricao": "Urgente.", "prioridade": 1}'
```

---

## 6. Enviar Anexo

Faz upload de um arquivo como anexo de uma requisicao. Usa `multipart/form-data`. Atencao: a rota de upload e `/anexo` (singular).

```bash
#!/usr/bin/env bash
# enviar-anexo.sh — Envia um arquivo como anexo de uma requisicao

BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"
REQUISICAO_ID=35174
ARQUIVO="/caminho/para/relatorio.pdf"

if [ ! -f "$ARQUIVO" ]; then
  echo "Arquivo nao encontrado: $ARQUIVO"
  exit 1
fi

echo "Enviando anexo: $(basename "$ARQUIVO")"

RESPOSTA=$(curl -s -X POST \
  "$BASE_URL/v1/requisicoes/$REQUISICAO_ID/anexo" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@$ARQUIVO")

echo "Resposta: $RESPOSTA"

echo "$RESPOSTA" | python3 -c "
import sys, json
resp = json.load(sys.stdin)
if resp.get('MensagensErro'):
    print('Erro:', resp['MensagensErro'])
else:
    print(f\"Anexo enviado. ID: {resp.get('Id', '?')}\")
"
```

**Estrutura da resposta:**

```json
{
  "Id": 5678,
  "MensagensErro": []
}
```

**Listar anexos existentes:**

```bash
curl -s -X GET \
  "$BASE_URL/v1/requisicoes/$REQUISICAO_ID/anexos" \
  -H "Authorization: Bearer $TOKEN"
```

> **Nota:** Listagem usa `/anexos` (plural); upload usa `/anexo` (singular).

---

## 7. Listar Catalogo

Retorna todas as areas e formularios disponiveis para abertura de requisicoes.

```bash
#!/usr/bin/env bash
# listar-catalogo.sh — Lista areas e formularios do catalogo de servicos

BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"

echo "=== Catalogo de Servicos ==="
curl -s -X GET "$BASE_URL/v1/cardapio" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "
import sys, json
resp = json.load(sys.stdin)
for area in resp.get('records', []):
    print(f\"[Area {area['Id']}] {area['Nome']}\")
    for frm in area.get('Formularios', []):
        print(f\"  Formulario {frm['Id']}: {frm['Nome']}\")
        print(f\"    URL: {frm['Url']}\")
"
```

**Estrutura da resposta:**

```json
{
  "_metadata": { "Release": "9.8.0", "MensagensErro": null },
  "records": [
    {
      "Id": 1,
      "Nome": "Suporte de TI",
      "Formularios": [
        {
          "Id": 101,
          "Nome": "Solicitacao de Suporte de PC",
          "Url": "https://sua-empresa.bdesk.com.br/askrest/v1/cardapio/formularios/101"
        }
      ]
    }
  ]
}
```

**Buscar detalhes de um formulario especifico:**

```bash
FORMULARIO_ID=101

curl -s -X GET "$BASE_URL/v1/cardapio/formularios/$FORMULARIO_ID" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "
import sys, json
resp = json.load(sys.stdin)
print(f\"Formulario: {resp.get('Nome', '-')} (v{resp.get('Versao', '?')})\")
print('Conjuntos:')
for conj in resp.get('Conjuntos', []):
    print(f\"  [{conj['Chave']}] {conj['Nome']}\")
    for campo in conj.get('Campos', []):
        obrig = '*' if campo.get('Obrigatoriedade') else ' '
        print(f\"    {obrig} {campo['Chave']} ({campo['TipoDeDado']})\")
"
```

---

## 8. Buscar Participante

Pesquisa participantes por formulario-papel e termo de busca. Util para preencher campos do tipo participante ao abrir requisicoes.

```bash
#!/usr/bin/env bash
# buscar-participante.sh — Pesquisa participantes por nome

BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"

# formularioPapelId: obtido nos campos do formulario (campo de participante)
FORMULARIO_PAPEL_ID=42
TERMO="silva"

echo "Pesquisando participantes com termo: '$TERMO'"

curl -s -X GET \
  "$BASE_URL/v1/participantes/$FORMULARIO_PAPEL_ID/pesquisar/$TERMO" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "
import sys, json
resultado = json.load(sys.stdin)

# Retorno e array direto (sem envelope _metadata)
if not resultado:
    print('Nenhum participante encontrado.')
    sys.exit(0)

for p in resultado:
    # O campo Id contem JSON serializado com IdParticipante e IdTipoPapel
    id_info = json.loads(p['Id'])
    print(f\"Nome : {p['Texto']}\")
    print(f\"  IdParticipante : {id_info['IdParticipante']}\")
    print(f\"  IdTipoPapel    : {id_info['IdTipoPapel']}\")
    print()
"
```

**Estrutura da resposta:**

```json
[
  {
    "Id": "{\"IdParticipante\":1803,\"IdTipoPapel\":1}",
    "Texto": "Joao Silva",
    "Legenda": null,
    "IdDominioPai": null
  }
]
```

> **Atenção:** O campo `Id` contem JSON serializado (nao um inteiro). O nome de exibicao esta em `Texto` (nao `Nome`). A resposta e um array direto, sem envelope `_metadata`.

---

## Script Completo: Fluxo de Integracao

Script de exemplo que encadeia todos os passos: login, consulta do catalogo, criacao de requisicao e envio de anexo.

```bash
#!/usr/bin/env bash
# fluxo-completo.sh — Exemplo de integracao completa

set -e

BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
LOGIN="seu-usuario"
SENHA="sua-senha"

echo "=== 1. Login ==="
TOKEN=$(curl -s -X POST "$BASE_URL/v1/login/entrar" \
  -H "Content-Type: application/json" \
  -d "{\"Login\": \"$LOGIN\", \"Senha\": \"$SENHA\"}" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); t=json.loads(d['Dados']); print(t['access_token'])")
echo "Autenticado. Token: ${TOKEN:0:20}..."

echo
echo "=== 2. Listar Catalogo ==="
curl -s "$BASE_URL/v1/cardapio" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "
import sys,json
for area in json.load(sys.stdin).get('records',[]):
    print(f\"  [{area['Id']}] {area['Nome']}: {len(area.get('Formularios',[]))} formularios\")
"

echo
echo "=== 3. Criar Requisicao ==="
REQ_ID=$(curl -s -X POST "$BASE_URL/v1/requisicoes/abrir" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"Formulario":101,"Conjuntos":{"DadosBasicos":{"Assunto":"Teste via API","Descricao":"Requisicao criada pelo script de integracao."}}}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin))")
echo "Requisicao criada: #$REQ_ID"

echo
echo "=== 4. Consultar Requisicao ==="
curl -s "$BASE_URL/v1/requisicoes/$REQ_ID" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "
import sys,json
resp = json.load(sys.stdin)
det = resp.get('Conjuntos',{}).get('Detalhes Do Pedido',{})
print(f\"  Assunto: {det.get('Assunto','-')}\")
print(f\"  Status : {det.get('Status','-')}\")
"
echo "Concluido."
```
