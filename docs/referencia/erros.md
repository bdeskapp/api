# Tratamento de Erros

## Dois padrões de erro

A API BDesk sinaliza falhas de negócio de duas formas diferentes, dependendo do endpoint. Um cliente robusto precisa tratar as duas:

| Padrão | Status HTTP | Onde está a mensagem | Como detectar |
|--------|-------------|----------------------|---------------|
| **(a) Erro com 406** | 406 | Corpo da resposta em **texto puro** (não é JSON) | Verifique o status; leia o texto do corpo |
| **(b) Erro com 200** | 200 | Campo `MensagensErro` dentro do JSON | Verifique o status **e** leia `MensagensErro` mesmo quando o status é 200 |

> **Importante:** um HTTP 200 nem sempre significa sucesso. Em vários endpoints, a falha de negócio volta com status 200 e a mensagem em `MensagensErro`. Sempre confira esse campo antes de considerar a operação concluída.

---

## (a) HTTP 406 com corpo em texto puro

Quando a operação foi recebida corretamente, mas não pôde ser executada por uma regra de negócio (campo obrigatório ausente, permissão insuficiente, requisição inexistente ou sem acesso, JSON malformado), a API responde **406** e o corpo é o **texto da mensagem**, com várias mensagens separadas por quebra de linha. O corpo **não é JSON** e **não** tem `_metadata`.

**Exemplo de resposta:**

```
HTTP/1.1 406 Not Acceptable
Content-Type: text/plain

Requisição inexistente ou usuário sem acesso
```

**Python:**
```python
import requests

resp = requests.post(url, json=payload, headers=headers)
if resp.status_code == 406:
    # O corpo é texto puro, não JSON
    for erro in resp.text.splitlines():
        print(f"Erro: {erro}")
```

**PowerShell:**
```powershell
try {
    $resp = Invoke-RestMethod -Uri $url -Method Post -Headers $headers -Body $body -ContentType "application/json"
} catch {
    if ($_.Exception.Response.StatusCode.value__ -eq 406) {
        # O corpo é texto puro
        Write-Warning $_.ErrorDetails.Message
    } else { throw }
}
```

**cURL:**
```bash
HTTP_CODE=$(curl -s -o resposta.txt -w "%{http_code}" -X POST "$URL" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "$BODY")
# Se for 406, resposta.txt contém a mensagem em texto puro
```

---

## (b) HTTP 200 com a mensagem em `MensagensErro`

Alguns endpoints não usam o 406: respondem **200** e informam o problema em um campo `MensagensErro`. O local desse campo varia:

| Formato da resposta | Onde fica `MensagensErro` | Exemplo |
|---------------------|---------------------------|---------|
| Envelope padrão (`_metadata` + `records`) | `_metadata.MensagensErro` | `{"_metadata":{"MensagensErro":["..."]},"records":[]}` |
| Login | Na raiz da resposta | `{"Dados":null,"MensagensErro":["..."],...}` |
| Detalhes da requisição (`GET /v1/requisicoes/{id}`) | Na raiz da resposta | `{"Conjuntos":null,"MensagensErro":["..."]}` |
| Upload de anexo (`POST /v1/requisicoes/{id}/anexo`) | Na raiz da resposta | `{"Id":"","MensagensErro":["..."]}` |

Em caso de sucesso, `MensagensErro` vem como lista vazia (`[]`). Em alguns endpoints, como `GET /v1/cardapio`, pode vir `null`: trate `null` e `[]` do mesmo modo.

**Principais endpoints que usam o padrão (b):**

- `POST /v1/login/entrar` e `POST /v1/login/EntrarApp`: falhas de autenticação voltam com 200, `Dados: null` e a mensagem em `MensagensErro`.
- `GET /v1/requisicoes/{id}`: requisição inexistente ou sem acesso volta com 200, `Conjuntos: null` e `MensagensErro` preenchido.
- `POST /v1/requisicoes/{id}/acoes`: a maioria dos erros (ação não encontrada, usuário sem permissão na ação neste status, data inválida) volta com 200 e a mensagem em `_metadata.MensagensErro`. Só a falta de acesso à requisição responde 406.
- `POST /v1/requisicoes/{id}/anexo`: falha ao gravar o arquivo volta com 200, `Id` vazio e `MensagensErro` preenchido.
- `POST /v1/requisicoes/responderpesquisasatisfacao`: a resposta é um texto JSON; em caso de falha ele começa com `"erro: ..."`.
- `GET /v1/cardapio/{id}`: item inexistente volta com 200 e um corpo reduzido a `LogAmigavel` e `MensagensErro`.
- `POST /v1/ics/{ic}/listaspapeis/{lista}/membros`: sempre 200, com o resultado individual de cada nome informado (veja [Gestão de ICs](../guias/gestao-ics.md)).

**Python:**
```python
resp = requests.post(url, json=payload, headers=headers)

if resp.status_code == 406:
    raise RuntimeError(resp.text)            # padrão (a)
resp.raise_for_status()                      # 401, 404, 500 etc.

corpo = resp.json()
if isinstance(corpo, dict):
    mensagens = (corpo.get("_metadata") or {}).get("MensagensErro") or corpo.get("MensagensErro")
    if mensagens:                            # padrão (b)
        raise RuntimeError("; ".join(mensagens))
```

**PowerShell:**
```powershell
$resp = Invoke-RestMethod -Uri $url -Method Post -Headers $headers -Body $body -ContentType "application/json"
$mensagens = if ($resp._metadata.MensagensErro) { $resp._metadata.MensagensErro } else { $resp.MensagensErro }
if ($mensagens) { throw ($mensagens -join "; ") }
```

---

## Códigos de Status HTTP

| Código | Significado | O que fazer |
|--------|-------------|-------------|
| 200 | Requisição processada | Confira `MensagensErro` (padrão b): o 200 não garante sucesso de negócio |
| 401 | Não autenticado | Token ausente, inválido, ou usuário desativado. Faça login novamente (ver [Autenticação](../autenticacao.md)) |
| 403 | Sem permissão | Ocorre em poucos endpoints, como `GET /v1/cardapio/formularios/{formulario}` (formulário inexistente ou sem acesso). O corpo é texto puro |
| 404 | Não encontrado | Rota inexistente; parâmetro de query obrigatório não informado tende a responder 404 (a seleção de ação do Web API descarta a rota) |
| 406 | Erro de negócio | O corpo é texto puro com a mensagem (padrão a) |
| 500 | Erro interno do servidor | Ocorre, por exemplo, quando um endpoint exige corpo e ele não foi enviado, ou quando o `Id` de um arquivo enviado não existe. Revise o que foi enviado; se persistir, contate o suporte BDesk |

---

## Peculiaridade do Endpoint de Login

O endpoint de login (`POST /v1/login/entrar`) usa um envelope próprio (`Dados`, `MensagensErro`, `LogAmigavel`, `Versao`) e **responde 200 mesmo em falha**. Exemplo de resposta a um login incorreto:

```json
{
  "Dados": null,
  "LogAmigavel": [],
  "MensagensErro": ["Usuário ou senha inválidos"],
  "Versao": null
}
```

As mensagens ficam em `MensagensErro` na raiz do objeto, **não** dentro de `_metadata`. Detecte a falha por `Dados == null` ou por `MensagensErro` não vazio.

**Python — tratando erro de login:**
```python
import json
import requests

resp = requests.post(url_login, json={"Login": "joao.silva", "Senha": "senha"})
resp.raise_for_status()
dados = resp.json()

if dados.get("Dados") is None or dados.get("MensagensErro"):
    print("Falha no login:", dados.get("MensagensErro"))
else:
    token_data = json.loads(dados["Dados"])  # Dados é uma string JSON: parse duplo
    token = token_data["access_token"]
```

---

## Erros Comuns e Soluções

| Situação | Resposta | Causa provável | Solução |
|----------|----------|----------------|---------|
| Login incorreto | 200, `Dados: null` | Usuário ou senha incorretos | Leia `MensagensErro` na raiz |
| Token ausente ou inválido | 401 | Header `Authorization` incorreto, token desconhecido ou usuário desativado | Refaça o login e obtenha um novo token |
| Parâmetro de query obrigatório ausente | 404 (tende a) | A seleção de ação do Web API descarta a rota sem o parâmetro (ex.: `termo` em `ObterParticipantes2`) | Informe todos os parâmetros obrigatórios |
| Campo obrigatório ausente na abertura | 406 | Formulário exige campos não enviados | Leia o texto do corpo para saber quais campos faltam |
| Requisição não encontrada em `GET /v1/requisicoes/{id}` | 200, `Conjuntos: null` | ID inexistente ou sem acesso | Leia `MensagensErro` e confirme o ID |
| Requisição não encontrada nas demais rotas | 406 | ID inexistente ou sem acesso | Confirme o ID e se o usuário tem acesso |
| Ação não executada | 200, `_metadata.MensagensErro` | `Id` da ação sem o código entre colchetes, ou usuário sem permissão no status atual | Use o `Id` exato de `GET /v1/requisicoes/{id}/acoes` (ex.: `Encerrar [ENC]`) |
| Dados adicionais rejeitados | 406 | Um conjunto de `Conjuntos` não é objeto nem lista de objetos | Veja o formato correto em [Criar Requisições](../guias/criar-requisicoes.md) |
| JSON malformado | 406 | Estrutura inválida no corpo (a mensagem começa com "Erro no JSON") | Valide o JSON antes de enviar |

---

## Dicas de Diagnóstico

- **Campos com PascalCase**: os campos da API usam PascalCase (`Assunto`, `Descricao`, `Conjuntos`). Algumas exceções existem na execução de ações (`prioridade`, `tipoAvaliacao`, `usuResponsavelId`); use a grafia exata indicada em cada guia.
- **Content-Type obrigatório**: endpoints que recebem corpo JSON exigem o header `Content-Type: application/json`.
- **Array vs. objeto**: o campo `Conjuntos` (dados adicionais) é um objeto/dicionário, não um array.
- **Múltiplos erros**: `MensagensErro` pode conter mais de uma mensagem, e o corpo de um 406 pode ter várias linhas. Exiba todas ao usuário final ou ao log.
- **Sucesso não é prova de efeito**: no envio de anexos, o upload responde 200 mesmo sem anexar nada; só a submissão vincula o arquivo. Veja [Anexos](../guias/anexos.md).
