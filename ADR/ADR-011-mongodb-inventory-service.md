# ADR-011 — Adoção do MongoDB no Inventory Service

## Status

Aceito.

## Contexto

Na decomposição do sistema **Lata Velha** em microsserviços, o `inventory-service` será responsável pelo **cadastro de peças e serviços, definição de preços de referência, disponibilidade, reservas e movimentações de estoque**.

Os itens do catálogo apresentam características heterogêneas. Peças mecânicas podem ter fabricante, modelo compatível, código do fornecedor e especificações técnicas; serviços podem possuir descrição, categoria, tempo estimado e preço de mão de obra. Essa diversidade gera campos opcionais e estruturas que evoluem conforme novos tipos de itens são cadastrados.

Além da flexibilidade do catálogo, o serviço precisa preservar regras críticas de estoque, evitando reservas duplicadas, saldos negativos e inconsistências em operações concorrentes.

## Decisão

Adotar o **MongoDB** como banco de dados exclusivo do `inventory-service`, utilizando seu **modelo orientado a documentos** para representar peças, serviços e respectivos atributos específicos.

O `inventory-service` será a **única fonte de verdade para catálogo e estoque**. O `os-service` consultará preços e disponibilidade por APIs e receberá eventos relacionados a reservas e consumo por **Amazon SNS/SQS**. Valores de itens já incorporados a um orçamento aprovado serão preservados no `os-service`, independentemente de alterações posteriores no catálogo.

## Alternativas consideradas

- **PostgreSQL (relacional):** oferece excelente integridade transacional e consultas estruturadas, mas exigiria modelagem com diversas tabelas auxiliares, campos opcionais ou uso extensivo de JSON para acomodar as diferenças entre peças e serviços.
- **DynamoDB:** proporciona escalabilidade gerenciada e baixa latência, porém exige uma modelagem fortemente orientada a padrões de acesso e apresenta restrições distintas para consultas flexíveis do catálogo.

O MongoDB foi escolhido pelo equilíbrio entre **flexibilidade do modelo documental, capacidade de consulta e suporte a operações atômicas**, mantendo a responsabilidade pelas invariantes de estoque na aplicação.

## Consequências

### Positivas

- **Flexibilidade de modelagem:** peças e serviços com atributos diferentes podem coexistir sem alterações frequentes de esquema relacional.
- **Evolução do catálogo:** novas categorias e especificações podem ser introduzidas com menor impacto no modelo de persistência.
- **Autonomia do domínio:** catálogo, preços de referência e estoque permanecem sob controle de um único microsserviço.
- **Consultas orientadas a documentos:** informações de um item podem ser recuperadas de forma agregada, conforme o desenho do documento e dos índices.

### Negativas

- **Maior responsabilidade pela integridade:** consistência de saldo, concorrência e reservas exigem regras explícitas, atualizações condicionais e testes específicos.
- **Risco de esquemas inconsistentes:** a flexibilidade exige validação de documentos, convenções e evolução controlada do modelo.
- **Complexidade de transações multidocumento:** movimentações que envolvem múltiplos documentos podem demandar transações locais e maior cuidado com desempenho.
- **Custo operacional e de capacitação:** a equipe precisará administrar, monitorar, proteger e realizar backups do MongoDB, além dos bancos PostgreSQL dos demais serviços.
