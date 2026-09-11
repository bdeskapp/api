# Evento de Dado Basico

O BDesk envia este evento para o endpoint do seu sistema sempre que um **campo padrao** de uma
requisicao e alterado — titulo, prioridade, status, grupo responsavel, datas, SLA e outros.
Esta pagina descreve o que o seu consumidor recebe e como interpretar cada campo.

---

## O que voce vai aprender

- Em que situacoes o evento de dado basico e disparado
- Quais sao os 21 campos monitorados e o que cada valor contem
- Por que `old_value` e `new_value` chegam sempre como texto legivel
- Como rotear o evento usando `field_name`

---

## Pre-requisitos

- **Endpoint de recebimento configurado** — uma URL do seu sistema apta a receber requisicoes
  `POST` com corpo JSON.
- **Familiaridade com o envelope comum** — todos os eventos do BDesk compartilham a mesma
  estrutura externa. Veja a [visao geral de eventos](index.md) antes de continuar.
- **Token de autenticacao valido** — necessario apenas se voce for consultar a API para
  complementar o evento. Veja [Autenticacao](../autenticacao.md).

---

## Quando este evento e disparado

O evento e enviado sempre que um dos campos padrao da requisicao tem seu valor alterado. Isso
inclui alteracoes feitas por um usuario na interface, por uma integracao via API ou por rotinas
automaticas do proprio BDesk (por exemplo, a marcacao de uma requisicao como fora do SLA).

Cada campo alterado gera **um evento independente**. Se um usuario altera a prioridade e o grupo
na mesma operacao, o seu sistema recebe dois eventos distintos.

O `event_type` deste evento e **sempre** `ticket.data.basicdatachanged`, qualquer que seja o campo
alterado.

!!! warning "Nao roteie pelo `event_type`"
    Como todos os campos compartilham o mesmo `event_type`, ele **nao** serve para decidir o que
    fazer com o evento. Use sempre `data.execution.change.field_name` para identificar qual campo
    mudou e direcionar o tratamento no seu consumidor.

---

## Estrutura do evento

Alem do envelope comum, o bloco `data.execution` traz quem executou a alteracao e o que mudou.

### `data.execution.executor`

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `id` | inteiro | Identificador do usuario que realizou a alteracao |
| `name` | texto | Nome do usuario. Alteracoes feitas automaticamente pelo sistema chegam identificadas como acao do sistema |

### `data.execution.change`

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `field_name` | texto | Identificador tecnico do campo alterado. **Use este campo para rotear o evento** |
| `field_description` | texto | Nome amigavel do campo, adequado para exibicao |
| `old_value` | texto | Valor anterior. Pode ser `null` quando o campo estava vazio |
| `new_value` | texto | Novo valor. Pode ser `null` quando o campo foi esvaziado |

---

## Valores chegam como texto legivel

!!! warning "Nao converta `new_value` para numero"
    `old_value` e `new_value` chegam **sempre como texto ja formatado para leitura humana**, nunca
    como identificador numerico. Para `field_name` igual a `idFrmStatus`, o valor e
    `"Em Atendimento"` — e nao o codigo do status. Para `nPrioridade`, o valor e `"Alta"`.

    Varios `field_name` comecam com o prefixo `id` por convencao interna do BDesk, mas **isso nao
    significa que o valor seja um identificador**. Um consumidor que faca
    `parseInt(new_value)` porque `field_name` comeca com `id` vai obter um resultado invalido.

Essa escolha torna o evento imediatamente util para exibicao, notificacao e auditoria, sem
necessidade de consultas adicionais.

!!! tip "Quando voce precisa do identificador"
    Se o seu sistema precisa do codigo interno de um status, grupo ou area — e nao apenas do nome —
    consulte os detalhes da requisicao pela API usando o `data.ticket.id` recebido no evento.
    Veja [Consultar Requisicoes](../guias/consultar-requisicoes.md).

---

## Campos monitorados

Os 21 campos abaixo geram evento quando alterados.

| `field_name` | `field_description` | Conteudo do valor |
|--------------|---------------------|-------------------|
| `dsTitulo` | Assunto | Texto livre |
| `dsReq` | Descricao | Texto livre |
| `nPrioridade` | Prioridade | `"Alta"`, `"Media"` ou `"Baixa"` |
| `idFrmStatus` | Status | Nome do status |
| `idAre` | Area | Nome da area |
| `idTar` | Tarefa | Nome completo da tarefa |
| `dtAbertura` | Dt. Abertura | Data e hora |
| `dtIniPrev` | Dt. Inicio Prevista | Data e hora |
| `dtFimPrev` | Dt. Fim Prevista | Data e hora |
| `dtIniReal` | Dt. Inicio Real | Data e hora |
| `dtFimReal` | Dt. Fim Real | Data e hora |
| `sla` | SLA | Duracao formatada |
| `idGru` | Grupo | Nome do grupo |
| `idGruAtual` | Grupo Atual | Nome do grupo |
| `idFrmOrigem` | Origem | Nome da origem |
| `idLoc` | Id da Localizacao | Nome da localizacao |
| `dsLoc` | Localizacao | Texto livre |
| `idPat` | Patrimonio | Descricao do patrimonio |
| `idHie` | Hierarquia | Nome completo da hierarquia |
| `idEmp` | Empresa | Razao social |
| `flgForaSla` | Fora do SLA | `"Sim"` ou `"Nao"` |

!!! note "Formato das datas"
    Os campos iniciados por `dt` chegam no formato `AAAA-MM-DD HH:MM:SS`, sem fuso horario.
    Esse formato e **diferente** do campo `timestamp` do envelope, que segue o padrao ISO 8601
    com offset (por exemplo, `2026-09-11T14:32:07-03:00`). Trate os dois formatos separadamente
    ao converter datas no seu consumidor.

---

## Exemplo: alteracao de prioridade

```json
{
  "event_id": "3f2a8c14-9e7b-4d51-a0c6-8b1f2e5d7a93",
  "event_type": "ticket.data.basicdatachanged",
  "event_version": "1.0",
  "timestamp": "2026-09-11T14:23:07-03:00",
  "source": "Bdesk/Tickets",
  "data": {
    "ticket": {
      "id": 184527,
      "title": "Impressora do 3o andar sem tinta",
      "status": "Em Atendimento"
    },
    "execution": {
      "executor": {
        "id": 4412,
        "name": "Maria Silva Santos"
      },
      "change": {
        "field_name": "nPrioridade",
        "field_description": "Prioridade",
        "old_value": "Media",
        "new_value": "Alta"
      }
    },
    "customer_context": {
      "client_id": "sistema-externo",
      "client_name": "Sistema Externo Ltda"
    }
  }
}
```

## Exemplo: mudanca de status

```json
{
  "event_id": "c81d5f60-4b2a-47e9-9d13-6f0a2c7e8b45",
  "event_type": "ticket.data.basicdatachanged",
  "event_version": "1.0",
  "timestamp": "2026-09-11T15:47:22-03:00",
  "source": "Bdesk/Tickets",
  "data": {
    "ticket": {
      "id": 184527,
      "title": "Impressora do 3o andar sem tinta",
      "status": "Em Atendimento"
    },
    "execution": {
      "executor": {
        "id": 4412,
        "name": "Maria Silva Santos"
      },
      "change": {
        "field_name": "idFrmStatus",
        "field_description": "Status",
        "old_value": "Aguardando Atendimento",
        "new_value": "Em Atendimento"
      }
    },
    "customer_context": {
      "client_id": "sistema-externo",
      "client_name": "Sistema Externo Ltda"
    }
  }
}
```

!!! note "`ticket.status` e `new_value`"
    No exemplo acima os dois coincidem porque o evento e justamente sobre a mudanca de status.
    O bloco `data.ticket` sempre reflete o estado da requisicao no momento em que o evento foi
    gerado, enquanto `change` descreve especificamente o que foi alterado.

---

## Dicas de consumo

- **Roteie por `field_name`.** Monte um mapa de tratamento no seu consumidor e aplique um
  comportamento padrao — ignorar ou registrar em log — para campos que voce ainda nao trata.
- **Trate `new_value` como texto.** Use os valores para exibicao, notificacao e trilha de
  auditoria. Se precisar de identificadores, consulte a requisicao pela API.
- **Aceite `null` nos dois lados.** Um campo preenchido pela primeira vez chega com `old_value`
  nulo; um campo esvaziado chega com `new_value` nulo.
- **Espere eventos em paralelo.** Uma unica operacao do usuario pode alterar varios campos e
  gerar varios eventos, sem ordem garantida entre eles. Ordene por `timestamp` quando a sequencia
  for relevante.

!!! warning "Uma alteracao pode gerar dois eventos de tipos diferentes"
    Algumas operacoes produzem tanto um evento de **Dado Basico** quanto um evento de **Acao**.
    Alterar a prioridade, por exemplo, pode gerar um `ticket.data.basicdatachanged` com
    `field_name` igual a `nPrioridade` **e** um `ticket.action.priority_updated`.

    Os dois descrevem a mesma operacao sob perspectivas diferentes: o evento de dado basico
    informa o valor anterior e o novo; o evento de acao informa o contexto da acao executada.
    Decida qual dos dois o seu fluxo deve consumir e ignore o outro, para nao processar a mesma
    mudanca duas vezes.

---

## Proximos Passos

- [Evento de Acao](acao.md) — acoes de workflow executadas na requisicao
- [Evento de Dado Adicional](dado-adicional.md) — alteracoes em campos personalizados
- [Evento de Participante](participante.md) — mudancas na composicao de participantes
- [Configurar o Recebimento de Eventos](configuracao.md) — como habilitar o envio
