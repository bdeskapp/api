# Integracoes de Monitoramento

## O que voce vai aprender

Como integrar ferramentas de monitoramento — como Zabbix, PRTG, Nagios e similares — com a API BDesk
para criar e atualizar requisicoes automaticamente a partir de alertas de infraestrutura.

Ao final deste guia voce sera capaz de:

- Escolher o endpoint de abertura adequado para cada ferramenta de monitoramento.
- Criar requisicoes automaticamente quando um alerta for disparado.
- Atualizar uma requisicao existente quando o alerta for atualizado ou resolvido.
- Implementar scripts Python e PowerShell prontos para producao com tratamento de erros.

---

## Pre-requisitos

Antes de comecar, voce precisa ter:

- **Token de autenticacao valido** — veja [Autenticacao](../autenticacao.md) para obter o seu token.
- **ID do formulario** que sera usado para os alertas — veja [Catalogo de Servicos](catalogo-servicos.md)
  para identificar o formulario correto na sua instancia.
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
| Template Zabbix | `POST /v1/requisicoes/abrirFormatoZabbix` | Zabbix e ferramentas com payload dinamico similar |
| Template Zabbix com variante | `POST /v1/requisicoes/abrirFormatoZabbix/{variante}` | Quando sua instancia possui templates especificos por tipo de incidente |
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

Apos criar a requisicao, armazene o ID retornado pela API. Voce vai precisar dele para atualizar ou
encerrar a requisicao quando o alerta for resolvido.

---

## Metodo 1: Template Zabbix

### POST /v1/requisicoes/abrirFormatoZabbix

Este endpoint foi projetado para receber o payload dinamico no formato de macros do Zabbix. Ele
interpreta automaticamente os campos do alerta e cria a requisicao com o formulario e as
categorizacoes configuradas no template do BDesk.

**Payload tipico enviado pelo Zabbix:**

```json
{
  "host": "srv-app-01.empresa.com.br",
  "hostip": "10.0.1.15",
  "trigger": "Servidor indisponivel",
  "triggerid": "12345",
  "triggerurl": "https://zabbix.empresa.com.br/tr_events.php?triggerid=12345",
  "severity": "High",
  "status": "PROBLEM",
  "eventid": "98765",
  "eventdate": "2025-08-15",
  "eventtime": "14:32:00",
  "itemkey": "agent.ping",
  "itemvalue": "0",
  "description": "O host srv-app-01 nao esta respondendo ao agente Zabbix."
}
```

**Exemplo com cURL:**

```bash
curl -s -X POST "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abrirFormatoZabbix" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -H "Content-Type: application/json" \
  -d '{
    "host": "srv-app-01.empresa.com.br",
    "hostip": "10.0.1.15",
    "trigger": "Servidor indisponivel",
    "severity": "High",
    "status": "PROBLEM",
    "eventid": "98765",
    "description": "O host srv-app-01 nao esta respondendo ao agente Zabbix."
  }'
```

---

### POST /v1/requisicoes/abrirFormatoZabbix/{variante}

Use este endpoint quando sua instancia BDesk possui templates especificos por tipo de incidente.
A `{variante}` identifica o template de formulario e as regras de categorização que serao aplicadas.

**Variantes comuns:**

| Variante | Uso tipico |
|----------|------------|
| `RompimentoFibra` | Incidentes de link de fibra optica rompido |
| `InvestigacaoSaturacao` | Alertas de saturacao de banda ou CPU |

Consulte a equipe BDesk da sua empresa para saber quais variantes estao configuradas na sua instancia.

**Exemplo com cURL:**

```bash
curl -s -X POST \
  "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abrirFormatoZabbix/RompimentoFibra" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -H "Content-Type: application/json" \
  -d '{
    "host": "roteador-sp-01",
    "hostip": "200.175.42.10",
    "trigger": "Link de fibra DOWN",
    "severity": "Disaster",
    "status": "PROBLEM",
    "eventid": "110022",
    "description": "Interface GigabitEthernet0/1 sem link ha 3 minutos."
  }'
```

---

## Metodo 2: Abertura Generica (Outras Ferramentas)

Para ferramentas que nao seguem o formato Zabbix — como PRTG, Nagios, Checkmk, Dynatrace ou scripts
proprios — use o endpoint padrao de abertura de requisicao.

**Endpoint:** `POST /v1/requisicoes/abrir`

Monte o payload mapeando os campos do alerta para os campos do BDesk:

| Campo do Alerta | Campo do BDesk | Observacao |
|-----------------|---------------|------------|
| Titulo do alerta | `Assunto` | Use um prefixo claro, ex.: `[PRTG] Sensor offline` |
| Descricao detalhada | `Descricao` | Inclua host, IP, valor medido e horario do alerta |
| Severidade | Definida pelo formulario | Mapeie severidades para prioridades via workflow |
| ID do alerta externo | `Conjuntos` | Armazene em campo adicional para deduplicacao |

**Payload para PRTG:**

```json
{
  "Formulario": 456,
  "Assunto": "[PRTG] Sensor offline: Ping - srv-db-02",
  "Descricao": "Sensor: Ping\nHost: srv-db-02.empresa.com.br (10.0.2.30)\nStatus: Down\nHorario: 2025-08-15 14:35:00\nMensagem: Request timeout for icmp_seq 1",
  "Conjuntos": {
    "DadosDoAlerta": {
      "FerramentaOrigem": "PRTG",
      "IdAlertaExterno": "sensor-4421",
      "Severidade": "Error"
    }
  }
}
```

**Exemplo com cURL:**

```bash
curl -s -X POST "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/abrir" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -H "Content-Type: application/json" \
  -d '{
    "Formulario": 456,
    "Assunto": "[PRTG] Sensor offline: Ping - srv-db-02",
    "Descricao": "Host: srv-db-02.empresa.com.br\nStatus: Down\nHorario: 2025-08-15 14:35:00"
  }'
```

A resposta e o ID da requisicao criada, retornado como texto simples:

```
"RQ-20250815-042"
```

---

## Atualizando Requisicoes Existentes

Quando o status do alerta mudar — por exemplo, quando o Zabbix atualizar o evento ou o problema for
resolvido — voce pode atualizar a requisicao correspondente no BDesk.

**Endpoint:** `POST /v1/requisicoes/{id}/atualizarFormatoZabbix/{variante}`

Substitua `{id}` pelo ID da requisicao retornado na abertura e `{variante}` pelo mesmo valor usado na
abertura (ex.: `RompimentoFibra`).

**Payload de atualizacao:**

```json
{
  "host": "roteador-sp-01",
  "trigger": "Link de fibra DOWN",
  "status": "RESOLVED",
  "eventid": "110022",
  "description": "Interface GigabitEthernet0/1 restaurada. Link UP confirmado."
}
```

**Exemplo com cURL:**

```bash
curl -s -X POST \
  "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/RQ-20250815-042/atualizarFormatoZabbix/RompimentoFibra" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -H "Content-Type: application/json" \
  -d '{
    "host": "roteador-sp-01",
    "trigger": "Link de fibra DOWN",
    "status": "RESOLVED",
    "eventid": "110022",
    "description": "Interface restaurada. Link UP confirmado."
  }'
```

---

## Exemplos Completos

### Script Python — Integracao com Zabbix

```python
import os
import json
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
    Retorna o ID da requisicao criada, ou None em caso de falha.
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
        id_requisicao = resp.json() if resp.headers.get("Content-Type", "").startswith("application/json") else resp.text.strip('"')
        logger.info("Requisicao criada: %s", id_requisicao)
        return id_requisicao
    except requests.exceptions.HTTPError as e:
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
        logger.info("Requisicao %s atualizada com sucesso", id_requisicao)
        return True
    except requests.exceptions.HTTPError as e:
        logger.error("Erro HTTP ao atualizar requisicao: %s — %s", e.response.status_code, e.response.text)
    except Exception as e:
        logger.error("Erro inesperado ao atualizar requisicao: %s", e)
    return False


# --- Ponto de entrada: simula disparo do Zabbix ---
if __name__ == "__main__":
    alerta = {
        "host":        "srv-app-01.empresa.com.br",
        "hostip":      "10.0.1.15",
        "trigger":     "Servidor indisponivel",
        "triggerid":   "12345",
        "severity":    "High",
        "status":      "PROBLEM",
        "eventid":     "98765",
        "description": "O host srv-app-01 nao esta respondendo ao agente Zabbix."
    }

    id_req = abrir_requisicao_zabbix(alerta, variante="InvestigacaoSaturacao")

    if id_req:
        # Persistir id_req no seu sistema para uso posterior (ex.: banco, arquivo)
        logger.info("Salvar mapeamento: eventid=%s -> requisicao=%s", alerta["eventid"], id_req)
```

---

### Script PowerShell — Integracao com Zabbix

```powershell
param(
    [string]$Host         = "srv-app-01.empresa.com.br",
    [string]$HostIp       = "10.0.1.15",
    [string]$Trigger      = "Servidor indisponivel",
    [string]$Severity     = "High",
    [string]$Status       = "PROBLEM",
    [string]$EventId      = "98765",
    [string]$Description  = "O host nao esta respondendo ao agente Zabbix.",
    [string]$Variante     = "",
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

$Payload = @{
    host        = $Host
    hostip      = $HostIp
    trigger     = $Trigger
    severity    = $Severity
    status      = $Status
    eventid     = $EventId
    description = $Description
} | ConvertTo-Json

try {
    if ($RequisicaoId) {
        # Atualizar requisicao existente
        $Url = "$BaseUrl/v1/requisicoes/$RequisicaoId/atualizarFormatoZabbix/$Variante"
        $Resp = Invoke-RestMethod -Uri $Url -Method Post -Headers $Headers -Body $Payload -ContentType "application/json"
        Write-Host "Requisicao $RequisicaoId atualizada com sucesso."
    } elseif ($Variante) {
        # Abrir com variante
        $Url = "$BaseUrl/v1/requisicoes/abrirFormatoZabbix/$Variante"
        $IdRequisicao = Invoke-RestMethod -Uri $Url -Method Post -Headers $Headers -Body $Payload -ContentType "application/json"
        Write-Host "Requisicao criada: $IdRequisicao"
    } else {
        # Abrir formato padrao
        $Url = "$BaseUrl/v1/requisicoes/abrirFormatoZabbix"
        $IdRequisicao = Invoke-RestMethod -Uri $Url -Method Post -Headers $Headers -Body $Payload -ContentType "application/json"
        Write-Host "Requisicao criada: $IdRequisicao"
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
Armazene o mapeamento `eventid → RequisicaoId` no banco de dados ou em um arquivo de estado local.
Se o mapeamento existir e a requisicao ainda estiver aberta, use o endpoint de atualizacao em vez
de criar uma nova.

### Mapeamento de Severidade para Prioridade

Configure no BDesk um campo ou regra de workflow que mapeie a severidade do alerta para a prioridade
interna. Uma sugestao de mapeamento:

| Severidade Zabbix | Prioridade BDesk |
|-------------------|-----------------|
| Not Classified    | Baixa           |
| Information       | Baixa           |
| Warning           | Media           |
| Average           | Media           |
| High              | Alta            |
| Disaster          | Critica         |

Discuta o mapeamento com a equipe responsavel pelas categorias do BDesk antes de colocar em producao.

### Categorizacao Correta

Escolha o formulario (campo `Formulario`) e a variante mais especifica para cada tipo de alerta.
Formularios genericos resultam em triagem manual desnecessaria e SLA incorreto.

### Logging

Registre em log ao menos: o ID do evento externo, o ID da requisicao criada, o timestamp e o status
da chamada (sucesso ou codigo de erro). Isso facilita auditoria e resolucao de falhas de integracao.

### Timeout e Retry

Configure timeout de no maximo 15 segundos nas chamadas HTTP. Implemente no maximo 3 tentativas com
intervalo exponencial (ex.: 5s, 15s, 45s) antes de abandonar e registrar o erro. Alertas que falham
repetidamente devem gerar notificacao para o administrador da integracao.

### Segredos

Nunca incorpore o token diretamente no codigo-fonte. Use variaveis de ambiente, cofre de segredos
(HashiCorp Vault, AWS Secrets Manager) ou mecanismos nativos da ferramenta de monitoramento para
injetar o token em tempo de execucao.

---

## Proximos Passos

Com a integracao de monitoramento configurada, voce pode:

- [Acoes de Workflow](acoes-workflow.md) — encerrar ou redirecionar requisicoes criadas automaticamente
- [Consultar Requisicoes](consultar-requisicoes.md) — verificar o status das requisicoes criadas via alerta
