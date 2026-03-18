# Trabalhar com Anexos

## O que voce vai aprender

Como listar os anexos de uma requisicao, baixar um arquivo especifico e enviar novos arquivos
como anexo — tudo via API BDesk.

---

## Pre-requisitos

- **Token de autenticacao valido** — veja o guia de [Autenticacao](../autenticacao.md).
- **ID da requisicao** — o numero da requisicao a qual os anexos pertencem (ex: `12345`).
  Obtenha-o ao consultar a lista de requisicoes abertas ou encerradas.

---

## Passo a Passo

### Passo 1: Listar os anexos da requisicao

Use `GET /v1/requisicoes/{id}/anexos` para obter todos os arquivos vinculados a uma requisicao.
A resposta inclui a URL de download de cada anexo.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/12345/anexos" \
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
    {
      "Id": 5678,
      "Nome": "contrato-servico.pdf",
      "Tamanho": 204800,
      "DataUpload": "2025-10-01T14:22:00",
      "UsuarioUpload": "Maria Santos",
      "UrlDownload": "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/12345/anexos/5678"
    },
    {
      "Id": 5679,
      "Nome": "evidencia-erro.png",
      "Tamanho": 98304,
      "DataUpload": "2025-10-01T15:10:00",
      "UsuarioUpload": "Joao Silva",
      "UrlDownload": "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/12345/anexos/5679"
    }
  ]
}
```

> **Nota sobre a rota:** O endpoint de listagem usa plural `/anexos`. O endpoint de upload usa
> singular `/anexo`. Essa diferenca e intencional — use a rota correta para cada operacao.

---

### Passo 2: Baixar um anexo

Use a URL do campo `UrlDownload` retornada na listagem para fazer o download do arquivo.
O servidor retorna o binario do arquivo com o `Content-Type` apropriado.

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/12345/anexos/5678" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -o "contrato-servico.pdf"
```

---

### Passo 3: Enviar um arquivo como anexo (upload)

Use `POST /v1/requisicoes/{id}/anexo` com `Content-Type: multipart/form-data` para anexar
um arquivo a uma requisicao existente. O campo de formulario deve se chamar `file`.

```bash
curl -X POST "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/12345/anexo" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -F "file=@/caminho/para/arquivo.pdf"
```

**Resposta:**

```json
{
  "Id": 5680,
  "MensagensErro": []
}
```

O campo `Id` contem o identificador do anexo criado. Se `MensagensErro` estiver preenchido,
o upload falhou — leia as mensagens para entender o motivo.

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

# 3. Enviar novo anexo
curl -X POST "$BASE/v1/requisicoes/$REQ_ID/anexo" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@/caminho/para/arquivo.pdf"
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
    nome_arquivo = anexos[0]["Nome"]
    download = requests.get(url_download, headers=HEADERS)
    download.raise_for_status()
    with open(nome_arquivo, "wb") as f:
        f.write(download.content)
    print(f"Arquivo salvo: {nome_arquivo}")

# 3. Enviar novo anexo
caminho = "/caminho/para/arquivo.pdf"
with open(caminho, "rb") as f:
    upload = requests.post(
        f"{BASE}/v1/requisicoes/{REQ_ID}/anexo",
        files={"file": f},
        headers=HEADERS
    )
upload.raise_for_status()
resultado = upload.json()
if resultado.get("MensagensErro"):
    print("Erro no upload:", resultado["MensagensErro"])
else:
    print(f"Anexo enviado com Id: {resultado['Id']}")
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
  -OutFile $primeiro.Nome
Write-Host "Arquivo salvo: $($primeiro.Nome)"

# 3. Enviar novo anexo
$caminho = "C:\caminho\para\arquivo.pdf"
$form = @{ file = Get-Item -Path $caminho }
$upload = Invoke-RestMethod -Method Post `
  -Uri "$Base/v1/requisicoes/$ReqId/anexo" `
  -Headers $Headers `
  -Form $form
if ($upload.MensagensErro) {
    Write-Error "Erro no upload: $($upload.MensagensErro -join ', ')"
} else {
    Write-Host "Anexo enviado com Id: $($upload.Id)"
}
```

> **Versao minima do PowerShell:** O parametro `-Form` no `Invoke-RestMethod` esta disponivel
> a partir do **PowerShell 6.1**. Em versoes anteriores, use `Invoke-WebRequest` com
> `MultipartFormDataContent` manualmente.

---

## Erros Comuns

| Codigo HTTP | Mensagem | Causa provavel | Como resolver |
|-------------|----------|----------------|---------------|
| 406 | "Extensao nao permitida" | O tipo do arquivo (ex: `.exe`, `.js`) esta na lista de extensoes bloqueadas do formulario | Use um formato permitido. A lista de extensoes bloqueadas aparece no campo `ExtensoesNaoPermitidas` ao consultar `GET /v1/cardapio/{id}` |
| 406 | "Arquivo muito grande" | O tamanho do arquivo excede o limite configurado | Compacte o arquivo ou divida em partes menores antes do upload |
| 401 | Token invalido ou ausente | O header `Authorization` esta ausente ou o token expirou | Obtenha um novo token via `POST /v1/login/entrar` |
| 404 | Requisicao nao encontrada | O `{id}` da requisicao nao existe ou o usuario nao tem acesso | Confirme o ID e as permissoes do usuario autenticado |

---

## Proximos Passos

- [Criar Requisicoes](criar-requisicoes.md) — abra uma requisicao antes de anexar arquivos
- [Acoes de Workflow](acoes-workflow.md) — execute acoes como encerrar ou redirecionar apos o upload
