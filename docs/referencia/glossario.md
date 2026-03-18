# Glossário — Termos BDesk

Referência rápida dos termos usados na API BDesk e na documentação do usuário.

## Termos Gerais

| Termo | Descrição |
|-------|-----------|
| **Atividade** | Tipo de serviço dentro de uma área do catálogo. Cada atividade está associada a um formulário específico que define os campos da requisição. |
| **Cardápio** | Catálogo de serviços disponíveis para abertura de requisições. Organizado em áreas e atividades. Retornado pelo endpoint `GET /v1/cardapio`. |
| **Conjunto** | Grupo de campos adicionais em um formulário. Os dados adicionais de uma requisição são organizados por conjuntos. No JSON, `Conjuntos` é um objeto cujas chaves são os nomes dos conjuntos. |
| **Contexto** | Perspectiva do usuário em relação a uma requisição. Determina quais requisições são listadas. Valores possíveis: `Solicitei` (1), `Atendo` (2), `Acompanho` (3), `Administro` (4). |
| **Dado Adicional** | Campo personalizado em um formulário de requisição. Pode ser texto, número, data, seleção única, múltipla seleção, entre outros. Identificado por nome dentro de um conjunto. |
| **Direcionar** | Ação de encaminhar uma requisição para outro grupo de atendimento ou analista. Código de ação: `DIR`. |
| **Encerrar** | Ação de concluir e fechar uma requisição. Código de ação: `ENC`. |
| **Fluxo** | Sequência de etapas e ações definidas para um tipo de requisição. Define quais ações estão disponíveis em cada fase do atendimento. |
| **Formulário** | Modelo usado para abertura de requisições. Define os campos obrigatórios, dados adicionais, validações e o fluxo de atendimento associado. |
| **IC (Item de Configuração)** | Ativo de TI registrado no CMDB do BDesk. Pode ser um servidor, estação de trabalho, software, equipamento de rede ou qualquer outro ativo gerenciado. |
| **Papel** | Função de um participante em uma requisição (ex.: quem solicitou, quem está atendendo, quem está copiado). Veja a tabela de papéis abaixo. |
| **Requisição** | Chamado ou ticket no sistema BDesk. Pode estar aberta (em andamento) ou encerrada (concluída). Identificada por um número inteiro único (`IdRequisicao`). |
| **Solicitado** | Analista ou grupo responsável por atender a requisição. Papel ID: 3. |
| **Solicitante** | Pessoa para quem o serviço foi solicitado. Papel ID: 2. Pode ser diferente do registrador. |
| **Registrador** | Pessoa que registrou a requisição no sistema. Papel ID: 1. Em autoatendimento, coincide com o solicitante. |
| **Token** | Chave de autenticação obtida via `POST /v1/login/entrar`. Deve ser enviada em todas as chamadas no header `Authorization: Bearer <token>`. |

## Códigos de Ação

Ações de workflow são operações que avançam ou modificam o estado de uma requisição. São identificadas por códigos curtos enviados no campo `CdAcao`. Para detalhes de uso, consulte [Ações de Workflow](../guias/acoes-workflow.md).

| Código | Ação | Descrição resumida |
|--------|------|--------------------|
| `DIR` | Direcionar | Encaminha a requisição para outro grupo ou analista |
| `ENC` | Encerrar | Conclui e fecha a requisição |
| `ATR` | Atribuir | Atribui a requisição a um analista específico |
| `ATRR` | Atribuir Responsabilidade | Atribui responsabilidade a um participante |
| `ALTPRI` | Alterar Prioridade | Muda a prioridade da requisição |
| `ALTDES` | Alterar Descrição | Edita o texto da descrição da requisição |
| `RECAT` | Recategorizar | Muda a categoria/formulário da requisição |
| `DEVD` | Devolver Direto | Devolve a requisição ao solicitante sem encerrar |
| `AVAL` | Avaliar | Registra a avaliação de atendimento |
| `VINC` | Vincular | Vincula a requisição a outra requisição relacionada |
| `ENC_EXT` | Encerrar Externo | Encerra requisição integrada com sistema externo |
| `CANC_EXT` | Cancelar Externo | Cancela integração com sistema externo |

## Papéis (IDs)

O campo `IdPapel` identifica a função de um participante em uma requisição.

| ID | Papel | Descrição |
|----|-------|-----------|
| 1 | Registrador | Quem registrou a requisição |
| 2 | Solicitante | Para quem o serviço foi solicitado |
| 3 | Solicitado | Analista ou grupo responsável pelo atendimento |
| 9 | Copiado | Participante informado sobre o andamento, sem responsabilidade de atendimento |
| 12 | Sistema | Ações automáticas realizadas pelo sistema |
| 14 | Administrador do Processo | Gestor com permissões ampliadas sobre o fluxo |
| 21 | E-Mail | Participante adicionado via resposta de e-mail |

## Contextos de Listagem

O parâmetro `Contexto` (ou `IdContexto`) filtra as requisições conforme a perspectiva do usuário autenticado:

| Valor | Nome | Retorna |
|-------|------|---------|
| 1 | Solicitei | Requisições abertas pelo usuário |
| 2 | Atendo | Requisições em que o usuário é o analista responsável |
| 3 | Acompanho | Requisições em que o usuário está copiado |
| 4 | Administro | Requisições sob gestão do usuário (perfil gestor) |

## Formatos de Data

Todas as datas na API BDesk usam o formato ISO 8601:

```
YYYY-MM-DDTHH:MM:SS
```

Exemplos: `"2025-03-17T14:30:00"`, `"2025-12-01T00:00:00"`

Ao filtrar por período, use os parâmetros `DtInicio` e `DtFim` neste formato.
