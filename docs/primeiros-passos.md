# Primeiros Passos com a API BDesk

## O que voce vai aprender

Como configurar seu acesso e fazer sua primeira chamada a API BDesk.

---

## O que e a API BDesk

A API BDesk e uma interface REST que permite automatizar operacoes da sua plataforma de service desk. Com ela, voce pode criar requisicoes, consultar dados, executar acoes de fluxo de trabalho e integrar o BDesk com outros sistemas da sua empresa. Toda a troca de dados e feita em formato JSON. A autenticacao utiliza tokens Bearer obtidos no endpoint de login.

---

## Pre-requisitos

- Credenciais de usuario BDesk com permissao de acesso a API (solicite ao administrador da sua instancia)
- Uma ferramenta HTTP: [Postman](https://www.postman.com/), cURL (disponivel no terminal) ou qualquer linguagem de scripting (Python, PowerShell, etc.)

---

## URL Base da Sua Instalacao

Todas as chamadas partem da URL base da sua empresa:

```
https://sua-empresa.bdesk.com.br/askrest/v1/
```

Substitua `sua-empresa` pelo subdominio da sua instalacao BDesk. Se voce nao souber qual e, consulte o administrador do sistema.

---

## Passo a Passo

### Passo 1: Fazer Login

Antes de qualquer outra chamada, voce precisa obter um token de acesso. Faca uma requisicao `POST` para o endpoint de login:

```bash
curl -X POST "https://sua-empresa.bdesk.com.br/askrest/v1/login/entrar" \
  -H "Content-Type: application/json" \
  -d '{"Login": "seu-usuario", "Senha": "sua-senha"}'
```

A resposta sera semelhante a esta:

```json
{
  "Dados": "{\"access_token\":\"eyJhbGciOi...\",\"token_type\":\"bearer\",\"expires_in\":\"1799999999\",\"refresh_token\":null,\"scope\":\"admin\",\"error\":null}",
  "LogAmigavel": [],
  "MensagensErro": [],
  "Versao": null
}
```

> **Atencao:** O campo `Dados` contem uma string JSON, nao um objeto direto. Voce precisa fazer um parse adicional para extrair o token. Nao e possivel acessar `Dados.access_token` diretamente — primeiro converta o valor de `Dados` de string para objeto JSON.

> **O login responde 200 mesmo quando falha.** Se usuario ou senha estiverem incorretos, a resposta continua sendo HTTP 200, mas com `"Dados": null` e a mensagem em `MensagensErro`. Confira esses dois campos antes de usar o token.

Para extrair o token automaticamente no terminal, use o comando abaixo (requer Python 3):

```bash
# Extrair token (usando python)
TOKEN=$(curl -s -X POST "https://sua-empresa.bdesk.com.br/askrest/v1/login/entrar" \
  -H "Content-Type: application/json" \
  -d '{"Login": "seu-usuario", "Senha": "sua-senha"}' \
  | python3 -c "import sys,json; d=json.load(sys.stdin); t=json.loads(d['Dados']) if d.get('Dados') else sys.exit('Falha no login: %s' % d.get('MensagensErro')); print(t['access_token'])")
```

Agora a variavel `$TOKEN` contem o token Bearer. Use-a nas proximas chamadas.

O servidor nao expira o token por tempo: ele vale enquanto o usuario estiver ativo (o campo `expires_in` e apenas informativo). Se uma chamada responder HTTP 401, o token e invalido ou o usuario foi desativado: repita este passo para obter um novo token.

---

### Passo 2: Fazer Sua Primeira Chamada

Com o token em maos, faca uma chamada ao painel resumido de requisicoes (os totais sao do ambiente inteiro, nao apenas do seu usuario):

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

Se tudo estiver correto, voce recebera uma resposta como esta:

```json
{
  "Abertas": 42,
  "Encerradas": 1580,
  "UrlRequisicoesAbertas": "/v1/requisicoes/abertas",
  "UrlRequisicoesEncerradas": "/v1/requisicoes/encerradas"
}
```

Os campos `UrlRequisicoesAbertas` e `UrlRequisicoesEncerradas` indicam os endpoints para listar cada grupo de requisicoes.

---

### Passo 3: Listar Requisicoes Abertas

Para ver a lista de requisicoes abertas, use o endpoint `/v1/requisicoes/abertas`. A API nao pagina os resultados; use o parametro `LimiteRequisicoes` (padrao 500) para limitar a quantidade devolvida:

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abertas?LimiteRequisicoes=5" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI"
```

Exemplo de resposta:

```json
{
  "_metadata": {
    "MensagensErro": [],
    "LogAmigavel": []
  },
  "records": [
    {
      "RequisicaoId": 35174,
      "Assunto": "Falha no servidor de email",
      "DataAbertura": "2026-03-15T10:30:00"
    }
  ]
}
```

Para reduzir o volume, combine `LimiteRequisicoes` com outros filtros (assunto, status, periodo de abertura). Veja [Paginacao e Limites](referencia/paginacao.md).

> **Erros:** falhas de negocio podem voltar como HTTP 406 com a mensagem em texto puro, ou como HTTP 200 com a mensagem em `MensagensErro`. Veja [Tratamento de Erros](referencia/erros.md).

---

### Usando o Postman

Se preferir uma interface grafica, importe a colecao Postman disponivel no repositorio da documentacao:

1. Abra o Postman e clique em **Import**
2. Selecione o arquivo `postman_collection.json`
3. Apos importar, configure as seguintes variaveis de ambiente no Postman:
   - `{{baseUrl}}` — ex: `https://sua-empresa.bdesk.com.br/askrest`
   - `{{bearerToken}}` — o token obtido no Passo 1

Todas as requisicoes da colecao ja estao configuradas para usar essas variaveis.

---

## Proximos Passos

- [Autenticacao](autenticacao.md) — Detalhes sobre tokens, validade e re-login
- [Tratamento de Erros](referencia/erros.md) — Os dois padroes de erro da API
- [Criar Requisicoes](guias/criar-requisicoes.md) — Como criar requisicoes via API
- [Consultar Requisicoes](guias/consultar-requisicoes.md) — Como listar e filtrar requisicoes
- [Exemplos Python](exemplos/python.md) e [Exemplos PowerShell](exemplos/powershell.md) — Scripts completos prontos para usar

---

Todos os exemplos neste guia usam cURL. Para exemplos em Python e PowerShell, consulte a secao [Exemplos de Codigo](exemplos/curl.md).
