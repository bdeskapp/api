# Tratamento de Erros

## Como a API Retorna Erros

A API BDesk retorna erros de negócio no campo `_metadata.MensagensErro` da resposta. Este campo é um array de strings com as mensagens de erro.

**Exemplo de resposta com erro:**
```json
{
  "_metadata": {
    "MensagensErro": [
      "Usuário ou senha inválidos"
    ],
    "LogAmigavel": []
  }
}
```

## Códigos de Status HTTP

| Código | Significado | O que fazer |
|--------|-------------|-------------|
| 200 | Sucesso | Requisição processada com sucesso |
| 400 | Requisição inválida | Verifique o formato do JSON e os parâmetros enviados |
| 401 | Não autenticado | Faça login ou renove o token (ver [Autenticação](../autenticacao.md)) |
| 403 | Sem permissão | Solicite permissão ao administrador BDesk |
| 406 | Erro de negócio | Leia as mensagens em `_metadata.MensagensErro` |
| 500 | Erro interno do servidor | Contate o suporte BDesk |

## Erros de Negócio (HTTP 406)

O código 406 é o principal indicador de erros de negócio na API BDesk. Isso ocorre quando a operação foi recebida corretamente, mas não pôde ser executada por uma regra de negócio — por exemplo, campos obrigatórios ausentes, permissão insuficiente ou estado inválido da requisição.

Sempre verifique o array `MensagensErro` para entender o motivo da rejeição:

**Python:**
```python
import requests

resp = requests.post(url, json=payload, headers=headers)
if resp.status_code == 406:
    erros = resp.json()["_metadata"]["MensagensErro"]
    for erro in erros:
        print(f"Erro: {erro}")
```

**PowerShell:**
```powershell
try {
    $resp = Invoke-RestMethod -Uri $url -Method Post -Headers $headers -Body $body -ContentType "application/json"
} catch {
    $detalhes = $_.ErrorDetails.Message | ConvertFrom-Json
    $detalhes._metadata.MensagensErro | ForEach-Object { Write-Warning $_ }
}
```

**cURL:**
```bash
curl -s -o resposta.json -w "%{http_code}" -X POST "$URL" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "$BODY"
# Se retornar 406, leia resposta.json e inspecione _metadata.MensagensErro
```

## Peculiaridade do Endpoint de Login

O endpoint de login (`POST /v1/login/entrar`) usa um formato de resposta diferente dos demais endpoints. Em caso de erro, a estrutura retornada é:

```json
{
  "Dados": null,
  "LogAmigavel": [],
  "MensagensErro": ["Usuário ou senha inválidos"],
  "Versao": "9.8.0"
}
```

Note que as mensagens de erro ficam em `MensagensErro` diretamente no objeto raiz — **não** dentro de `_metadata`.

**Python — tratando erro de login:**
```python
resp = requests.post(url_login, json={"Login": "usuario", "Senha": "senha"})
dados = resp.json()

# No login, as mensagens ficam fora do _metadata
erros = dados.get("MensagensErro", [])
if erros:
    print("Falha no login:", erros)
else:
    token_data = json.loads(dados["Dados"])  # Parse duplo!
    token = token_data["access_token"]
```

## Erros Comuns e Soluções

| Situação | Código HTTP | Causa provável | Solução |
|----------|-------------|----------------|---------|
| Token ausente ou inválido | 401 | Header `Authorization` incorreto ou token expirado | Refaça o login e obtenha um novo token |
| Token expirado | 401 | Tokens têm validade limitada | Chame `POST /v1/login/entrar` novamente |
| Sem acesso ao recurso | 403 | Usuário sem permissão no BDesk | Solicite permissão ao administrador |
| Campo obrigatório ausente | 406 | Formulário exige campos não enviados | Leia `MensagensErro` para identificar quais campos faltam |
| Requisição não encontrada | 406 | ID inexistente ou sem acesso | Confirme o ID e se o usuário tem acesso à requisição |
| JSON malformado | 400 | Estrutura inválida no corpo da requisição | Valide o JSON antes de enviar |
| Dados adicionais rejeitados | 406 | `Conjuntos` enviado como array em vez de objeto | Veja o formato correto em [Criar Requisições](../guias/criar-requisicoes.md) |

## Dicas de Diagnóstico

- **Campos com PascalCase**: todos os campos da API usam PascalCase (`Assunto`, `IdSolicitante`, `Conjuntos`). Campos em minúsculas serão ignorados silenciosamente.
- **Content-Type obrigatório**: endpoints que recebem corpo JSON exigem o header `Content-Type: application/json`. Sem ele, a requisição pode ser rejeitada com 400.
- **Array vs. objeto**: o campo `Conjuntos` (dados adicionais) é um objeto/dicionário, não um array. Enviar como array retornará 406.
- **Múltiplos erros**: `MensagensErro` pode conter mais de uma mensagem. Exiba todas ao usuário final ou ao log.
