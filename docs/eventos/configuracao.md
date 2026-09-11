# Configurar o Recebimento de Eventos

## O que voce vai aprender

Como solicitar a habilitacao do Envio de Eventos para a sua organizacao, quais informacoes
precisam ser passadas ao suporte BDesk, quais requisitos o seu endpoint deve atender e como
validar que a integracao esta funcionando.

---

## Pre-requisitos

- **Leitura previa da visao geral** — veja [Envio de Eventos](index.md) para entender o
  formato da mensagem e as garantias de entrega.
- **Endpoint HTTP ja disponivel** (ainda que em ambiente de homologacao) para receber os
  eventos.
- **Provedor OAuth2 configurado do seu lado**, com credenciais que possam ser compartilhadas
  com o suporte BDesk.
- **Lista dos formularios** cujas requisicoes devem gerar eventos.

---

## Passo a Passo

### Passo 1: Abrir um chamado para o suporte BDesk

O Envio de Eventos **nao e auto-atendimento**: ele e habilitado pela equipe BDesk mediante
solicitacao. Nao ha tela de configuracao no produto nem endpoint de API para ativar ou alterar
o envio.

!!! note "Quem faz o que"
    **Voce** disponibiliza o endpoint e as credenciais; **a equipe BDesk** configura o envio no
    ambiente da sua organizacao. Qualquer alteracao posterior (nova URL, novas credenciais,
    inclusao de formularios) tambem passa por chamado.

Abra um chamado com o suporte BDesk pedindo a habilitacao do **Envio de Eventos** e informe os
dados do Passo 2.

---

### Passo 2: Informar os dados da integracao

Reuna as informacoes abaixo antes de abrir o chamado.

#### Obrigatorio

| Informacao | Descricao | Exemplo |
|---|---|---|
| **URL do endpoint** | Endereco que recebera os `POST` com os eventos | `https://seu-sistema.exemplo.com.br/webhooks/bdesk` |
| **URL do token OAuth2** | Endpoint do seu provedor onde o BDesk obtera o token de acesso | `https://auth.exemplo.com.br/oauth2/token` |
| **Credenciais OAuth2** | `client_id` e o segredo correspondente, ou usuario/senha, conforme o fluxo do seu provedor | — |
| **Formularios** | Quais formularios devem gerar eventos | "Solicitacao de Acesso", "Incidente de TI" |

#### Opcional (filtros)

| Informacao | Descricao | Efeito se omitido |
|---|---|---|
| **Tipos de evento** | Quais dos quatro tipos voce quer receber: Acao, Dado Basico, Dado Adicional ou Participante | Todos os tipos sao enviados |
| **Acoes especificas** | Restringe os eventos de Acao a determinadas acoes (ex: apenas encerramento) | Todas as acoes geram evento |
| **Campos especificos** | Restringe os eventos de Dado Basico / Dado Adicional a determinados campos | Todos os campos monitorados geram evento |
| **Papeis especificos** | Restringe os eventos de Participante a determinados papeis (ex: apenas o responsavel) | Todas as mudancas de participante geram evento |

!!! tip "Comece amplo, depois refine"
    Em ambiente de homologacao, vale habilitar todos os tipos de evento para ver o que a sua
    operacao realmente produz. Depois de mapear o volume, peca os filtros que reduzem o ruido.

---

### Passo 3: Indicar os formularios — filtro obrigatorio

!!! warning "Sem formularios, nenhum evento e enviado"
    O filtro por formulario e **obrigatorio**. Se nenhum formulario for indicado na
    configuracao, o BDesk **nao envia evento algum** — mesmo que a URL, as credenciais e os
    demais filtros estejam corretos. Este e o motivo mais comum de "configurei e nao recebo
    nada".

Ao listar os formularios, lembre-se de que:

- Requisicoes abertas em formularios **fora** da lista nunca geram eventos.
- Incluir um novo formulario no escopo depois exige um novo chamado ao suporte.
- Se a sua organizacao usa formularios distintos para o mesmo processo (por exemplo, um por
  unidade de negocio), inclua **todos** eles.

---

### Passo 4: Preparar o endpoint que recebera os eventos

O endpoint informado no Passo 2 precisa atender aos requisitos abaixo.

| Requisito | Detalhe |
|---|---|
| **Aceitar `POST` com JSON** | Corpo `application/json`, um evento por requisicao |
| **Responder HTTP 200 rapidamente** | Qualquer outro codigo ou timeout conta como falha e agenda retentativa. Enfileire e processe em segundo plano |
| **Ser idempotente** | A entrega e *at-least-once*: o mesmo evento pode chegar mais de uma vez. Deduplique por `event_id` |
| **Estar acessivel pela internet** | O BDesk precisa alcancar o endereco a partir do ambiente onde esta hospedado. Endpoints em rede interna nao funcionam sem exposicao publica |
| **Ter TLS valido** | Certificado HTTPS emitido por autoridade confiavel e dentro da validade. Certificados autoassinados ou expirados fazem a entrega falhar |
| **Aceitar `Authorization: Bearer`** | O BDesk obtem o token no seu provedor OAuth2 e o envia em cada requisicao |

!!! tip "Responda 200 tambem ao que voce ignora"
    Se um evento nao interessa ao seu sistema, descarte-o e responda 200 mesmo assim.
    Responder erro apenas consome as tentativas de entrega sem beneficio.

---

## Como testar a integracao

Apos o suporte confirmar a habilitacao, valide a integracao nesta ordem.

### 1. Receber o primeiro evento

Abra uma requisicao em um dos formularios configurados e execute uma mudanca coberta pelos
filtros (por exemplo, altere a prioridade para gerar um evento de Dado Basico).

Confirme no log do seu endpoint que chegou um `POST` com:

- Cabecalho `Authorization: Bearer ...` valido.
- Corpo JSON com `event_id`, `event_type`, `event_version`, `timestamp`, `source` e `data`.
- O `ticket.id` correspondente a requisicao que voce alterou.

!!! note "Nao espere entrega instantanea"
    O envio e assincrono e em lote. Alguns minutos de atraso entre a mudanca e a chegada do
    evento sao normais — aguarde antes de concluir que a integracao falhou.

### 2. Conferir a deduplicacao por `event_id`

Reenvie manualmente ao seu proprio endpoint um payload ja processado, com o **mesmo**
`event_id`. O comportamento esperado e:

- O sistema responde HTTP 200.
- **Nenhum** registro duplicado e criado, nenhuma acao e disparada novamente.

Este teste simula exatamente o que acontece quando o BDesk reenvia um evento.

### 3. Checar o comportamento de retentativa

Configure temporariamente o endpoint para responder um erro (por exemplo, HTTP 500) e provoque
uma nova mudanca em uma requisicao. Em seguida, volte o endpoint ao normal.

O esperado e:

- A primeira tentativa falha.
- Aproximadamente 10 minutos depois, o BDesk tenta de novo — desta vez com sucesso.
- O evento chega com o **mesmo** `event_id` da tentativa que falhou.

!!! warning "Limite de 3 tentativas"
    Se o endpoint ficar indisponivel por mais de aproximadamente 30 minutos, as 3 tentativas
    daquele evento se esgotam e ele **nao sera mais reenviado**. Nao ha reprocessamento
    automatico posterior — planeje janelas de manutencao considerando isso.

---

## Solucao de Problemas

| Sintoma | Causa provavel | Como resolver |
|---|---|---|
| **Nao recebo nenhum evento** | Nenhum formulario foi indicado na configuracao (filtro obrigatorio); ou os filtros de tipo/acao/campo estao restritivos demais; ou o endpoint nao e alcancavel pela internet; ou o TLS e invalido | Confirme com o suporte quais formularios e filtros estao ativos. Teste o endpoint a partir de uma rede externa e verifique a validade do certificado |
| **Recebo o mesmo evento duas vezes** | Comportamento **esperado** — a entrega e *at-least-once* | Deduplique por `event_id`, que e estavel entre reenvios. Nao trate como erro |
| **Eventos chegam com atraso de minutos** | Comportamento **esperado** — o envio e assincrono e processado em lote | Nao construa fluxos que dependam de entrega instantanea. Se o atraso for de horas, abra chamado |
| **Eventos param de chegar depois de um tempo** | O endpoint deixou de responder HTTP 200. Apos 3 falhas consecutivas, as tentativas daquele evento sao interrompidas | Verifique se o endpoint responde 200 (e nao 401, 403, 500, redirecionamento ou timeout). Cheque tambem a validade das credenciais OAuth2 e do certificado TLS |
| **Eventos chegam fora de ordem** | Comportamento **esperado** — a ordem de entrega nao e garantida | Ordene pelo campo `timestamp`, nao pela ordem de chegada |
| **Recebo eventos de formularios que nao pedi** | O escopo configurado e mais amplo que o esperado | Peca ao suporte o ajuste da lista de formularios. Enquanto isso, filtre no seu lado e responda 200 aos que ignorar |

---

## Proximos Passos

- [Envio de Eventos](index.md) — envelope comum, garantias de entrega e boas praticas do consumidor
- [Evento de Acao](acao.md) — acoes de fluxo de trabalho executadas na requisicao
- [Evento de Dado Basico](dado-basico.md) — mudancas em campos padrao da requisicao
- [Evento de Dado Adicional](dado-adicional.md) — mudancas em campos personalizados do formulario
- [Evento de Participante](participante.md) — mudancas na composicao de participantes
