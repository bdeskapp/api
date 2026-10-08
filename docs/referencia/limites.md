# Limites da API BDesk

Esta pagina lista os limites operacionais da API BDesk. Respeitar esses limites evita erros e garante integracao estavel.

---

## Autenticacao

| Item                   | Valor                          |
|------------------------|--------------------------------|
| Validade do token      | Nao ha prazo de expiracao aplicado pelo servidor |
| Refresh token          | Nao disponivel                 |
| Header de autenticacao | `Authorization: Bearer <token>` |

O token obtido via login continua valido enquanto o usuario estiver ativo no BDesk. O campo `expires_in` da resposta de login e apenas informativo e nao e aplicado pelo servidor. Se o usuario for desativado, ou se o token for invalido, a API responde HTTP `401 Unauthorized`.

**Dica:** Monitore respostas HTTP `401` no seu codigo para disparar o fluxo de re-login automaticamente:

```python
resp = requests.get(url, headers=headers)
if resp.status_code == 401:
    token = refazer_login()
    headers["Authorization"] = f"Bearer {token}"
    resp = requests.get(url, headers=headers)
```

---

## Limites de Listagem

A API nao possui paginacao. Cada listagem tem um teto de registros proprio:

| Endpoint                                 | Limite / Padrao                          |
|------------------------------------------|------------------------------------------|
| `abertas` e `encerradas`                 | `LimiteRequisicoes`, padrao 500          |
| `busca`                                  | `maximoPorSituacao` (query), padrao 100  |
| `porcondicao/{condicao}/{quantidade}`    | `{quantidade}`, informado na rota        |
| `GET /v1/ics`                            | Sem limite (devolve todos os ICs)        |
| Pesquisa de participantes                | Ate 100 itens                            |

Consulte [Paginacao e Limites](./paginacao.md) para filtros e exemplos de como dividir uma consulta grande.

---

## Upload de Arquivos

| Item                       | Valor                              |
|----------------------------|------------------------------------|
| Tamanho maximo por arquivo | 4 MB na API (padrao do ASP.NET; a instalacao pode ajustar) |
| Content-Type               | `multipart/form-data`              |
| Extensoes bloqueadas       | Lista informada em `ExtensoesNaoPermitidas`, validada na submissao |

O limite de 4 MB e o padrao do ASP.NET para a API (`maxRequestLength`), e a instalacao pode ter um valor diferente. Arquivos maiores sao recusados com erro do servidor (normalmente HTTP 500, "Maximum request length exceeded"), nao com 413. O limite exibido na tela web do BDesk (por exemplo, 30 MB) e de outra aplicacao e nao vale para a API.

O envio e feito em duas etapas (upload e submissao; veja [Anexos](../guias/anexos.md)). O nome e a extensao do arquivo so sao validados na submissao. Para conhecer as extensoes bloqueadas, consulte o campo `ExtensoesNaoPermitidas` ao obter um item do catalogo:

```
GET https://sua-empresa.bdesk.com.br/askrest/v1/cardapio/{id}
```

---

## Formato de Dados

| Item                    | Valor                                        |
|-------------------------|----------------------------------------------|
| Content-Type (geral)    | `application/json`                           |
| Content-Type (upload)   | `multipart/form-data`                        |
| Encoding                | UTF-8                                        |
| Convencao de nomes      | PascalCase (ex: `Assunto`, `RequisicaoId`)   |

Todos os endpoints aceitam e retornam JSON com encoding UTF-8. A unica excecao sao os endpoints de upload de anexos, que utilizam `multipart/form-data`.

Os nomes de campos seguem a convencao **PascalCase** — por exemplo: `Assunto`, `Descricao`, `RequisicaoId`, `DataAbertura`. Atente-se a essa convencao ao construir payloads de requisicao ou ao ler campos da resposta.
