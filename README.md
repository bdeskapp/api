# API BDesk — Guia do Usuario

Documentacao da API REST do [BDesk](https://herocorp.bdesk.com.br), uma plataforma ITSM/service-desk empresarial. Este repositorio contem o site de documentacao publicado em **[bdeskapp.github.io/api](https://bdeskapp.github.io/api)**.

## Conteudo

- **Primeiros Passos** — Login, obtencao de token e primeira chamada
- **Guias** — Criar/consultar requisicoes, workflow, anexos, catalogo de servicos, ICs, monitoramento
- **Referencia** — Erros, paginacao, limites, glossario
- **Exemplos de Codigo** — cURL, Python, PowerShell
- **Referencia de Endpoints** — Todos os 72 endpoints documentados via [Redoc](https://redocly.com/redoc)
- **OpenAPI 3.1** — Especificacao completa em `docs/openapi.yaml`
- **Colecao Postman** — Pronta para importar em `docs/postman_collection.json`

## Desenvolvimento Local

Requisitos: Python 3.12+

```bash
# Instalar dependencias
pip install mkdocs-material

# Iniciar servidor local com live reload (http://127.0.0.1:8000)
mkdocs serve

# Build de producao (mesmo comando usado no CI)
mkdocs build --strict
```

## Deploy

O deploy e automatico. Todo push na branch `main` dispara o workflow [deploy-docs.yml](.github/workflows/deploy-docs.yml), que faz build com `mkdocs build --strict` e publica no GitHub Pages.

## Estrutura do Repositorio

```
mkdocs.yml                    # Configuracao do site (nav, tema, extensoes)
docs/
  index.md                    # Pagina inicial
  primeiros-passos.md         # Guia de inicio rapido
  autenticacao.md             # Autenticacao e tokens
  guias/                      # Guias praticos (8 guias)
  referencia/                 # Referencia tecnica (erros, paginacao, limites, glossario)
  exemplos/                   # Exemplos de codigo (cURL, Python, PowerShell)
  openapi.yaml                # Especificacao OpenAPI 3.1.0
  postman_collection.json     # Colecao Postman
  redoc.html                  # Visualizacao interativa dos endpoints (Redoc)
```

## Contribuindo

1. Edite ou crie arquivos `.md` dentro de `docs/`
2. Se adicionar ou remover paginas, atualize o `nav` em `mkdocs.yml`
3. Valide localmente com `mkdocs build --strict` antes de fazer push
4. Abra um Pull Request para `main`

> **Nota:** O arquivo `docs/redoc.html` carrega `openapi.yaml` por caminho relativo. Se o arquivo OpenAPI for renomeado ou movido, a pagina Redoc quebrara sem aviso no build.

## Licenca

Uso interno — documentacao da API BDesk.
