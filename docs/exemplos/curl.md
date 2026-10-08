# Exemplos cURL — API BDesk

Scripts completos e prontos para execução em shell (bash/zsh). Copie, ajuste as variáveis de configuração e execute.

> **Pré-requisitos:** `curl` e `python3` instalados. Não é necessário `jq`.

> **Lembrete sobre erros:** a API sinaliza falhas de duas formas: HTTP 406 com a mensagem em **texto puro**, ou HTTP 200 com a mensagem em `MensagensErro`. Os scripts abaixo tratam os dois casos. Veja [Tratamento de Erros](../referencia/erros.md).

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

O endpoint de login retorna o token dentro do campo `Dados`, que é uma **string JSON escapada** — não um objeto direto. É necessário fazer dois níveis de parse para extrair o `access_token`. O login responde **HTTP 200 mesmo quando falha**: confira `Dados` e `MensagensErro`.

```bash
#!/usr/bin/env bash
# login.sh — Autentica e exibe o token obtido

BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
LOGIN="seu-usuario"
SENHA="sua-senha"

RESPOSTA=$(curl -s -X POST "$BASE_URL/v1/login/entrar" \
  -H "Content-Type: application/json" \
  -d "{\"Login\": \"$LOGIN\", \"Senha\": \"$SENHA\"}")

# O login responde HTTP 200 mesmo em falha: confira Dados e MensagensErro.
# Dados e uma string JSON escapada — requer dois niveis de parse
TOKEN=$(echo "$RESPOSTA" | python3 -c "
import sys, json
resp = json.load(sys.stdin)
if resp.get('Dados') is None or resp.get('MensagensErro'):
    print('ERRO:', resp.get('MensagensErro'), file=sys.stderr)
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
  "Dados": "{\"access_token\":\"eyJhbGci...\",\"token_type\":\"bearer\",\"expires_in\":\"1799999999\",\"refresh_token\":null,\"scope\":\"admin\",\"error\":null}",
  "LogAmigavel": [],
  "MensagensErro": [],
  "Versao": null
}
```

> **Atenção:** `Dados` é uma string (não um objeto). Execute `JSON.parse(resp.Dados)` para extrair o token. Erros de autenticação **também retornam HTTP 200**: nesse caso `Dados` vem `null` e `MensagensErro` vem preenchido. O token não expira por tempo no servidor (`expires_in` é apenas informativo); HTTP 401 em uma chamada posterior indica token inválido ou usuário desativado.

---

## 2. Listar Requisicoes Abertas

Lista as requisições abertas do usuário autenticado. A API **não pagina**: a resposta traz as requisições até o limite `LimiteRequisicoes` (padrão 500). Use filtros para reduzir o volume (veja [Paginação e Limites](../referencia/paginacao.md)).

```bash
#!/usr/bin/env bash
# listar-abertas.sh — Lista requisicoes abertas com limite explicito

BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"

LIMITE=20

curl -s -X GET \
  "$BASE_URL/v1/requisicoes/abertas?LimiteRequisicoes=$LIMITE" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "
import sys, json
resp = json.load(sys.stdin)

erros = resp.get('_metadata', {}).get('MensagensErro')
if erros:
    print('Erro:', erros)
    sys.exit(1)

registros = resp.get('records', [])
print(f'{len(registros)} requisicoes (limite: $LIMITE)')
print()
for req in registros:
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
    "MensagensErro": [],
    "LogAmigavel": []
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

> **Nota:** O campo é `RequisicaoId` (não `Id`). Para filtros mais ricos (status, período de abertura, dados adicionais), use `POST /v1/requisicoes/abertas` com os filtros no corpo JSON.

---

## 3. Buscar Requisicao por ID

Retorna os detalhes completos de uma requisição específica. A resposta usa `Conjuntos` como dicionário (diferente do catálogo, onde é um array). Se a requisição não existir ou o usuário não tiver acesso, a resposta é **HTTP 200** com `Conjuntos` nulo e a mensagem em `MensagensErro`.

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

# Requisicao inexistente ou sem acesso: HTTP 200, Conjuntos nulo e MensagensErro preenchido
if resp.get('MensagensErro') or resp.get('Conjuntos') is None:
    print('Erro:', resp.get('MensagensErro'))
    sys.exit(1)

conjuntos = resp['Conjuntos']

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

Abre uma nova requisição. O campo `Formulario` recebe o ID do formulário (obtido via `GET /v1/cardapio`). Os dados são organizados em `Conjuntos` — um dicionário onde cada chave é a `Chave` do conjunto no formulário. Assunto e descrição ficam no conjunto `DadosBasicos`.

```bash
#!/usr/bin/env bash
# criar-requisicao.sh — Abre uma nova requisicao

BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"

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

RESPOSTA=$(curl -s -w "\n%{http_code}" -X POST "$BASE_URL/v1/requisicoes/abrir" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD")

HTTP_CODE=$(echo "$RESPOSTA" | tail -1)
CORPO=$(echo "$RESPOSTA" | sed '$d')

# Erro de negocio: HTTP 406 com a mensagem em texto puro (nao e JSON)
if [ "$HTTP_CODE" != "200" ]; then
  echo "Erro (HTTP $HTTP_CODE): $CORPO"
  exit 1
fi

# A resposta de sucesso e apenas o numero da requisicao, como texto JSON (ex: "12345")
REQUISICAO_ID=$(echo "$CORPO" | tr -d '"')
echo "Requisicao criada com ID: $REQUISICAO_ID"
```

> **Dica:** Use `POST /v1/requisicoes/abrirRequisicao` (mesmo corpo) para receber o número dentro do envelope padrão, em `records[0].IdRequisicaoAberta`. O endpoint `/abrir` retorna apenas o número como texto.

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

Executa uma ação de workflow em uma requisição existente. O campo `Id` deve ser **exatamente** o valor devolvido por `GET /v1/requisicoes/{id}/acoes`, que traz o código entre colchetes (por exemplo, `Encerrar [ENC]`). Enviar apenas `ENC` não funciona: a API responde 200 com a mensagem "Ação não encontrada".

Atenção ao tratamento de erro: quando a ação falha por regra de negócio (ação inexistente, usuário sem permissão neste status), a API responde **HTTP 200** com a mensagem em `_metadata.MensagensErro`. Só a falta de acesso à requisição responde 406.

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
  "tipoAvaliacao": 2
}
ENDJSON
)

RESPOSTA=$(curl -s -w "\n%{http_code}" -X POST \
  "$BASE_URL/v1/requisicoes/$REQUISICAO_ID/acoes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD")

HTTP_CODE=$(echo "$RESPOSTA" | tail -1)
CORPO=$(echo "$RESPOSTA" | sed '$d')

if [ "$HTTP_CODE" = "406" ]; then
  # Padrao (a): corpo em texto puro
  echo "Erro de negocio (HTTP 406): $CORPO"
elif [ "$HTTP_CODE" = "200" ]; then
  # Padrao (b): HTTP 200 pode trazer a mensagem em _metadata.MensagensErro
  echo "$CORPO" | python3 -c "
import sys, json
resp = json.load(sys.stdin)
erros = resp.get('_metadata', {}).get('MensagensErro') if isinstance(resp, dict) else None
if erros:
    print('A acao NAO foi executada:')
    for msg in erros:
        print(f'  - {msg}')
    sys.exit(1)
print('Acao executada com sucesso.')
"
else
  echo "Erro HTTP $HTTP_CODE: $CORPO"
fi
```

**Outros exemplos de acoes** (use o `Id` exato devolvido pela listagem de ações):

```bash
# Direcionar: NovoSolicitado recebe o Id do destino, copiado de
# GET /v1/requisicoes/$REQUISICAO_ID/acoes/DIR/grupos (envie exatamente como veio)
curl -s -X POST "$BASE_URL/v1/requisicoes/$REQUISICAO_ID/acoes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"Id": "Direcionar [DIR]", "Descricao": "Direcionando para infra.", "NovoSolicitado": "{ IdParticipante : 5, IdTipoPapel : 2 } "}'

# Alterar prioridade (1 = Alta, 2 = Media, 3 = Baixa)
curl -s -X POST "$BASE_URL/v1/requisicoes/$REQUISICAO_ID/acoes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"Id": "Alterar Prioridade [ALTPRI]", "Descricao": "Urgente.", "prioridade": 1}'
```

Veja todos os campos de cada ação em [Ações de Workflow](../guias/acoes-workflow.md).

---

## 6. Enviar Anexo (2 etapas)

Enviar um arquivo exige **duas chamadas**. O upload (etapa 1) apenas grava o arquivo em uma área temporária e responde 200: **nada é anexado à requisição ainda**. É a submissão (etapa 2) que vincula o arquivo. Atenção às rotas: o upload usa `/anexo` (singular) e a submissão usa `/anexos/submeter`.

```bash
#!/usr/bin/env bash
# enviar-anexo.sh — Envia um arquivo como anexo de uma requisicao (upload + submeter)

BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"
REQUISICAO_ID=35174
ARQUIVO="/caminho/para/relatorio.pdf"
NOME=$(basename "$ARQUIVO")

if [ ! -f "$ARQUIVO" ]; then
  echo "Arquivo nao encontrado: $ARQUIVO"
  exit 1
fi

# Etapa 1: upload (area temporaria). O campo do formulario deve se chamar "file".
echo "Etapa 1: enviando $NOME"
UPLOAD=$(curl -s -X POST \
  "$BASE_URL/v1/requisicoes/$REQUISICAO_ID/anexo" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@$ARQUIVO")

echo "Resposta do upload: $UPLOAD"

# O Id devolvido e um GUID em texto (nao e o id de um anexo da requisicao).
# Se Id vier vazio ou MensagensErro preenchido, o upload falhou.
GUID=$(echo "$UPLOAD" | python3 -c "
import sys, json
resp = json.load(sys.stdin)
if resp.get('MensagensErro') or not resp.get('Id'):
    print('Erro no upload:', resp.get('MensagensErro'), file=sys.stderr)
    sys.exit(1)
print(resp['Id'])
") || exit 1

# Etapa 2: submeter — vincula o arquivo a requisicao
echo "Etapa 2: submetendo o anexo (GUID $GUID)"
RESPOSTA=$(curl -s -w "\n%{http_code}" -X POST \
  "$BASE_URL/v1/requisicoes/$REQUISICAO_ID/anexos/submeter" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"CodigoAcao\":\"ANDOC\",\"Anexos\":[{\"Id\":\"$GUID\",\"NomeDuranteUpload\":\"$NOME\",\"Titulo\":\"$NOME\"}]}")

HTTP_CODE=$(echo "$RESPOSTA" | tail -1)
CORPO=$(echo "$RESPOSTA" | sed '$d')

if [ "$HTTP_CODE" != "200" ]; then
  # HTTP 406: mensagem em texto puro (extensao nao permitida, sem permissao de anexar etc.)
  echo "Falha ao submeter (HTTP $HTTP_CODE): $CORPO"
  exit 1
fi

echo "Anexo vinculado a requisicao. Confira a lista:"
curl -s "$BASE_URL/v1/requisicoes/$REQUISICAO_ID/anexos" \
  -H "Authorization: Bearer $TOKEN"
```

**Resposta do upload (etapa 1):**

```json
{
  "Id": "3f2c9a1e-7b4d-4c1a-9e55-0a1b2c3d4e5f",
  "MensagensErro": []
}
```

**Resposta da submissão (etapa 2, sucesso):**

```json
{
  "_metadata": { "MensagensErro": [] },
  "records": null
}
```

> **Notas:**
> - `NomeDuranteUpload` deve ter o nome do arquivo **com a extensão**; ele é validado (extensões como `.exe` e `.js` são recusadas com HTTP 406).
> - Para vários arquivos, faça um upload por arquivo e envie todos os `Id` em uma única chamada de submissão (lista `Anexos`).
> - Listagem e download usam `/anexos` (plural). Veja o guia completo em [Anexos](../guias/anexos.md).

---

## 7. Listar Catalogo

Retorna todas as áreas e formulários disponíveis para abertura de requisições.

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

**Buscar detalhes de um formulario especifico** (se o formulário não existir ou não for permitido, a API responde HTTP 403 com a mensagem em texto puro):

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

Pesquisa participantes por formulário-papel e termo de busca. Útil para preencher campos do tipo participante ao abrir requisições.

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

> **Atenção:** O campo `Id` contém JSON serializado (não um inteiro). O nome de exibição está em `Texto` (não `Nome`). A resposta é um array direto, sem envelope `_metadata`.

---

## Script Completo: Fluxo de Integracao

Script de exemplo que encadeia os passos: login, consulta do catálogo, criação de requisição, consulta e envio de anexo em 2 etapas. O script para ao primeiro erro e trata os dois padrões de erro da API.

```bash
#!/usr/bin/env bash
# fluxo-completo.sh — Exemplo de integracao completa

set -e

BASE_URL="https://sua-empresa.bdesk.com.br/askrest"
LOGIN="seu-usuario"
SENHA="sua-senha"
ARQUIVO="/caminho/para/relatorio.pdf"

echo "=== 1. Login ==="
TOKEN=$(curl -s -X POST "$BASE_URL/v1/login/entrar" \
  -H "Content-Type: application/json" \
  -d "{\"Login\": \"$LOGIN\", \"Senha\": \"$SENHA\"}" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
# O login responde 200 mesmo em falha
if d.get('Dados') is None or d.get('MensagensErro'):
    sys.exit('Falha no login: %s' % d.get('MensagensErro'))
print(json.loads(d['Dados'])['access_token'])
")
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
RESP=$(curl -s -w "\n%{http_code}" -X POST "$BASE_URL/v1/requisicoes/abrir" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"Formulario":101,"Conjuntos":{"DadosBasicos":{"Assunto":"Teste via API","Descricao":"Requisicao criada pelo script de integracao."}}}')
CODE=$(echo "$RESP" | tail -1)
CORPO=$(echo "$RESP" | sed '$d')
if [ "$CODE" != "200" ]; then
  echo "Erro ao criar (HTTP $CODE): $CORPO"
  exit 1
fi
REQ_ID=$(echo "$CORPO" | tr -d '"')
echo "Requisicao criada: #$REQ_ID"

echo
echo "=== 4. Consultar Requisicao ==="
curl -s "$BASE_URL/v1/requisicoes/$REQ_ID" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "
import sys,json
resp = json.load(sys.stdin)
if resp.get('MensagensErro') or resp.get('Conjuntos') is None:
    sys.exit('Erro: %s' % resp.get('MensagensErro'))
det = resp['Conjuntos'].get('Detalhes Do Pedido',{})
print(f\"  Assunto: {det.get('Assunto','-')}\")
print(f\"  Status : {det.get('Status','-')}\")
"

echo
echo "=== 5. Anexar arquivo (upload + submeter) ==="
NOME=$(basename "$ARQUIVO")
GUID=$(curl -s -X POST "$BASE_URL/v1/requisicoes/$REQ_ID/anexo" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@$ARQUIVO" \
  | python3 -c "
import sys,json
r = json.load(sys.stdin)
if r.get('MensagensErro') or not r.get('Id'):
    sys.exit('Erro no upload: %s' % r.get('MensagensErro'))
print(r['Id'])
")
CODE=$(curl -s -o /tmp/submeter.txt -w "%{http_code}" -X POST "$BASE_URL/v1/requisicoes/$REQ_ID/anexos/submeter" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"CodigoAcao\":\"ANDOC\",\"Anexos\":[{\"Id\":\"$GUID\",\"NomeDuranteUpload\":\"$NOME\",\"Titulo\":\"$NOME\"}]}")
if [ "$CODE" != "200" ]; then
  echo "Falha ao submeter (HTTP $CODE): $(cat /tmp/submeter.txt)"
  exit 1
fi
echo "Anexo vinculado."
echo "Concluido."
```
