# # ADR-010 — Adoção do Saga Pattern Coreografado

## Status

Aceito.

## Contexto

O sistema **Lata Velha** está sendo decomposto em três microsserviços independentes: `os-service`, `inventory-service` e `billing-service`, cada um responsável por seus próprios dados.

O ciclo de uma ordem de serviço (OS) envolve operações distribuídas, como aprovação do orçamento, reserva e consumo de peças, finalização do reparo e cobrança pelo Mercado Pago. Como essas operações não compartilham uma transação de banco de dados, é necessário assegurar a continuidade do processo e tratar falhas parciais sem comprometer a consistência do negócio.

## Decisão

Adotar o **Saga Pattern coreografado**, no qual cada microsserviço executa transações locais e publica eventos de negócio que desencadeiam as próximas etapas, **sem um orquestrador central**.

A troca de eventos será realizada com **Amazon SNS** (publicação e distribuição), **Amazon SQS** (filas por consumidor) e **dead-letter queues — DLQ** (mensagens que excederem as tentativas de processamento). Chamadas REST permanecem para consultas e operações que exigem resposta imediata.

Em linhas gerais, após a aprovação do orçamento pelo `os-service`, o `inventory-service` reserva as peças necessárias. A execução ocorre com a disponibilidade confirmada. Depois do reparo e da confirmação do consumo de estoque, a OS é finalizada e o `billing-service` emite a cobrança pelo **Mercado Pago Checkout Pro**. A confirmação do pagamento, recebida e validada por webhook, gera um evento que permite ao `os-service` liberar a entrega do veículo.

A solução adotará **consistência eventual** e **compensações de negócio**, como liberação de reservas em cancelamentos, manutenção da OS pendente quando faltar estoque e reembolso quando um pagamento confirmado precisar ser desfeito. Para evitar inconsistências em falhas e reentregas, serão utilizados **Transactional Outbox**, processamento idempotente, identificadores de correlação, retentativas e monitoramento das DLQs. A conclusão de um reparo físico não poderá ser revertida automaticamente.

## Alternativas consideradas

- **Saga orquestrada:** facilitaria a centralização das regras e o acompanhamento do processo, porém exigiria um componente coordenador e criaria dependência adicional entre os serviços.
- **Transações distribuídas (2PC):** ofereceriam coordenação transacional entre participantes compatíveis, mas aumentariam o acoplamento e não resolveriam adequadamente as etapas externas e de longa duração, como o pagamento no Mercado Pago.

## Consequências

### Positivas

- **Baixo acoplamento:** cada microsserviço reage aos eventos de seu interesse e evolui de forma independente.
- **Resiliência:** o uso de filas permite absorver indisponibilidades temporárias e reprocessar eventos.
- **Escalabilidade:** cada serviço e seus consumidores podem ser dimensionados conforme a carga.
- **Responsabilidades claras:** cada domínio mantém controle sobre suas transações e compensações.

### Negativas

- **Maior complexidade de rastreamento:** o processo completo fica distribuído entre serviços e eventos, exigindo observabilidade e correlação.
- **Consistência eventual:** os estados dos serviços podem divergir temporariamente até o processamento das mensagens.
- **Tratamento de falhas mais elaborado:** exige idempotência, controle de eventos duplicados ou fora de ordem, retentativas e DLQs.
- **Compensações limitadas:** operações físicas e pagamentos confirmados requerem ações de negócio específicas, nem sempre reversíveis automaticamente.
