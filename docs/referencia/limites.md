# Limites da API BDesk

Esta pagina lista os limites operacionais da API BDesk. Respeitar esses limites evita erros e garante integracao estavel.

---

## Autenticacao

| Item                   | Valor                          |
|------------------------|--------------------------------|
| Validade do token      | 3600 segundos (1 hora)         |
| Refresh token          | Nao disponivel                 |
| Header de autenticacao | `Authorization: Bearer <token>` |

O token obtido via login e valido por **1 hora**. Nao existe mecanismo de refresh token — quando o token expirar, e necessario fazer login novamente para obter um novo.

**Dica:** Monitore respostas HTTP `401 Unauthorized` no seu codigo para disparar o fluxo de re-login automaticamente:

```python
resp = requests.get(url, headers=headers)
if resp.status_code == 401:
    token = refazer_login()
    headers["Authorization"] = f"Bearer {token}"
    resp = requests.get(url, headers=headers)
```

---

## Paginacao

| Parametro            | Limite / Padrao |
|----------------------|-----------------|
| `pageSize` maximo    | 100             |
| `pageSize` padrao    | 20              |
| `pageNumber` inicial | 1               |

Valores de `pageSize` acima de 100 serao rejeitados. Consulte a secao [Paginacao](./paginacao.md) para exemplos de iteracao completa.

---

## Upload de Arquivos

| Item                       | Valor                              |
|----------------------------|------------------------------------|
| Tamanho maximo por arquivo | 4 MB (configuracao padrao do servidor) |
| Content-Type               | `multipart/form-data`              |
| Extensoes permitidas       | Configuravel por formulario        |

O limite de 4 MB e a configuracao padrao do servidor. Arquivos maiores resultam em erro HTTP `413 Request Entity Too Large`.

As extensoes de arquivo bloqueadas sao definidas por formulario de catalogo. Para verificar quais extensoes sao permitidas em um servico especifico, consulte o campo `ExtensoesNaoPermitidas` ao obter o catalogo:

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
