# Trabalhar com Anexos

## O que voce vai aprender

Como listar os anexos de uma requisicao, baixar um arquivo especifico e enviar novos arquivos
como anexo — tudo via API BDesk. O envio de um arquivo acontece em **dois passos**: primeiro o
upload do arquivo, depois a submissao que o vincula a requisicao.

---

## Pre-requisitos

- **Token de autenticacao valido** — veja o guia de [Autenticacao](../autenticacao.md).
- **ID da requisicao** — o numero da requisicao a qual os anexos pertencem (ex: `12345`).
  Obtenha-o ao consultar a lista de requisicoes abertas ou encerradas.
- **Permissao de anexar** — o usuario autenticado precisa poder executar a acao "anexar
  documento" (codigo `ANDOC`) na requisicao, no status em que ela esta. Consulte
  `GET /v1/requisicoes/{id}/acoes` para ver as acoes disponiveis.

---

## Passo a Passo

### Passo 1: Listar os anexos da requisicao

Use `GET /v1/requisicoes/{id}/anexos` para obter todos os arquivos ja vinculados a uma
requisicao. A resposta inclui a URL de download de cada anexo.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/12345/anexos" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

**Resposta:**

```json
{
  "_metadata": {
    "MensagensErro": []
  },
  "records": [
    {
      "Id": 5678,
      "Titulo": "Contrato de servico",
      "NomeDocumentoFisico": "00012345_001_contrato-servico.pdf",
      "NomeUsuario": "Maria Santos",
      "DataInclusao": "2025-10-01T14:22:00",
      "Versao": 1,
      "Publico": true,
      "Invalido": false,
      "Excluido": false,
      "UrlDownload": "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/12345/anexos/5678"
    },
    {
      "Id": 5679,
      "Titulo": "Evidencia do erro",
      "NomeDocumentoFisico": "00012345_001_evidencia-erro.png",
      "NomeUsuario": "Joao Silva",
      "DataInclusao": "2025-10-01T15:10:00",
      "Versao": 1,
      "Publico": true,
      "Invalido": false,
      "Excluido": false,
      "UrlDownload": "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/12345/anexos/5679"
    }
  ]
}
```

O `Id` do registro (numero inteiro) e o que identifica o anexo no download. Cada registro traz
ainda outros campos, como `NomeTipoDocumento`, `Inline` e `IdAnexoOriginal`.

> **Nota sobre as rotas:** a listagem e o download usam o plural `/anexos`. O upload usa o
> singular `/anexo`, e a submissao usa `/anexos/submeter`. Use a rota correta para cada operacao.

---

### Passo 2: Baixar um anexo

Use a URL do campo `UrlDownload` retornada na listagem para fazer o download do arquivo.
O servidor retorna o binario do arquivo com o `Content-Type` apropriado e o nome do arquivo no
cabecalho `Content-Disposition` (e o nome fisico gravado no servidor, nao o `Titulo`).

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/12345/anexos/5678" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -o "contrato-servico.pdf"
```

Se a requisicao ou o documento nao existirem, ou o usuario nao tiver acesso, a resposta e
HTTP 406 com uma mensagem em texto puro (nao e um arquivo nem um JSON).

---

### Passo 3: Enviar um arquivo como anexo (2 etapas)

Enviar um anexo exige duas chamadas. **Apenas o upload nao anexa nada.** Se voce parar na
primeira etapa, a API responde 200 e o arquivo fica numa area temporaria, mas a requisicao
continua sem o anexo.

#### Etapa 1 — Upload do arquivo

Use `POST /v1/requisicoes/{id}/anexo` com `Content-Type: multipart/form-data`. O campo de
formulario recomendado e `file` (o servidor le o primeiro arquivo enviado, qualquer que seja o nome do campo).

```bash
curl -X POST "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/12345/anexo" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -F "file=@/caminho/para/arquivo.pdf"
```

**Resposta:**

```json
{
  "Id": "3f2c9a1e-7b4d-4c1a-9e55-0a1b2c3d4e5f",
  "MensagensErro": []
}
```

O campo `Id` e um **GUID em texto** que identifica o arquivo na area temporaria. Ele **nao** e o
id de um anexo da requisicao. Se `MensagensErro` vier preenchido (ou `Id` vazio), o upload
falhou — leia as mensagens. Guarde o `Id` para a etapa 2.

Nesta etapa a API nao valida o nome nem a extensao do arquivo; isso acontece na etapa 2.

#### Etapa 2 — Submeter o anexo a requisicao

Use `POST /v1/requisicoes/{id}/anexos/submeter` com corpo JSON (`Content-Type: application/json`)
para vincular o arquivo enviado a requisicao:

```json
{
  "CodigoAcao": "ANDOC",
  "Anexos": [
    {
      "Id": "3f2c9a1e-7b4d-4c1a-9e55-0a1b2c3d4e5f",
      "NomeDuranteUpload": "arquivo.pdf",
      "Titulo": "arquivo.pdf"
    }
  ]
}
```

| Campo | Descricao |
|-------|-----------|
| `CodigoAcao` | Codigo da acao "anexar documento" do formulario. Normalmente `ANDOC`. |
| `Anexos[].Id` | O GUID devolvido pela etapa 1. |
| `Anexos[].NomeDuranteUpload` | Nome do arquivo **com a extensao** (ex.: `relatorio.pdf`). E validado e define o nome final do arquivo no servidor. |
| `Anexos[].Titulo` | Titulo exibido do documento na requisicao. |

Para enviar varios arquivos, faca um upload (etapa 1) para cada arquivo e envie todos os `Id`
recebidos numa unica chamada de submissao, como itens de `Anexos`.

```bash
curl -X POST "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/12345/anexos/submeter" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -H "Content-Type: application/json" \
  -d '{"CodigoAcao":"ANDOC","Anexos":[{"Id":"3f2c9a1e-7b4d-4c1a-9e55-0a1b2c3d4e5f","NomeDuranteUpload":"arquivo.pdf","Titulo":"arquivo.pdf"}]}'
```

**Resposta (sucesso):** HTTP 200 com o envelope abaixo e `MensagensErro` vazio.

```json
{
  "_metadata": {
    "Release": "9.8.0",
    "LogAmigavel": [],
    "MensagensErro": []
  },
  "records": null
}
```

Depois da submissao, confira o resultado com `GET /v1/requisicoes/{id}/anexos`: o novo
documento deve aparecer na lista.

---

## Exemplos Completos

### cURL — Listar, baixar e enviar

```bash
BASE="https://sua-empresa.bdesk.com.br/askrest"
TOKEN="SEU_TOKEN_AQUI"
REQ_ID=12345

# 1. Listar anexos
curl -s "$BASE/v1/requisicoes/$REQ_ID/anexos" \
  -H "Authorization: Bearer $TOKEN"

# 2. Baixar o primeiro anexo (substitua 5678 pelo Id real)
curl -s "$BASE/v1/requisicoes/$REQ_ID/anexos/5678" \
  -H "Authorization: Bearer $TOKEN" \
  -o "arquivo-baixado.pdf"

# 3. Enviar novo anexo — etapa 1: upload (devolve o Id em GUID)
curl -s -X POST "$BASE/v1/requisicoes/$REQ_ID/anexo" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@/caminho/para/arquivo.pdf"
# Resposta: {"Id":"3f2c9a1e-7b4d-4c1a-9e55-0a1b2c3d4e5f","MensagensErro":[]}

# 4. Enviar novo anexo — etapa 2: submeter (use o Id recebido na etapa 1)
GUID="3f2c9a1e-7b4d-4c1a-9e55-0a1b2c3d4e5f"
curl -s -X POST "$BASE/v1/requisicoes/$REQ_ID/anexos/submeter" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"CodigoAcao\":\"ANDOC\",\"Anexos\":[{\"Id\":\"$GUID\",\"NomeDuranteUpload\":\"arquivo.pdf\",\"Titulo\":\"arquivo.pdf\"}]}"
```

---

### Python — Listar, baixar e enviar

```python
import requests

BASE = "https://sua-empresa.bdesk.com.br/askrest"
TOKEN = "SEU_TOKEN_AQUI"
REQ_ID = 12345
HEADERS = {"Authorization": f"Bearer {TOKEN}"}

# 1. Listar anexos
resp = requests.get(f"{BASE}/v1/requisicoes/{REQ_ID}/anexos", headers=HEADERS)
resp.raise_for_status()
anexos = resp.json()["records"]
print(f"{len(anexos)} anexo(s) encontrado(s)")

# 2. Baixar o primeiro anexo
if anexos:
    url_download = anexos[0]["UrlDownload"]
    nome_arquivo = anexos[0]["NomeDocumentoFisico"]
    download = requests.get(url_download, headers=HEADERS)
    download.raise_for_status()
    with open(nome_arquivo, "wb") as f:
        f.write(download.content)
    print(f"Arquivo salvo: {nome_arquivo}")

# 3. Enviar novo anexo — etapa 1: upload (area temporaria)
caminho = "/caminho/para/arquivo.pdf"
nome = "arquivo.pdf"
with open(caminho, "rb") as f:
    upload = requests.post(
        f"{BASE}/v1/requisicoes/{REQ_ID}/anexo",
        files={"file": f},
        headers=HEADERS
    )
upload.raise_for_status()
resultado = upload.json()
if resultado.get("MensagensErro") or not resultado.get("Id"):
    raise SystemExit(f"Erro no upload: {resultado.get('MensagensErro')}")
guid = resultado["Id"]  # GUID em texto, ainda NAO e um anexo da requisicao

# 4. Enviar novo anexo — etapa 2: submeter (vincula o arquivo a requisicao)
submissao = {
    "CodigoAcao": "ANDOC",
    "Anexos": [{"Id": guid, "NomeDuranteUpload": nome, "Titulo": nome}],
}
resp = requests.post(
    f"{BASE}/v1/requisicoes/{REQ_ID}/anexos/submeter",
    json=submissao,
    headers=HEADERS
)
if resp.status_code != 200:
    # 406: a mensagem vem em texto puro
    raise SystemExit(f"Falha ao submeter ({resp.status_code}): {resp.text}")
print("Anexo vinculado a requisicao")
```

---

### PowerShell — Listar, baixar e enviar

```powershell
$Base  = "https://sua-empresa.bdesk.com.br/askrest"
$Token = "SEU_TOKEN_AQUI"
$ReqId = 12345
$Headers = @{ Authorization = "Bearer $Token" }

# 1. Listar anexos
$lista = Invoke-RestMethod -Uri "$Base/v1/requisicoes/$ReqId/anexos" -Headers $Headers
Write-Host "$($lista.records.Count) anexo(s) encontrado(s)"

# 2. Baixar o primeiro anexo
$primeiro = $lista.records[0]
Invoke-RestMethod -Uri $primeiro.UrlDownload -Headers $Headers `
  -OutFile $primeiro.NomeDocumentoFisico
Write-Host "Arquivo salvo: $($primeiro.NomeDocumentoFisico)"

# 3. Enviar novo anexo — etapa 1: upload (area temporaria)
$caminho = "C:\caminho\para\arquivo.pdf"
$nome = "arquivo.pdf"
$form = @{ file = Get-Item -Path $caminho }
$upload = Invoke-RestMethod -Method Post `
  -Uri "$Base/v1/requisicoes/$ReqId/anexo" `
  -Headers $Headers `
  -Form $form
if ($upload.MensagensErro -or -not $upload.Id) {
    throw "Erro no upload: $($upload.MensagensErro -join ', ')"
}
$guid = $upload.Id   # GUID em texto, ainda NAO e um anexo da requisicao

# 4. Enviar novo anexo — etapa 2: submeter (vincula o arquivo a requisicao)
$submissao = @{
    CodigoAcao = "ANDOC"
    Anexos     = @(@{ Id = $guid; NomeDuranteUpload = $nome; Titulo = $nome })
} | ConvertTo-Json -Depth 5
$resp = Invoke-RestMethod -Method Post `
  -Uri "$Base/v1/requisicoes/$ReqId/anexos/submeter" `
  -Headers $Headers `
  -ContentType "application/json" `
  -Body $submissao
Write-Host "Anexo vinculado a requisicao"
```

> **Versao minima do PowerShell:** O parametro `-Form` no `Invoke-RestMethod` esta disponivel
> a partir do **PowerShell 6.1**. Em versoes anteriores, use `Invoke-WebRequest` com
> `MultipartFormDataContent` manualmente.

---

## Erros Comuns

Os erros de negocio (HTTP 406) trazem a mensagem em **texto puro** no corpo da resposta, nao em JSON.

| Codigo HTTP | Etapa | Mensagem / situacao | Causa provavel | Como resolver |
|-------------|-------|---------------------|----------------|---------------|
| 200 com `Id` vazio e `MensagensErro` preenchido | Upload | Falha ao gravar o arquivo no servidor | Area de armazenamento indisponivel | Leia `MensagensErro` e tente novamente; se persistir, acione o suporte |
| 406 | Upload | "Requisicao inexistente ou usuario sem acesso" | O `{id}` nao existe ou o usuario nao tem acesso | Confirme o ID e as permissoes do usuario autenticado |
| 500 | Upload | Corpo sem nenhum arquivo | A requisicao nao trouxe o arquivo no `multipart/form-data` | Envie o arquivo no campo `file` |
| 406 | Submissao | "Caractere nao permitido no nome do arquivo" | `NomeDuranteUpload` contem quebra de linha, tabulacao, barra ou barra invertida | Envie apenas o nome do arquivo, sem caminho |
| 406 | Submissao | "Extensao nao permitida: .ext" | A extensao (ex: `.exe`, `.js`) esta na lista de bloqueadas, ou o nome nao tem extensao | Use um formato permitido e inclua a extensao em `NomeDuranteUpload`. A lista de extensoes bloqueadas aparece no campo `ExtensoesNaoPermitidas` ao consultar `GET /v1/cardapio/{id}` |
| 406 | Submissao | "Usuario sem permissao de anexar documento neste status" | A acao de anexar nao esta disponivel para o usuario no status atual | Consulte `GET /v1/requisicoes/{id}/acoes` |
| 406 | Submissao | Mensagem informando que o formulario nao tem a acao de anexar | O `CodigoAcao` nao existe no formulario da requisicao | Confira o `CodigoAcao` (normalmente `ANDOC`) |
| 406 | Submissao | Falha ao gravar o anexo | Erro ao registrar o documento na requisicao | Leia a mensagem e tente novamente; se persistir, acione o suporte |
| 500 | Submissao | `Id` inexistente | O GUID nao existe na area temporaria (digitado errado ou ja submetido) | Refaca o upload (etapa 1) e use o `Id` recebido |
| 401 | Qualquer | Token invalido ou ausente | O header `Authorization` esta ausente, o token e invalido ou o usuario foi desativado | Obtenha um novo token via `POST /v1/login/entrar` |

---

## Proximos Passos

- [Criar Requisicoes](criar-requisicoes.md) — abra uma requisicao antes de anexar arquivos
- [Acoes de Workflow](acoes-workflow.md) — execute acoes como encerrar ou redirecionar apos o upload
