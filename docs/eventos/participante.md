# Evento de Participante

O BDesk envia este evento para o endpoint do seu sistema sempre que a composicao de participantes
de uma requisicao muda — alguem passa a participar, deixa de participar ou o responsavel e trocado.
Esta pagina descreve o que o seu consumidor recebe e como interpretar cada campo.

---

## O que voce vai aprender

- Em que situacoes o evento de participante e disparado
- Quais campos chegam em `data.execution` e o que cada um significa
- Como usar `role_code` para identificar o papel do participante
- Por que o evento deve ser tratado como um **sinal para reconsultar** o estado atual

---

## Pre-requisitos

- **Endpoint de recebimento configurado** — uma URL do seu sistema apta a receber requisicoes
  `POST` com corpo JSON.
- **Familiaridade com o envelope comum** — todos os eventos do BDesk compartilham a mesma
  estrutura externa. Veja a
  [Visao Geral do Envio de Eventos](index.md) antes de continuar.
- **Token de autenticacao valido** — necessario apenas se voce for consultar a API para
  complementar o evento. Veja [Autenticacao](../autenticacao.md).

---

## Quando este evento e disparado

O evento e enviado quando a composicao de participantes de uma requisicao e alterada. Os casos
mais comuns sao:

- Um analista ou grupo passa a atender a requisicao (papel **ADO**)
- A requisicao e redirecionada para outro grupo ou pessoa
- Um responsavel e definido ou substituido (papel **RESP**)
- Alguem e incluido em copia para acompanhar o andamento (papel **COP**)
- Um administrador do processo e vinculado a requisicao (papel **ADP**)

O `event_type` deste evento e **sempre** `ticket.participant.changed`, independentemente de qual
dos casos acima ocorreu.

---

## Estrutura do evento

O envelope segue o padrao comum a todos os eventos (`event_id`, `event_type`, `event_version`,
`timestamp`, `source` e `data`). O conteudo especifico deste evento fica em `data.execution`.

### `data.execution.executor`

Identifica quem provocou a mudanca.

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `id` | inteiro | Identificador do usuario que executou a acao |
| `name` | texto | Nome do usuario que executou a acao |

### `data.execution.change`

Descreve a mudanca de participante.

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `role_code` | texto | Codigo do papel do participante na requisicao. **Use este campo para identificar o papel** |
| `participant_id` | inteiro | Identificador do participante envolvido — pode ser um usuario ou um grupo |
| `new_value` | texto | Nome do participante envolvido na mudanca |

!!! tip "Usuario ou grupo?"
    O campo `participant_id` identifica tanto usuarios quanto grupos. O campo `new_value` traz o
    nome legivel correspondente — util para exibicao e para registro em log, mas **nao** como
    chave de correlacao. Para correlacionar com cadastros do seu lado, prefira `participant_id`
    combinado com `role_code`.

!!! warning "Use apenas os campos documentados acima"
    O bloco `change` pode trazer campos alem dos tres descritos nesta tabela. Eles nao fazem parte
    do contrato publico e podem mudar sem aviso — ignore-os.

    Em especial: **para identificar o papel do participante, use sempre `role_code`**, e nao outro
    campo com prefixo `role_`. O `role_code` e o unico identificador de papel com significado
    documentado e estavel.

---

## Codigos de papel (`role_code`)

A tabela abaixo lista os papeis mais comuns e seu significado:

| `role_code` | Papel | Significado |
|-------------|-------|-------------|
| `REG` | Registrador | Quem registrou a requisicao |
| `ANTE` | Solicitante | Em nome de quem a requisicao foi aberta |
| `ADO` | Solicitado | Quem atende a requisicao |
| `RESP` | Responsavel | Responsavel pela requisicao |
| `COP` | Copiado | Acompanha a requisicao, sem atuar nela |
| `ADM` | Administrador | Administrador da requisicao |
| `ADP` | Administrador do Processo | Administra o fluxo da requisicao |
| `SYS` | Sistema | Participacao gerada automaticamente pelo proprio sistema |
| `EMA` | E-mail | Participante que interage por e-mail |

!!! note "Outros codigos podem existir"
    A lista acima cobre os papeis padrao, mas **outros codigos podem aparecer** conforme a
    configuracao de cada cliente. Escreva um consumidor **tolerante**: ao receber um `role_code`
    desconhecido, registre-o e siga o processamento normalmente, em vez de rejeitar o evento ou
    interromper a fila. Trate a lista como referencia, nao como enumeracao fechada.

---

## Ponto de atencao: o evento nao descreve a natureza da mudanca

!!! warning "O evento sinaliza que mudou — nao o que mudou"
    Este evento informa que a **composicao de participantes da requisicao foi alterada**, mas
    **nao informa a natureza da mudanca**: nao ha indicacao de que o participante tenha *entrado*,
    *saido* ou *substituido* outra pessoa.

    Essa e uma caracteristica do contrato: o evento funciona como um **sinal de mudanca**, nao como
    um diario de alteracoes.

    **Orientacao pratica:** ha dois caminhos, conforme a sua necessidade.

    - **Para distinguir inclusao de remocao**, assine tambem os eventos de Acao
      `ticket.action.participant_added` e `ticket.action.participant_removed`, que identificam
      explicitamente cada caso. Veja [Evento de Acao](acao.md).
    - **Para saber quem esta participando naquele momento**, consulte o endpoint de participantes
      da requisicao e use a resposta como fonte da verdade.

    O evento de participante diz *quando* olhar; o evento de acao ou a consulta dizem *o que* mudou.

Consulte o guia de [Participantes](../guias/participantes.md) para saber como buscar participantes
e interpretar os papeis retornados pela API.

---

## Exemplo completo

Atribuicao de um solucionador — um analista passa a atender a requisicao no papel **ADO**
(Solicitado):

```json
{
  "event_id": "9f3c1d2a-7b45-4e18-a6c0-2d91f8e4b573",
  "event_type": "ticket.participant.changed",
  "event_version": "1.0",
  "timestamp": "2026-09-11T14:32:07-03:00",
  "source": "Bdesk/Tickets",
  "data": {
    "ticket": {
      "id": 184527,
      "title": "Falha de acesso ao portal de faturamento",
      "status": "Em Atendimento"
    },
    "execution": {
      "executor": {
        "id": 1803,
        "name": "Maria Santos"
      },
      "change": {
        "role_code": "ADO",
        "participant_id": 2417,
        "new_value": "Joao Silva"
      }
    },
    "customer_context": {
      "client_id": "ACME-BR",
      "client_name": "ACME Industria e Comercio Ltda"
    }
  }
}
```

Neste exemplo:

- **Maria Santos** (`executor`) realizou a acao que alterou os participantes
- **Joao Silva** (`new_value`, `participant_id` 2417) esta envolvido na mudanca no papel **ADO**
- A requisicao **184527** esta em **Em Atendimento**

Lembre-se de que o evento **nao afirma** que Joao Silva entrou na requisicao — apenas que a
composicao de participantes mudou e que ele esta envolvido nessa mudanca no papel `ADO`. Para
confirmar o estado atual, consulte a requisicao.

---

## Dicas de consumo

**Responda rapido e processe depois.** Confirme o recebimento com uma resposta HTTP de sucesso o
quanto antes e empurre o processamento para uma fila interna. Consultas a API para enriquecer o
evento devem acontecer fora do ciclo de recebimento.

**Use `event_id` para deduplicacao.** Reentregas podem acontecer. Guarde os `event_id` ja
processados e descarte repeticoes, garantindo que o processamento seja idempotente.

**Nao presuma ordem de chegada.** Varios eventos da mesma requisicao podem chegar fora de ordem.
Ao reconsultar o estado atual, voce elimina a dependencia da sequencia de entrega.

**Agrupe reconsultas por requisicao.** Uma unica acao pode gerar mais de um evento de participante.
Se o seu consumidor reconsulta o estado a cada evento, considere agrupar por `data.ticket.id` dentro
de uma pequena janela de tempo para reduzir chamadas desnecessarias a API.

**Valide `event_type` antes de processar.** Direcione o evento pelo campo `event_type` e ignore
com seguranca os tipos que o seu consumidor ainda nao trata.

**Registre o payload bruto.** Guardar o JSON original facilita muito a investigacao de divergencias
entre o que o BDesk enviou e o que o seu sistema interpretou.

---

## Proximos Passos

- [Participantes](../guias/participantes.md) — consultar participantes e papeis via API
- [Evento de Dado Adicional](dado-adicional.md) — eventos de campos personalizados do formulario
- [Consultar Requisicoes](../guias/consultar-requisicoes.md) — obter o estado atual de uma requisicao
