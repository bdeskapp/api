# Exemplos Python — API BDesk

Exemplos completos de integracao com a API BDesk usando Python 3.

> **Pre-requisito:** `pip install requests`

---

## 1. Classe Helper `BDeskApi`

Classe reutilizavel que encapsula autenticacao, tratamento de erros e todas as operacoes principais da API.

```python
"""
bdesk_api.py — Cliente Python para a API BDesk

Pre-requisito: pip install requests
"""

import json
import os
import requests


class BDeskApiError(Exception):
    """Excecao para erros de negocio retornados pela API (HTTP 406)."""

    def __init__(self, mensagens):
        self.mensagens = mensagens if isinstance(mensagens, list) else [mensagens]
        super().__init__("; ".join(self.mensagens))


class BDeskApi:
    """
    Cliente para a API REST do BDesk.

    Exemplo de uso:
        api = BDeskApi("https://sua-empresa.bdesk.com.br/askrest", "usuario", "senha")
        abertas = api.listar_abertas(page_size=10)
    """

    def __init__(self, base_url: str, login: str, senha: str):
        """
        Inicializa o cliente e realiza login automaticamente.

        Args:
            base_url: URL base da API (ex: https://sua-empresa.bdesk.com.br/askrest)
            login: Nome de usuario
            senha: Senha do usuario
        """
        self.base_url = base_url.rstrip("/")
        self.token = None
        self._login(login, senha)

    # ------------------------------------------------------------------
    # Autenticacao
    # ------------------------------------------------------------------

    def _login(self, login: str, senha: str) -> None:
        """
        Autentica o usuario e armazena o token internamente.

        O campo Dados da resposta e uma string JSON escapada — nao um objeto.
        Sao necessarios dois niveis de parse para extrair o access_token.

        Raises:
            BDeskApiError: Se as credenciais forem invalidas ou a licenca estiver expirada.
            requests.HTTPError: Para erros HTTP nao-406.
        """
        url = f"{self.base_url}/v1/login/entrar"
        resp = requests.post(
            url,
            json={"Login": login, "Senha": senha},
            timeout=30,
        )

        if resp.status_code == 406:
            corpo = resp.json()
            raise BDeskApiError(corpo.get("MensagensErro", ["Credenciais invalidas."]))

        resp.raise_for_status()
        corpo = resp.json()

        # Dados e uma string JSON — nao um objeto direto
        dados = json.loads(corpo["Dados"])
        self.token = dados["access_token"]

    def _headers(self) -> dict:
        """Retorna o dicionario de headers com autenticacao Bearer."""
        return {
            "Authorization": f"Bearer {self.token}",
            "Content-Type": "application/json",
        }

    def _check_errors(self, resp: requests.Response) -> dict:
        """
        Verifica a resposta por erros de negocio (HTTP 406) e MensagensErro.

        Args:
            resp: Objeto de resposta do requests.

        Returns:
            O corpo da resposta como dicionario.

        Raises:
            BDeskApiError: Se houver mensagens de erro ou status 406.
            requests.HTTPError: Para outros erros HTTP (401, 500, etc.).
        """
        if resp.status_code == 406:
            try:
                corpo = resp.json()
                mensagens = (
                    corpo.get("_metadata", {}).get("MensagensErro")
                    or corpo.get("MensagensErro")
                    or ["Erro de negocio (HTTP 406)."]
                )
                raise BDeskApiError(mensagens)
            except (ValueError, KeyError):
                raise BDeskApiError([f"Erro HTTP 406: {resp.text}"])

        resp.raise_for_status()
        corpo = resp.json()

        # Verificar MensagensErro no envelope (alguns endpoints retornam 200 com erro)
        mensagens = (
            corpo.get("_metadata", {}).get("MensagensErro")
            or corpo.get("MensagensErro")
        )
        if mensagens:
            raise BDeskApiError(mensagens)

        return corpo

    # ------------------------------------------------------------------
    # Requisicoes
    # ------------------------------------------------------------------

    def listar_abertas(self, page: int = 1, page_size: int = 20) -> dict:
        """
        Lista requisicoes abertas do usuario autenticado com paginacao.

        Args:
            page: Numero da pagina (base 1, padrao 1).
            page_size: Itens por pagina (padrao 20, maximo 100).

        Returns:
            Dicionario com 'records' (lista) e '_metadata' (paginacao, etc.).
        """
        url = f"{self.base_url}/v1/requisicoes/abertas"
        resp = requests.get(
            url,
            headers=self._headers(),
            params={"pageNumber": page, "pageSize": page_size},
            timeout=30,
        )
        return self._check_errors(resp)

    def buscar_requisicao(self, req_id: int) -> dict:
        """
        Retorna detalhes completos de uma requisicao pelo ID.

        Args:
            req_id: ID da requisicao.

        Returns:
            Dicionario com 'Conjuntos' (dados da requisicao) e URLs de navegacao.
            Conjuntos e um dicionario cujas chaves sao nomes das secoes.
        """
        url = f"{self.base_url}/v1/requisicoes/{req_id}"
        resp = requests.get(url, headers=self._headers(), timeout=30)
        return self._check_errors(resp)

    def criar_requisicao(
        self,
        formulario_id: int,
        assunto: str,
        descricao: str,
        conjuntos: dict = None,
    ) -> int:
        """
        Abre uma nova requisicao.

        Args:
            formulario_id: ID do formulario (obtido via listar_catalogo()).
            assunto: Titulo/assunto da requisicao.
            descricao: Descricao detalhada do problema ou solicitacao.
            conjuntos: Dicionario adicional de conjuntos para o formulario.
                       Se None, usa apenas DadosBasicos com assunto e descricao.

        Returns:
            ID numerico da requisicao criada (int).

        Raises:
            BDeskApiError: Se houver erros de validacao (campos obrigatorios, etc.).
        """
        payload_conjuntos = {
            "DadosBasicos": {
                "Assunto": assunto,
                "Descricao": descricao,
            }
        }
        if conjuntos:
            payload_conjuntos.update(conjuntos)

        payload = {
            "Formulario": formulario_id,
            "Conjuntos": payload_conjuntos,
        }

        url = f"{self.base_url}/v1/requisicoes/abrir"
        resp = requests.post(url, headers=self._headers(), json=payload, timeout=30)

        if resp.status_code == 406:
            corpo = resp.json()
            raise BDeskApiError(
                corpo.get("_metadata", {}).get("MensagensErro")
                or corpo.get("MensagensErro", ["Erro ao criar requisicao."])
            )

        resp.raise_for_status()

        # /abrir retorna apenas o ID como string simples (ex: "12345")
        return int(resp.json())

    def executar_acao(
        self,
        req_id: int,
        acao_id: str,
        descricao: str = "",
        **kwargs,
    ) -> dict:
        """
        Executa uma acao de workflow em uma requisicao.

        Args:
            req_id: ID da requisicao.
            acao_id: Identificador da acao no formato "Nome [CODIGO]"
                     (ex: "Encerrar [ENC]", "Direcionar [DIR]").
                     Use listar_acoes() para obter os IDs disponiveis.
            descricao: Comentario/descricao da acao.
            **kwargs: Parametros adicionais da acao:
                - Motivo (int): ID do motivo (usado com ENC)
                - prioridade (int): Nova prioridade (usado com ALTPRI)
                - NovoSolicitado (str): ID do novo solicitado (usado com DIR)
                - GrupoId (int): ID do novo grupo (usado com DIR)
                - IdRequisicaoAVincular (int): ID a vincular (usado com VINC)
                - DescricaoRequisicao (str): Nova descricao (usado com ALTDES)
                - AssuntoRequisicao (str): Novo assunto (usado com ALTDES)

        Returns:
            Dicionario com a resposta da API.

        Raises:
            BDeskApiError: Se a acao nao for permitida ou os dados estiverem incompletos.
        """
        payload = {"Id": acao_id, "Descricao": descricao}
        payload.update(kwargs)

        url = f"{self.base_url}/v1/requisicoes/{req_id}/acoes"
        resp = requests.post(url, headers=self._headers(), json=payload, timeout=30)
        return self._check_errors(resp)

    def listar_acoes(self, req_id: int) -> list:
        """
        Lista as acoes disponiveis para o usuario autenticado na requisicao.

        Args:
            req_id: ID da requisicao.

        Returns:
            Lista de dicionarios com 'Nome', 'Id' e 'Campos' de cada acao.
        """
        url = f"{self.base_url}/v1/requisicoes/{req_id}/acoes"
        resp = requests.get(url, headers=self._headers(), timeout=30)
        corpo = self._check_errors(resp)
        return corpo.get("records", [])

    def enviar_anexo(self, req_id: int, caminho_arquivo: str) -> dict:
        """
        Faz upload de um arquivo como anexo de uma requisicao.

        Args:
            req_id: ID da requisicao.
            caminho_arquivo: Caminho absoluto ou relativo do arquivo a enviar.

        Returns:
            Dicionario com 'Id' (ID do anexo criado) e 'MensagensErro'.

        Raises:
            FileNotFoundError: Se o arquivo nao for encontrado.
            BDeskApiError: Se o tipo de arquivo nao for permitido.
        """
        if not os.path.isfile(caminho_arquivo):
            raise FileNotFoundError(f"Arquivo nao encontrado: {caminho_arquivo}")

        url = f"{self.base_url}/v1/requisicoes/{req_id}/anexo"

        # Para upload multipart, nao incluir Content-Type no header (requests define automaticamente)
        headers = {"Authorization": f"Bearer {self.token}"}

        with open(caminho_arquivo, "rb") as f:
            nome_arquivo = os.path.basename(caminho_arquivo)
            resp = requests.post(
                url,
                headers=headers,
                files={"file": (nome_arquivo, f)},
                timeout=60,
            )

        return self._check_errors(resp)

    # ------------------------------------------------------------------
    # Catalogo
    # ------------------------------------------------------------------

    def listar_catalogo(self) -> list:
        """
        Retorna todas as areas e formularios do catalogo de servicos.

        Returns:
            Lista de areas, cada uma contendo 'Id', 'Nome' e 'Formularios'.
        """
        url = f"{self.base_url}/v1/cardapio"
        resp = requests.get(url, headers=self._headers(), timeout=30)
        resp.raise_for_status()
        corpo = resp.json()
        return corpo.get("records", [])

    def buscar_formulario(self, formulario_id: int) -> dict:
        """
        Retorna os conjuntos e campos de um formulario especifico.

        Args:
            formulario_id: ID do formulario (obtido via listar_catalogo()).

        Returns:
            Dicionario com 'Nome', 'Versao', 'Conjuntos' e URLs de navegacao.
        """
        url = f"{self.base_url}/v1/cardapio/formularios/{formulario_id}"
        resp = requests.get(url, headers=self._headers(), timeout=30)
        resp.raise_for_status()
        return resp.json()

    def pesquisar_participantes(self, formulario_papel_id: int, termo: str) -> list:
        """
        Pesquisa participantes por formulario-papel e termo de busca.

        Args:
            formulario_papel_id: ID do formulario-papel (do campo de participante no formulario).
            termo: Texto para pesquisa (nome parcial do participante).

        Returns:
            Lista de participantes com 'Id' (JSON serializado) e 'Texto' (nome).
            O campo 'Id' contem {"IdParticipante": N, "IdTipoPapel": N}.
        """
        url = f"{self.base_url}/v1/participantes/{formulario_papel_id}/pesquisar/{termo}"
        resp = requests.get(url, headers=self._headers(), timeout=30)
        resp.raise_for_status()
        # Retorna array direto (sem envelope _metadata)
        resultado = resp.json()
        # Fazer parse do campo Id (JSON serializado) para facilitar uso
        for item in resultado:
            try:
                item["_id_parsed"] = json.loads(item["Id"])
            except (ValueError, TypeError):
                item["_id_parsed"] = {}
        return resultado
```

---

## 2. Exemplos de Uso

### Configuracao e Login

```python
from bdesk_api import BDeskApi, BDeskApiError

# O login e feito automaticamente no construtor
api = BDeskApi(
    base_url="https://sua-empresa.bdesk.com.br/askrest",
    login="seu-usuario",
    senha="sua-senha",
)
print("Autenticado com sucesso.")
```

### Listar Requisicoes Abertas

```python
# Listar as 10 primeiras requisicoes abertas
abertas = api.listar_abertas(page_size=10)

paginacao = abertas.get("_metadata", {}).get("Pagination", {})
print(f"Total: {paginacao.get('TotalRecords', '?')} requisicoes abertas")
print(f"Pagina {paginacao.get('CurrentPage', 1)} de {paginacao.get('TotalPages', 1)}")
print()

for req in abertas["records"]:
    print(f"#{req['RequisicaoId']} - {req['Assunto']}")
    print(f"  Status: {req['Status']} | Responsavel: {req.get('Responsavel', '-')}")
    print(f"  Abertura: {req['DataAbertura'][:10]}")
```

### Buscar Detalhes de uma Requisicao

```python
detalhes = api.buscar_requisicao(35174)

# Conjuntos e um dicionario (diferente do catalogo, onde e um array)
conjuntos = detalhes.get("Conjuntos", {})
info = conjuntos.get("Detalhes Do Pedido", {})

print(f"Assunto  : {info.get('Assunto', '-')}")
print(f"Status   : {info.get('Status', '-')}")
print(f"Descricao: {info.get('Descricao', '-')}")

participantes = conjuntos.get("Participantes", {})
for papel, nome in participantes.items():
    print(f"  {papel}: {nome}")
```

### Criar Requisicao

```python
try:
    req_id = api.criar_requisicao(
        formulario_id=101,
        assunto="Impressora nao liga",
        descricao="A impressora do setor financeiro nao liga desde esta manha.",
    )
    print(f"Requisicao criada: #{req_id}")
except BDeskApiError as e:
    print(f"Erro ao criar requisicao: {e}")
    for msg in e.mensagens:
        print(f"  - {msg}")
```

**Com conjuntos adicionais (formularios multi-secao):**

```python
req_id = api.criar_requisicao(
    formulario_id=318,
    assunto="Compra de equipamentos",
    descricao="Solicitacao de compra para o departamento de TI.",
    conjuntos={
        # Conjuntos de linhas multiplas (Multiplo: true)
        "Itens": [
            {"Produto": "Teclado", "Quantidade": 2},
            {"Produto": "Mouse", "Quantidade": 5},
        ]
    },
)
print(f"Requisicao de compra criada: #{req_id}")
```

### Listar e Executar Acoes

```python
# Ver quais acoes estao disponiveis
acoes = api.listar_acoes(req_id)
print("Acoes disponiveis:")
for acao in acoes:
    print(f"  {acao['Id']}")

# Encerrar a requisicao
try:
    api.executar_acao(
        req_id=req_id,
        acao_id="Encerrar [ENC]",
        descricao="Problema resolvido. Equipamento substituido.",
        Motivo=1,
    )
    print(f"Requisicao #{req_id} encerrada.")
except BDeskApiError as e:
    print(f"Nao foi possivel encerrar: {e}")
```

**Outros exemplos de acoes:**

```python
# Direcionar para outro grupo
api.executar_acao(
    req_id=req_id,
    acao_id="Direcionar [DIR]",
    descricao="Direcionando para equipe de infraestrutura.",
    GrupoId=10,
)

# Alterar prioridade
api.executar_acao(
    req_id=req_id,
    acao_id="Alterar Prioridade [ALTPRI]",
    descricao="Impacto em producao — prioridade elevada.",
    prioridade=1,
)

# Vincular requisicoes
api.executar_acao(
    req_id=req_id,
    acao_id="Vincular [VINC]",
    descricao="",
    IdRequisicaoAVincular=12300,
)
```

### Upload de Anexo

```python
try:
    resultado = api.enviar_anexo(
        req_id=req_id,
        caminho_arquivo="/caminho/para/relatorio.pdf",
    )
    print(f"Anexo enviado. ID do anexo: {resultado['Id']}")
except FileNotFoundError as e:
    print(f"Arquivo nao encontrado: {e}")
except BDeskApiError as e:
    print(f"Tipo de arquivo nao permitido: {e}")
```

### Explorar o Catalogo

```python
# Listar todas as areas e formularios
areas = api.listar_catalogo()
for area in areas:
    print(f"[{area['Id']}] {area['Nome']}")
    for frm in area.get("Formularios", []):
        print(f"  Formulario {frm['Id']}: {frm['Nome']}")

# Ver campos de um formulario especifico
formulario = api.buscar_formulario(101)
print(f"\nFormulario: {formulario['Nome']} (v{formulario['Versao']})")
for conj in formulario.get("Conjuntos", []):
    print(f"\n  Conjunto: {conj['Nome']} (chave: {conj['Chave']})")
    for campo in conj.get("Campos", []):
        obrig = "[obrig]" if campo.get("Obrigatoriedade") else ""
        print(f"    {campo['Chave']:30s} {campo['TipoDeDado']:15s} {obrig}")
```

### Pesquisar Participantes

```python
# Util para preencher campos de participante em formularios
participantes = api.pesquisar_participantes(
    formulario_papel_id=42,
    termo="silva",
)

for p in participantes:
    id_info = p.get("_id_parsed", {})
    print(f"{p['Texto']} — IdParticipante: {id_info.get('IdParticipante', '?')}")
```

### Tratamento de Erros

```python
from bdesk_api import BDeskApi, BDeskApiError
import requests

try:
    api = BDeskApi("https://sua-empresa.bdesk.com.br/askrest", "usuario", "senha123")
    abertas = api.listar_abertas()

except BDeskApiError as e:
    # Erros de negocio: credenciais invalidas, acao nao permitida, campos obrigatorios, etc.
    print(f"Erro de negocio: {e}")
    for msg in e.mensagens:
        print(f"  - {msg}")

except requests.exceptions.ConnectionError:
    print("Nao foi possivel conectar a API. Verifique a URL e a conectividade.")

except requests.exceptions.Timeout:
    print("Timeout na requisicao. Tente novamente.")

except requests.exceptions.HTTPError as e:
    print(f"Erro HTTP inesperado: {e.response.status_code} — {e.response.text}")
```

---

## Referencia Rapida

| Metodo | Endpoint | Descricao |
|--------|----------|-----------|
| `listar_abertas(page, page_size)` | `GET /v1/requisicoes/abertas` | Lista requisicoes abertas paginadas |
| `buscar_requisicao(req_id)` | `GET /v1/requisicoes/{id}` | Detalhes de uma requisicao |
| `criar_requisicao(...)` | `POST /v1/requisicoes/abrir` | Abre nova requisicao; retorna ID (int) |
| `listar_acoes(req_id)` | `GET /v1/requisicoes/{id}/acoes` | Lista acoes disponiveis |
| `executar_acao(req_id, acao_id, ...)` | `POST /v1/requisicoes/{id}/acoes` | Executa acao de workflow |
| `enviar_anexo(req_id, caminho)` | `POST /v1/requisicoes/{id}/anexo` | Upload de arquivo |
| `listar_catalogo()` | `GET /v1/cardapio` | Areas e formularios do catalogo |
| `buscar_formulario(id)` | `GET /v1/cardapio/formularios/{id}` | Campos de um formulario |
| `pesquisar_participantes(fp_id, termo)` | `GET /v1/participantes/{id}/pesquisar/{termo}` | Busca participantes |
