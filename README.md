# lata-velha-pos-tech-fiap-documentation

Repositório central de documentação do **Lata Velha**, sistema de gestão para oficinas mecânicas desenvolvido no contexto do **Tech Challenge da Pós-Tech FIAP**, ao longo das fases 1 a 4.

O projeto utiliza princípios de **Domain-Driven Design (DDD)** para organizar regras de negócio e evoluiu de uma aplicação monolítica para uma arquitetura distribuída em microsserviços. Seu escopo contempla o cadastro de clientes e veículos, a gestão de ordens de serviço, o catálogo e estoque de peças, os serviços da oficina e o processo de cobrança.

## Objetivo deste repositório

Centralizar a **documentação geral do sistema**, oferecendo uma visão consistente das decisões arquiteturais, dos domínios de negócio e da evolução do sistema ao longo das fases do projeto.

Aqui estão reunidos registros de decisões arquiteturais (**ADRs**), propostas e definições de arquitetura (**RFCs**), diagramas de arquitetura, modelos de dados, diagramas de sequência, fluxos de negócio e materiais de apoio à linguagem do domínio, como o dicionário de termos e os *storytellings*.

## Organização da documentação

O Lata Velha é composto por diferentes repositórios, dedicados a componentes como **infraestrutura, microsserviços e funções AWS Lambda**. Para evitar duplicação de informações, adotamos a seguinte divisão:

- **Este repositório (`documentation`):** decisões, modelos, fluxos e documentação que descrevem o sistema como um todo ou envolvem mais de um componente.
- **Repositórios dos componentes:** instruções de execução, configuração, APIs, implementação, testes e demais detalhes técnicos específicos de cada projeto.

Os documentos são organizados por assunto em diretórios próprios e podem refletir diferentes momentos da evolução da solução. Para conhecer uma decisão ou fluxo, navegue pela categoria correspondente.
