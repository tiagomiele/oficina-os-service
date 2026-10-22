# Evidências de Atendimento aos Requisitos Obrigatórios — Oficina Fase 3

Este documento centraliza as evidências de forma sucinta e objetivo para os atendimentos aos requisitos obrigatórios do Tech Challenge — Oficina Fase 3.

## Projetos da solução contidos dentro da estrutura de 4 Repositórios

| Projeto | Repositório |
|---|---|
| Backend — aplicação principal | [fiap-tech-challenge-fase3-oficina-backend](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend) |
| Auth Serverless | [fiap-tech-challenge-fase3-oficina-auth-serverless](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless) |
| Kubernetes Infra | [fiap-tech-challenge-fase3-oficina-kubernetes-infra](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra) |
| Database Infra | [fiap-tech-challenge-fase3-oficina-database-infra](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra) |

## 1. Autenticação Serverless e API Gateway

A autenticação por CPF valida a existência e o status ativo do cliente, emite um JWT e protege as rotas sensíveis por meio do API Gateway.

### Arquivos técnicos da implementação da autenticação

Os links abaixo comprovam como o API Gateway, as funções serverless e o contrato de autenticação foram definidos na aplicação.

- [Arquitetura do Auth](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/blob/main/docs/arquitetura.md)
- [Terraform do API Gateway](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/blob/main/apigateway.tf)
- [Terraform das funções Lambda](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/blob/main/lambda.tf)
- [Contrato OpenAPI do Auth](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/blob/main/docs/openapi/oficina-auth.yaml)

### Evidências de execução da autenticação

| Ambiente ou contexto | Tipo | Execução | Log dos testes E2E |
|---|---|---|---|
| Homologação | CI | [Run 34794303074](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/actions/runs/34794303074) | — |
| Homologação | Deploy Terraform | [Run 34794302992](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/actions/runs/34794302992) | — |
| Homologação | Reexecução manual do deploy | [Run 34796371489](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/actions/runs/34796371489) | — |
| Produção | CI | [Run 34799909581](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/actions/runs/34799909581) | — |
| Produção | Deploy Terraform | [Run 34799909603](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/actions/runs/34799909603) | — |
| Produção | Reexecução manual do deploy | [Run 34802109160](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/actions/runs/34802109160) | — |
| Execução a partir de `homolog` | Validação funcional E2E | [Run 34796659485](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/actions/runs/34796659485) | [Abrir log Postman/Newman](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/actions/runs/34796659485/job/103830971683#step:6:1) |
| Execução a partir de `main` | Validação funcional E2E | [Run 34770966649](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/actions/runs/34770966649) | [Abrir log Postman/Newman](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/actions/runs/34770966649/job/103760366305#step:6:1) |

Os runs foram concluídos com sucesso. As validações E2E executam a collection Postman/Newman e comprovam a emissão e o uso do JWT na integração entre Auth Serverless, Backend e Database.

Para visualizar os cenários, abra o link da última coluna. O GitHub exibirá o job `Postman/Newman` e a etapa **Run functional and security collection**, que contém o log da execução.

### Perfis de acesso

| Perfil | Acesso esperado |
|---|---|
| `CLIENTE` | Acessa somente as rotas destinadas ao cliente e os recursos que lhe pertencem; não acessa rotas administrativas ou técnicas. |
| `FUNCIONARIO_DA_OFICINA` | Acessa rotas administrativas e, pela hierarquia de papéis, também rotas técnicas; não acessa rotas exclusivas de cliente. |

### Evidências complementares de execução

- [Swagger — autenticação pelo perfil Cliente](docs-fase3/evidencias/01-autenticacao-api-gateway/Swagger-Perfil-Cliente-Oficina.pdf)
- [Swagger — autenticação pelo perfil Funcionário da Oficina](docs-fase3/evidencias/01-autenticacao-api-gateway/Swagger-Perfil-Funcionario-Oficina.pdf)
- [Vídeo auxiliar — proteção de rotas sensíveis e acesso por perfis](https://vimeo.com/1226754719) — duração: 3min50s.

## 2. Quatro repositórios e CI/CD

A solução está dividida em quatro repositórios independentes, cada um com validações de CI e entrega automatizada para homologação e produção.

### Arquivos técnicos de configuração dos workflows

Os links abaixo apresentam os manifestos que implementam as validações e os deploys automatizados de cada projeto.

| Projeto     | Workflows configurados                                                                                                                                                                                                                                                               |
|-------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Auth        | [CI](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/blob/main/.github/workflows/ci.yml) · [Deploy Terraform](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/blob/main/.github/workflows/terraform-deploy.yml)      |
| Kubernetes  | [CI](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/blob/main/.github/workflows/ci.yml) · [Deploy de produção](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/blob/main/.github/workflows/deploy-production.yml) |
| Database    | [CI](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/blob/main/.github/workflows/ci.yml) · [Terraform apply](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/blob/main/.github/workflows/terraform-apply.yml)          |
| Backend     | [CI](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/blob/main/.github/workflows/ci.yml) · [CD](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/blob/main/.github/workflows/cd.yml)                                                  |
| Fluxo CI/CD | [Fluxo deploy aplicações](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/blob/homolog/docs/cicd.md)                                                                                                                                                |


### Evidências de execução do CI/CD

Os links abaixo comprovam execuções aprovadas das pipelines em produção.

| Projeto | CI aprovado | Deploy aprovado |
|---|---|---|
| Auth | [Run 34799909581](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/actions/runs/34799909581) | [Run 34799909603](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/actions/runs/34799909603) |
| Kubernetes | [Run 34797594998](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/actions/runs/34797594998) | [Run 34797595003](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/actions/runs/34797595003) |
| Database | [Run 34798911957](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/actions/runs/34798911957) | [Run 34798911837](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/actions/runs/34798911837) |
| Backend | [Run 34800781548](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/actions/runs/34800781548) | [Run 34800781717](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/actions/runs/34800781717) |

Os runs do quadro foram concluídos com `success`.

As execuções de homologação e produção estão consolidadas na pasta de evidências ci-cd em: [documentação de CI/CD](docs-fase3/evidencias/02-cicd/CI-CD.md).

## 3. Governança de branches

As branches `homolog` e `main` dos quatro repositórios possuem rulesets ativos. Eles exigem Pull Request, execução bem-sucedida dos checks configurados. A promoção para produção ocorre por Pull Request de `homolog` para `main`.

### Configuração técnica da proteção das branches

Os links abaixo apresentam os rulesets que aplicam as regras de governança às branches `homolog` e `main`.

| Projeto | `homolog` | `main` |
|---|---|---|
| Auth | [Ruleset de homologação](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/rules/23168852) | [Ruleset de produção](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/rules/23169021) |
| Kubernetes | [Ruleset de homologação](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/rules/23075975) | [Ruleset de produção](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/rules/23076125) |
| Database | [Ruleset de homologação](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/rules/23075472) | [Ruleset de produção](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/rules/23075731) |
| Backend | [Ruleset de homologação](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/rules/23074518) | [Ruleset de produção](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/rules/23070178) |

### Evidências de execução da promoção para produção

Os últimos Pull Requests concluídos comprovam que a promoção ocorreu de `homolog` para `main`, sem commit direto na branch de produção.

| Projeto | Último Pull Request de promoção | Histórico de Pull Requests |
|---|---|---|
| Auth | [PR #3 — `homolog` → `main`](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/pull/3) | [Histórico](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/pulls?q=is%3Apr) |
| Kubernetes | [PR #7 — `homolog` → `main`](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/pull/7) | [Histórico](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/pulls?q=is%3Apr) |
| Database | [PR #5 — `homolog` → `main`](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/pull/5) | [Histórico](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/pulls?q=is%3Apr) |
| Backend | [PR #5 — `homolog` → `main`](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/pull/5) | [Histórico](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/pulls?q=is%3Apr) |

### Evidências de execução da proteção contra commit direto na `main`

Os testes abaixo tentaram iniciar uma alteração a partir da `main`. A proteção impediu a gravação direta, e o GitHub direcionou o commit para a branch separada `tiagomiele-patch-1`. O CI foi executado com sucesso nessa nova branch, sem alterar a `main`.

| Projeto | Run de validação | Branch que recebeu o commit | Resultado |
|---|---|---|---|
| Auth | [Run 34976228182](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/actions/runs/34976228182) | `tiagomiele-patch-1` | `success` |
| Kubernetes | [Run 34976482274](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/actions/runs/34976482274) | `tiagomiele-patch-1` | `success` |
| Database | [Run 34976362476](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/actions/runs/34976362476) | `tiagomiele-patch-1` | `success` |
| Backend | [Run 34976005252](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/actions/runs/34976005252) | `tiagomiele-patch-1` | `success` |

Esses runs comprovam o bloqueio do commit direto, mas não representam deploy ou promoção para produção. A atualização legítima da `main` continua sendo realizada pelos Pull Requests de `homolog` para `main` listados acima.

## 4. Infraestrutura em nuvem com Terraform

API Gateway, funções serverless, Amazon RDS e Amazon EKS são provisionados com Terraform para homologação e produção.

### Arquivos técnicos da infraestrutura como código

Os arquivos abaixo comprovam como os componentes de nuvem foram definidos e configurados na aplicação.

| Componente | Arquivo de implementação |
|---|---|
| Rede e EKS | [network.tf](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/blob/main/network.tf) · [eks.tf](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/blob/main/eks.tf) |
| Add-ons Kubernetes | [kubernetes/addons](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/tree/main/kubernetes/addons) |
| RDS PostgreSQL | [main.tf](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/blob/main/main.tf) |
| Telemetria do RDS | [telemetry.tf](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/blob/main/telemetry.tf) |
| API Gateway | [apigateway.tf](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/blob/main/apigateway.tf) |
| Funções serverless | [lambda.tf](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/blob/main/lambda.tf) |
| Notificações assíncronas | [notification.tf](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/blob/main/notification.tf) |
| Configuração do backend remoto do Terraform | [versions.tf — Kubernetes](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/blob/main/versions.tf) · [versions.tf — Database](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/blob/main/versions.tf) · [versions.tf — Auth](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/blob/main/versions.tf) |

Os deploys apresentados na seção 2 comprovam a execução automatizada do Terraform. A solução utiliza HCP Terraform e workspaces separados por componente e ambiente.

### Evidências de execução no HCP Terraform

#### Produção

- [Auth — execução](docs-fase3/evidencias/04-terraform-hcp/PRD-TF-auth1.pdf)
- [Auth — reaplicação](docs-fase3/evidencias/04-terraform-hcp/PRD-TF-auth2.pdf)
- [Database](docs-fase3/evidencias/04-terraform-hcp/PRD-TF-Db.pdf)
- [Kubernetes](docs-fase3/evidencias/04-terraform-hcp/PRD-TF-kubernetes.pdf)
- [New Relic](docs-fase3/evidencias/04-terraform-hcp/PRD-TF-newrelic.pdf)

#### Workspaces

- [Workspaces da solução](docs-fase3/evidencias/04-terraform-hcp/TF-Workspaces.pdf)

#### Evidências complementares do HCP Terraform para os ambientes de homologação e produção, encontram-se no índice abaixo:

- [Índice das evidências de homologação e produção para implementação do HCP Terraform](docs-fase3/evidencias/04-terraform-hcp/Terraform-hcp.md)

## 5. Aplicação principal no Kubernetes

O Backend é conteinerizado e executado no Kubernetes, com deploy automatizado, healthchecks e escalabilidade.

### Arquivos técnicos da implantação no Kubernetes

Os links abaixo comprovam a conteinerização e a configuração da aplicação para execução, exposição e escalabilidade no cluster.

- [Dockerfile do Backend](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/blob/main/Dockerfile)
- [Deployment Kubernetes](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/blob/main/k8s/app-deployment.yaml)
- [Service Kubernetes](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/blob/main/k8s/app-service.yaml)
- [HPA-Horizontal Pod Autoscaler](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/blob/main/k8s/hpa.yaml)
- [Collection Postman](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/blob/main/tests/postman/oficina-weeks4-5.postman_collection.json)
- [Probes e réplicas no Deployment](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/blob/main/k8s/app-deployment.yaml)
- [PodDisruptionBudget](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/blob/main/k8s/pdb.yaml)

  As réplicas, probes e o PodDisruptionBudget são evidências complementares de alta disponibilidade e resiliência. O HPA comprova a estratégia de escalabilidade exigida.

### Evidências de execução da implantação

- [Deploy de homologação aprovado](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/actions/runs/34795569734)
- [Deploy de produção aprovado](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/actions/runs/34800781717)
- [Diretório de evidências do Kubernetes Backend](docs-fase3/evidencias/05-kubernetes-backend/)
- [Vídeo auxiliar — AWS-AWSacademy, EKS, aplicação e infra estrutura em cloud](https://vimeo.com/1226631740) — duração: 4min44s.

## 6. Banco de dados gerenciado e modelo relacional

A solução utiliza PostgreSQL gerenciado no Amazon RDS, com modelo relacional documentado e controles de consistência e desempenho.

### Justificativa da escolha do PostgreSQL

O PostgreSQL oferece transações ACID, integridade referencial, maturidade e suporte gerenciado no Amazon RDS. Sua integração com Spring Data JPA e Flyway atende aos requisitos da aplicação com consistência e baixo custo operacional.

### Arquivos e documentos técnicos do banco de dados

Os links abaixo comprovam a implementação, a modelagem e as decisões técnicas do banco gerenciado.

- [Justificativa PostgreSQL/RDS — ADR 0001 (arquivo.docx)](docs-fase3/evidencias/06-banco-de-dados/ADR_0001_Banco_Relacional_PostgreSQL.docx)
- [Diagrama entidade-relacionamento](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/blob/homolog/docs/assets/modelo-relacional-database.png)
- [Modelo relacional](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/blob/main/docs/modelo-relacional.md)
- [Índices e desempenho](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/blob/main/docs/indices-desempenho.md)
- [ADR de consistência do modelo](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/blob/main/docs/adr/0001-consistencia-modelo.md)
- [Terraform do RDS PostgreSQL](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/blob/main/main.tf)

### Evidências de execução do provisionamento

- [Apply de homologação aprovado](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/actions/runs/34792773246)
- [Apply de produção aprovado](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/actions/runs/34798911837)

### Evidência de execução da conexão ao PostgreSQL

- [Conexão aos bancos PostgreSQL de homologação e produção (arquivo.docx)](docs-fase3/evidencias/06-banco-de-dados/Evidencia-Conexao-BD.docx)

## 7. Monitoramento, observabilidade e logs

O New Relic centraliza métricas, logs e traces do Backend, das funções serverless, do Kubernetes e do RDS. Os dashboards acompanham saúde, desempenho e requisições correlacionadas por `correlationId` e `traceId`.

### Arquivos técnicos da implementação da observabilidade

- [Dashboard New Relic](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/blob/main/observability/newrelic/dashboard.tf)
- [Alertas da plataforma](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/blob/main/observability/newrelic/alerts.tf)
- [Alertas do RDS](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/blob/main/observability/newrelic/rds-alerts.tf)
- [Monitor sintético](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/blob/main/observability/newrelic/synthetics.tf)
- [Instrumentação do Auth](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/blob/main/newrelic.tf)
- [Telemetria do Database](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/blob/main/telemetry.tf)
- [Documentação de observabilidade do Backend](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/blob/main/docs/observabilidade-evidencias.md)
- [ADR de logs, correlação e traces](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/blob/main/docs/decisions/adr/0004-logs-correlacao-traces.md)

### Evidências de execução da observabilidade

- [New Relic — consumo de CPU](docs-fase3/evidencias/07-Monitoramento/NR-CPU.png)
- [New Relic — disponibilidade da aplicação](docs-fase3/evidencias/07-Monitoramento/NR-Disponibilidade.png)
- [New Relic — falhas nas ordens de serviço](docs-fase3/evidencias/07-Monitoramento/NR-Falhas-OS.png)
- [New Relic — latência das APIs](docs-fase3/evidencias/07-Monitoramento/NR-latencia.png)
- [New Relic — logs da aplicação](docs-fase3/evidencias/07-Monitoramento/NR-Logs.png)
- [New Relic — tempo médio por status da ordem de serviço](docs-fase3/evidencias/07-Monitoramento/NR-Media-Status-OS.png)
- [New Relic — volume diário de ordens de serviço](docs-fase3/evidencias/07-Monitoramento/NR-Vol-OS-dia.png)
- [Vídeo auxiliar analises a dashboard/logs e traces New Relic (Negócio, Aplicação, Kubernets, Serveless/Api Gateway e RDS PostgreSql](https://vimeo.com/1226611174) — duração: 4min08s.

## 8. Documentação arquitetural, RFCs e ADRs

A documentação apresenta a arquitetura, os fluxos principais e as decisões técnicas por meio de diagramas, RFCs e ADRs.

### Documentação técnica e decisões arquiteturais

| Documento técnico | Link                                                                                                                                                             |
|---|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Diagrama de componentes | [componentes.md](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/blob/main/docs-fase3/architecture/componentes.md)                       |
| Sequência de autenticação | [autenticacao.md](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/blob/main/docs-fase3/architecture/autenticacao.md)                     |
| Sequência de abertura da OS | [abertura-ordem-servico.md](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/blob/main/docs-fase3/architecture/abertura-ordem-servico.md) |
| RFCs | [docs/decisions/rfc](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/tree/main/docs-fase3/decisions/rfc)                                 |
| ADRs | [docs/decisions/adr](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/tree/main/docs-fase3/decisions/adr)                                 |
| Diagrama ER | [diagrama entidade-relacionamento](docs-fase3/assets/modelo-relacional-database.png)                                                                             |
| Arquitetura do Auth | [Arquitetura do Auth](docs-fase3/assets/arquitetura-integrada-oficina-fase3.png)                                                                                 |
| Infraestrutura e observabilidade | [docs-fase3/evidencias/07-Monitoramento](docs-fase3/evidencias/07-Monitoramento)                                                                                        |


## 9. READMEs para os quatro repositórios

Cada repositório documenta propósito, tecnologias, pré-requisitos, execução, deploy, pipeline, arquitetura e contratos aplicáveis.

| Projeto | README |
|---|---|
| Auth | [README do Auth](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/blob/main/README.md) |
| Kubernetes | [README do Kubernetes](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/blob/main/README.md) |
| Database | [README do Database](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/blob/main/README.md) |
| Backend | [README do Backend](https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/blob/main/README.md) |

### Evidências de execução dos contratos de API

- [Swagger em execução — perfil Cliente](docs-fase3/evidencias/01-autenticacao-api-gateway/Swagger-Perfil-Cliente-Oficina.pdf)
- [Swagger em execução — perfil Funcionário](docs-fase3/evidencias/01-autenticacao-api-gateway/Swagger-Perfil-Funcionario-Oficina.pdf)

## 10. Documentação central e fechamento da entrega

A pasta `docs-fase3` reúne diagramas, decisões arquiteturais, evidências de execução e materiais de apoio da solução.

### Acesso centralizado

- [Abrir a pasta de documentação e evidências](docs-fase3/evidencias)
- [Voltar ao README principal](README.md)
