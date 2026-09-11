# Envio de Eventos

## O que voce vai aprender

O que e o Envio de Eventos do BDesk, quais tipos de evento existem, qual e o formato da
mensagem recebida e quais garantias de entrega voce precisa considerar ao construir um
sistema consumidor.

---

## Pre-requisitos

- **Um endpoint HTTP proprio** capaz de receber requisicoes `POST` com corpo JSON, acessivel
  pela internet e com certificado TLS valido.
- **Um provedor OAuth2 do seu lado** — o BDesk autentica no *seu* endpoint antes de entregar
  cada evento (detalhes em [Configurar o Recebimento de Eventos](configuracao.md)).
- **Solicitacao de habilitacao junto ao suporte BDesk** — o envio nao e ativado
  automaticamente.

---

## Passo a Passo

### Passo 1: Entender a direcao da integracao

O Envio de Eventos inverte o sentido da integracao em relacao a API REST.

| | Quem chama | Quem responde | Quando acontece |
|---|---|---|---|
| **API REST** | Seu sistema | BDesk | Quando voce decide consultar ou alterar algo |
| **Envio de Eventos** | BDesk | Seu sistema | Quando algo muda em uma requisicao |

Ou seja: em vez de o seu sistema consultar o BDesk de tempos em tempos (*polling*) para
descobrir se algo mudou, o BDesk **notifica proativamente** o seu sistema assim que a mudanca
ocorre.

!!! tip "Quando usar cada um"
    Use o Envio de Eventos para reagir a mudancas (sincronizar um CRM, alimentar um painel,
    disparar uma automacao). Use a [API REST](../index.md) quando precisar consultar dados sob
    demanda ou executar acoes no BDesk.

---

### Passo 2: Conhecer os quatro tipos de evento

Cada evento descreve **um tipo de mudanca** ocorrida em uma requisicao.

| Evento | `event_type` | O que significa | Exemplos tipicos |
|---|---|---|---|
| Acao | `ticket.action.*` | Uma acao de fluxo de trabalho foi executada na requisicao | Encerrar, direcionar, atribuir, recategorizar, avaliar |
| Dado Basico | `ticket.data.basicdatachanged` | Um campo padrao da requisicao mudou de valor | Titulo, descricao, prioridade, status, grupo responsavel, datas previstas, SLA |
| Dado Adicional | `ticket.data.additionaldatachanged` | Um campo personalizado do formulario mudou de valor | Centro de custo, placa do veiculo, numero do contrato — qualquer campo criado pela sua organizacao |
| Participante | `ticket.participant.changed` | A composicao de participantes da requisicao mudou | Inclusao de um solicitado, troca de responsavel, adicao de alguem em copia |

Cada tipo tem sua propria pagina com o detalhamento do bloco `data` e exemplos completos de
payload — veja [Proximos Passos](#proximos-passos).

---

### Passo 3: Entender o envelope comum

Todos os eventos, independentemente do tipo, chegam no mesmo envelope JSON. O que varia e o
conteudo de `data`.

| Campo | Tipo | Descricao |
|---|---|---|
| `event_id` | string | Identificador unico do evento. **Use este campo para deduplicacao** — ele e estavel entre reenvios do mesmo evento |
| `event_type` | string | Identifica o que aconteceu, no formato `ticket.<area>.<evento>`. Veja a tabela de tipos acima |
| `event_version` | string | Versao do contrato do envelope. Atualmente `"1.0"` |
| `timestamp` | string | Momento em que a mudanca ocorreu, em ISO 8601 com offset de fuso (ex: `2025-10-01T14:22:00-03:00`) |
| `source` | string | Origem do evento. Sempre `"Bdesk/Tickets"` |
| `data` | objeto | Carga util do evento — o conteudo varia conforme `event_type` |

Dentro de `data`, alguns blocos aparecem em **todos** os tipos de evento:

| Bloco | Campos | Descricao |
|---|---|---|
| `ticket` | `id`, `title`, `status` | Identificacao da requisicao afetada |
| `customer_context` | `client_id`, `client_name` | Contexto do cliente/organizacao a qual a requisicao pertence |
| `execution.executor` | `id`, `name` | Quem provocou a mudanca |

**Exemplo de envelope (campos comuns apenas):**

```json
{
  "event_id": "b5f3a1c2-8e44-4d7a-9f21-6c0e2b3d5a90",
  "event_type": "ticket.data.basicdatachanged",
  "event_version": "1.0",
  "timestamp": "2025-10-01T14:22:00-03:00",
  "source": "Bdesk/Tickets",
  "data": {
    "ticket": {
      "id": 12345,
      "title": "Solicitacao de acesso ao sistema financeiro",
      "status": "Em Atendimento"
    },
    "customer_context": {
      "client_id": "sistema-externo",
      "client_name": "Diretoria Administrativa"
    },
    "execution": {
      "executor": {
        "id": 987,
        "name": "Maria Santos"
      }
    }
  }
}
```

!!! note "Campos adicionais podem surgir"
    Trate o JSON de forma tolerante: novos campos podem ser incluidos em versoes futuras sem
    que `event_version` mude. Ignore o que voce nao reconhece, em vez de rejeitar a mensagem.

---

### Passo 4: Entender o transporte

A entrega de cada evento e uma requisicao HTTP simples ao endpoint que voce informou ao
suporte BDesk:

- **Metodo:** `POST`
- **Cabecalho:** `Content-Type: application/json`
- **Autenticacao:** OAuth2 no **seu** provedor. O BDesk obtem um token de acesso junto ao
  endpoint de token que voce informou e o envia no cabecalho `Authorization: Bearer ...`
- **Um evento por requisicao HTTP** — nao ha agrupamento de varios eventos num mesmo corpo

```http
POST /webhooks/bdesk HTTP/1.1
Host: seu-sistema.exemplo.com.br
Content-Type: application/json
Authorization: Bearer eyJhbGciOi...

{ "event_id": "b5f3a1c2-8e44-4d7a-9f21-6c0e2b3d5a90", "event_type": "ticket.action.closed", ... }
```

---

### Passo 5: Considerar as garantias de entrega

Esta e a parte mais importante para quem constroi o consumidor. Projete o seu sistema
assumindo as garantias abaixo — e **apenas** elas.

#### Entrega at-least-once (pelo menos uma vez)

O mesmo evento **pode chegar mais de uma vez**. Isso e esperado e nao indica falha.

!!! warning "Deduplique por `event_id`"
    Guarde os `event_id` ja processados e descarte repetidos. O `event_id` e **estavel entre
    reenvios** do mesmo evento — e a unica chave confiavel para deduplicacao. Nao use o
    `timestamp` nem o ID da requisicao para essa finalidade.

#### Retentativas

- O BDesk faz **ate 3 tentativas** de entrega para cada evento.
- O intervalo entre tentativas e de **aproximadamente 10 minutos**.
- Apos a terceira falha, aquele evento **nao e mais tentado**.

#### O que conta como sucesso

- **Sucesso:** resposta **HTTP 200**.
- **Falha:** qualquer outro codigo de resposta (301, 400, 401, 403, 404, 500, 502...),
  timeout ou erro de conexao. Toda falha agenda uma nova tentativa, respeitando o limite de 3.

!!! warning "Responda 200 somente apos aceitar o evento"
    Se voce responder 200 e depois perder a mensagem internamente, o BDesk considera a entrega
    concluida e **nao reenviara**. Persista ou enfileire antes de responder.

#### Ordem nao garantida

Os eventos **nao chegam necessariamente na ordem em que ocorreram**. Se a ordem importa para a
sua logica de negocio, **ordene pelo campo `timestamp`** — e nao pela ordem de chegada.

#### Entrega assincrona e em lote

O envio nao e sincrono com a mudanca: ele acontece em ciclos periodicos de processamento.
**Pode haver atraso de alguns minutos** entre a mudanca no BDesk e a chegada do evento no seu
endpoint. Nao construa fluxos que dependam de entrega instantanea.

---

### Passo 6: Aplicar as boas praticas do consumidor

| Pratica | Por que |
|---|---|
| **Responda rapido** — enfileire e processe em segundo plano | Respostas lentas podem gerar timeout, que conta como falha e consome uma das 3 tentativas |
| **Seja idempotente** | Consequencia direta da entrega at-least-once: processar o mesmo evento duas vezes nao pode duplicar registros nem disparar acoes repetidas |
| **Registre `event_id` e `timestamp` em log** | Sao as chaves para investigar qualquer divergencia junto ao suporte BDesk |
| **Aceite campos desconhecidos** | Evita quebra quando novos campos forem adicionados ao payload |
| **Responda 200 tambem para eventos que voce ignora** | Se um tipo de evento nao interessa ao seu sistema, descarte-o e responda 200. Responder erro so gera retentativas inuteis |

---

## Proximos Passos

- [Evento de Acao](acao.md) — acoes de fluxo de trabalho executadas na requisicao
- [Evento de Dado Basico](dado-basico.md) — mudancas em campos padrao da requisicao
- [Evento de Dado Adicional](dado-adicional.md) — mudancas em campos personalizados do formulario
- [Evento de Participante](participante.md) — mudancas na composicao de participantes
- [Configurar o Recebimento de Eventos](configuracao.md) — o que informar ao suporte BDesk para habilitar o envio
