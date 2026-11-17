# RFC-001 — Decomposição do Monólito Lata Velha em Microsserviços

| Campo | Definição |
|---|---|
| Status | Proposta |
| Data | 16/11/2026 |
| Sistema | Lata Velha — Gestão de Oficina Mecânica |
| Escopo | Limites dos serviços, comunicação, consistência e migração |
| Decisões relacionadas | ADR de MongoDB no `inventory-service`; ADR de Saga coreografada (a elaborar) |

## 1. Contexto e motivação

O Lata Velha concentra em um monólito o cadastro de clientes e veículos, a gestão das ordens de serviço (OS), o catálogo e estoque de peças, a execução dos reparos e o acompanhamento do atendimento. O crescimento da operação e a introdução do pagamento digital exigem maior autonomia de evolução e implantação dessas capacidades.

Esta RFC propõe a decomposição em **três microsserviços orientados a domínio**, com dados exclusivos, deploy independente e integração por APIs e eventos. Os objetivos são reduzir o acoplamento, isolar falhas, permitir escalabilidade por capacidade de negócio e manter a rastreabilidade do ciclo completo da OS.

## 2. Decisão e Bounded Contexts

A divisão segue os princípios de Domain-Driven Design (DDD). Um *Bounded Context* define a propriedade do modelo de negócio; não precisa corresponder individualmente a um microsserviço.

| Microsserviço | Domínio e responsabilidades | Persistência proposta |
|---|---|---|
| **`os-service`** | Gestão da OS, usuários, clientes e veículos; diagnóstico; composição e aprovação do orçamento; execução dos serviços; estados operacionais e entrega do veículo. | **Amazon RDS PostgreSQL** |
| **`inventory-service`** | Catálogo de peças, insumos e serviços; atributos variáveis dos itens; preços de referência; disponibilidade por filial, reserva, consumo e movimentação de estoque. | **MongoDB**, executado no cluster conforme a arquitetura proposta |
| **`billing-service`** | Cobrança de OS finalizada; criação de preferência no **Mercado Pago Checkout Pro**; disponibilização do checkout; recebimento e validação de webhooks; confirmação, conciliação e histórico de pagamentos. | **Amazon RDS PostgreSQL** |

Cada serviço é proprietário exclusivo de seu banco e não acessa tabelas ou coleções de outros serviços. Identificadores de OS, filial, itens e cobrança são compartilhados por contratos de integração, sem chaves estrangeiras entre bancos.

### Limites de responsabilidade

- O **catálogo de peças e os preços de referência** pertencem ao inventário; o **orçamento aprovado e seus valores congelados** pertencem à OS. Alterações posteriores no catálogo não modificam valores já aprovados.
- O `billing-service` **não calcula orçamentosm, apenas aprova/prova**: o serviço da OS se encarrega de criar os orçamentos, permitindo que o cliente aprove ou reprove.
- O status **`FINALIZADA`** indica conclusão técnica. A **entrega** depende da confirmação do pagamento e da efetiva retirada do veículo. O estado financeiro pertence ao `billing-service`.
- A consulta prévia de disponibilidade não equivale à reserva: a **reserva efetiva ocorre após a aprovação** do orçamento e antes do início da execução que necessita das peças.

A escolha do **MongoDB** considera a diversidade de atributos e estruturas de peças e serviços. Modelo documental, índices, integridade de reservas e comparação com alternativas serão detalhados em uma **ADR específica**, evitando duplicação nesta RFC.

## 3. Infraestrutura e integrações

A solução será executada na **AWS**, com microsserviços em namespaces separados no **Amazon EKS** e bancos próprios. O tráfego HTTP externo entra pelo **Amazon API Gateway**, com autorização por **AWS Lambda/JWT Authorizer**, seguindo por **VPC Link** e **Application Load Balancer interno** até os serviços. O webhook do Mercado Pago é encaminhado ao `billing-service` por rota pública controlada, com validação adicional da notificação.

As integrações combinam:

- **REST síncrono:** operações externas de cadastro e acompanhamento, consulta imediata de catálogo/disponibilidade e obtenção do link do checkout.
- **Amazon SNS:** publicação de eventos de domínio e distribuição aos consumidores interessados.
- **Amazon SQS:** filas próprias de cada consumidor, desacoplando o produtor da disponibilidade dos demais serviços.
- **Dead-letter queues (DLQ):** isolamento de mensagens que excedam a política de tentativas, para inspeção e reprocessamento controlado.

Cada serviço terá pipeline independente no **GitHub Actions**, imagem no **Amazon ECR** e provisionamento via **Terraform** (estado remoto no S3). **CloudWatch** apoiará logs e alarmes; **Datadog**, com instrumentação **OpenTelemetry**, permitirá observar requisições e eventos entre os serviços.

## 4. Fluxo integrado da ordem de serviço

1. **Abertura e diagnóstico:** o `os-service` registra cliente, veículo e OS; o mecânico identifica serviços e peças necessários. Para compor o orçamento, consulta informações e preços de referência no `inventory-service` e registra os valores na OS.
2. **Aprovação e estoque:** o `os-service` registra o aceite do orçamento e publica um evento no SNS. O `inventory-service`, consumidor via SQS, verifica e reserva as peças. Quando há estoque insuficiente, informa a pendência e a execução dependente dessas peças permanece bloqueada até a regularização.
3. **Execução e baixa:** confirmada a reserva necessária, os serviços são executados. Ao concluir os reparos, o `os-service` solicita por evento a confirmação do consumo das peças reservadas. Após o inventário confirmar a baixa, a OS é marcada como `FINALIZADA`.
4. **Emissão da cobrança:** o `os-service` publica o evento de OS finalizada com valor aprovado e referência do orçamento. O `billing-service` cria a cobrança de modo idempotente, solicita uma preferência ao Mercado Pago Checkout Pro e disponibiliza a URL de pagamento para o cliente (meios conforme habilitação da conta).
5. **Confirmação e entrega:** o Mercado Pago notifica o `billing-service` por webhook. Após validar a notificação e consultar o status no provedor, o serviço publica a aprovação do pagamento via SNS/SQS. O `os-service` registra a quitação operacional e autoriza a retirada; a OS passa a `ENTREGUE` quando o veículo é efetivamente retirado.

Os nomes de eventos serão formalizados em contratos versionados. Como referência, o fluxo pode utilizar `OrcamentoAprovado`, `EstoqueReservado`, `ConsumoConfirmado`, `OSFinalizada`, `CobrancaEmitida` e `PagamentoConfirmado`.

## 5. Saga coreografada e consistência distribuída

As etapas que atravessam serviços serão tratadas por **Saga Pattern coreografado**: cada participante confirma sua transação local e reage aos eventos recebidos, **sem orquestrador central**. Essa estratégia permite consistência eventual entre OS, inventário e cobrança. A justificativa comparativa com orquestração, seus trade-offs e o detalhamento dos contratos ficam para a **ADR de Saga coreografada**.

Para garantir confiabilidade operacional:

- **Transactional Outbox:** persistir a mudança local e o evento pendente na mesma transação; um publicador entrega o evento ao SNS posteriormente.
- **Idempotência:** evitar reserva, consumo e cobrança duplicados em caso de redelivery de SQS ou webhook repetido.
- **Controle de estados:** cada consumidor valida o identificador, a versão e a transição permitida da OS, reserva ou cobrança; eventos atrasados não podem sobrescrever estados mais recentes.
- **Retentativas e DLQ:** falhas transitórias seguem uma política limitada de novas tentativas; falhas persistentes exigem análise e reprocessamento seguro.
- **Rastreabilidade e reconciliação:** IDs de correlação acompanham as mensagens; rotinas verificam divergências entre OS, reservas, cobranças e situação real do Mercado Pago.

### Compensações relevantes

| Falha ou exceção | Comportamento esperado |
|---|---|
| Orçamento recusado ou OS cancelada antes da execução | Não iniciar o reparo e liberar reservas eventualmente existentes. |
| Estoque parcial ou indisponível | Registrar pendência, aguardar reposição e impedir execução que dependa das peças faltantes. |
| Falha na confirmação de consumo | Retentar e conciliar; não emitir cobrança como se a finalização estivesse confirmada. |
| Falha na emissão da preferência de pagamento | Manter cobrança pendente e retentar com identificação idempotente, sem duplicar lançamentos. |
| Pagamento recusado, expirado ou ainda pendente | Manter OS concluída e não entregue; permitir nova tentativa conforme política de cobrança. |
| Webhook duplicado ou fora de ordem | Validar autenticidade, consultar estado atual no Mercado Pago e aplicar somente transições financeiras válidas. |
| Pagamento aprovado que precisa ser desfeito | Iniciar reembolso/estorno conforme regras do provedor e registrar o resultado; não apagar o pagamento. |

**Compensações não equivalem a rollback físico.** Peças já instaladas e reparos concluídos não podem ser desfeitos automaticamente; cancelamentos posteriores requerem regras administrativas, ajustes de estoque e, quando cabível, estorno financeiro.

## 6. Estratégia de migração

A decomposição seguirá uma abordagem **incremental (Strangler Fig)**:

1. Estabelecer contratos de APIs e eventos, infraestrutura AWS, pipelines independentes, monitoramento e bancos segregados.
2. Extrair catálogo e estoque para `inventory-service`, migrando dados e reconciliando saldos e reservas.
3. Extrair a gestão da OS para `os-service`, mantendo históricos, vínculos e estados e substituindo dependências internas por integrações definidas.
4. Introduzir `billing-service` e validar Checkout Pro e webhooks em ambiente de testes antes de habilitar cobranças reais.
5. Ativar progressivamente os fluxos entre os três serviços e remover rotas e escritas legadas após validação.

Durante a transição, **cada agregado terá um único proprietário de escrita**. Ordens em andamento deverão ser concluídas no fluxo antigo ou migradas mediante procedimento controlado; a mesma OS não pode ser modificada simultaneamente pelo monólito e pelo serviço extraído.

## 7. Riscos e consequências

| Risco / custo | Mitigação |
|---|---|
| Complexidade operacional e de observabilidade maior que no monólito | Pipelines por serviço, CloudWatch, Datadog e correlação ponta a ponta. |
| Consistência eventual e mensagens repetidas ou fora de ordem | Outbox, idempotência, estados explícitos, filas SQS/DLQ e reconciliação. |
| Concorrência na reserva de peças | Escritas atômicas condicionais e verificação de saldos no MongoDB. |
| Indisponibilidade ou atraso de confirmação do Mercado Pago | Cobrança pendente, retentativas controladas, conciliação e bloqueio de entrega até confirmação. |
| Divergência de dados durante a migração | Migração faseada, comparação de registros/saldos e definição clara de propriedade de escrita. |

**Consequências positivas:** autonomia de manutenção e implantação; escalabilidade específica por domínio; isolamento entre estoque, operação e pagamentos; inclusão de pagamento digital; maior auditabilidade dos processos.

**Consequências negativas:** aumento da infraestrutura e do esforço de testes, monitoramento, segurança, versionamento de contratos e tratamento de falhas distribuídas.

## 8. Decisões relacionadas e pontos em aberto

- **ADR — MongoDB no Inventory Service:** justificativa da persistência documental para peças e serviços, modelagem e mecanismos de consistência de estoque.
- **ADR — Saga Pattern Coreografado:** motivos da escolha, alternativas consideradas e governança de eventos e compensações.
- **Detalhamento posterior:** nomes e payloads definitivos dos eventos; políticas de timeout e redrive das DLQs; critérios de cancelamento e estorno; regras multi-filial e migração das OS já abertas.

---

**Situação:** proposta sujeita à validação da equipe antes de sua aceitação e implementação. Esta RFC apresenta decisões arquiteturais, não a garantia de que todos os fluxos descritos já estejam implementados.
