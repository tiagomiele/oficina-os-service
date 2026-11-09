# Índice resumido de evidências e documentações — FIAP Tech Challenge Fase 4 | Oficina

Este índice reúne em um só lugar a documentação técnica, os requisitos obrigatórios e as principais evidências da Oficina Fase 4.

## Requisitos obrigatórios e evidências

O documento abaixo liga cada requisito obrigatório à sua implementação técnica e às evidências de execução.

Leia-o por inteiro para entender a solução completa. Ele é o guia principal de evidências da Fase 4.

- [Consultar os requisitos obrigatórios e as evidências](README-requisitos-obrigatorios-fase4.md)

## Perguntas e respostas do enunciado

As 40 questões do enunciado da Fase 4, cada uma com uma resposta curta e a referência ao projeto.

- [Consultar as perguntas e respostas da Fase 4](README-perguntas-respostas-fase4.md)

## READMEs dos seis repositórios

Cada repositório documenta seu propósito, tecnologias, arquitetura, estrutura de pastas, pré-requisitos, validações, execução e implantação.

| Projeto | README |
|---|---|
| OS Service (orquestrador da Saga) | [README do OS Service](https://github.com/tiagomiele/oficina-os-service/blob/main/README.md) |
| Orçamento e Pagamento | [README do Orçamento e Pagamento](https://github.com/tiagomiele/oficina-orcamento-pagamento-service/blob/main/README.md) |
| Diagnóstico e Reparo | [README do Diagnóstico e Reparo](https://github.com/tiagomiele/oficina-diagnostico-reparo-service/blob/main/README.md) |
| Auth Serverless | [README do Auth](https://github.com/tiagomiele/oficina-auth-serverless/blob/main/README.md) |
| Database Infra | [README do Database](https://github.com/tiagomiele/oficina-database-infra/blob/main/README.md) |
| Kubernetes Infra | [README do Kubernetes](https://github.com/tiagomiele/oficina-kubernetes-infra/blob/main/README.md) |

## Vídeo de demonstração

O vídeo mostra o fluxo completo de uma OS, a Saga com tratamento de falhas, o deploy automatizado e o monitoramento dos fluxos distribuídos.

- Vídeo de demonstração da Fase 4: link a incluir depois da publicação, com duração de até 15 min.

## Documentação complementar

As pastas abaixo reúnem os diagramas de arquitetura, o contrato da Saga, os contratos de API e a documentação de cada componente.

- [Saga orquestrada: estratégia, fluxo e compensações](docs/saga/README.md)
- [Contrato das mensagens da Saga](docs/saga/contrato-mensagens.md)
- [Diagramas de arquitetura e da Saga](docs/assets/)
- [Cenários BDD da Saga](bdd/src/test/resources/features/saga.feature)
- [Collection Postman/Newman do fluxo E2E](tests/postman/oficina-weeks4-5.postman_collection.json)
- [Arquitetura do Auth](https://github.com/tiagomiele/oficina-auth-serverless/blob/main/docs/arquitetura.md) · [Contrato OpenAPI do Auth](https://github.com/tiagomiele/oficina-auth-serverless/blob/main/docs/openapi/oficina-auth.yaml)
- [Modelo relacional](https://github.com/tiagomiele/oficina-database-infra/blob/main/docs/modelo-relacional.md) · [ADRs dos bancos](https://github.com/tiagomiele/oficina-database-infra/tree/main/docs/adr)
- [Infraestrutura e observabilidade](https://github.com/tiagomiele/oficina-kubernetes-infra/blob/main/docs/infraestrutura-observabilidade.md) · [Fluxo de CI/CD](https://github.com/tiagomiele/oficina-kubernetes-infra/blob/main/docs/cicd.md)

## Bootstrap das aplicações

Os guias abaixo mostram como configurar, implantar e destruir os seis projetos na AWS.

- [Bootstrap da solução na AWS](docs/bootstrap-aws-do-zero-fase4.md)
- [Bootstrap de uma única vez: provisionar e destruir um ambiente com um único script](README-bootstrap-unica-vez.md)
