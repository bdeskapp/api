# Evento de Acao

O BDesk envia um evento de **Acao** toda vez que uma acao de fluxo de trabalho e executada
sobre uma requisicao — encerrar, direcionar, atribuir, aprovar, registrar acompanhamento e
assim por diante. Este documento descreve o contrato que o seu sistema **recebe** no endpoint
de webhook configurado.

---

## O que voce vai aprender

- Quando o evento de Acao e disparado e o que ele significa.
- Como interpretar o campo `event_type` e a tabela de acoes suportadas.
- Quais campos chegam em `data.action`, `data.ticket` e `data.execution`.
- Como tratar a variacao de **anexo de documento**, que traz um objeto `document`.
- Como construir um consumidor tolerante a acoes novas ou desconhecidas.

---

## Pre-requisitos

- **Endpoint HTTPS publico** capaz de receber `POST` com corpo `application/json`.
- **Endereco do webhook cadastrado** junto a equipe BDesk que administra o seu ambiente.
- Familiaridade com o envelope comum dos eventos — veja a visao geral de eventos na pagina
  inicial da documentacao.

---

## Quando este evento e disparado

O evento e emitido **imediatamente apos a conclusao bem-sucedida de uma acao de workflow**
em uma requisicao. Ou seja:

- A acao ja foi validada e aplicada quando o evento sai. Voce nunca recebe evento de acao
  que falhou ou foi abortada.
- Uma unica interacao do usuario pode resultar em **mais de um evento** — por exemplo,
  alterar a prioridade dispara tanto um evento de Acao quanto um evento de
  [Dado Basico](dado-basico.md). Decida no consumidor qual dos dois e a fonte de verdade
  para o seu caso de uso.
- Acoes executadas via API REST, via portal web, via aplicativo ou por automacoes internas
  do BDesk geram o mesmo evento, com o mesmo formato. Nao ha como distinguir a origem pelo
  contrato.

!!! note "Entrega assincrona"
    O evento e entregue de forma assincrona. Um pequeno atraso entre a acao e a chegada do
    `POST` no seu endpoint e esperado e nao indica falha.

---

## Envelope

O evento usa o envelope padrao de todos os eventos enviados pelo BDesk:

```json
{
  "event_id": "<uuid>",
  "event_type": "ticket.action.<acao>",
  "event_version": "1.0",
  "timestamp": "2026-09-11T14:32:07-03:00",
  "source": "Bdesk/Tickets",
  "data": { }
}
```

| Campo | Descricao |
|---|---|
| `event_id` | Identificador unico (UUID) do evento. Use-o para idempotencia. |
| `event_type` | Varia conforme a acao executada — veja a tabela abaixo. |
| `event_version` | Versao do contrato. Atualmente `"1.0"`. |
| `timestamp` | Momento do evento em ISO 8601 com offset de fuso horario. |
| `source` | Origem do evento. Sempre `"Bdesk/Tickets"` para eventos de requisicao. |
| `data` | Carga util do evento — detalhada nas secoes seguintes. |

---

## Tabela de acoes e `event_type`

Cada acao de workflow tem um codigo curto e um `event_type` correspondente:

| Codigo | `event_type` | Significado |
|---|---|---|
| `ENV` | `ticket.action.submitted` | Enviada |
| `ENC` | `ticket.action.closed` | Encerrada |
| `CONC` | `ticket.action.completed` | Concluida |
| `DIR` | `ticket.action.routed` | Direcionada |
| `ATR` | `ticket.action.assigned` | Atribuida |
| `ATRR` | `ticket.action.responsible_assigned` | Responsavel atribuido |
| `ASS` | `ticket.action.assumed` | Assumida |
| `AC` | `ticket.action.accepted` | Aceita |
| `REC` | `ticket.action.rejected` | Rejeitada |
| `CANC` | `ticket.action.cancelled` | Cancelada |
| `SUSP` | `ticket.action.suspended` | Suspensa |
| `RETOM` | `ticket.action.resumed` | Retomada |
| `REAT` | `ticket.action.reactivated` | Reativada |
| `CAT` | `ticket.action.categorized` | Categorizada |
| `RECAT` | `ticket.action.recategorized` | Recategorizada |
| `ALTPRI` | `ticket.action.priority_updated` | Prioridade alterada |
| `ALTDES` | `ticket.action.description_updated` | Descricao alterada |
| `ALTPRZ` | `ticket.action.deadline_updated` | Prazo alterado |
| `APR` | `ticket.action.approved` | Aprovada |
| `REPR` | `ticket.action.approval_rejected` | Aprovacao rejeitada |
| `AVAL` | `ticket.action.evaluated` | Avaliada |
| `SOL` | `ticket.action.resolved` | Solucionada |
| `ACO` | `ticket.action.followup_logged` | Acompanhamento registrado |
| `DESD` | `ticket.action.split` | Desdobrada |
| `VINC` | `ticket.action.linked` | Vinculada |
| `APONT` | `ticket.action.time_logged` | Apontamento de horas |
| `INCPAR` | `ticket.action.participant_added` | Participante incluido |
| `EXCPAR` | `ticket.action.participant_removed` | Participante removido |
| `CONV` | `ticket.action.participants_summoned` | Participantes convocados |

Alem dessas, existe a variacao de anexo, descrita mais adiante:

| Situacao | `event_type` |
|---|---|
| Acao que anexa um documento, com os dados do arquivo | `ticket.action.attached_by_user` |
| Acao de anexo sem os dados do arquivo | `ticket.action.document_attached` ou `ticket.action.document_attached_by_user` |

!!! warning "Trate `event_type` desconhecido de forma tolerante"
    A lista acima **nao e exaustiva**. O BDesk suporta acoes adicionais, e acoes que nao
    possuem mapeamento especifico chegam como **`ticket.action.generic`**. Acoes novas podem
    ser introduzidas a qualquer momento sem alterar a versao do contrato.

    Seu consumidor **nunca deve falhar** ao encontrar um `event_type` que nao reconhece.
    Use um `switch`/`match` com um ramo padrao que apenas registra o evento em log (ou o
    ignora silenciosamente) e responde `2xx`. Rejeitar o evento com erro so faz com que ele
    seja reentregue indefinidamente.

!!! tip "Quer saber exatamente qual acao foi?"
    Mesmo quando o `event_type` chega como `ticket.action.generic`, o campo
    `data.action.description` traz o nome legivel da acao executada (ex.: `"Encerrar"`).
    Use-o para log e diagnostico.

---

## Estrutura de `data`

### `data.action`

Identifica qual acao foi executada.

| Campo | Tipo | Descricao |
|---|---|---|
| `description` | string | Nome legivel da acao, como exibido ao usuario. Ex.: `"Encerrar"`. |
| `id_action` | inteiro | Identificador numerico da acao. |
| `id_routine_action` | inteiro | Identificador da execucao da acao dentro do fluxo da requisicao. |

### `data.ticket`

Identifica a requisicao afetada.

| Campo | Tipo | Descricao |
|---|---|---|
| `id` | inteiro | Numero da requisicao. |
| `title` | string | Assunto da requisicao no momento do evento. |
| `status` | string | Situacao da requisicao **apos** a acao, em texto legivel. Ex.: `"Encerrada"`. |

### `data.execution`

Descreve **como** e **por quem** a acao foi executada.

| Campo | Tipo | Descricao |
|---|---|---|
| `description` | string \| null | Texto que o usuario escreveu ao executar a acao (justificativa, solucao, comentario). Pode vir `null` quando a acao nao exige texto. |
| `executor.id` | inteiro | Identificador do usuario que executou a acao. |
| `executor.name` | string | Nome do usuario que executou a acao. |

!!! note "`description` pode ser `null`"
    Nem toda acao pede um texto ao usuario. Trate `data.execution.description` como campo
    opcional e nao assuma que existe uma string para exibir.

### `data.customer_context`

Identifica o cliente/tenant de origem do evento.

| Campo | Tipo | Descricao |
|---|---|---|
| `client_id` | string | Identificador do cliente no BDesk. |
| `client_name` | string | Nome do cliente. |

---

## Exemplo 1 — Encerramento de requisicao (`ENC`)

```json
{
  "event_id": "8f3b21c4-5d7e-4a19-b0c6-2e41f9a7d830",
  "event_type": "ticket.action.closed",
  "event_version": "1.0",
  "timestamp": "2026-09-11T14:32:07-03:00",
  "source": "Bdesk/Tickets",
  "data": {
    "action": {
      "description": "Encerrar",
      "id_action": 12,
      "id_routine_action": 984217
    },
    "ticket": {
      "id": 184527,
      "title": "Impressora do setor financeiro nao liga",
      "status": "Encerrada"
    },
    "execution": {
      "description": "Fonte da impressora substituida. Equipamento testado e liberado para uso.",
      "executor": {
        "id": 1803,
        "name": "Joao Silva"
      }
    },
    "customer_context": {
      "client_id": "ACME-BR",
      "client_name": "ACME Industria e Comercio Ltda"
    }
  }
}
```

---

## Variacao: acao com anexo de documento

Quando a acao envolve **anexar um documento** a requisicao, o evento chega com
`event_type` = **`ticket.action.attached_by_user`** e o objeto `data.execution` ganha um
campo adicional `document`:

| Campo | Tipo | Descricao |
|---|---|---|
| `execution.document.id` | inteiro | Identificador do anexo. Use este valor para referenciar o arquivo. |
| `execution.document.url` | string | Endereco de visualizacao do anexo no portal. |

!!! warning "Nao baixe o arquivo pela `url` do evento"
    A `url` presente em `document` aponta para a interface do portal e **exige uma sessao
    autenticada de navegador**. Uma requisicao automatizada com token de API para esse
    endereco nao retorna o arquivo.

    Para baixar o arquivo programaticamente, use o **endpoint REST de download de anexo**:

    ```
    GET /v1/requisicoes/{requisicao}/anexos/{idDoc}
    ```

    Onde `{requisicao}` e `data.ticket.id` e `{idDoc}` e `data.execution.document.id`.
    Envie o header `Authorization: Bearer SEU_TOKEN`. Detalhes e exemplos completos em
    [Trabalhar com Anexos](../guias/anexos.md).

### Exemplo 2 — Anexo de documento

```json
{
  "event_id": "c07d5e91-4b2a-4f38-9a15-6db3c8047e52",
  "event_type": "ticket.action.attached_by_user",
  "event_version": "1.0",
  "timestamp": "2026-09-11T15:04:22-03:00",
  "source": "Bdesk/Tickets",
  "data": {
    "action": {
      "description": "Registrar Acompanhamento",
      "id_action": 7,
      "id_routine_action": 984305
    },
    "ticket": {
      "id": 184527,
      "title": "Impressora do setor financeiro nao liga",
      "status": "Em Atendimento"
    },
    "execution": {
      "description": "Segue laudo tecnico do equipamento.",
      "executor": {
        "id": 1803,
        "name": "Joao Silva"
      },
      "document": {
        "id": "56789",
        "url": "https://sua-empresa.bdesk.com.br/Requisicao/Documento/56789"
      }
    },
    "customer_context": {
      "client_id": "ACME-BR",
      "client_name": "ACME Industria e Comercio Ltda"
    }
  }
}
```

Para baixar esse anexo:

```bash
curl -s "https://sua-empresa.bdesk.com.br/askrest/v1/requisicoes/184527/anexos/56789" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -o "laudo-tecnico.pdf"
```

---

## Dicas de consumo

### Responda rapido, processe depois

Devolva `200 OK` assim que receber e validar o formato do evento. Coloque o processamento
pesado (consultas a API, gravacao em banco, notificacoes) em uma fila. Endpoints lentos
aumentam o risco de timeout e reentrega.

### Garanta idempotencia por `event_id`

O mesmo evento pode chegar mais de uma vez. Guarde os `event_id` ja processados e descarte
duplicatas antes de aplicar qualquer efeito colateral.

### Roteie pelo `event_type`, mas com ramo padrao

```text
se event_type == "ticket.action.closed"        -> fechar chamado espelho
se event_type == "ticket.action.assigned"      -> atualizar responsavel
se event_type em ("ticket.action.attached_by_user",
                  "ticket.action.document_attached",
                  "ticket.action.document_attached_by_user") -> tratar anexo
    (o objeto data.execution.document so vem no primeiro caso)
senao                                          -> registrar em log e seguir
```

Nunca lance erro no ramo padrao.

### Nao confie apenas no evento para o estado completo

O evento traz um recorte: a acao, quem executou e o status resultante. Se voce precisa do
estado completo da requisicao (participantes, campos adicionais, prazos), consulte
`GET /v1/requisicoes/{id}` apos receber o evento.

### Considere a ordem de chegada

Eventos sao entregues de forma assincrona e **a ordem nao e garantida**. Use o campo
`timestamp` para ordenar eventos da mesma requisicao antes de aplicar mudancas de estado,
ou trate o evento apenas como gatilho para reconsultar a API.

### Valide o `customer_context`

Se o seu consumidor atende mais de um ambiente BDesk, use `data.customer_context.client_id`
para direcionar o processamento ao tenant correto.

---

## Proximos passos

- [Evento de Dado Basico](dado-basico.md) — alteracoes em campos padrao da requisicao
- [Trabalhar com Anexos](../guias/anexos.md) — baixar e enviar arquivos via API
- [Acoes de Workflow](../guias/acoes-workflow.md) — executar acoes pela API REST
- [Consultar Requisicoes](../guias/consultar-requisicoes.md) — buscar o estado completo apos o evento
