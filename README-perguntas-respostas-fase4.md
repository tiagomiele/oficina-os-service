# Perguntas e respostas do enunciado — Oficina Fase 4

Cada questão do enunciado da Fase 4 tem uma resposta curta e a referência ao projeto. O detalhe e as evidências de cada item estão em [README-requisitos-obrigatorios-fase4.md](README-requisitos-obrigatorios-fase4.md).

Repositórios:
- OS: [oficina-os-service](https://github.com/tiagomiele/oficina-os-service)
- Orçamento e Pagamento (O&P): [oficina-orcamento-pagamento-service](https://github.com/tiagomiele/oficina-orcamento-pagamento-service)
- Diagnóstico e Reparo (D&R): [oficina-diagnostico-reparo-service](https://github.com/tiagomiele/oficina-diagnostico-reparo-service)
- Auth: [oficina-auth-serverless](https://github.com/tiagomiele/oficina-auth-serverless)
- DB-Infra: [oficina-database-infra](https://github.com/tiagomiele/oficina-database-infra)
- K8s-Infra: [oficina-kubernetes-infra](https://github.com/tiagomiele/oficina-kubernetes-infra)

## Separação em Microsserviços

**1. A aplicação foi refatorada em, no mínimo, 3 microsserviços independentes, cada um com seu próprio repositório, infraestrutura e banco de dados?**
Resposta: Sim. São 3 microsserviços: OS, O&P e D&R. Cada um tem repositório próprio, com `Dockerfile`, `k8s/` e `.github/workflows/`, além de namespace, Deployment, LoadBalancer, HPA e banco próprios. Ressalva: o Terraform dos bancos fica no DB-Infra, mas com uma pasta, um state, um workspace HCP e um workflow por serviço (`os/`, `orcamento-pagamento/` e `diagnostico-reparo/`). O cluster EKS é compartilhado, com um namespace por serviço. Os dois formatos foram aceitos pela coordenação (OS `README-requisitos-obrigatorios-fase4.md` §1).

**2. O OS Service faz a abertura da OS, a atualização de status e a consulta de status e histórico?**
Resposta: Sim. A abertura fica em `AdministrativoOficinaController`. O status muda pela Saga, em `SagaOrdemServicoServiceImpl`. O cliente consulta o status e o histórico em `ClienteOficinaController`, que usa a tabela `historico_status_ordem_servico` (migration V3).

**3. O Billing Service gera e envia orçamentos para aprovação, registra e verifica pagamentos e atualiza o status da OS após o pagamento?**
Resposta: Sim. O comando `GERAR_ORCAMENTO` gera o orçamento (`SqsComandosConsumidor`). A aprovação fica em `OrcamentoController`, e o pagamento em `PagamentoController`. O evento `PAGAMENTO_CONFIRMADO` faz o OS mudar a OS para `PAGA`.

**4. O pagamento está integrado ao Mercado Pago?**
Resposta: Sim. Usa o Checkout Pro em sandbox, com webhook assinado (`MercadoPagoPagamentoGateway`, `MercadoPagoWebhookController` e `MercadoPagoAssinatura`). Os testes usam um adaptador fake. O fluxo está descrito no README do O&P, §6.

**5. O serviço de Execução e Produção gerencia a fila de execução, atualiza o status durante o diagnóstico e o reparo e comunica a finalização ao OS Service?**
Resposta: Sim. A fila fica em `GET /execucoes/fila`, pelo índice DynamoDB `etapa-entradaFilaEm-index`. As etapas mudam em `ExecucaoService`, e o fim é avisado ao OS com o evento `REPARO_CONCLUIDO`. Tudo está no D&R.

**6. Cada microsserviço tem banco próprio, com pelo menos um banco SQL e um NoSQL?**
Resposta: Sim. O OS usa o RDS PostgreSQL `oficina-<env>`, e o O&P usa um RDS PostgreSQL isolado. O D&R usa o DynamoDB `oficina-<env>-execucoes`. Ver as pastas `os/`, `orcamento-pagamento/` e `diagnostico-reparo/` do DB-Infra.

## Comunicação entre Microsserviços

**7. Há APIs RESTful síncronas quando necessário?**
Resposta: Sim. Os usuários chamam os controllers REST de cada serviço. O cliente passa pelo API Gateway com JWT (Auth `apigateway.tf`). O O&P chama o Mercado Pago, e o login do Auth consulta o OS em `InternoClienteController`.

**8. Há mensageria assíncrona para eventos e integração desacoplada?**
Resposta: Sim. Os 3 serviços se comunicam só por Amazon SQS, em 3 filas: `os-eventos`, `diagnostico-reparo-comandos` e `orcamento-pagamento-comandos`. Cada fila tem DLQ, que recebe a mensagem depois de 5 tentativas (DB-Infra `modules/sqs-com-dlq`). No OS, ver `SqsMensageriaGateway` e `SqsEventosConsumidor`.

**9. Algum serviço acessa diretamente o banco de outro serviço?**
Resposta: Não. O `k8s/app-deployment.yaml` de cada serviço só recebe as credenciais do próprio banco. Os dados de outro serviço chegam no `payload` das mensagens. O Auth consulta o OS por REST, em `OsServiceClientRepository`, e não acessa o banco dele.

## Implementação do Saga Pattern

**10. O Saga Pattern coordena o fluxo transacional das ordens de serviço?**
Resposta: Sim. O fluxo é abertura → diagnóstico → orçamento → aprovação → reparo → pagamento → entrega. Ver OS `SagaOrdemServicoServiceImpl` e `docs/saga/contrato-mensagens.md`.

**11. Há rollback e compensação em caso de falha em qualquer etapa?**
Resposta: Sim, nestes casos:
- **Estoque insuficiente:** cancela a OS e envia `CANCELAR_ORCAMENTO` e `CANCELAR_EXECUCAO`.
- **Orçamento rejeitado com cancelamento:** envia `CANCELAR_EXECUCAO`.
- **Rejeitado para refazer:** volta ao diagnóstico.
- **Pagamento recusado:** permite nova tentativa.
- **Erro técnico:** desfaz a transação, reprocessa a mensagem e, se continuar falhando, a envia para a DLQ.

Mensagens repetidas são ignoradas pelo `messageId` (tabela `mensagens_processadas`).

**12. A Saga é orquestrada ou coreografada?**
Resposta: Orquestrada. O orquestrador central é o OS Service (OS `docs/saga/README.md`).

**13. A escolha está documentada e justificada no README?**
Resposta: Sim, no OS `README.md` §7 e em `README-requisitos-obrigatorios-fase4.md` §3. O fluxo é sequencial e o OS já é o dono do status. Assim, o próximo passo e as compensações ficam num lugar só, o que facilita testar e rastrear.

## Testes e Qualidade

**14. Há testes unitários em todos os microsserviços?**
Resposta: Sim, com JUnit 5 e Mockito, em `src/test` de cada repositório. São 44 classes de teste no OS, 13 no O&P, 10 no D&R e 20 no Auth.

**15. Há pelo menos um fluxo completo testado com BDD?**
Resposta: Sim. São 5 cenários Cucumber, o fluxo feliz e 4 compensações, rodando com os 3 serviços reais (OS `bdd/src/test/resources/features/saga.feature` e `.github/workflows/bdd.yml`). Último run em `main`: https://github.com/tiagomiele/oficina-os-service/actions/runs/42539981730.

**16. A cobertura mínima é de 80% por serviço?**
Resposta: Sim. O JaCoCo falha o build com cobertura abaixo de 0,80 (`pom.xml`). No SonarCloud (`main`), a cobertura é de 84,3% no OS, 97,6% no O&P e 97,7% no D&R.

**17. A qualidade do código é validada via SonarQube ou similar no CI?**
Resposta: Sim. O job `SonarCloud` do `ci.yml` roda com `qualitygate.wait=true`, e o Quality Gate está OK nos 3 serviços:
- OS: https://sonarcloud.io/summary/overall?id=tiagomiele_oficina-os-service&branch=main
- O&P: https://sonarcloud.io/summary/overall?id=tiagomiele_oficina-orcamento-pagamento-service&branch=main
- D&R: https://sonarcloud.io/summary/overall?id=tiagomiele_oficina-diagnostico-reparo-service&branch=main

## CI/CD Automatizado

**18. Cada microsserviço tem pipeline independente de CI/CD com build, testes automatizados, verificação de qualidade e deploy automatizado em Kubernetes?**
Resposta: Sim. Cada serviço tem `ci.yml`, `cd.yml` e `deploy.yml` próprios:
- **Build e testes:** job `Build, test & coverage` (Maven, Testcontainers e JaCoCo).
- **Qualidade:** SonarCloud, Trivy, Gitleaks e SBOM (Dependency-Track).
- **Deploy:** um push em `homolog` publica na homologação, e um push em `main`, na produção. A imagem vai para o GHCR e o `k8s/` é aplicado no EKS.

Últimos CDs em `main`:
- OS: https://github.com/tiagomiele/oficina-os-service/actions/runs/42527530259
- O&P: https://github.com/tiagomiele/oficina-orcamento-pagamento-service/actions/runs/42528964889
- D&R: https://github.com/tiagomiele/oficina-diagnostico-reparo-service/actions/runs/42528501218

**19. Os repositórios estão protegidos, com branch main, PR obrigatório e checagens automáticas?**
Resposta: Sim. Os rulesets `Proteção main` e `Proteção homolog` estão ativos nos 6 repositórios. Eles exigem PR, bloqueiam force-push e exclusão e exigem estes checks: `Repository validation`, `Build, test & coverage`, `SonarCloud`, `SBOM (CycloneDX) + Dependency-Track` e `Security scan`. Os rulesets são criados por OS `scripts/configure-environment.ps1`. Ressalva: não exigem aprovação de um revisor.

## Infraestrutura

**20. Cada microsserviço está em seu próprio repositório?**
Resposta: Sim. Ver a resposta 1.

**21. O deploy para Kubernetes é automatizado?**
Resposta: Sim, no Amazon EKS 1.34 (K8s-Infra `eks.tf`), pelo `cd.yml` de cada serviço. Cada serviço tem Deployment, Service LoadBalancer, HPA, PDB e probes em `k8s/`.

**22. Há mensageria para orquestração ou coreografia?**
Resposta: Sim. A Saga orquestrada usa SQS com DLQ. Ver a resposta 8.

**23. As ferramentas de monitoramento e observabilidade da Fase 3 são usadas?**
Resposta: Sim, com New Relic:
- APM no `Dockerfile` de cada serviço.
- Logs em JSON com `correlationId`.
- `nri-bundle` no cluster.
- Dashboard `Oficina-fase4-<env>`, com página da Saga, alertas e Synthetic.

Ver K8s-Infra `observability/newrelic/` e README §6.4.

## Entregáveis da Fase 4: repositórios

**24. Cada repositório tem código-fonte?**
Resposta: Sim, em `src/` de cada serviço.

**25. Cada repositório tem Dockerfile e manifestos Kubernetes?**
Resposta: Sim, o `Dockerfile` e a pasta `k8s/` de cada serviço.

**26. Cada repositório tem pipelines de CI/CD?**
Resposta: Sim, em `.github/workflows/` (`ci.yml`, `cd.yml` e `deploy.yml`).

**27. Há evidências de cobertura de testes (prints ou links no README)?**
Resposta: Sim. O README de cada serviço tem a seção "Evidência de cobertura de testes", com links do CI e do SonarCloud.

**28. Há documentação da arquitetura do serviço?**
Resposta: Sim, no README de cada serviço. Os diagramas ficam em OS `docs/assets/`: `arquitetura-fase4.png`, `arquitetura-macro-fase4.png`, `arquitetura-hexagonal-fase4.png`, `saga-fluxo.png` e `saga-orquestrada-eventos-fase4.png`.

**29. Há Swagger ou collection Postman atualizada?**
Resposta: Sim, as duas coisas:
- Swagger em `/swagger-ui.html` nos 3 serviços.
- Collection Postman em OS `tests/postman/oficina-weeks4-5.postman_collection.json`.
- OpenAPI do Auth em `docs/openapi/oficina-auth.yaml`.

## Entregáveis da Fase 4: vídeo de demonstração

**30. Há vídeo de até 15 minutos no YouTube ou Vimeo (público ou não listado)?**
Resposta: Pendente. O grupo ainda precisa gravar e publicar o vídeo. Antes, suba o ambiente com `provision-environment.ps1` (OS `README-bootstrap-unica-vez.md`).

**31. O vídeo demonstra o fluxo completo de uma OS passando pelos microsserviços?**
Resposta: Pendente. Para mostrar, rode o E2E (`e2e.yml`) ou o `scripts/pagamento-mercadopago.ps1` e exiba o histórico da OS.

**32. O vídeo demonstra a execução do Saga Pattern e o tratamento de falhas?**
Resposta: Pendente. Para mostrar, use os cenários de compensação do BDD, uma rejeição de orçamento e uma mensagem na DLQ.

**33. O vídeo demonstra o deploy automatizado de pelo menos um microsserviço, com validação de testes?**
Resposta: Pendente. Para mostrar, faça um merge em `homolog` e exiba o `cd.yml` rodando os testes e publicando no EKS.

**34. O vídeo demonstra o monitoramento e o rastreamento dos fluxos distribuídos?**
Resposta: Pendente. Para mostrar, abra o dashboard `Oficina-fase4-<env>` e a página da Saga, e busque os logs por `correlationId` no New Relic.

## Documentação entregue no portal (PDF)

**35. O PDF tem o nome e a identificação dos participantes?**
Resposta: Pendente. O grupo precisa preencher.

**36. O PDF tem os links dos repositórios?**
Resposta: Sim. São os links do início deste documento.

**37. O PDF tem o link do vídeo de até 15 minutos?**
Resposta: Pendente. Fica disponível depois da publicação do vídeo.

**38. O PDF tem o diagrama geral da arquitetura final, com microsserviços, bancos e comunicação?**
Resposta: Sim. Use OS `docs/assets/arquitetura-fase4.png` e `arquitetura-macro-fase4.png`.

**39. O PDF descreve a estratégia escolhida para o Saga Pattern?**
Resposta: Sim. A Saga é orquestrada pelo OS Service. Ver as respostas 12 e 13 e OS `docs/saga/README.md`.

**40. O PDF tem a justificativa da divisão dos microsserviços e das tecnologias usadas?**
Resposta: Sim. A divisão segue as áreas da oficina: atendimento e OS, diagnóstico e execução, e cobrança. Usa PostgreSQL para dados transacionais e DynamoDB para a fila de execução, SQS para desacoplar os serviços e EKS para o deploy. O texto completo está em OS `README-requisitos-obrigatorios-fase4.md` §10.
