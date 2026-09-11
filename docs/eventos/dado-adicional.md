# Evento de Dado Adicional

O BDesk envia este evento para o endpoint do seu sistema sempre que um **campo personalizado**
de um formulario — tambem chamado de dado adicional — e preenchido ou alterado em uma requisicao.
Esta pagina descreve o que o seu consumidor recebe e como interpretar cada campo.

---

## O que voce vai aprender

- Em que situacoes o evento de dado adicional e disparado
- A diferenca entre `field_id_dad` e `field_id_dad_con` e qual deles usar
- Como interpretar `new_value` para cada tipo de campo
- Como tratar campos de lista com multipla selecao

---

## Pre-requisitos

- **Endpoint de recebimento configurado** — uma URL do seu sistema apta a receber requisicoes
  `POST` com corpo JSON.
- **Familiaridade com o envelope comum** — todos os eventos do BDesk compartilham a mesma
  estrutura externa. Veja a [visao geral de eventos](index.md) antes de continuar.
- **Conhecimento dos formularios do seu ambiente** — os campos personalizados sao definidos por
  formulario. Veja [Criar Requisicoes](../guias/criar-requisicoes.md) para entender como os
  conjuntos de dados adicionais sao organizados.

---

## Quando este evento e disparado

O evento e enviado quando o valor de um campo personalizado do formulario e gravado — no momento
da abertura da requisicao, em uma edicao posterior ou durante a execucao de uma acao que atualize
o formulario.

Cada campo alterado gera **um evento independente**. Se um usuario preenche tres campos
personalizados na mesma tela, o seu sistema recebe tres eventos distintos.

O `event_type` deste evento e **sempre** `ticket.data.additionaldatachanged`.

---

## Estrutura do evento

Alem do envelope comum, o bloco `data.execution` traz quem executou a alteracao e o que mudou.

### `data.execution.executor`

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `id` | inteiro | Identificador do usuario que realizou a alteracao |
| `name` | texto | Nome do usuario. Alteracoes automaticas chegam identificadas como acao do sistema |

### `data.execution.change`

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `field_id_dad` | inteiro | Identificador do campo personalizado. **Estavel entre formularios** |
| `field_id_dad_con` | inteiro | Identificador da ocorrencia do campo dentro de um formulario especifico |
| `field_description` | texto | Nome amigavel do campo, conforme configurado no formulario |
| `new_value` | texto | Novo valor do campo |

!!! tip "Qual identificador usar no seu mapeamento"
    Use **`field_id_dad`** para mapear o campo no seu sistema. Ele identifica a definicao do campo
    e permanece o mesmo ainda que o campo seja utilizado em varios formularios diferentes.

    **`field_id_dad_con`** identifica a ocorrencia daquele campo dentro de um formulario
    especifico. O mesmo campo, usado em dois formularios, tem um `field_id_dad` unico e dois
    `field_id_dad_con` distintos. Use-o quando precisar diferenciar a origem da alteracao.

---

## Como interpretar `new_value`

Este e o ponto que mais exige atencao ao construir o consumidor.

### O valor chega sempre como texto

`new_value` e transmitido como **texto (string)**, mesmo para campos numericos, de data ou
booleanos. Converta o valor conforme o tipo esperado para aquele campo no seu sistema.

| Tipo do campo | Formato recebido | Exemplo |
|---------------|------------------|---------|
| Texto | Texto livre | `"Contrato renovado"` |
| Numerico | Texto sem zeros a direita desnecessarios, ponto como separador decimal | `"7.5"` |
| Booleano | `"true"` ou `"false"` como texto | `"true"` |
| Data | `AAAA-MM-DD HH:MM:SS` | `"2026-09-11 14:32:10"` |
| Lista / selecao | Rotulo exibido, nao o codigo interno | `"Faturamento"` |
| Lista com multipla selecao | Rotulos separados por virgula e espaco | `"Faturamento, Portal do Cliente"` |

!!! note "Formato das datas"
    Campos de data usam `AAAA-MM-DD HH:MM:SS`, sem fuso horario — formato **diferente** do campo
    `timestamp` do envelope, que segue ISO 8601 com offset. Trate os dois separadamente ao
    converter datas.

### Campos de lista trazem o rotulo, nao o codigo

Para campos de selecao, `new_value` traz o **rotulo exibido ao usuario**, nao o codigo interno da
opcao. Isso torna o valor diretamente utilizavel para exibicao e notificacao.

!!! warning "Multipla selecao e rotulos com virgula"
    Quando o campo permite selecionar mais de uma opcao, os rotulos chegam concatenados e
    separados por virgula e espaco:

    ```json
    "new_value": "Faturamento, Portal do Cliente"
    ```

    Se algum rotulo configurado no seu ambiente **contiver uma virgula**, a separacao se torna
    ambigua e nao e possivel reconstruir a lista com seguranca a partir do texto.

    Quando a precisao for critica, use o evento como sinal e consulte os dados adicionais da
    requisicao pela API, utilizando o `data.ticket.id` recebido.

### Campos esvaziados

Um campo que tenha sido limpo pode chegar como `null` ou como texto vazio (`""`), dependendo do
tipo do campo. Trate as duas formas como ausencia de valor.

!!! warning "Este evento nao traz o valor anterior"
    O bloco `change` informa apenas o valor resultante (`new_value`). Diferente do
    [Evento de Dado Basico](dado-basico.md), que traz os dois lados da alteracao, aqui nao ha
    valor anterior.

    Se o seu fluxo depende de comparar antes e depois, o consumidor precisa manter o ultimo valor
    conhecido de cada campo no proprio armazenamento. Ao planejar isso, considere que:

    - **O primeiro evento de um campo chega sem base de comparacao.** Para ter um ponto de partida,
      consulte os dados adicionais da requisicao pela API antes de comecar a processar eventos
      daquele campo.
    - **Um evento nao entregue nao e reenviado** depois que as tentativas se esgotam (veja as
      [garantias de entrega](index.md)). O estado local pode divergir sem aviso — reconcilie
      periodicamente pela API.

---

## Exemplo: campo de lista com multipla selecao

```json
{
  "event_id": "8f3c1d2e-5a7b-4e09-9c11-2b6d4a8f0e33",
  "event_type": "ticket.data.additionaldatachanged",
  "event_version": "1.0",
  "timestamp": "2026-09-11T14:32:10-03:00",
  "source": "Bdesk/Tickets",
  "data": {
    "ticket": {
      "id": 184552,
      "title": "Falha intermitente na emissao de boletos",
      "status": "Em Atendimento"
    },
    "execution": {
      "executor": {
        "id": 4471,
        "name": "Maria Silva"
      },
      "change": {
        "field_id_dad": 318,
        "field_id_dad_con": 2907,
        "field_description": "Sistemas Impactados",
        "new_value": "Faturamento, Portal do Cliente"
      }
    },
    "customer_context": {
      "client_id": "sistema-externo",
      "client_name": "Sistema Externo Ltda"
    }
  }
}
```

## Exemplo: campo numerico

```json
{
  "event_id": "5b9e0a72-1c84-4f36-8e27-3d5a1b6c9f40",
  "event_type": "ticket.data.additionaldatachanged",
  "event_version": "1.0",
  "timestamp": "2026-09-11T16:05:41-03:00",
  "source": "Bdesk/Tickets",
  "data": {
    "ticket": {
      "id": 184552,
      "title": "Falha intermitente na emissao de boletos",
      "status": "Em Atendimento"
    },
    "execution": {
      "executor": {
        "id": 4471,
        "name": "Maria Silva"
      },
      "change": {
        "field_id_dad": 91,
        "field_id_dad_con": 1440,
        "field_description": "Horas Estimadas",
        "new_value": "7.5"
      }
    },
    "customer_context": {
      "client_id": "sistema-externo",
      "client_name": "Sistema Externo Ltda"
    }
  }
}
```

---

## Dicas de consumo

- **Mapeie por `field_id_dad`.** Monte a correspondencia entre os campos do BDesk e os campos do
  seu sistema usando esse identificador, que nao muda entre formularios.
- **Converta `new_value` conforme o tipo esperado.** O campo chega sempre como texto; a conversao
  para numero, data ou booleano e responsabilidade do consumidor.
- **Seja tolerante com campos desconhecidos.** Novos campos personalizados podem ser criados no
  BDesk a qualquer momento e passarao a gerar eventos. Ignore ou registre em log os
  `field_id_dad` que voce ainda nao trata, em vez de falhar.
- **Trate `null` e texto vazio da mesma forma** ao interpretar um campo esvaziado.
- **Espere varios eventos por operacao.** O preenchimento de um formulario com multiplos campos
  personalizados gera um evento por campo, sem ordem garantida entre eles.

---

## Proximos Passos

- [Evento de Dado Basico](dado-basico.md) — alteracoes nos campos padrao da requisicao
- [Evento de Acao](acao.md) — acoes de workflow executadas na requisicao
- [Evento de Participante](participante.md) — mudancas na composicao de participantes
- [Configurar o Recebimento de Eventos](configuracao.md) — como habilitar o envio
