# Guia do Usuario — API BDesk

A API REST do BDesk permite automatizar operacoes da plataforma diretamente dos seus sistemas: abrir requisicoes, consultar chamados em andamento, executar acoes de fluxo de trabalho, gerenciar anexos e muito mais. Todas as chamadas sao feitas via HTTP para o endereco base `https://sua-empresa.bdesk.com.br/askrest`, retornando respostas em JSON.

---

## O que voce quer fazer?

| Objetivo | Documento |
|---|---|
| Comecar a usar a API | [primeiros-passos.md](primeiros-passos.md) |
| Entender a autenticacao | [autenticacao.md](autenticacao.md) |
| Criar requisicoes automaticamente | [guias/criar-requisicoes.md](guias/criar-requisicoes.md) |
| Consultar requisicoes | [guias/consultar-requisicoes.md](guias/consultar-requisicoes.md) |
| Executar acoes em requisicoes | [guias/acoes-workflow.md](guias/acoes-workflow.md) |
| Gerenciar anexos | [guias/anexos.md](guias/anexos.md) |
| Navegar o catalogo de servicos | [guias/catalogo-servicos.md](guias/catalogo-servicos.md) |
| Integrar com ferramentas de monitoramento | [guias/integracoes-monitoramento.md](guias/integracoes-monitoramento.md) |
| Gerenciar itens de configuracao | [guias/gestao-ics.md](guias/gestao-ics.md) |
| Buscar participantes | [guias/participantes.md](guias/participantes.md) |

---

## Referencia

- [referencia/erros.md](referencia/erros.md) — Codigos de erro e troubleshooting
- [referencia/paginacao.md](referencia/paginacao.md) — Paginacao
- [referencia/glossario.md](referencia/glossario.md) — Glossario de termos BDesk
- [referencia/limites.md](referencia/limites.md) — Limites da API

---

## Exemplos de Codigo

- [exemplos/curl.md](exemplos/curl.md) — cURL
- [exemplos/python.md](exemplos/python.md) — Python
- [exemplos/powershell.md](exemplos/powershell.md) — PowerShell

---

## Referencia Completa de Endpoints

Consulte a [Referencia de Endpoints](referencia/endpoints.md) para navegar todos os 72 endpoints com exemplos de requisicao e resposta.

---

## Recursos Adicionais

A especificacao completa da API esta disponivel no arquivo `openapi.yaml` (formato OpenAPI 3.1.0), que pode ser importado em ferramentas como Swagger UI ou Insomnia. Uma colecao Postman pronta para uso tambem esta disponivel em `postman_collection.json` — importe-a no Postman para ter todos os endpoints pre-configurados com exemplos de requisicao.
