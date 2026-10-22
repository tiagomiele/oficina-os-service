> **Tech Challenge Fase 4 — `oficina-os-service`**
>
> Serviço de Ordem de Serviço (OS): clientes, veículos, OS, catálogo, estoque, notas fiscais de fornecedor, financeiro, usuários, relatórios e orquestração da Saga.
>
> Base importada de [`fiap-tech-challenge-fase3-oficina-backend`](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend) (commit `9623b23`). O conteúdo abaixo ainda descreve a Fase 3, com as mudanças da Fase 4 listadas aqui.
>
> **Saga orquestrada (Fase 4):** o OS Service orquestra a Saga pelo status da OS, envia comandos e consome eventos pelo Amazon SQS, aplica as compensações e garante idempotência por `messageId`. As rotas de enviar para aprovação, concluir reparo, aprovar, rejeitar e confirmar pagamento saíram deste serviço e passaram para Diagnóstico e Reparo e Orçamento e Pagamento. Contrato das mensagens: [`docs/saga/contrato-mensagens.md`](docs/saga/contrato-mensagens.md).

# Oficina Fase 3 — documentação central

Este repositório é o ponto de entrada funcional, técnico e acadêmico da Oficina Fase 3. Ele permite compreender o negócio, a arquitetura, as decisões, o CI/CD, o bootstrap e as evidências, sem substituir ou alterar os quatro projetos originais.

## Visão de negócio

A Oficina organiza o ciclo completo de atendimento de uma oficina mecânica: cadastro do cliente e do veículo, recepção, diagnóstico, orçamento, aprovação, execução do reparo, consumo de peças, pagamento e entrega. Também apoia estoque, compras de fornecedores, movimentações financeiras, notificações e relatórios operacionais.

O sistema atende clientes, funcionários, técnicos e gestores. A **Ordem de Serviço (OS)** é o agregado central e mantém o histórico da jornada:

```text
Recebida → Em diagnóstico → Aguardando aprovação → Em execução
→ Aguardando pagamento → Paga → Entregue
```

Rejeições e cancelamentos seguem regras próprias. O histórico permite acompanhar o atendimento e calcular indicadores como volume de OS e tempos médios por etapa.

## Evolução para a Fase 3

A Fase 2 consolidou as regras de negócio em um Backend Java com Clean Architecture, banco relacional, testes, Docker, Kubernetes, Terraform e CI/CD. A Fase 3 preserva essas funcionalidades e responde ao cenário de crescimento da base de clientes e expansão para múltiplas unidades, acrescentando segurança, escalabilidade, alta disponibilidade e observabilidade corporativa.

A entrada utiliza o **Amazon API Gateway**. Uma **Function Serverless** valida o CPF, consulta a existência e o status do cliente no PostgreSQL e emite um **JWT RSA de curta duração**. O Lambda Authorizer protege as APIs do cliente antes de encaminhar a chamada ao Backend no EKS.

A evolução também separa aplicação, autenticação, banco e Kubernetes em quatro projetos com pipelines independentes, ambientes de homologação e produção e telemetria centralizada no New Relic.

- [Visão detalhada de negócio, evolução e engenharia](docs-fase3/evidencias/visao-negocio-e-engenharia.md)

## Aplicações da solução

| Componente | Contribuição para o negócio | Responsabilidade técnica | Documentação |
|---|---|---|---|
| Backend | Executa o atendimento e as regras da oficina. | APIs, domínio, migrations, Docker e deploy no EKS. | Este repositório · [Projeto original](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend) |
| Auth | Permite acesso seguro do cliente e notificações desacopladas. | CPF, JWT, API Gateway, Authorizer, SNS e DLQ. | [Auth](https://github.com/tiagomiele/auth) · [Projeto original](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless) |
| Database | Preserva os dados operacionais com consistência e recuperação. | RDS privado, backup, segurança, logs e telemetria. | [Database](https://github.com/tiagomiele/database) · [Projeto original](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra) |
| Kubernetes | Mantém a aplicação disponível, escalável e observável. | VPC, EKS, nodes, add-ons, HPA e New Relic. | [Kubernetes](https://github.com/tiagomiele/kubernetes) · [Projeto original](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra) |

## Modelo arquitetural e práticas

A solução distribui responsabilidades entre quatro repositórios. O Backend aplica **DDD e Clean Architecture em quatro anéis** (`Domain`, `Usecase`, `Adapter` e `Infrastructure`), com dependências apontando para o domínio e limites verificados por ArchUnit.

O Auth utiliza camadas leves de domínio, aplicação, handlers e infraestrutura, adequadas às Lambdas. Database e Kubernetes adotam **Infrastructure as Code declarativa**, com Terraform, states separados e plans revisáveis. Clean Code, SOLID, testes automatizados, formatação, análise de segurança, revisão por Pull Request e observabilidade são práticas transversais.

> Clean Code e SOLID são princípios de desenvolvimento; o modelo estrutural comprovado do Backend é Clean Architecture. Os projetos de infraestrutura não simulam camadas de aplicação: seguem organização própria de IaC.

## Arquitetura específica do Backend

![Arquitetura integrada da Oficina Fase 3 com ícones dos serviços AWS](docs-fase3/assets/arquitetura-integrada-oficina-fase3.png)

O Backend concentra o domínio da oficina. API Gateway, autenticação, banco e plataforma Kubernetes permanecem desacoplados em projetos independentes.

- [Diagrama completo de componentes](docs-fase3/architecture/componentes.md)
- [Sequência de autenticação por CPF](docs-fase3/architecture/autenticacao.md)
- [Sequência de abertura da ordem de serviço](docs-fase3/architecture/abertura-ordem-servico.md)

## Tecnologias

| Área | Tecnologias e finalidade |
|---|---|
| Arquitetura | DDD, Clean Architecture, monólito modular, serverless e Infrastructure as Code |
| Aplicação | Java 21, Spring Boot 3.3, Spring Security, JPA/Hibernate e Flyway |
| APIs e segurança | API Gateway v2, AWS Lambda, Lambda Authorizer e JWT RSA |
| Dados | Amazon RDS PostgreSQL 16, constraints, índices, migrations, backup e SSL |
| Plataforma | Amazon EKS, Kubernetes, Helm, HPA, PDB, Metrics Server e Load Balancer |
| Entrega | GitHub Actions, Docker, GHCR, Terraform, HCP Terraform e GitHub Environments |
| Qualidade | JUnit 5, Mockito, RestAssured, Testcontainers, ArchUnit, JaCoCo, Newman e k6 |
| Segurança de código e IaC | SBOM CycloneDX, Trivy, Checkov, TFLint, actionlint, ShellCheck e Gitleaks |
| Observabilidade | New Relic APM, logs JSON, correlação, traces, dashboards, alertas e sintéticos |

## Estrutura de pastas

```text
.
├── .github/workflows/        # CI, deploy e testes E2E
├── docs-fase1/               # documentação preservada da Fase 1
├── docs-fase2/               # documentação preservada da Fase 2
├── docs-fase3/               # arquitetura, decisões e evidências da Fase 3
├── k8s/                      # manifests e scripts de implantação do Backend
├── scripts/                  # configuração e sincronização entre projetos
├── src/main/java/br/com/oficina/
│   ├── domain/               # entidades, regras e exceções de negócio
│   ├── usecase/              # casos de uso e portas
│   ├── adapter/              # controllers, DTOs, persistência e integrações
│   └── infrastructure/       # configuração do Spring e composição da aplicação
├── src/main/resources/
│   ├── db/migration/         # migrations Flyway V1–V4
│   ├── application.yml       # configuração da aplicação
│   └── logback-spring.xml    # logs estruturados
├── src/test/                 # testes unitários, integração e arquitetura
├── tests/postman/            # collection E2E integrada
├── tests/k6/                 # teste de carga smoke
├── Dockerfile
├── docker-compose.yml
└── pom.xml
```

## Pré-requisitos

- Java 21;
- Docker e Docker Compose;
- Git;
- acesso à AWS, ao cluster EKS e aos ambientes GitHub somente para implantação remota.

O Maven Wrapper está incluído; não é necessário instalar Maven globalmente.

## Executar localmente com Docker

```bash
git clone https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend.git
cd fiap-tech-challenge-fase3-oficina-backend
docker compose up --build
```

Serviços locais:

- API: `http://localhost:8080`;
- Swagger UI: `http://localhost:8080/swagger-ui.html`;
- OpenAPI JSON: `http://localhost:8080/v3/api-docs`;
- healthcheck: `http://localhost:8080/actuator/health`;
- Adminer: `http://localhost:8081`.

As credenciais presentes no `docker-compose.yml` destinam-se exclusivamente ao desenvolvimento local. Valores reais devem permanecer em Secrets, GitHub Environments ou HCP Terraform.

## Executar a aplicação com o Maven Wrapper

Inicie somente o PostgreSQL:

```bash
docker compose up -d db
```

Depois execute:

```bash
./mvnw spring-boot:run
```

No Windows PowerShell:

```powershell
.\mvnw.cmd spring-boot:run
```

As variáveis disponíveis estão documentadas em [.env.example](.env.example). Não versione arquivos `.env` com valores reais.

## Testes e validações

```bash
./mvnw -B clean verify spotless:check
```

A suíte inclui testes unitários, integração, persistência, segurança, contrato e arquitetura. O pipeline também executa SBOM, análise de vulnerabilidades e testes funcionais Postman/Newman.

- [Collection Postman integrada](tests/postman/oficina-weeks4-5.postman_collection.json)
- [Ambiente Postman de exemplo](tests/postman/oficina-homolog.postman_environment.example.json)
- [Teste de carga smoke](tests/k6/smoke-load.js)

## Configuração

Principais variáveis:

| Variável | Finalidade |
|---|---|
| `DB_URL`, `DB_USER`, `DB_PASSWORD` | conexão PostgreSQL |
| `JWT_SECRET` | autenticação interna da equipe |
| `SERVERLESS_JWT_PUBLIC_KEY` | validação dos tokens emitidos pelo Auth |
| `SERVERLESS_JWT_ISSUER`, `SERVERLESS_JWT_AUDIENCE` | validação do emissor e da audiência |
| `AUTH_BASE_URL` | URL do Auth Serverless |
| `NOTIFICATION_ENDPOINT`, `NOTIFICATION_API_KEY` | integração de notificações |
| `OFICINA_ENVIRONMENT` | identificação do ambiente |
| `OFICINA_OBSERVABILITY_NEW_RELIC_ENABLED` | habilitação da integração New Relic |

## CI/CD e implantação

| Workflow | Finalidade |
|---|---|
| `.github/workflows/ci.yml` | build, testes, cobertura, SBOM e segurança |
| `.github/workflows/deploy.yml` | build da imagem e implantação por ambiente |
| `.github/workflows/e2e.yml` | validação funcional e de segurança integrada |
| `.github/workflows/cd.yml` | compatibilidade com o fluxo de entrega existente |

Fluxo de promoção:

```text
feature → Pull Request → homolog → Pull Request → main
```

- Pull Requests executam validações sem implantar produção;
- `homolog` publica o ambiente de homologação;
- `main` promove a versão validada para produção;
- commits diretos, exclusões e force pushes são controlados pelos rulesets;
- a implantação completa respeita a ordem `Kubernetes → Database → Auth → Backend`.

- [Fluxo completo de CI/CD](docs-fase3/evidencias/02-cicd/cicd-promocao.md)
- [Bootstrap da solução na AWS](docs-fase3/bootstrap-aws-do-zero.md)

## Documentação e evidências

- [Requisitos obrigatórios da Fase 3](README-requisitos-obrigatorios-fase3.md)

## Projetos relacionados

- [Auth Serverless](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless)
- [Database Infra](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra)
- [Kubernetes Infra](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra)

## Licença

Consulte o arquivo [LICENSE](LICENSE).
