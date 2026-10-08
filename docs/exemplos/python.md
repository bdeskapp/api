# Exemplos Python — API BDesk

Exemplos completos de integracao com a API BDesk usando Python 3.

> **Pre-requisito:** `pip install requests`

> **Sobre erros:** a API sinaliza falhas de duas formas: HTTP 406 com a mensagem em **texto puro**, ou HTTP 200 com a mensagem em `MensagensErro` (no envelope `_metadata`, ou na raiz da resposta no login e em `GET /v1/requisicoes/{id}`). A classe abaixo trata os dois casos. Veja [Tratamento de Erros](../referencia/erros.md).

---

## 1. Classe Helper `BDeskApi`

Classe reutilizavel que encapsula autenticacao, tratamento de erros e as operacoes principais da API.

```python
"""
bdesk_api.py — Cliente Python para a API BDesk

Pre-requisito: pip install requests
"""

import json
import os
import requests


class BDeskApiError(Exception):
    """Excecao para erros de negocio retornados pela API.

    Cobre os dois padroes da API: HTTP 406 (mensagem em texto puro) e
    HTTP 200 com a mensagem em MensagensErro.
    """

    def __init__(self, mensagens):
        self.mensagens = mensagens if isinstance(mensagens, list) else [mensagens]
        super().__init__("; ".join(str(m) for m in self.mensagens))


class BDeskApi:
    """
    Cliente para a API REST do BDesk.

    Exemplo de uso:
        api = BDeskApi("https://sua-empresa.bdesk.com.br/askrest", "usuario", "senha")
        abertas = api.listar_abertas(limite=10)
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

        O login responde HTTP 200 mesmo quando falha: nesse caso Dados vem
        nulo e a mensagem fica em MensagensErro (na raiz da resposta).
        Quando funciona, Dados e uma string JSON escapada — nao um objeto —
        e sao necessarios dois niveis de parse para extrair o access_token.

        O token nao expira por tempo no servidor (expires_in e apenas
        informativo): guarde-o e reutilize-o. Se uma chamada futura responder
        401, faca login novamente.

        Raises:
            BDeskApiError: Se as credenciais forem invalidas ou o login nao for permitido.
            requests.HTTPError: Para erros HTTP.
        """
        url = f"{self.base_url}/v1/login/entrar"
        resp = requests.post(
            url,
            json={"Login": login, "Senha": senha},
            timeout=30,
        )
        resp.raise_for_status()
        corpo = resp.json()

        if corpo.get("Dados") is None or corpo.get("MensagensErro"):
            raise BDeskApiError(corpo.get("MensagensErro") or ["Falha no login."])

        # Dados e uma string JSON — nao um objeto direto
        dados = json.loads(corpo["Dados"])
        self.token = dados["access_token"]

    def _headers(self) -> dict:
        """Retorna o dicionario de headers com autenticacao Bearer."""
        return {
            "Authorization": f"Bearer {self.token}",
            "Content-Type": "application/json",
        }

    def _checar(self, resp: requests.Response):
        """
        Verifica a resposta pelos dois padroes de erro da API.

        - HTTP 406: a mensagem vem em texto puro no corpo (nao e JSON).
        - HTTP 200 com MensagensErro: em _metadata.MensagensErro (envelope padrao)
          ou na raiz da resposta (login, detalhes da requisicao, upload).
        - Outros erros HTTP (401, 404, 500): levantados por raise_for_status().

        Returns:
            O corpo da resposta ja convertido de JSON (dict, lista ou texto).

        Raises:
            BDeskApiError: Em erro de negocio (406, ou 200 com MensagensErro).
            requests.HTTPError: Para os demais erros HTTP.
        """
        if resp.status_code == 406:
            raise BDeskApiError(resp.text.strip().splitlines() or ["Erro de negocio (HTTP 406)."])

        resp.raise_for_status()
        corpo = resp.json()

        if isinstance(corpo, dict):
            mensagens = (corpo.get("_metadata") or {}).get("MensagensErro") or corpo.get("MensagensErro")
            if mensagens:
                raise BDeskApiError(mensagens)

        return corpo

    # ------------------------------------------------------------------
    # Requisicoes
    # ------------------------------------------------------------------

    def listar_abertas(self, limite: int = 500, filtros: dict = None) -> dict:
        """
        Lista requisicoes abertas do usuario autenticado.

        A API NAO pagina: a resposta traz tudo ate o limite informado
        (LimiteRequisicoes, padrao 500). Para reduzir o volume, use filtros.

        Args:
            limite: Maximo de requisicoes devolvidas (LimiteRequisicoes).
            filtros: Filtros opcionais, enviados no corpo de um POST.
                     Ex.: {"DescricoesStatus": ["Aberta"],
                           "AbertoEntre": {"Inicio": "2026-01-01T00:00:00",
                                           "Fim": "2026-01-31T23:59:59"}}

        Returns:
            Dicionario com 'records' (lista) e '_metadata'.
        """
        url = f"{self.base_url}/v1/requisicoes/abertas"
        if filtros:
            corpo = dict(filtros, LimiteRequisicoes=limite)
            resp = requests.post(url, headers=self._headers(), json=corpo, timeout=30)
        else:
            resp = requests.get(
                url,
                headers=self._headers(),
                params={"LimiteRequisicoes": limite},
                timeout=30,
            )
        return self._checar(resp)

    def buscar_requisicao(self, req_id: int) -> dict:
        """
        Retorna detalhes completos de uma requisicao pelo ID.

        Requisicao inexistente ou sem acesso volta com HTTP 200, Conjuntos nulo
        e MensagensErro preenchido; _checar() converte isso em BDeskApiError.

        Returns:
            Dicionario com 'Conjuntos' (dados da requisicao) e URLs de navegacao.
            Conjuntos e um dicionario cujas chaves sao nomes das secoes.
        """
        url = f"{self.base_url}/v1/requisicoes/{req_id}"
        resp = requests.get(url, headers=self._headers(), timeout=30)
        return self._checar(resp)

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
            Numero da requisicao criada (int).

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

        # /abrir devolve apenas o numero como texto JSON (ex: "12345")
        return int(self._checar(resp))

    def executar_acao(
        self,
        req_id: int,
        acao_id: str,
        descricao: str = "",
        **kwargs,
    ):
        """
        Executa uma acao de workflow em uma requisicao.

        Args:
            req_id: ID da requisicao.
            acao_id: Identificador EXATO da acao, como devolvido por listar_acoes()
                     (ex: "Encerrar [ENC]", "Direcionar [DIR]"). O codigo entre
                     colchetes e obrigatorio: "ENC" sozinho resulta em
                     "Acao nao encontrada".
            descricao: Comentario/descricao da acao.
            **kwargs: Parametros adicionais da acao:
                - tipoAvaliacao (int): 1, 2 ou 3 (usado com ENC e AVAL)
                - prioridade (int): Nova prioridade (usado com ALTPRI)
                - NovoSolicitado (str): Id do destino, copiado de listar_destinos_direcionar() (usado com DIR)
                - usuResponsavelId (int): ID do usuario (usado com ATR e ATRR)
                - IdRequisicaoAVincular (int): ID a vincular (usado com VINC)
                - DescricaoRequisicao / AssuntoRequisicao (str): Novos textos (usado com ALTDES)

        Returns:
            O corpo da resposta (envelope com records nulo, ou texto simples).

        Raises:
            BDeskApiError: Se a acao nao for permitida ou os dados estiverem incompletos.
                           Atencao: a maioria dos erros desta rota volta com HTTP 200
                           e a mensagem em _metadata.MensagensErro; _checar() cobre isso.
        """
        payload = {"Id": acao_id, "Descricao": descricao}
        payload.update(kwargs)

        url = f"{self.base_url}/v1/requisicoes/{req_id}/acoes"
        resp = requests.post(url, headers=self._headers(), json=payload, timeout=30)
        return self._checar(resp)

    def listar_acoes(self, req_id: int) -> list:
        """
        Lista as acoes disponiveis para o usuario autenticado na requisicao.

        Returns:
            Lista de dicionarios com 'Nome', 'Id', 'CodigoAcao' e 'Campos' de cada acao.
        """
        url = f"{self.base_url}/v1/requisicoes/{req_id}/acoes"
        resp = requests.get(url, headers=self._headers(), timeout=30)
        corpo = self._checar(resp)
        return corpo.get("records", [])

    def listar_destinos_direcionar(self, req_id: int) -> list:
        """
        Lista os grupos/usuarios que podem receber a requisicao na acao DIR.

        Returns:
            Lista de {'Id': '<texto pronto>', 'Texto': '<nome>'}. Use o 'Id',
            sem alterar, em executar_acao(..., NovoSolicitado=<Id>).
        """
        url = f"{self.base_url}/v1/requisicoes/{req_id}/acoes/DIR/grupos"
        resp = requests.get(url, headers=self._headers(), timeout=30)
        corpo = self._checar(resp)
        return corpo.get("records", [])

    def enviar_arquivo(self, req_id: int, caminho_arquivo: str) -> str:
        """
        ETAPA 1 do anexo: faz upload do arquivo para a area temporaria.

        ATENCAO: isto NAO anexa o arquivo a requisicao. A resposta e 200 mesmo
        assim. Depois do upload, chame submeter_anexos() (etapa 2) com o GUID.

        Args:
            req_id: ID da requisicao.
            caminho_arquivo: Caminho do arquivo a enviar.

        Returns:
            GUID (texto) que identifica o arquivo na area temporaria.

        Raises:
            FileNotFoundError: Se o arquivo nao for encontrado.
            BDeskApiError: Se o upload falhar (Id vazio ou MensagensErro preenchido).
        """
        if not os.path.isfile(caminho_arquivo):
            raise FileNotFoundError(f"Arquivo nao encontrado: {caminho_arquivo}")

        url = f"{self.base_url}/v1/requisicoes/{req_id}/anexo"

        # Upload multipart: nao incluir Content-Type (requests define sozinho)
        headers = {"Authorization": f"Bearer {self.token}"}

        with open(caminho_arquivo, "rb") as f:
            resp = requests.post(
                url,
                headers=headers,
                files={"file": (os.path.basename(caminho_arquivo), f)},
                timeout=60,
            )

        corpo = self._checar(resp)  # levanta erro se MensagensErro vier preenchido
        if not corpo.get("Id"):
            raise BDeskApiError(["O upload nao devolveu o identificador do arquivo."])
        return corpo["Id"]

    def submeter_anexos(self, req_id: int, anexos: list, codigo_acao: str = "ANDOC"):
        """
        ETAPA 2 do anexo: vincula os arquivos ja enviados a requisicao.

        Args:
            req_id: ID da requisicao.
            anexos: Lista de dicionarios {"Id": <GUID da etapa 1>,
                    "NomeDuranteUpload": "relatorio.pdf" (COM extensao),
                    "Titulo": "Relatorio de marco"}.
            codigo_acao: Codigo da acao "anexar documento" (normalmente "ANDOC").

        Raises:
            BDeskApiError: Em HTTP 406 (extensao nao permitida, usuario sem permissao
                           de anexar neste status etc.) ou 200 com MensagensErro.
        """
        url = f"{self.base_url}/v1/requisicoes/{req_id}/anexos/submeter"
        payload = {"CodigoAcao": codigo_acao, "Anexos": anexos}
        resp = requests.post(url, headers=self._headers(), json=payload, timeout=60)
        return self._checar(resp)

    def enviar_anexo(self, req_id: int, caminho_arquivo: str, titulo: str = None) -> None:
        """
        Anexa um arquivo a uma requisicao (upload + submissao, as 2 etapas).

        Depois de chamar, confira com listar_anexos() que o documento apareceu.
        """
        nome = os.path.basename(caminho_arquivo)
        guid = self.enviar_arquivo(req_id, caminho_arquivo)
        self.submeter_anexos(
            req_id,
            [{"Id": guid, "NomeDuranteUpload": nome, "Titulo": titulo or nome}],
        )

    def listar_anexos(self, req_id: int) -> list:
        """Lista os anexos ja vinculados a requisicao (rota no plural: /anexos)."""
        url = f"{self.base_url}/v1/requisicoes/{req_id}/anexos"
        resp = requests.get(url, headers=self._headers(), timeout=30)
        return self._checar(resp).get("records", [])

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
        return self._checar(resp).get("records", [])

    def buscar_formulario(self, formulario_id: int) -> dict:
        """
        Retorna os conjuntos e campos de um formulario especifico.

        Formulario inexistente ou nao permitido responde HTTP 403 (texto puro);
        isso levanta requests.HTTPError.

        Returns:
            Dicionario com 'Nome', 'Versao', 'Conjuntos' e URLs de navegacao.
        """
        url = f"{self.base_url}/v1/cardapio/formularios/{formulario_id}"
        resp = requests.get(url, headers=self._headers(), timeout=30)
        return self._checar(resp)

    def pesquisar_participantes(self, formulario_papel_id: int, termo: str) -> list:
        """
        Pesquisa participantes por formulario-papel e termo de busca.

        Returns:
            Lista de participantes com 'Id' (JSON serializado) e 'Texto' (nome).
            O campo 'Id' contem {"IdParticipante": N, "IdTipoPapel": N}.
        """
        url = f"{self.base_url}/v1/participantes/{formulario_papel_id}/pesquisar/{termo}"
        resp = requests.get(url, headers=self._headers(), timeout=30)
        resp.raise_for_status()
        # Retorna array direto (sem envelope _metadata)
        resultado = resp.json()
        # Parse do campo Id (JSON serializado) para facilitar o uso
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
# A API nao pagina: informe um limite (padrao 500) e use filtros para reduzir o volume
abertas = api.listar_abertas(limite=10)

registros = abertas["records"]
print(f"{len(registros)} requisicoes abertas")
if len(registros) >= 10:
    print("Atencao: o limite foi atingido; pode haver mais requisicoes.")
print()

for req in registros:
    print(f"#{req['RequisicaoId']} - {req['Assunto']}")
    print(f"  Status: {req['Status']} | Responsavel: {req.get('Responsavel', '-')}")
    print(f"  Abertura: {req['DataAbertura'][:10]}")
```

**Com filtros (periodo de abertura e status):**

```python
janeiro = api.listar_abertas(
    limite=200,
    filtros={
        "DescricoesStatus": ["Aberta"],
        "AbertoEntre": {"Inicio": "2026-01-01T00:00:00", "Fim": "2026-01-31T23:59:59"},
    },
)
print(len(janeiro["records"]))
```

### Buscar Detalhes de uma Requisicao

```python
try:
    detalhes = api.buscar_requisicao(35174)
except BDeskApiError as e:
    # Requisicao inexistente ou sem acesso: HTTP 200 com MensagensErro
    print(f"Nao foi possivel consultar: {e}")
else:
    # Conjuntos e um dicionario (diferente do catalogo, onde e um array)
    conjuntos = detalhes["Conjuntos"]
    info = conjuntos.get("Detalhes Do Pedido", {})

    print(f"Assunto  : {info.get('Assunto', '-')}")
    print(f"Status   : {info.get('Status', '-')}")
    print(f"Descricao: {info.get('Descricao', '-')}")

    for papel, nome in conjuntos.get("Participantes", {}).items():
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
        # Conjuntos de linhas multiplas (Multiplo: true) recebem uma lista de objetos
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
# Ver quais acoes estao disponiveis (o Id inclui o codigo entre colchetes)
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
        tipoAvaliacao=2,
    )
    print(f"Requisicao #{req_id} encerrada.")
except BDeskApiError as e:
    # Cobre HTTP 406 e tambem HTTP 200 com MensagensErro
    print(f"Nao foi possivel encerrar: {e}")
```

**Outros exemplos de acoes:**

```python
# Direcionar: o Id do destino vem de acoes/DIR/grupos e vai, sem alteracao, em NovoSolicitado
destinos = api.listar_destinos_direcionar(req_id)
api.executar_acao(
    req_id=req_id,
    acao_id="Direcionar [DIR]",
    descricao="Direcionando para equipe de infraestrutura.",
    NovoSolicitado=destinos[0]["Id"],
)

# Alterar prioridade (1 = Alta, 2 = Media, 3 = Baixa)
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

### Anexar Arquivo (2 etapas)

O envio de um anexo tem duas chamadas: o **upload** apenas guarda o arquivo numa area temporaria; a **submissao** e que o vincula a requisicao. Parar no upload deixa a requisicao sem anexo, mesmo com resposta 200.

```python
try:
    # Atalho: faz as duas etapas
    api.enviar_anexo(req_id, "/caminho/para/relatorio.pdf", titulo="Relatorio de marco")

    # Confira que o documento apareceu na lista
    for anexo in api.listar_anexos(req_id):
        print(f"{anexo['Id']}: {anexo['Titulo']} ({anexo['NomeDocumentoFisico']})")
except FileNotFoundError as e:
    print(f"Arquivo nao encontrado: {e}")
except BDeskApiError as e:
    # Ex.: extensao nao permitida, usuario sem permissao de anexar neste status
    print(f"Anexo nao vinculado: {e}")
```

**Passo a passo, com varios arquivos numa unica submissao:**

```python
arquivos = ["/caminho/para/a.pdf", "/caminho/para/b.png"]

itens = []
for caminho in arquivos:
    guid = api.enviar_arquivo(req_id, caminho)            # etapa 1 (um upload por arquivo)
    nome = os.path.basename(caminho)
    itens.append({"Id": guid, "NomeDuranteUpload": nome, "Titulo": nome})

api.submeter_anexos(req_id, itens)                        # etapa 2 (uma chamada para todos)
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
    # Erros de negocio: login recusado, acao nao permitida, campos obrigatorios etc.
    # Cobre HTTP 406 (texto puro) e HTTP 200 com MensagensErro.
    print(f"Erro de negocio: {e}")
    for msg in e.mensagens:
        print(f"  - {msg}")

except requests.exceptions.ConnectionError:
    print("Nao foi possivel conectar a API. Verifique a URL e a conectividade.")

except requests.exceptions.Timeout:
    print("Timeout na requisicao. Tente novamente.")

except requests.exceptions.HTTPError as e:
    # 401: token invalido ou usuario desativado (refaca o login)
    # 404: rota inexistente ou parametro de query obrigatorio ausente
    print(f"Erro HTTP inesperado: {e.response.status_code} — {e.response.text}")
```

---

## Referencia Rapida

| Metodo | Endpoint | Descricao |
|--------|----------|-----------|
| `listar_abertas(limite, filtros)` | `GET` ou `POST /v1/requisicoes/abertas` | Lista requisicoes abertas (sem paginacao; limite padrao 500) |
| `buscar_requisicao(req_id)` | `GET /v1/requisicoes/{id}` | Detalhes de uma requisicao |
| `criar_requisicao(...)` | `POST /v1/requisicoes/abrir` | Abre nova requisicao; retorna o numero (int) |
| `listar_acoes(req_id)` | `GET /v1/requisicoes/{id}/acoes` | Lista acoes disponiveis |
| `listar_destinos_direcionar(req_id)` | `GET /v1/requisicoes/{id}/acoes/DIR/grupos` | Destinos para a acao DIR |
| `executar_acao(req_id, acao_id, ...)` | `POST /v1/requisicoes/{id}/acoes` | Executa acao de workflow |
| `enviar_arquivo(req_id, caminho)` | `POST /v1/requisicoes/{id}/anexo` | Anexo, etapa 1: upload (area temporaria) |
| `submeter_anexos(req_id, anexos)` | `POST /v1/requisicoes/{id}/anexos/submeter` | Anexo, etapa 2: vincula a requisicao |
| `enviar_anexo(req_id, caminho)` | as duas rotas acima | Faz as 2 etapas |
| `listar_anexos(req_id)` | `GET /v1/requisicoes/{id}/anexos` | Anexos ja vinculados |
| `listar_catalogo()` | `GET /v1/cardapio` | Areas e formularios do catalogo |
| `buscar_formulario(id)` | `GET /v1/cardapio/formularios/{id}` | Campos de um formulario |
| `pesquisar_participantes(fp_id, termo)` | `GET /v1/participantes/{id}/pesquisar/{termo}` | Busca participantes |
