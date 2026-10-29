# Roteiro atualizado — subir toda a Oficina Fase 3 na AWS do zero

## 1. Objetivo

Este roteiro recria a Oficina Fase 3 completa na AWS, começando por um AWS Academy Learner Lab vazio e usando os workflows unificados atuais.

A ordem obrigatória de homologação é:

```text
Preparar credenciais e configuração base uma vez
→ Kubernetes/EKS + add-ons + New Relic inicial
→ Database/RDS
→ Auth/Lambdas/API Gateway
→ Backend/EKS/LoadBalancer
→ reaplicar Auth com o LoadBalancer final
→ reaplicar Kubernetes/New Relic com o health check final
→ validação E2E
```

Cada projeto executa no próprio run:

```text
validar → plan interno → apply/deploy → capturar outputs → sincronizar consumidores → summary
```

Portanto, não execute um workflow manual de plan antes de cada deploy e não reexecute o PowerShell depois de cada projeto apenas para sincronizar outputs.

> **Custos:** EKS, RDS, NAT Gateway, LoadBalancer, logs e telemetria podem consumir rapidamente o orçamento do AWS Academy ou gerar cobrança em uma conta AWS real. Acompanhe o orçamento e destrua o ambiente quando ele não for mais necessário, usando os fluxos versionados do projeto — nunca apagando recursos isolados pelo console.

## 2. Regras de segurança

1. Faça primeiro toda a homologação na branch `homolog`.
2. Execute somente um workflow por vez e aguarde-o terminar verde.
3. Não selecione `main` ou `production` durante a homologação.
4. Não envie no chat, prints ou documentos:
- credenciais do AWS Academy;
- tokens HCP Terraform;
- chaves New Relic;
- conteúdo do arquivo `.pem` da GitHub App;
- senhas, JWTs ou chaves RSA.
5. Não versione `C:\fiap-secrets`, arquivos `.pem`, `.env`, credenciais ou states Terraform.
6. Não ative Auto Apply nos workspaces HCP Terraform.
7. Se um plan interno mostrar destruição inesperada, interrompa o run antes de continuar para o próximo projeto.
8. Se a sessão AWS expirar, renove as credenciais com o script central e reexecute somente o workflow que falhou.
9. Um run anterior ao último run do projeto precedente não vale para a reconstrução atual. A cronologia obrigatória é Kubernetes → Database → Auth → Backend.
10. Não exclua recursos manualmente pelo console para corrigir divergências de Terraform.
11. Produção é uma etapa separada e exige aprovação explícita no GitHub Environment `production`.

## 3. Repositórios e recursos

| Projeto | Repositório |
|---|---|
| Kubernetes/EKS/New Relic | `tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra` |
| Database/RDS | `tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra` |
| Auth/API Gateway/Lambdas | `tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless` |
| Backend Spring Boot | `tiagomiele/fiap-tech-challenge-fase3-oficina-backend` |

Configuração esperada:

```text
Região AWS: us-west-2
HCP Organization: oficina-fiap-soat-fase-2
HCP Project: soat-fase3
Cluster homolog: oficina-homolog
Namespace homolog: oficina-homolog
Cluster production: oficina-production
Namespace production: oficina-production
```

Workspaces HCP:

```text
oficina-kubernetes-homolog
oficina-database-homolog
oficina-auth-homolog
oficina-newrelic-homolog

oficina-kubernetes-production
oficina-database-production
oficina-auth-production
oficina-newrelic-production
```

## 4. Preparação permanente — fazer uma vez por computador/conta

### 4.1 Instalar ferramentas no Windows

Abra PowerShell como usuário normal:

```powershell
winget install --id Git.Git -e
winget install --id GitHub.cli -e
winget install --id Hashicorp.Terraform -e --architecture x64
winget install --id Amazon.AWSCLI -e
winget install --id Kubernetes.kubectl -e
winget install --id Helm.Helm -e
winget install --id EclipseAdoptium.Temurin.21.JDK -e
winget install --id Docker.DockerDesktop -e
```

Feche e abra o PowerShell. O Docker Desktop pode exigir logout, reinicialização e aceite dos termos na primeira abertura. Valide:

```powershell
git --version
gh --version
terraform version
aws --version
kubectl version --client
helm version
java -version
docker --version

$TerraformInfo = terraform version -json | ConvertFrom-Json
if ($TerraformInfo.platform -ne 'windows_amd64') {
  throw "Terraform incompatível: $($TerraformInfo.platform). Instale windows_amd64."
}

$OpenSsl = (Get-Command openssl -ErrorAction SilentlyContinue).Source
if (-not $OpenSsl) {
  $OpenSsl = Join-Path $env:ProgramFiles 'Git\usr\bin\openssl.exe'
}
if (-not (Test-Path $OpenSsl)) {
  throw 'OpenSSL não encontrado. Reinstale o Git for Windows.'
}
& $OpenSsl version
```

O deploy remoto prepara Java, Maven, Docker, Helm e Node nos runners. As instalações locais acima deixam o computador apto também para diagnóstico, build e validações manuais; Maven e Node podem ser adicionados depois, pois o Backend usa Maven Wrapper e os workflows instalam Node automaticamente.

### 4.2 Autenticar GitHub e HCP Terraform

```powershell
gh auth login --hostname github.com --git-protocol https --web
gh auth status --hostname github.com
terraform login
```

Use uma conta com acesso administrativo aos quatro repositórios e um token HCP com acesso à organização `oficina-fiap-soat-fase-2`.

### 4.3 Clonar ou atualizar os quatro repositórios

```powershell
$Root = 'C:\fiap-fase3'
New-Item -ItemType Directory -Force $Root | Out-Null

$Repositories = @(
  'fiap-tech-challenge-fase3-oficina-kubernetes-infra',
  'fiap-tech-challenge-fase3-oficina-database-infra',
  'fiap-tech-challenge-fase3-oficina-auth-serverless',
  'fiap-tech-challenge-fase3-oficina-backend'
)

foreach ($Name in $Repositories) {
  $Path = Join-Path $Root $Name
  if (-not (Test-Path (Join-Path $Path '.git'))) {
    git clone "https://github.com/tiagomiele/$Name.git" $Path
    if ($LASTEXITCODE -ne 0) { throw "Falha ao clonar $Name" }
  }

  $Changes = @(git -C $Path status --porcelain)
  if ($Changes.Count -gt 0) {
    throw "$Name possui alterações locais. Preserve-as antes de atualizar."
  }

  git -C $Path fetch origin
  if ($LASTEXITCODE -ne 0) { throw "Falha ao buscar atualizações de $Name" }

  git -C $Path switch homolog
  if ($LASTEXITCODE -ne 0) { throw "Falha ao selecionar homolog em $Name" }

  git -C $Path pull --ff-only origin homolog
  if ($LASTEXITCODE -ne 0) { throw "Falha ao atualizar $Name" }
}
```

### 4.4 Confirmar a GitHub App de sincronização

A GitHub App é permanente e independente do Devin. Consulte:

- configuração e Client ID: https://github.com/settings/apps/oficina-database-sync-tiagomiele;
- instalação e repositórios autorizados: https://github.com/settings/installations.

Use o campo **Client ID** exibido em **General**. Não use o **App ID** numérico legado (por exemplo, `4862665`): ele causa falha `401` com `Issuer claim (iss) must be an Integer` no fluxo atual.

Ela precisa ter:

```text
Nome: oficina-database-sync-tiagomiele
Repository permission: Environments — Read and write
Webhook: desativado
Eventos: nenhum
Repository access:
  - fiap-tech-challenge-fase3-oficina-backend
  - fiap-tech-challenge-fase3-oficina-kubernetes-infra
```

Nos repositórios de origem abaixo, devem existir em nível de repositório:

```text
fiap-tech-challenge-fase3-oficina-database-infra:
  variável SYNC_APP_CLIENT_ID
  secret SYNC_APP_PRIVATE_KEY

fiap-tech-challenge-fase3-oficina-auth-serverless:
  variável SYNC_APP_CLIENT_ID
  secret SYNC_APP_PRIVATE_KEY

fiap-tech-challenge-fase3-oficina-backend:
  variável SYNC_APP_CLIENT_ID
  secret SYNC_APP_PRIVATE_KEY
```

Valide apenas os nomes, sem exibir o conteúdo do secret:

```powershell
$SyncSources = @(
  'tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra',
  'tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless',
  'tiagomiele/fiap-tech-challenge-fase3-oficina-backend'
)

foreach ($Repo in $SyncSources) {
  Write-Host "Verificando $Repo"
  gh variable list --repo $Repo | Select-String '^SYNC_APP_CLIENT_ID'
  gh secret list --repo $Repo | Select-String '^SYNC_APP_PRIVATE_KEY'
}
```

Se algum item estiver ausente, use o Client ID e o `.pem` guardado fora dos repositórios:

```powershell
$ClientId = Read-Host 'Informe o Client ID da GitHub App'
if ([string]::IsNullOrWhiteSpace($ClientId)) {
  throw 'Client ID não informado.'
}
if ($ClientId -match '^\d+$') {
  throw 'O valor informado parece ser o App ID numérico. Copie o Client ID na seção General da GitHub App.'
}

$PemLocation = Read-Host 'Informe o arquivo .pem ou a pasta que o contém'
$PemPath = $PemLocation

if (Test-Path -LiteralPath $PemLocation -PathType Container) {
  $PemFiles = @(Get-ChildItem -LiteralPath $PemLocation -File -Filter '*.pem')
  if ($PemFiles.Count -eq 0) { throw 'Nenhum arquivo PEM encontrado na pasta.' }
  if ($PemFiles.Count -gt 1) {
    $PemFiles | Select-Object Name, FullName
    throw 'Há mais de um PEM. Execute novamente e informe o caminho completo do arquivo correto.'
  }
  $PemPath = $PemFiles[0].FullName
}

if (-not (Test-Path -LiteralPath $PemPath -PathType Leaf)) {
  throw 'O caminho informado não é um arquivo PEM.'
}

foreach ($Repo in $SyncSources) {
  gh variable set SYNC_APP_CLIENT_ID --body $ClientId --repo $Repo
  if ($LASTEXITCODE -ne 0) { throw "Falha ao configurar SYNC_APP_CLIENT_ID em $Repo" }

  Get-Content -Raw -LiteralPath $PemPath |
    gh secret set SYNC_APP_PRIVATE_KEY --repo $Repo
  if ($LASTEXITCODE -ne 0) { throw "Falha ao configurar SYNC_APP_PRIVATE_KEY em $Repo" }
}

foreach ($Repo in $SyncSources) {
  Write-Host "Confirmando $Repo"
  gh variable list --repo $Repo | Select-String '^SYNC_APP_CLIENT_ID'
  gh secret list --repo $Repo | Select-String '^SYNC_APP_PRIVATE_KEY'
}
```

Nunca cole o conteúdo do `.pem` no terminal como texto, no chat ou em documentação.

### 4.5 Preparar o New Relic

Tenha disponíveis, sem colocá-los neste documento:

- New Relic Account ID;
- User API key;
- Ingest License key.

A primeira execução com `-ConfigureNewRelic` solicitará o Account ID em texto normal e as duas chaves em prompts protegidos. Os valores serão armazenados fora dos repositórios em `C:\fiap-secrets`; não os envie ao chat nem os inclua em capturas.

## 5. Iniciar uma nova sessão AWS Academy

1. Abra o AWS Academy Learner Lab.
2. Clique em **Start Lab**.
3. Aguarde o indicador ficar verde.
4. Abra **AWS Details**.
5. Copie o bloco `[default]` completo para o clipboard.
6. Não cole o bloco no chat, em arquivo de projeto ou em prints.
7. Mantenha o Lab ativo durante todos os deploys.

Validação opcional antes do bootstrap:

```powershell
Set-Location C:\fiap-fase3\fiap-tech-challenge-fase3-oficina-backend
.\scripts\configure-environment.ps1 `
  -Environment homolog `
  -ValidateOnly
```

A validação deve terminar com acesso confirmado a ferramentas, AWS STS, HCP Terraform e GitHub.

## 6. Configurar homologação uma única vez

Ainda com o bloco `[default]` no clipboard:

```powershell
Set-Location C:\fiap-fase3\fiap-tech-challenge-fase3-oficina-backend

.\scripts\configure-environment.ps1 `
  -Environment homolog `
  -ConfigureNewRelic
```

Na primeira execução:

- pressione Enter nos prompts de senha/secret que oferecem geração automática, se quiser gerar valores novos;
- informe Account ID, User API key e License key do New Relic somente nos prompts protegidos;
- no AWS Academy, não use `-EnableSesDelivery` nem `-CreateSesIdentity`; as notificações permanecerão assíncronas em modo técnico `log`.

Resultado obrigatório:

```text
Configuração automática concluída para homolog.
Credenciais AWS: computador local, HCP Terraform e GitHub Environments atualizados.
```

Confirme imediatamente a conta e a sessão do Learner Lab atual:

```powershell
$Identity = aws sts get-caller-identity | ConvertFrom-Json
$Identity | Select-Object Account, Arn

if ($Identity.Arn -notmatch ':assumed-role/voclabs/') {
  throw "Sessão AWS Academy/LabRole inesperada: $($Identity.Arn). Não prossiga."
}
```

Compare `Account` com a conta exibida em **AWS Details** no Learner Lab atual. Não fixe o Account ID de uma execução anterior: o AWS Academy pode entregar outra conta ao reiniciar, trocar ou renovar o laboratório. O ARN STS deve conter `assumed-role/voclabs/`, e a role IAM derivada pelo projeto deve terminar em `voclabs/LabRole`.

Quando a conta mudar, o `configure-environment.ps1` propaga as credenciais novas para os GitHub Environments e para o Variable Set HCP. Os states HCP continuam sendo a fonte de verdade; no primeiro run da conta nova, revise o plan e confirme que ele está recriando a infraestrutura ausente sem exclusões inesperadas na conta atual.

Antes da infraestrutura existir, são normais avisos de outputs ausentes e:

```text
DEPLOY_ENABLED=false
```

O script cria/configura os oito workspaces HCP, Variable Set AWS, GitHub Environments e secrets estáveis. Ele não executa `terraform apply` nem deploy.

A partir daqui, não execute novamente o script apenas para propagar outputs; cada workflow fará sua própria sincronização.

Runs verdes executados antes deste bootstrap, durante correções de workflow ou antes da recriação do Kubernetes não substituem os passos seguintes. Registre os links dos runs desta reconstrução e confirme esta ordem cronológica antes do Backend:

| Ordem | Projeto | Condição para o run ser válido |
|---|---|---|
| 1 | Kubernetes | executado após o bootstrap da sessão AWS atual |
| 2 | Database | iniciado depois da conclusão do Kubernetes atual |
| 3 | Auth | iniciado depois da conclusão do Database atual |
| 4 | Backend | iniciado depois da conclusão do Auth atual |

Se o Kubernetes criar ou alterar VPC, subnets ou security group, qualquer run anterior do Database e do Auth fica obsoleto e deve ser refeito na ordem acima.

### 6.1 Confirmar HCP Terraform e GitHub Environments

Execute novamente em modo somente leitura:

```powershell
.\scripts\configure-environment.ps1 `
  -Environment homolog `
  -ValidateOnly
```

O comando deve confirmar a organização HCP e não deve mais listar workspaces ausentes. No HCP Terraform, confirme os oito nomes da seção 3, execução remota e **Auto Apply desativado**.

No GitHub, confirme que os quatro repositórios possuem os Environments:

```text
homolog
production
```

O Environment `production` deve possuir required reviewer/aprovação humana. O Environment `homolog` pode continuar sem gate manual para permitir a reconstrução sequencial.

## 7. Implantar Kubernetes/EKS e observabilidade inicial

Workflow: **Deploy homolog**.

Abra:

https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/actions/workflows/deploy-homolog.yml

1. Clique em **Run workflow**.
2. Em **Use workflow from**, selecione `homolog`.
3. Clique em **Run workflow**.
4. Aguarde o run terminar totalmente verde e registre seu link; ele inicia a cadeia válida desta reconstrução.

O mesmo run executa:

```text
Validate configuration and AWS
→ Terraform plan/apply da infraestrutura
→ sincronizar VPC/subnets/security group com Database e Auth
→ instalar Metrics Server e New Relic nri-bundle
→ verificar rollouts
→ Terraform plan/apply da observabilidade
→ summary
```

Critérios de avanço:

- cluster `oficina-homolog` criado;
- outputs de rede sincronizados;
- add-ons aprovados;
- Terraform observability aprovado;
- nenhuma destruição inesperada.

Nesta primeira execução, o Synthetic pode permanecer desativado porque o Backend ainda não possui LoadBalancer.

Não execute o workflow manual `Terraform plan`; o deploy já contém seu próprio plan.

## 8. Implantar Database/RDS

Workflow: **Terraform deploy**.

Abra:

https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/actions/workflows/terraform-apply.yml

1. Confirme que o run verde do passo 7 pertence à reconstrução atual.
2. Clique em **Run workflow**.
3. Selecione a branch `homolog`.
4. Inicie o workflow somente depois de o Kubernetes terminar.
5. Aguarde o run terminar totalmente verde e registre seu link.

Um run do Database anterior ao último Kubernetes não vale: os inputs de VPC, subnets e security group foram produzidos e sincronizados pelo passo 7.

O run executa:

```text
Validate configuration and AWS
→ Terraform init/plan/apply
→ gerar token temporário da GitHub App
→ sincronizar JDBC com Auth e Backend
→ definir DEPLOY_ENABLED=true no Backend
→ revogar token da GitHub App
→ summary
```

Critérios de avanço:

- RDS PostgreSQL criado em subnets privadas;
- `jdbc_url` produzido;
- sincronização com `oficina-auth-homolog` aprovada;
- `APP_DB_URL` e `DEPLOY_ENABLED=true` atualizados no Backend;
- nenhum secret exibido.

Não execute o PowerShell novamente após o RDS.

## 9. Implantar Auth, API Gateway e notificações

Workflow: **Terraform deploy**.

Abra:

https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/actions/workflows/terraform-deploy.yml

1. Confirme que o run verde do Database foi iniciado depois do Kubernetes atual.
2. Clique em **Run workflow**.
3. Selecione a branch `homolog`.
4. Inicie o workflow somente depois de o Database terminar e sincronizar o JDBC.
5. Aguarde o run terminar totalmente verde e registre seu link.

O run executa:

```text
Validate configuration and AWS
→ package/test da Lambda Java 21
→ Terraform init/plan/apply
→ criar Lambdas, API Gateway, authorizer, SNS, SQS e DLQ
→ gerar token temporário da GitHub App
→ sincronizar API_GATEWAY_BASE_URL, AUTH_BASE_URL e NOTIFICATION_ENDPOINT com Backend
→ revogar token
→ summary
```

Critérios de avanço:

- quatro Lambdas criadas/atualizadas;
- API Gateway criado;
- login CPF e Lambda Authorizer provisionados;
- mensageria SNS/SQS/DLQ criada;
- notificação configurada no modo `log` no AWS Academy;
- URLs sincronizadas com o GitHub Environment `homolog` do Backend.

Nesta primeira aplicação, o Auth pode ainda estar usando uma URL anterior/vazia de Backend. Isso será corrigido automaticamente no passo 11, após o LoadBalancer existir.

## 10. Implantar Backend no EKS

Workflow: **CD**.

Abra:

https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/actions/workflows/cd.yml

1. Confirme pelos horários que os runs atuais estão na ordem Kubernetes → Database → Auth.
2. Confirme no run do Database que `Sincronizar JDBC com Auth e Backend` passou; essa etapa define `DEPLOY_ENABLED=true`.
3. Clique em **Run workflow**.
4. Selecione a branch `homolog`.
5. Inicie o workflow somente depois de o Auth terminar.
6. Aguarde o run terminar totalmente verde e registre seu link.

Não configure `DEPLOY_ENABLED` manualmente. Se o gate receber `false`, pare e identifique qual run precedente foi omitido ou executado fora de ordem.

O run executa:

```text
Validate application
→ build/test Java 21
→ construir e publicar imagem no GHCR
→ validar DEPLOY_ENABLED=true
→ aplicar ConfigMap, Secret, Deployment, Service, HPA e PDB
→ aguardar rollout
→ smoke test interno de readiness
→ capturar hostname/IP do LoadBalancer
→ sincronizar URL com Auth, Backend e New Relic
→ ativar configuração do Synthetic
→ revogar token da GitHub App
→ summary
```

Critérios de avanço:

- `Validate application` verde;
- `Docker build & push (GHCR)` verde;
- `Deploy HOMOLOG` verde;
- rollout do `deployment/oficina-app` concluído;
- smoke test retornando `UP`;
- LoadBalancer público capturado;
- sincronização Backend/Auth/New Relic/GitHub aprovada.

Não execute novamente `configure-environment.ps1` após esse deploy.

## 11. Reaplicar Auth com o LoadBalancer final

O Backend acabou de gravar `backend_base_url` no workspace `oficina-auth-homolog`. Agora o API Gateway precisa materializar essa nova URL.

Abra novamente:

https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/actions/workflows/terraform-deploy.yml

1. Selecione `homolog`.
2. Execute o workflow uma segunda vez.
3. Aguarde o run ficar verde.

Resultado esperado:

- atualizações nas integrações do API Gateway;
- nenhum recurso destruído;
- rotas protegidas apontando para o novo LoadBalancer do Backend.

Essa segunda execução não é duplicação do mesmo projeto: ela consome um output novo produzido pelo Backend.

## 12. Reaplicar Kubernetes/New Relic com o health final

O Backend também gravou:

```text
HEALTH_CHECK_URL=<LoadBalancer>/actuator/health
SYNTHETIC_MONITOR_ENABLED=true
```

Abra novamente:

https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/actions/workflows/deploy-homolog.yml

1. Selecione `homolog`.
2. Execute o workflow uma segunda vez.
3. Aguarde o run ficar verde.

Resultado esperado:

- infraestrutura EKS sem mudanças destrutivas;
- add-ons saudáveis;
- Synthetic Monitor criado/atualizado com a URL final;
- condição de alerta NRQL criada/atualizada;
- observabilidade concluída.

## 13. Validar saúde técnica

### 13.1 Obter URLs não sensíveis do GitHub Environment

```powershell
$BackendRepo = 'tiagomiele/fiap-tech-challenge-fase3-oficina-backend'
$BackendBaseUrl = gh variable get BACKEND_BASE_URL --repo $BackendRepo --env homolog
$ApiGatewayBaseUrl = gh variable get API_GATEWAY_BASE_URL --repo $BackendRepo --env homolog

if (-not $BackendBaseUrl) { throw 'BACKEND_BASE_URL ausente.' }
if (-not $ApiGatewayBaseUrl) { throw 'API_GATEWAY_BASE_URL ausente.' }
```

### 13.2 Validar Backend público

```powershell
curl.exe -fsS "$BackendBaseUrl/actuator/health"
curl.exe -fsS "$BackendBaseUrl/actuator/health/readiness"
curl.exe -I "$BackendBaseUrl/swagger-ui/index.html"
```

Resultados esperados:

```json
{"status":"UP"}
```

O health geral pode também listar os grupos `liveness` e `readiness`.

### 13.3 Validar Kubernetes

```powershell
aws eks update-kubeconfig `
  --region us-west-2 `
  --name oficina-homolog

kubectl get nodes
kubectl get pods -n oficina-homolog
kubectl get deployment,service,hpa,pdb -n oficina-homolog
kubectl get deployment oficina-app -n oficina-homolog -o yaml |
  Select-String 'topologySpreadConstraints'

kubectl rollout status deployment/oficina-app `
  -n oficina-homolog `
  --timeout=120s
```

Critérios:

- nodes `Ready`;
- pods do Backend `Running` e `Ready`;
- Service `oficina-app` com endereço externo;
- HPA e PDB presentes;
- `topologySpreadConstraints` presente no Deployment;
- rollout concluído.

## 14. Executar validação funcional E2E

Workflow: **Validacao E2E**.

Abra:

https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/actions/workflows/e2e.yml

1. Clique em **Run workflow**.
2. Selecione a branch `homolog`.
3. Em `environment`, escolha `homolog`.
4. Deixe `run_load_test=false` na primeira execução.
5. Inicie o workflow.
6. Aguarde `Postman/Newman (homolog)` terminar verde.

A coleção valida o fluxo funcional e de segurança usando URLs e credenciais do GitHub Environment sem exibi-las, incluindo autenticação por CPF, emissão/uso do JWT, rejeição de CPF inválido, rota protegida sem token, token adulterado e chamadas integradas ao Backend.

Depois, opcionalmente, reexecute com:

```text
run_load_test=true
```

Isso acrescenta um smoke de carga limitado com k6 após o fluxo funcional.

## 15. Validar New Relic

Esta validação visual é opcional para o provisionamento, mas recomendada para evidência de observabilidade.

Em https://one.newrelic.com confirme:

- APM: `oficina-backend-homolog`;
- cluster: `oficina-homolog`;
- telemetria das Lambdas do Auth;
- telemetria do RDS;
- Synthetic Monitor usando a URL final `/actuator/health`;
- resultado recente saudável;
- dashboards e alert conditions criados pelo Terraform.

Pode haver atraso de alguns minutos entre o deploy e a primeira telemetria.

## 16. Checklist final de homologação

Considere a Oficina Fase 3 completamente implantada somente quando todos estiverem aprovados:

- [ ] sessão AWS Academy ativa durante os deploys;
- [ ] bootstrap terminou com código zero;
- [ ] Kubernetes run inicial verde;
- [ ] Database run verde e JDBC sincronizado;
- [ ] Auth run inicial verde e URLs sincronizadas;
- [ ] Backend run verde, smoke aprovado e LoadBalancer capturado;
- [ ] Auth run final verde com integrações atualizadas;
- [ ] Kubernetes/New Relic run final verde com Synthetic ativado;
- [ ] `/actuator/health` retornando `UP`;
- [ ] pods, Service, HPA e PDB saudáveis;
- [ ] workflow E2E Newman verde;
- [ ] nenhuma destruição inesperada;
- [ ] nenhum secret exposto.

## 17. Resumo operacional mínimo

Depois das configurações permanentes, uma reconstrução de homologação exige somente:

```text
1. Start Lab
2. copiar [default]
3. executar configure-environment.ps1 uma vez
4. Run Kubernetes homolog
5. Run Database homolog
6. Run Auth homolog
7. Run Backend homolog
8. Run Auth homolog novamente
9. Run Kubernetes homolog novamente
10. Run E2E homolog
```

Não são necessários:

- plan manual antes de cada deploy;
- PowerShell após cada apply;
- PAT pessoal;
- atualização manual de outputs;
- alteração manual de variables no HCP/GitHub;
- workflow orquestrador único entre os quatro repositórios.

## 18. Se as credenciais AWS expirarem

1. Pare de iniciar novos workflows.
2. Reinicie/renove o Learner Lab.
3. Copie o novo bloco `[default]` completo.
4. Execute novamente:

```powershell
Set-Location C:\fiap-fase3\fiap-tech-challenge-fase3-oficina-backend
.\scripts\configure-environment.ps1 `
  -Environment homolog `
  -ConfigureNewRelic
```

5. Os valores New Relic já armazenados serão reutilizados; não os redigite, exceto se tiverem sido rotacionados.
6. Reexecute somente o workflow que falhou.
7. Não recomece toda a sequência se os projetos anteriores terminaram verdes e seus states continuam íntegros.

## 19. Produção — somente após homologação aprovada

Produção repete a mesma dependência:

```text
Kubernetes production
→ Database main
→ Auth main
→ Backend main
→ Auth main novamente
→ Kubernetes production novamente
→ E2E production
```

### 19.1 Pré-condições

- os quatro repositórios devem possuir PRs de promoção de `homolog` para `main` revisados e prontos;
- faça os merges **um por vez**, na ordem Kubernetes → Database → Auth → Backend, aguardando o deploy verde antes do próximo merge;
- GitHub Environment `production` deve possuir aprovação humana;
- uma nova sessão AWS Academy deve estar ativa;
- não reutilize valores locais específicos de homologação.

### 19.2 Configurar produção

Para uma produção com proteção e alta disponibilidade:

```powershell
Set-Location C:\fiap-fase3\fiap-tech-challenge-fase3-oficina-backend
.\scripts\configure-environment.ps1 `
  -Environment production `
  -ConfigureNewRelic
```

O perfil padrão de produção configura RDS Multi-AZ, proteção contra exclusão e snapshot final.

Somente para uma demonstração descartável no AWS Academy, quando limites do Lab impedirem o perfil real, existe o override:

```powershell
.\scripts\configure-environment.ps1 `
  -Environment production `
  -ConfigureNewRelic `
  -UseAwsAcademyDisposableProductionProfile
```

Esse override reduz a proteção/alta disponibilidade e não representa uma produção real.

### 19.3 Workflows de produção

Kubernetes:

https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-kubernetes-infra/actions/workflows/deploy-production.yml

Database, usando branch `main`:

https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-database-infra/actions/workflows/terraform-apply.yml

Auth, usando branch `main`:

https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-auth-serverless/actions/workflows/terraform-deploy.yml

Backend, usando branch `main`:

https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/actions/workflows/cd.yml

E2E, usando branch `main` e environment `production`:

https://github.com/tiagomiele/fiap-tech-challenge-fase3-oficina-backend/actions/workflows/e2e.yml

Se o código já estiver em `main` e a tarefa for apenas recriar a AWS, execute manualmente os workflows acima na ordem da seção 19. Se houver promoção de código, o merge em `main` inicia automaticamente o workflow correspondente; por isso, faça um merge por vez e não dispare uma execução manual duplicada.

A aprovação do GitHub Environment ocorre **antes** do job de produção. Depois da aprovação, `plan` e `apply/deploy` executam no mesmo run, sem uma segunda pausa. Antes de aprovar, revise o código, o workspace selecionado e as evidências verdes de homologação; acompanhe o plan no run e cancele imediatamente se aparecer destruição ou substituição inesperada.

### 19.4 Sequência operacional validada de produção

Execute e aguarde cada run terminar verde antes de iniciar o seguinte:

```text
1. Kubernetes production
2. Database main/production
3. Auth main/production — aplicação inicial
4. Backend main/production
5. Auth main/production — reaplicação obrigatória com o LoadBalancer final
6. Kubernetes production — reaplicação do Synthetic/New Relic com o health final
7. E2E main + environment production + run_load_test=true
```

O passo 5 é obrigatório quando o Backend acabou de criar ou alterar o LoadBalancer. O deploy do Backend sincroniza `backend_base_url` no workspace `oficina-auth-production`, mas somente uma nova aplicação do Terraform Auth atualiza as integrações `HTTP_PROXY` do API Gateway.

Se essa reaplicação for omitida, o Backend direto pode estar saudável enquanto uma rota protegida pelo API Gateway retorna `503 Service Unavailable`, porque a integração continua apontando para um LoadBalancer antigo.

Na reaplicação do Auth, confirme no plan que as integrações usam o mesmo hostname registrado como `BACKEND_BASE_URL` no GitHub Environment `production`. Não avance ao E2E se as URLs forem diferentes.

### 19.5 Validação final de produção

No workflow E2E:

1. selecione a branch `main`;
2. escolha `environment=production`;
3. marque `run_load_test=true`;
4. aguarde os jobs Newman e k6 terminarem verdes.

Critérios mínimos:

- Newman: 24 requisições sem falha;
- Newman: 25 assertions sem falha;
- envio da OS para aprovação retorna HTTP 200 e estado `AGUARDANDO_APROVACAO`;
- outro cliente consulta a OS pela rota protegida e recebe HTTP 404, sem revelar propriedade;
- k6 sem requisições/checks com falha;
- todos os thresholds de latência aprovados.

## 20. Acessar simultaneamente os bancos de homologação e produção

Os RDS são privados. Não torne o banco público e não configure o endpoint RDS diretamente no DBeaver. O acesso validado usa um pod temporário `socat` dentro de cada cluster e `kubectl port-forward` limitado a `127.0.0.1`.

Arquivos necessários em `C:\fiap-fase3`:

```text
CONECTAR-DBEAVER-HOMOLOGACAO.ps1
CONECTAR-DBEAVER-PRODUCAO.ps1
```

Pré-requisitos:

- sessão AWS Academy ativa;
- `aws`, `kubectl` e DBeaver instalados;
- acesso aos clusters `oficina-homolog` e `oficina-production`;
- scripts salvos fora dos repositórios ou mantidos apenas como utilitários locais, sem secrets embutidos.

### 20.1 Janela PowerShell 1 — homologação

Abra uma janela exclusiva e execute:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force

$env:KUBECONFIG = "$HOME\.kube\config-oficina-homolog"
aws eks update-kubeconfig `
  --region us-west-2 `
  --name oficina-homolog `
  --alias oficina-homolog `
  --kubeconfig $env:KUBECONFIG

Unblock-File -LiteralPath C:\fiap-fase3\CONECTAR-DBEAVER-HOMOLOGACAO.ps1
C:\fiap-fase3\CONECTAR-DBEAVER-HOMOLOGACAO.ps1 -LocalPort 15432
```

### 20.2 Janela PowerShell 2 — produção

Abra uma segunda janela exclusiva e execute:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force

$env:KUBECONFIG = "$HOME\.kube\config-oficina-production"
aws eks update-kubeconfig `
  --region us-west-2 `
  --name oficina-production `
  --alias oficina-production `
  --kubeconfig $env:KUBECONFIG

Unblock-File -LiteralPath C:\fiap-fase3\CONECTAR-DBEAVER-PRODUCAO.ps1
C:\fiap-fase3\CONECTAR-DBEAVER-PRODUCAO.ps1 -LocalPort 25432
```

Os kubeconfigs separados evitam que a troca do contexto global interrompa, limpe ou direcione comandos para o cluster incorreto.

### 20.3 Configurar as conexões no DBeaver

Crie duas conexões PostgreSQL:

| Ambiente | Host | Porta | Database | Usuário | Senha | SSL mode |
|---|---|---:|---|---|---|---|
| Homologação | `localhost` | `15432` | exibido pelo script, normalmente `oficina` | exibido pelo script | copiada pelo script | `require` |
| Produção | `localhost` | `25432` | exibido pelo script, normalmente `oficina` | exibido pelo script | copiada pelo script | `require` |

Em cada janela:

1. aguarde a mensagem de túnel ativo;
2. cole a senha na conexão correspondente do DBeaver;
3. teste a conexão;
4. volte ao PowerShell e pressione Enter para limpar a área de transferência;
5. mantenha a janela aberta enquanto usar o banco;
6. ao terminar, pressione Enter novamente para encerrar o `port-forward` e remover o pod temporário.

O pod possui duração máxima de duas horas. Para continuar depois disso, execute novamente o script do ambiente. O `Bypass` da política de execução vale somente para a janela atual e desaparece ao fechá-la.

Nunca copie a senha para documentos, logs ou mensagens. Em produção, evite alterações manuais e prefira consultas somente leitura.

## 21. Validar a aplicação pelo Swagger

O Swagger é publicado pelo Backend no LoadBalancer do ambiente:

```text
<BACKEND_BASE_URL>/swagger-ui/index.html
```

O contrato OpenAPI bruto está em:

```text
<BACKEND_BASE_URL>/v3/api-docs
```

O Swagger deve ser aberto pela URL direta do Backend, não pela URL base do API Gateway. A operação `POST /auth/cpf` exibida no Swagger possui uma URL de servidor própria e chama o Auth Serverless/API Gateway do mesmo ambiente.

### 21.1 Descobrir e abrir o Swagger de homologação

Com a sessão AWS Academy ativa:

```powershell
$HomologKubeconfig = "$HOME\.kube\config-oficina-homolog"
aws eks update-kubeconfig `
  --region us-west-2 `
  --name oficina-homolog `
  --alias oficina-homolog `
  --kubeconfig $HomologKubeconfig

$HomologBackendHost = kubectl `
  --kubeconfig $HomologKubeconfig `
  get service oficina-app `
  --namespace oficina-homolog `
  --output jsonpath='{.status.loadBalancer.ingress[0].hostname}'

if ([string]::IsNullOrWhiteSpace($HomologBackendHost)) {
  throw 'O Backend de homologação ainda não possui LoadBalancer.'
}

$HomologBackendUrl = "http://$HomologBackendHost"
Invoke-RestMethod "$HomologBackendUrl/actuator/health"
Start-Process "$HomologBackendUrl/swagger-ui/index.html"
```

### 21.2 Descobrir e abrir o Swagger de produção

```powershell
$ProductionKubeconfig = "$HOME\.kube\config-oficina-production"
aws eks update-kubeconfig `
  --region us-west-2 `
  --name oficina-production `
  --alias oficina-production `
  --kubeconfig $ProductionKubeconfig

$ProductionBackendHost = kubectl `
  --kubeconfig $ProductionKubeconfig `
  get service oficina-app `
  --namespace oficina-production `
  --output jsonpath='{.status.loadBalancer.ingress[0].hostname}'

if ([string]::IsNullOrWhiteSpace($ProductionBackendHost)) {
  throw 'O Backend de produção ainda não possui LoadBalancer.'
}

$ProductionBackendUrl = "http://$ProductionBackendHost"
Invoke-RestMethod "$ProductionBackendUrl/actuator/health"
Start-Process "$ProductionBackendUrl/swagger-ui/index.html"
```

Resultado esperado do health check: `status` igual a `UP`. Se a sessão AWS Academy tiver sido recriada, execute primeiro toda a cadeia do ambiente; não reutilize um hostname antigo salvo no navegador.

### 21.3 Obter o JWT administrativo

A senha é `APP_ADMIN_PASSWORD`, definida na preparação do ambiente e armazenada localmente em CLIXML protegido pelo usuário do Windows. Para copiá-la temporariamente sem exibi-la no terminal:

```powershell
$Environment = 'homolog' # troque para 'production' quando necessário
$SecureAdminPassword = Import-Clixml "C:\fiap-secrets\oficina-$Environment\backend-admin-password.clixml"
$PasswordPointer = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($SecureAdminPassword)
try {
  [Runtime.InteropServices.Marshal]::PtrToStringBSTR($PasswordPointer) | Set-Clipboard
} finally {
  [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($PasswordPointer)
}
```

No Swagger:

1. abra `POST /auth/login`;
2. clique em **Try it out**;
3. envie:

```json
{
  "email": "admin@oficina.local",
  "senha": "<cole APP_ADMIN_PASSWORD>"
}
```

4. copie somente o valor de `accessToken` da resposta;
5. clique em **Authorize** no topo da página;
6. no campo `bearerAuth`, cole somente o JWT, sem acrescentar a palavra `Bearer`;
7. confirme em **Authorize** e feche a janela;
8. volte ao PowerShell e execute `Set-Clipboard ''`.

O resultado esperado do login administrativo é HTTP 200 com `papel=FUNCIONARIO_DA_OFICINA`. Um HTTP 403 nos testes posteriores normalmente indica JWT expirado; faça login novamente e substitua o token no botão **Authorize**.

### 21.4 Autenticar um cliente pelo CPF

Use apenas um CPF sintético de cliente ativo já cadastrado no ambiente:

1. abra `POST /auth/cpf` no grupo `01-Autenticação e Logins para Aplicação`;
2. clique em **Try it out**;
3. informe o CPF com ou sem máscara;
4. execute e espere HTTP 200;
5. copie somente `accessToken`;
6. abra **Authorize**, substitua o token anterior pelo JWT do cliente e confirme.

Essa operação é executada pelo API Gateway configurado em `AUTH_BASE_URL`. Se o endereço exibido na caixa **Servers** estiver ausente, apontar para outro ambiente ou terminar em `.invalid`, não continue: reaplique o Auth e depois o Backend para sincronizar as URLs.

### 21.5 Checklist funcional manual em homologação

Use dados sintéticos e execute as operações na ordem indicada pelo Swagger:

```text
1. POST /auth/login — obter JWT administrativo
2. POST /usuarios — criar técnico com e-mail sintético único ou usar um já existente
3. POST /clientes — criar proprietário e segundo cliente
4. POST /veiculos — cadastrar veículo do proprietário
5. POST /servicos — cadastrar serviço
6. POST /ordens-servico — abrir OS
7. POST /auth/login — obter JWT do técnico e atualizar Authorize
8. POST /ordens-servico/{numeroOs}/servicos — incluir serviço
9. POST /ordens-servico/{numeroOs}/enviar-para-aprovacao
10. POST /auth/cpf — obter JWT do proprietário e atualizar Authorize
11. GET /consulta/ordens-servico/{numeroOs}/status — esperar HTTP 200
12. POST /auth/cpf — obter JWT do segundo cliente e atualizar Authorize
13. GET /consulta/ordens-servico/{numeroOs}/status — esperar HTTP 404
14. POST /auth/cpf — restaurar JWT do proprietário
15. POST /ordens-servico/{numeroOs}/aprovar
16. POST /auth/login — restaurar JWT do técnico
17. POST /ordens-servico/{numeroOs}/concluir-reparo
18. POST /auth/cpf — restaurar JWT do proprietário
19. POST /ordens-servico/{numeroOs}/confirmar-pagamento
20. POST /auth/login — restaurar JWT do técnico
21. POST /ordens-servico/{numeroOs}/entregar
22. POST /auth/cpf — restaurar JWT do proprietário
23. GET /ordens-servico/{numeroOs}/historico
```

Valide principalmente:

- HTTP 200/201 conforme documentado em cada operação;
- `enviar-para-aprovacao` retorna sem timeout e muda a OS para `AGUARDANDO_APROVACAO`;
- o proprietário consegue consultar e aprovar sua OS;
- um segundo cliente recebe HTTP 404 ao tentar consultar a OS do proprietário;
- o histórico contém as transições na ordem executada;
- nenhum CPF completo, senha ou JWT é incluído em screenshots ou evidências.

O Swagger valida o Backend e o login serverless por CPF. As demais operações do Swagger são enviadas diretamente ao LoadBalancer do Backend; portanto, ele não substitui o workflow E2E para validar o Lambda Authorizer e todas as rotas protegidas do API Gateway.

### 21.6 Uso seguro em produção

Em produção:

- prefira health, Swagger/OpenAPI, login e operações somente leitura;
- não reutilize dados pessoais reais em demonstrações;
- só execute o fluxo completo quando houver autorização para criar e alterar dados;
- use valores sintéticos facilmente identificáveis e não apague registros manualmente;
- finalize com o workflow E2E em `main`, `environment=production` e `run_load_test=true`, que permanece como evidência oficial da integração completa.

## 22. Critérios de parada

Não inicie o próximo projeto se ocorrer qualquer uma destas situações:

- `aws sts get-caller-identity` falha, retorna outra conta ou outra role;
- as credenciais AWS do GitHub e do HCP pertencem a sessões diferentes;
- o workflow informa configuração incompleta;
- o plan apresenta exclusão ou substituição não prevista;
- Kubernetes não produz VPC, subnets privadas e security group;
- EKS, nodes, add-ons ou rollouts não ficam saudáveis;
- Database não produz `jdbc_url`;
- token da GitHub App não é gerado ou a sincronização retorna 403/404;
- Auth não produz `api_base_url` e `notification_endpoint`;
- `DEPLOY_ENABLED` não fica `true` depois de Kubernetes e Database;
- Backend falha em rollout, readiness, smoke test ou captura do LoadBalancer;
- a segunda aplicação do Auth não atualiza as integrações;
- New Relic não possui Account ID, User API key e License key válidos;
- qualquer job termina vermelho ou cancelado.

Corrija a causa, mantenha os states existentes e reexecute somente a etapa afetada. Não avance assumindo que uma sincronização falha será corrigida por um projeto posterior.

## 23. Troubleshooting rápido

| Sintoma | Causa provável | Ação |
|---|---|---|
| `ExpiredToken`, `InvalidClientTokenId` ou STS falha | Learner Lab expirou | Reinicie o Lab, copie `[default]`, execute o script central e reexecute o run afetado |
| Workspace HCP ausente | bootstrap não terminou ou token sem acesso | Execute o script sem `-ValidateOnly`; confirme organização e token HCP |
| `ENABLE_TERRAFORM_APPLY` ou `TF_APPLY_ENABLED` diferente de `true` | GitHub Environment incompleto | Reexecute o bootstrap e confirme o Environment correto |
| `DEPLOY_ENABLED=false` no Backend | Database não terminou a sincronização ou seu run é anterior ao Kubernetes atual | Execute Database depois do Kubernetes, reaplique Auth e só então Backend; não altere a variável manualmente |
| GitHub App retorna 401/403/404 | Client ID/PEM/permissão/instalação incorretos | Confirme que `SYNC_APP_CLIENT_ID` contém o **Client ID**, não o App ID numérico; valide `Environments: Read and write`, instalação no Backend/Kubernetes e o PEM nos três produtores |
| `jdbc_url` ausente | apply do Database incompleto | Revise o state/run do Database e reexecute seu workflow |
| API Gateway retorna 503, mas o Backend direto está saudável | integração `HTTP_PROXY` ainda aponta para o LoadBalancer anterior | Compare `backend_base_url` do plan Auth com `BACKEND_BASE_URL`; reaplique Auth após o Backend |
| Synthetic ausente ou URL antiga | Kubernetes/New Relic ainda não foi reaplicado | Execute novamente `Deploy homolog` após o Backend |
| Backend sem LoadBalancer após 30 tentativas | Service/ELB/EKS não ficou pronto | Examine eventos do Service e pods; não grave URL manual |
| Pod em `ImagePullBackOff` | imagem/referência GHCR inválida | Revise `Docker build & push` e o `image_ref` do mesmo run |
| New Relic sem dados logo após deploy | janela normal de ingestão | Gere tráfego e aguarde aproximadamente 5–15 minutos antes de investigar |
| script DBeaver informa que o contexto é do ambiente errado | ambas as janelas compartilham o kubeconfig global | use os kubeconfigs separados das seções 20.1 e 20.2 |
| script PowerShell não está assinado digitalmente | política de execução bloqueia arquivo baixado | use `Set-ExecutionPolicy -Scope Process Bypass` e `Unblock-File`; não altere `LocalMachine` |
| Swagger não abre | LoadBalancer ausente, URL antiga ou Backend indisponível | redescubra o hostname pela seção 21, valide `/actuator/health` e revise o rollout |
| `POST /auth/cpf` no Swagger aponta para `.invalid` ou outro ambiente | `AUTH_BASE_URL` não chegou ao deploy atual | reaplique Auth, Backend e abra novamente `/v3/api-docs` sem cache |
| Swagger retorna 401 no login administrativo | senha incorreta ou segredo do pod dessincronizado | use o CLIXML do mesmo ambiente e confirme que o último deploy do Backend terminou verde |
| Swagger retorna 403 depois de autenticar | JWT expirado ou papel incompatível com a operação | faça login novamente ou troque para o token do papel indicado no grupo da operação |

Tempos típicos, sem garantia:

```text
Kubernetes/EKS inicial: 20–40 min
Database/RDS: 10–25 min
Auth: 5–15 min
Backend: 8–20 min
Reaplicação Auth: 5–15 min
Reaplicação Kubernetes/New Relic: 5–20 min
Primeira ingestão New Relic: 5–15 min após tráfego
```

Se o mesmo erro de configuração persistir, preserve os logs do run e não tente contornar criando recursos manualmente.

## 24. Matriz de sincronização automática

| Projeto produtor | Output | Consumidor atualizado automaticamente |
|---|---|---|
| Kubernetes | VPC, subnets privadas, security group EKS | HCP Database e Auth |
| Database | JDBC do RDS | HCP Auth e GitHub Environment Backend |
| Auth | API Gateway e endpoint de notificações | GitHub Environment Backend |
| Backend | LoadBalancer e health URL | HCP Auth, HCP New Relic, Backend e Kubernetes GitHub Environments |

Essa matriz explica as duas reaplicações finais em homologação e produção: Auth e New Relic precisam consumir a URL criada somente durante o deploy do Backend.
