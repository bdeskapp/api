# Integracoes de Monitoramento

## O que voce vai aprender

Como integrar ferramentas de monitoramento — como Zabbix, PRTG, Nagios e similares — com a API BDesk
para criar e atualizar requisicoes automaticamente a partir de alertas de infraestrutura.

Ao final deste guia voce sera capaz de:

- Escolher o endpoint de abertura adequado para cada ferramenta de monitoramento.
- Criar requisicoes automaticamente quando um alerta for disparado.
- Atualizar uma requisicao existente quando o alerta for atualizado ou resolvido.
- Implementar scripts Python e PowerShell com tratamento de erros.

---

## Pre-requisitos

Antes de comecar, voce precisa ter:

- **Token de autenticacao valido** — veja [Autenticacao](../autenticacao.md) para obter o seu token.
- **Variante Zabbix configurada na sua instancia** (para o Metodo 1) ou o **ID do formulario** que sera
  usado para os alertas (para o Metodo 2) — veja [Catalogo de Servicos](catalogo-servicos.md) para
  identificar o formulario correto. Confirme com a equipe BDesk da sua empresa o que esta configurado.
- Conhecimento basico da ferramenta de monitoramento que sera integrada (Zabbix, PRTG, Nagios, etc.).

---

## Passo a Passo

### Visao Geral: Automacao de Alertas

O fluxo basico de integracao entre uma ferramenta de monitoramento e o BDesk segue este caminho:

```
[Ferramenta Monitoramento] --> [Script Python/PowerShell] --> [API BDesk] --> [Requisicao Criada]
```

1. A ferramenta de monitoramento detecta um problema (ex.: servidor fora do ar, link saturado).
2. A ferramenta dispara um script ou webhook configurado pelo administrador.
3. O script monta o payload e chama a API BDesk.
4. O BDesk cria uma requisicao e a encaminha para a equipe responsavel conforme as regras do formulario.

---

### Passo 1: Escolha o metodo de abertura

A API BDesk oferece dois caminhos para criar requisicoes a partir de alertas de monitoramento:

| Metodo | Endpoint | Quando usar |
|--------|----------|-------------|
| Template Zabbix | `POST /v1/requisicoes/abrirFormatoZabbix/{variante}` | Zabbix e ferramentas com payload dinamico similar. A `{variante}` e opcional |
| Abertura Generica | `POST /v1/requisicoes/abrir` | PRTG, Nagios, Checkmk e qualquer outra ferramenta |

---

### Passo 2: Configure a autenticacao no script

Todos os endpoints exigem o token no cabecalho `Authorization`:

```
Authorization: Bearer SEU_TOKEN_AQUI
```

Guarde o token em uma variavel de ambiente ou em um cofre de segredos — nunca o incorpore diretamente
no codigo-fonte.

---

### Passo 3: Monte o payload do alerta

O payload varia conforme o metodo escolhido. Veja os detalhes nas secoes abaixo.

---

### Passo 4: Envie a requisicao e registre o ID retornado

Apos criar a requisicao, armazene o ID retornado pela API (um numero inteiro). Voce vai precisar dele
para atualizar ou encerrar a requisicao quando o alerta for resolvido.

---

## Metodo 1: Template Zabbix

### POST /v1/requisicoes/abrirFormatoZabbix/{variante}

Este e o endpoint unico para as integracoes no formato Zabbix. Cada **variante** (por exemplo,
`RompimentoFibra`) e uma configuracao da sua instancia BDesk que define o formulario, as
categorizacoes, os campos obrigatorios e as acoes automaticas daquele tipo de incidente. Nao existe
uma rota separada por tipo de incidente: o tipo e sempre escolhido pelo ultimo segmento da URL.

- Sem `{variante}` na URL (`POST /v1/requisicoes/abrirFormatoZabbix`), a variante usada e
  `RompimentoFibra`.
- Se a variante informada nao estiver configurada na sua instancia, a API responde HTTP 406.
- Consulte a equipe BDesk da sua empresa para saber quais variantes estao configuradas.

**O payload depende da variante.** O corpo e um JSON livre cujos campos sao definidos pela
configuracao da variante: ela diz quais campos sao obrigatorios e como cada um e usado para montar
a requisicao. Se um campo obrigatorio faltar, a API responde 406 informando o nome do campo. Para a
variante `RompimentoFibra`, por exemplo, o payload usa campos como `olt`, `hostName`, `problem_id`,
`HorarioQueda` e `PosicoesAfetadas` (lista de `{ "Slot", "Pon" }`). Combine com a equipe BDesk o
conjunto exato de campos da sua variante.

**Exemplo com cURL:**

```bash
curl -s -X POST \
  "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abrirFormatoZabbix/RompimentoFibra" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -H "Content-Type: application/json" \
  -d '{
    "olt": "OLT-01",
    "hostName": "olt01",
    "problem_id": "123",
    "HorarioQueda": "2026-10-08 10:00:00",
    "PosicoesAfetadas": [ { "Slot": "1", "Pon": "2" } ]
  }'
```

**Resposta (HTTP 200):**

```json
{
  "_metadata": {
    "Release": "9.8.0",
    "LogAmigavel": [],
    "MensagensErro": []
  },
  "records": [
    "222886"
  ]
}
```

O **primeiro item de `records` e o numero da requisicao criada** (em texto). Se a variante tiver
acoes automaticas configuradas, os itens seguintes sao textos no formato `Retorno da acao <id>: ...`
com o resultado de cada acao, e eventuais mensagens delas aparecem em `_metadata.MensagensErro`
(mesmo com HTTP 200). Confira essa lista depois de cada chamada.

**Acoes automaticas assincronas:** quando a variante esta configurada para executar as acoes em
segundo plano, a resposta traz apenas o numero da requisicao e as acoes rodam depois. Nesse caso, o
resultado das acoes nao aparece na resposta e uma falha delas nao chega ao seu script.

**Erros:** campos obrigatorios ausentes, variante nao configurada e validacoes do formulario voltam
como HTTP 406 com a mensagem em **texto puro** (nao e JSON).

---

## Metodo 2: Abertura Generica (Outras Ferramentas)

Para ferramentas que nao seguem o formato Zabbix — como PRTG, Nagios, Checkmk, Dynatrace ou scripts
proprios — use o endpoint padrao de abertura de requisicao.

**Endpoint:** `POST /v1/requisicoes/abrir`

O corpo tem o numero do formulario em `Formulario` e os dados agrupados em `Conjuntos`. O assunto e a
descricao ficam dentro do conjunto `DadosBasicos`. Monte o payload mapeando os campos do alerta para
os campos do BDesk:

| Campo do Alerta | Campo do BDesk | Observacao |
|-----------------|---------------|------------|
| Titulo do alerta | `Conjuntos.DadosBasicos.Assunto` | Use um prefixo claro, ex.: `[PRTG] Sensor offline` |
| Descricao detalhada | `Conjuntos.DadosBasicos.Descricao` | Inclua host, IP, valor medido e horario do alerta |
| Severidade | Definida pelo formulario | Mapeie severidades para prioridades via workflow |
| ID do alerta externo | Conjunto de dados adicionais do formulario | Armazene em campo adicional para deduplicacao |

Veja o guia [Criar Requisicoes](criar-requisicoes.md) para os detalhes dos conjuntos aceitos pelo
seu formulario (consulte-os em `GET /v1/cardapio/{id}`).

**Payload para PRTG:**

```json
{
  "Formulario": 456,
  "Conjuntos": {
    "DadosBasicos": {
      "Assunto": "[PRTG] Sensor offline: Ping - srv-db-02",
      "Descricao": "Sensor: Ping\nHost: srv-db-02.empresa.com.br (10.0.2.30)\nStatus: Down\nHorario: 2025-08-15 14:35:00\nMensagem: Request timeout for icmp_seq 1"
    }
  }
}
```

Se o seu formulario tiver conjuntos de dados adicionais (por exemplo, para guardar a ferramenta de
origem e o ID do alerta externo), inclua-os em `Conjuntos` com os nomes definidos no formulario.

**Exemplo com cURL:**

```bash
curl -s -X POST "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abrir" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -H "Content-Type: application/json" \
  -d '{
    "Formulario": 456,
    "Conjuntos": {
      "DadosBasicos": {
        "Assunto": "[PRTG] Sensor offline: Ping - srv-db-02",
        "Descricao": "Host: srv-db-02.empresa.com.br\nStatus: Down\nHorario: 2025-08-15 14:35:00"
      }
    }
  }'
```

A resposta e o numero da requisicao criada, como um texto JSON simples (sem envelope):

```
"222886"
```

---

## Atualizando Requisicoes Existentes

Quando o status do alerta mudar — por exemplo, quando o Zabbix atualizar o evento ou o problema for
resolvido — voce pode atualizar a requisicao correspondente no BDesk.

**Endpoint:** `POST /v1/requisicoes/{id}/atualizarFormatoZabbix/{variante}`

Substitua `{id}` pelo **numero inteiro** da requisicao retornado na abertura (por exemplo, `222886`;
nao existe um id em outro formato).

**A configuracao de atualizacao e separada da de abertura.** Para atualizar, a sua instancia precisa
ter uma configuracao propria de atualizacao para a variante informada, alem da configuracao de
abertura. Usar o mesmo nome de variante nao basta: se a configuracao de atualizacao nao existir, a
API responde 406. Peca a equipe BDesk para confirmar que ela existe para a variante.

**Payload de atualizacao:** tambem depende da variante. Os campos mais comuns sao `PosicoesAfetadas`
(lista de `{ "Slot", "Pon" }`), `novasONUs`, `olt` e `qtd_afetado`. Informe sempre `qtd_afetado`
(numero): sem ele, a atualizacao pode falhar com HTTP 500.

```json
{
  "olt": "OLT-01",
  "PosicoesAfetadas": [ { "Slot": "1", "Pon": "2" } ],
  "novasONUs": "ONU-0042, ONU-0043",
  "qtd_afetado": 2
}
```

**Exemplo com cURL:**

```bash
curl -s -X POST \
  "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/222886/atualizarFormatoZabbix/RompimentoFibra" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -H "Content-Type: application/json" \
  -d '{
    "olt": "OLT-01",
    "PosicoesAfetadas": [ { "Slot": "1", "Pon": "2" } ],
    "novasONUs": "ONU-0042, ONU-0043",
    "qtd_afetado": 2
  }'
```

**Resposta (HTTP 200):**

```json
{
  "_metadata": { "Release": "9.8.0", "LogAmigavel": [], "MensagensErro": [] },
  "records": [
    "Retorno da acao X: ..."
  ]
}
```

`records` e uma lista de textos, um por acao executada. **Uma acao que falha nao gera erro HTTP:** a
resposta continua 200 e a mensagem aparece em `records` ou em `_metadata.MensagensErro`. Por isso,
sempre leia as duas listas depois de atualizar. Erros como falta de acesso a requisicao, JSON
invalido ou configuracao de atualizacao inexistente voltam como HTTP 406 com texto puro.

---

## Exemplos Completos

### Script Python — Integracao com Zabbix

```python
import os
import logging
import requests

# Configuracao
BASE_URL = "https://sua-empresa.bdesk.com.br/askrest"
TOKEN    = os.environ.get("BDESK_TOKEN", "SEU_TOKEN_AQUI")

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s"
)
logger = logging.getLogger(__name__)


def abrir_requisicao_zabbix(dados_alerta, variante=None):
    """
    Cria uma requisicao no BDesk a partir de um alerta do Zabbix.
    Retorna o numero da requisicao criada (texto), ou None em caso de falha.
    """
    headers = {
        "Authorization": f"Bearer {TOKEN}",
        "Content-Type": "application/json"
    }

    if variante:
        url = f"{BASE_URL}/v1/requisicoes/abrirFormatoZabbix/{variante}"
    else:
        url = f"{BASE_URL}/v1/requisicoes/abrirFormatoZabbix"

    try:
        resp = requests.post(url, json=dados_alerta, headers=headers, timeout=15)
        resp.raise_for_status()
        corpo = resp.json()
        # Mensagens de erro de acoes automaticas chegam com HTTP 200
        erros = corpo["_metadata"]["MensagensErro"]
        if erros:
            logger.warning("Avisos da abertura: %s", erros)
        records = corpo["records"]
        if not records:
            logger.error("Resposta sem o numero da requisicao")
            return None
        id_requisicao = records[0]   # primeiro item = numero da requisicao
        logger.info("Requisicao criada: %s", id_requisicao)
        return id_requisicao
    except requests.exceptions.HTTPError as e:
        # 406: a mensagem vem em texto puro
        logger.error("Erro HTTP ao criar requisicao: %s — %s", e.response.status_code, e.response.text)
    except requests.exceptions.ConnectionError:
        logger.error("Nao foi possivel conectar ao BDesk em %s", BASE_URL)
    except requests.exceptions.Timeout:
        logger.error("Timeout ao chamar a API BDesk")
    return None


def atualizar_requisicao_zabbix(id_requisicao, dados_alerta, variante):
    """
    Atualiza uma requisicao existente no BDesk com os dados mais recentes do alerta.
    """
    headers = {
        "Authorization": f"Bearer {TOKEN}",
        "Content-Type": "application/json"
    }
    url = f"{BASE_URL}/v1/requisicoes/{id_requisicao}/atualizarFormatoZabbix/{variante}"

    try:
        resp = requests.post(url, json=dados_alerta, headers=headers, timeout=15)
        resp.raise_for_status()
        corpo = resp.json()
        # Falhas de acoes individuais chegam com HTTP 200
        erros = corpo["_metadata"]["MensagensErro"]
        if erros:
            logger.warning("Requisicao %s atualizada com avisos: %s", id_requisicao, erros)
            return False
        logger.info("Requisicao %s atualizada: %s", id_requisicao, corpo["records"])
        return True
    except requests.exceptions.HTTPError as e:
        logger.error("Erro HTTP ao atualizar requisicao: %s — %s", e.response.status_code, e.response.text)
    except Exception as e:
        logger.error("Erro inesperado ao atualizar requisicao: %s", e)
    return False


# --- Ponto de entrada: simula disparo do Zabbix ---
if __name__ == "__main__":
    alerta = {
        "olt":              "OLT-01",
        "hostName":         "olt01",
        "problem_id":       "123",
        "HorarioQueda":     "2026-10-08 10:00:00",
        "PosicoesAfetadas": [{"Slot": "1", "Pon": "2"}]
    }

    id_req = abrir_requisicao_zabbix(alerta, variante="RompimentoFibra")

    if id_req:
        # Persistir id_req no seu sistema para uso posterior (ex.: banco, arquivo)
        logger.info("Salvar mapeamento: problem_id=%s -> requisicao=%s", alerta["problem_id"], id_req)
```

---

### Script PowerShell — Integracao com Zabbix

```powershell
param(
    [string]$Olt          = "OLT-01",
    [string]$HostName     = "olt01",
    [string]$ProblemId    = "123",
    [string]$HorarioQueda = "2026-10-08 10:00:00",
    [string]$Variante     = "RompimentoFibra",
    [string]$RequisicaoId = ""
)

$BaseUrl = "https://sua-empresa.bdesk.com.br/askrest"
$Token   = $env:BDESK_TOKEN

if (-not $Token) {
    Write-Error "Variavel de ambiente BDESK_TOKEN nao definida."
    exit 1
}

$Headers = @{
    Authorization  = "Bearer $Token"
    "Content-Type" = "application/json"
}

try {
    if ($RequisicaoId) {
        # Atualizar requisicao existente (RequisicaoId e o numero inteiro retornado na abertura)
        $Payload = @{
            olt              = $Olt
            PosicoesAfetadas = @(@{ Slot = "1"; Pon = "2" })
            novasONUs        = "ONU-0042"
            qtd_afetado      = 1
        } | ConvertTo-Json -Depth 5
        $Url = "$BaseUrl/v1/requisicoes/$RequisicaoId/atualizarFormatoZabbix/$Variante"
        $Resp = Invoke-RestMethod -Uri $Url -Method Post -Headers $Headers -Body $Payload -ContentType "application/json"
        # Falhas de acoes individuais chegam com HTTP 200
        if ($Resp._metadata.MensagensErro.Count -gt 0) {
            Write-Warning ("Atualizada com avisos: " + ($Resp._metadata.MensagensErro -join "; "))
        } else {
            Write-Host "Requisicao $RequisicaoId atualizada com sucesso."
        }
    } else {
        # Abrir requisicao
        $Payload = @{
            olt              = $Olt
            hostName         = $HostName
            problem_id       = $ProblemId
            HorarioQueda     = $HorarioQueda
            PosicoesAfetadas = @(@{ Slot = "1"; Pon = "2" })
        } | ConvertTo-Json -Depth 5
        $Url = "$BaseUrl/v1/requisicoes/abrirFormatoZabbix/$Variante"
        $Resp = Invoke-RestMethod -Uri $Url -Method Post -Headers $Headers -Body $Payload -ContentType "application/json"
        if ($Resp._metadata.MensagensErro.Count -gt 0) {
            Write-Warning ("Avisos da abertura: " + ($Resp._metadata.MensagensErro -join "; "))
        }
        Write-Host "Requisicao criada: $($Resp.records[0])"
    }
} catch {
    $StatusCode = $_.Exception.Response.StatusCode.value__
    Write-Error "Erro ao chamar a API BDesk (HTTP $StatusCode): $_"
    exit 1
}
```

---

## Boas Praticas

### Deduplicacao de Alertas

Antes de criar uma nova requisicao, verifique se ja existe uma requisicao aberta para o mesmo alerta.
Armazene o mapeamento `ID do evento → numero da requisicao` no banco de dados ou em um arquivo de
estado local. Se o mapeamento existir e a requisicao ainda estiver aberta, use o endpoint de
atualizacao em vez de criar uma nova.

### Mapeamento de Severidade para Prioridade

Configure no BDesk um campo ou regra de workflow que mapeie a severidade do alerta para a prioridade
interna. Uma sugestao de mapeamento (a prioridade tem tres niveis: 1 Alta, 2 Media, 3 Baixa):

| Severidade Zabbix | Prioridade BDesk |
|-------------------|-----------------|
| Not Classified    | Baixa           |
| Information       | Baixa           |
| Warning           | Media           |
| Average           | Media           |
| High              | Alta            |
| Disaster          | Alta            |

Discuta o mapeamento com a equipe responsavel pelas categorias do BDesk antes de colocar em producao.

### Categorizacao Correta

Use a variante (Metodo 1) ou o formulario (Metodo 2) mais especifico para cada tipo de alerta.
Configuracoes genericas resultam em triagem manual desnecessaria e SLA incorreto.

### Logging

Registre em log ao menos: o ID do evento externo, o numero da requisicao criada, o timestamp e o
status da chamada (sucesso ou codigo de erro), alem das mensagens de `_metadata.MensagensErro`.
Isso facilita auditoria e resolucao de falhas de integracao.

### Timeout e Retry

Configure timeout de no maximo 15 segundos nas chamadas HTTP. Implemente no maximo 3 tentativas com
intervalo exponencial (ex.: 5s, 15s, 45s) antes de abandonar e registrar o erro. Cuidado com tentativas
repetidas depois de um timeout: a requisicao pode ter sido criada mesmo assim, o que gera duplicidade.
Alertas que falham repetidamente devem gerar notificacao para o administrador da integracao.

### Segredos

Nunca incorpore o token diretamente no codigo-fonte. Use variaveis de ambiente, cofre de segredos
(HashiCorp Vault, AWS Secrets Manager) ou mecanismos nativos da ferramenta de monitoramento para
injetar o token em tempo de execucao.

---

## Proximos Passos

Com a integracao de monitoramento configurada, voce pode:

- [Acoes de Workflow](acoes-workflow.md) — encerrar ou redirecionar requisicoes criadas automaticamente
- [Consultar Requisicoes](consultar-requisicoes.md) — verificar o status das requisicoes criadas via alerta
