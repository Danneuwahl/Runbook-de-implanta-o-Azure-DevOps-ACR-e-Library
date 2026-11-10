# Pipeline universal e módulos Terraform — Lumiere Pricing

## Escopo e estado do modelo

`.pipelines/azure-pipelines.yml` orquestra uma execução manual para um cliente, provider e ambiente. O fluxo é **Validate → Terraform plan/apply → Build/Push → Deploy Compose**. O modelo de runtime documentado aqui usa **uma VM Linux por ambiente**, frontend e API em containers, Nginx no host e PostgreSQL local em Compose para DEV/HML; PRD continua usando PostgreSQL gerenciado (por exemplo, RDS). Não há alta disponibilidade. A integração das variáveis precisa ser confirmada no pipeline consumidor antes de afirmar que está ativa.

Os módulos Terraform estão em `infra/terraform/aws` e `infra/terraform/azure`. A pipeline calcula tags Docker únicas por cliente/ambiente/build/commit, usa autenticação federada para provisionamento e registry e bloqueia qualquer plano Terraform contendo `delete` (inclusive replacement) antes de `apply`.

A pipeline não cria o backend remoto de Terraform, DNS, certificado TLS, banco PostgreSQL gerenciado, Service Connections, Azure DevOps Environments ou um agente self-hosted. Em DEV/HML, o PostgreSQL é iniciado como serviço Compose com volume persistente; em PRD, o banco gerenciado/RDS é pré-existente. Esses são itens de bootstrap/onboarding descritos abaixo. Na primeira execução, `DEPLOY_POOL_READY=false` permite provisionar e publicar sem tentar deploy; a operação comum ocorre quando esse valor está `true`.

## 1. Como executar

Em **Run pipeline**, selecione `clientCode`, `cloudProvider` (`aws`/`azure`) e `environment` (`dev`/`hml`/`prod`). O slug deve ser minúsculo, até 20 caracteres. O pipeline seleciona automaticamente o Variable Group:

- Cliente atual `lumiere`: `LUMIERE-PRICING-DEV`, `LUMIERE-PRICING-HML` ou `LUMIERE-PRICING-PROD`.
- Outros clientes: `LUMIERE-PRICING-<CLIENT_CODE>-<ENV>`; por exemplo `LUMIERE-PRICING-CLIENTE-A-DEV`.

Os valores `CLIENT_CODE`, `ENVIRONMENT` e `CLOUD_PROVIDER` no grupo devem coincidir com as escolhas. O desenvolvedor não digita senhas, tokens ou valores de secrets durante a execução. O parâmetro `clientCode` é seletor de perfil e não mecanismo de autorização: restringir quem pode executar a pipeline e configurar checks/approvals nos recursos de cada cliente, especialmente PROD.

O Terraform é aplicado em toda execução normal, para reconciliar a infraestrutura declarada. O apply é idempotente quando não há drift. O plano roda no mesmo job que o apply; não há aprovação manual do conteúdo do plano antes do apply. Para reduzir risco, um guard bloqueia recursos cuja ação inclua `delete`; o Environment `lumiere-<cliente>-<ambiente>-infra` deve ter aprovação/check de PROD. O guard não substitui revisão do plano nem proteção de IAM/RBAC.

## 2. Recursos por provider

### AWS (`infra/terraform/aws`)

Cria VPC dedicada, subnet pública em uma AZ, Internet Gateway, rota pública, Security Group com entrada TCP 80/443 (sem SSH), uma EC2 Ubuntu 24.04, Elastic IP estável, ECR para `app` e `api` com scan-on-push e tags imutáveis, Instance Profile/IAM Role e dois containers vazios no Secrets Manager. A role da VM recebe SSM Core, pull somente nos dois repositórios e `GetSecretValue` somente nos dois secrets criados.

O Elastic IP permanece estável para DNS e allowlist do PostgreSQL externo, mas tem custo de IPv4 público. A VM é uma única instância, sem autoscaling, balanceador ou failover. `WEB_INGRESS_CIDRS` controla origens de HTTP/HTTPS; não há regra SSH.

### Azure (`infra/terraform/azure`)

Cria Resource Group dedicado, VNet/subnet, NSG com entrada somente TCP 80/443 (sem SSH), Public IP Standard estático, NIC, uma VM Linux Ubuntu 24.04, Azure Container Registry Basic sem admin user, Key Vault com RBAC e System-assigned Managed Identity. A identidade recebe `AcrPull` no ACR e `Key Vault Secrets User` no vault. O Key Vault é criado vazio: o administrador deve cadastrar os dois valores de secret após o primeiro apply.

As role assignments requerem `Microsoft.Authorization/roleAssignments/write` no escopo aplicável (normalmente Owner ou User Access Administrator). `Contributor` sozinho não basta. A propagação do RBAC pode atrasar a primeira leitura de ACR/Key Vault. O Key Vault usa purge protection; exclusão/recuperação deve seguir o procedimento administrativo do Azure.

### Itens que permanecem fora do Terraform

Os módulos Terraform não criam banco PostgreSQL gerenciado, registro DNS nem certificado TLS. O Compose da aplicação inicia PostgreSQL local somente em DEV/HML, sem publicar a porta 5432 no host e com volume persistente separado por ambiente. PRD usa PostgreSQL gerenciado/RDS; ele deve aceitar conexão somente da origem adequada (EIP AWS/Public IP Azure ou conectividade privada existente), com firewall, TLS e conectividade validados antes do deploy. Cloud-init instala Docker/Compose, Nginx e Certbot e deixa proxy HTTP configurado. Depois que os DNS apontarem para o IP estático e o host responder pela porta 80, o administrador executa Certbot e verifica a renovação.

## 3. Bootstrap obrigatório (uma vez por cliente/ambiente)

1. **Backend remoto de Terraform pré-criado e protegido.** AWS: bucket S3 dedicado/compartimentado, encryption e versioning, com permissões de lock via `use_lockfile=true`; Azure: storage account e container pré-existentes, com acesso Azure AD/OIDC. O pipeline não cria esses backends nem configura sua retenção/backup.
2. **Extensões/agentes do Azure DevOps.** Instalar AWS Toolkit para os tasks `AWSShellScript@1` e Terraform extension para `TerraformInstaller@1`. O Terraform fica fixado em 1.10.5; as extensões devem estar instaladas na organização.
3. **Service Connections protegidas**, autorizadas somente a esta pipeline e padronizadas por cliente/ambiente:
   - `sc-lumiere-<cliente>-<env>-aws-iac` — AWS OIDC, para Terraform. Menor privilégio para VPC, EC2, EIP, IAM/Instance Profile, ECR e Secrets Manager; inclui `iam:PassRole` limitado à role da VM.
   - `sc-lumiere-<cliente>-<env>-aws-registry` — AWS OIDC para build/push apenas nos dois ECR repos.
   - `sc-lumiere-<cliente>-<env>-azure-iac` — ARM Service Connection com Workload Identity Federation. Escopo adequado para RG, rede, VM, IP, ACR, Key Vault e role assignments.
   - `sc-lumiere-<cliente>-<env>-azure-registry` — ARM WIF com `AcrPush` no ACR daquele ambiente.
4. **Azure DevOps Environments** `lumiere-<cliente>-<env>-infra` e `lumiere-<cliente>-<env>`. Configurar approvals e checks em PROD; autorizar explicitamente a pipeline. Evitar autorização global de recursos para todas as pipelines.
5. **Variable Group preenchido** conforme a matriz abaixo. `POSTGRES_PASSWORD` é secret em DEV/HML; não cadastrar a senha ou connection string em texto aberto. Marcar `DEPLOY_POOL_READY=false` inicialmente.
6. Rodar a pipeline: Terraform provisiona recursos e o build publica as imagens. O log de Terraform mostra IPs, repositórios e referências/nomes dos secrets (identificadores, não seus valores).
7. **Configurar secrets de runtime** sem compartilhar seus valores. Em DEV/HML, `POSTGRES_PASSWORD` deve ser cadastrado como secret no Variable Group apropriado ou obtido do cofre pela identidade aprovada da VM, conforme o mecanismo efetivamente integrado ao template `PrepareVM`; o template documentado espera a variável secreta `POSTGRES_PASSWORD`. Não usar `DB_SECRET_REF` como se fosse uma connection string para o banco local. Em PRD, preservar `DB_SECRET_REF` apontando para o secret existente com a connection string PostgreSQL/RDS completa e manter `JWT_SECRET_REF` para o JWT. AWS usa os ARNs do Secrets Manager; Azure usa nomes individuais no Key Vault. Não usar JSON único. A senha local e a conexão RDS são formatos distintos.
8. **Configurar DNS/TLS e agente de deploy:** apontar `APP_DOMAIN` e `API_DOMAIN` para o IP público, executar Certbot após a propagação e validar Nginx. Registrar um agente self-hosted dedicado no pool `lumiere-pricing-<cliente>-<env>-deploy`, na VM, com Docker access. AWS usa SSM para administração; Azure pode usar Run Command. A pipeline Terraform não registra agentes Azure DevOps e não guarda PAT para isso.
9. Após confirmar que agente e secrets funcionam, mudar `DEPLOY_POOL_READY=true`. A partir daí o run faz também deploy e health checks.

O agente na VM acessa registries/secrets com a identidade do host: AWS Instance Role ou Azure System-assigned Managed Identity. Não criar access keys AWS, client secrets Azure nem senhas ACR.

## 4. Variable Group — nomes e valores

### Contrato comum (todos os providers)

| Nome | Conteúdo/regra |
|---|---|
| `CLIENT_CODE` | Igual ao `clientCode` selecionado; slug minúsculo, até 20 caracteres. |
| `ENVIRONMENT` | `dev`, `hml` ou `prod`. |
| `CLOUD_PROVIDER` | `aws` ou `azure`. |
| `RESOURCE_PREFIX` | Prefixo curto minúsculo (por exemplo `lumiere`). Em AWS é o `project_name`; em Azure é `resource_prefix`. |
| `NETWORK_CIDR`, `SUBNET_CIDR` | IPv4 dedicado; a subnet deve ser menor e contida na rede. |
| `WEB_INGRESS_CIDRS` | Lista CIDR IPv4 separada por vírgulas; restringe TCP 80/443 nos Security Group/NSG dos dois clouds. `0.0.0.0/0` somente para aplicação pública. |
| `APP_DOMAIN`, `API_DOMAIN` | Hostnames DNS distintos. `API_DOMAIN` deve corresponder ao host de `VITE_API_URL`. |
| `APP_PORT`, `API_PORT` | Portas de loopback distintas entre 1024–65535; precisam coincidir com Nginx/Terraform e Compose. |
| `VITE_API_URL` | HTTPS terminado em `/v1`, por exemplo `https://api.cliente.example/v1`; é embutida no bundle do frontend, portanto build é por ambiente. |
| `REGISTRY_LOGIN_SERVER` | Host sem esquema/barra: AWS `account.dkr.ecr.region.amazonaws.com`; Azure `<acr>.azurecr.io`. |
| `IMAGE_REPOSITORY_APP`, `IMAGE_REPOSITORY_API` | Namespaces dos repositórios. Em AWS devem ser `<RESOURCE_PREFIX>/<CLIENT_CODE>/<ENVIRONMENT>/app` e `.../api`, conforme Terraform. Em Azure devem coincidir com `app_repository`/`api_repository` passados pelo pipeline. |
| `CORS_ORIGINS` | Origins separadas por vírgula; deve incluir `https://APP_DOMAIN`. |
| `DEPLOY_PATH` | Exatamente `/opt/lumiere/<CLIENT_CODE>/<ENVIRONMENT>`. |
| `DEPLOY_POOL_READY` | `false` no bootstrap; `true` somente depois de pool/agent e secrets prontos. |
| `DATABASE_MODE` | `compose` em DEV/HML; `rds` em PRD. Deve corresponder ao ambiente selecionado. |
| `POSTGRES_DB` | DEV/HML: nome do banco criado pelo container, por exemplo `lumierepricing_dev` ou `lumierepricing_hml`; alfanumérico/underscore, 1–63 caracteres. Não requerido em PRD. |
| `POSTGRES_USER` | DEV/HML: usuário do PostgreSQL, por exemplo `lumiere_app`; alfanumérico/underscore, 1–63 caracteres. Não requerido em PRD. |
| `POSTGRES_PASSWORD` | **Secret obrigatório em DEV/HML**: senha aleatória exclusiva por ambiente, 32–128 caracteres base64url (`A-Z`, `a-z`, `0-9`, `_`, `-`), armazenada como variável secreta ou obtida de secret store. Nunca incluir valor no YAML/log/runbook. Não requerido em PRD. |
| `POSTGRES_VOLUME_NAME` | DEV/HML: volume Docker persistente e distinto por ambiente, por exemplo `lumiere-pricing-dev-postgres-data` e `lumiere-pricing-hml-postgres-data`. Não remover durante deploy/rollback. Em PRD não é usado. |
| `ConnectionStrings__Default` | DEV/HML: gerada pelo `PrepareVM` apontando para `Host=postgres;Port=5432` e para o database/user/senha acima; não definir uma connection string apontando a `localhost`. PRD: connection string PostgreSQL/RDS atual, preservada via `DB_SECRET_REF`. |
| `DB_SECRET_REF`, `JWT_SECRET_REF` | `DB_SECRET_REF`: em PRD, identificador da connection string RDS; em DEV/HML não substitui `POSTGRES_PASSWORD` salvo se o template estiver explicitamente adaptado para buscar a senha. `JWT_SECRET_REF`: identificador do secret JWT. Podem ficar vazios no primeiro apply enquanto `DEPLOY_POOL_READY=false`; validar requisitos do template consumidor. |

Exemplos abaixo são valores de configuração propostos para os Variable Groups, não valores lidos do Azure DevOps. Ajustar database/user se a aplicação exigir nomes diferentes. Criar uma senha secreta aleatória e diferente em cada ambiente; não reutilizar estes exemplos como credencial.

| Variable Group | `DATABASE_MODE` | `POSTGRES_DB` | `POSTGRES_USER` | `POSTGRES_PASSWORD` | `POSTGRES_VOLUME_NAME` |
|---|---|---|---|---|---|
| `LUMIERE-PRICING-DEV` | `compose` | `lumierepricing_dev` | `lumiere_app_dev` | Secret independente de DEV; formato exigido indicado acima | `lumiere-pricing-dev-postgres-data` |
| `LUMIERE-PRICING-HML` | `compose` | `lumierepricing_hml` | `lumiere_app_hml` | Secret independente de HML; formato exigido indicado acima | `lumiere-pricing-hml-postgres-data` |
| `LUMIERE-PRICING-PROD` | `rds` | Não definir/usar | Não definir/usar | Não definir/usar | Não definir/usar |

Em DEV/HML, o template deve montar `ConnectionStrings__Default` com `Host=postgres;Port=5432`, o valor do respectivo `POSTGRES_DB`/`POSTGRES_USER` e a senha secreta do mesmo grupo. Não cadastrar manualmente uma connection string com `localhost`. Em PRD, manter o fluxo atual de `DB_SECRET_REF` e o endpoint RDS.


> **Integração pendente antes de habilitar deploy DEV/HML:** o contrato acima documenta as variáveis necessárias, mas elas também precisam existir nos Variable Groups `LUMIERE-PRICING-DEV` e `LUMIERE-PRICING-HML` e ser passadas pelo pipeline consumidor aos templates `PrepareVM`/`DeployVM`. O pacote analisado contém a pipeline universal antiga; confirme a chamada real dos templates e o mecanismo de entrega de `POSTGRES_PASSWORD` antes de definir `DEPLOY_POOL_READY=true`. A API deve compartilhar a rede Compose e usar o hostname do serviço (`postgres`), nunca `localhost`.

### AWS adicionais

| Nome | Conteúdo/regra |
|---|---|
| `AWS_REGION` | Região de deployment. |
| `AWS_ACCOUNT_ID` | ID numérico de 12 dígitos; a pipeline compara com a conta efetiva da Service Connection de IaC e registry. |
| `AWS_STATE_BUCKET`, `AWS_STATE_REGION` | Bucket S3 remoto preexistente e sua região. Estado isolado na chave `lumiere/<cliente>/aws/<ambiente>/terraform.tfstate`. |
| `AWS_ECR_REGISTRY` | Igual ao `REGISTRY_LOGIN_SERVER`. |
| `AWS_SECRET_PATH_PREFIX` | Prefixo do path Secrets Manager, por exemplo `/lumiere`. O módulo cria `<prefix>/<cliente>/<ambiente>/app` e `/api`, sem valor inicial. |
| `INSTANCE_TYPE` | Tipo EC2 disponível na região (default recomendado no módulo `t3.small`). |
| `ROOT_VOLUME_SIZE_GIB` | Opcional; tamanho do volume raiz (default 30 GiB). |
| `ENABLE_DETAILED_MONITORING` | Opcional; `true`/`false` (default `false`). |

Permissões pipeline AWS: OIDC trust restringida à organização/projeto/service connection; endpoint `aws-iac` precisa criar EC2/VPC/EIP/ECR/Secrets Manager e role/profile da VM, com `iam:PassRole` delimitado. Endpoint `aws-registry` precisa `ecr:GetAuthorizationToken` e push somente nos dois repos. A role da VM não tem permissões de build nem escrita nos secrets. Se `POSTGRES_PASSWORD` for lida diretamente como secret do Azure DevOps Variable Group, ela é disponibilizada ao job de preparação; se for lida do Secrets Manager pela VM, conceder leitura somente ao secret correspondente e adaptar o template para esse mecanismo, sem credenciais estáticas.

### Azure adicionais

| Nome | Conteúdo/regra |
|---|---|
| `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` | UUIDs da tenant e subscription usadas pelo ARM WIF; a pipeline confere a subscription/tenant efetivamente autenticadas. |
| `AZURE_LOCATION` | Região do Resource Group/VM. |
| `AZURE_STATE_RESOURCE_GROUP`, `AZURE_STATE_STORAGE_ACCOUNT`, `AZURE_STATE_CONTAINER` | Backend remoto pré-existente. Chave `lumiere/<cliente>/azure/<ambiente>/terraform.tfstate`. |
| `AZURE_ACR_NAME`, `AZURE_ACR_LOGIN_SERVER` | Nome globalmente único e login server `<name>.azurecr.io`. O login server deve ser igual a `REGISTRY_LOGIN_SERVER`. |
| `AZURE_KEY_VAULT_NAME` | Nome globalmente único (3–24 caracteres). O vault é criado vazio. |
| `ADMIN_SSH_PUBLIC_KEY` | Chave pública para bootstrap Linux; nenhuma regra SSH é aberta. Não é chave privada/segredo. |
| `AZURE_VM_SIZE`, `ADMIN_USERNAME` | Opcionais; defaults no módulo são `Standard_B2s` e `azureuser`. |

Permissões Azure: endpoint `azure-iac` precisa criar recursos no escopo de subscription e atribuir RBAC (`roleAssignments/write` no ACR/Key Vault); revisar se precisa de Owner/User Access Administrator ou role customizada. Endpoint `azure-registry` tem somente `AcrPush` no ACR. A identidade da VM tem `AcrPull` e leitura de secrets, sem administração do vault. Se `POSTGRES_PASSWORD` for obtida do Key Vault pela VM, adicionar somente a leitura desse secret ao desenho de identidade/template; se vier como secret de Library, proteger o Variable Group e autorizar apenas a pipeline necessária.

## 5. Build, deploy, segredos e rollback

- As imagens `app` e `api` são tagueadas `<cliente>-<ambiente>-<BuildId>-<commit>`; o frontend recebe `VITE_API_URL` como build arg. AWS ECR aplica imutabilidade; no ACR as tags geradas são únicas pelo pipeline, mas a configuração do módulo Basic não estabelece uma policy de imutabilidade. Configurar retenção/limpeza dos registries de acordo com necessidade de rollback.
- Deploy usa Docker Compose sobre o agente self-hosted na VM. Para DEV/HML, o Compose inicia PostgreSQL local, mantém os dados em volume persistente e a API conecta pelo DNS interno `postgres:5432`; não publicar 5432 no host. O template `PrepareVM` gera `ConnectionStrings__Default` para esse endereço e grava o `.env` com modo `0600`. PRD mantém a connection string existente do RDS. O mecanismo de obtenção da senha em DEV/HML deve corresponder à integração efetiva (secret de Variable Group ou leitura do secret store pela identidade da VM); não documentar os dois como se estivessem simultaneamente implementados. As portas da aplicação continuam em `127.0.0.1` e Nginx faz o proxy. Segredos de ambiente continuam acessíveis a administradores Docker/root e como configuração do container; restringir acesso ao host.
- O container frontend oferece `/health`; o script exige HTTP 200. A API não implementa `/health` no código observado. O script apenas confirma uma resposta HTTP não-5xx em `/`, o que não valida banco de dados ou dependências. Criar health endpoint real no código é melhoria recomendada.
- Em sucesso, grava `.last-good-image-tag` (sem segredo). Em falha, refaz pull e sobe a tag anterior com os secrets atuais; se não existir versão anterior, para a tentativa inicial. O rollback não reverte schema/dados PostgreSQL; `ApplyMigrations` é forçado a `false`.
- `terraform plan` é convertido a JSON e verificado por `.pipelines/scripts/deny-destructive-plan.py`. Qualquer ação que contenha `delete` bloqueia apply. Para alteração legítima que exija replacement, fazer procedimento separado, revisar o plano e autorização de mudança; não remover o guard para liberar automaticamente.

## 6. Templates e pastas

- `.pipelines/azure-pipelines.yml`: selectors, carregamento de grupo e ordem dos estágios.
- `.pipelines/templates/validate-profile.yml`: contrato e checagens fail-fast.
- `.pipelines/templates/terraform-apply-aws.yml` / `terraform-apply-azure.yml`: inicialização de backend, plan, guard e apply do provider selecionado.
- `.pipelines/templates/build-push-aws.yml` / `build-push-azure.yml`: build e publicação para ECR/ACR.
- `.pipelines/templates/deploy-existing-vm.yml`: job de deploy em pool padronizado.
- `.pipelines/scripts/providers/`: login e fetch de secret pelo host.
- `.pipelines/scripts/deploy-compose.sh`: escrita protegida de env, Compose, checks e rollback.
- `infra/terraform/aws` e `infra/terraform/azure`: runtime de infraestrutura; `infra/terraform/scripts/deny-destructive-plan.py`: guarda de plano.

## 7. Verificação e limites de validação

A validação local prevista é `terraform fmt -check -recursive infra/terraform`, `terraform init -backend=false` e `terraform validate` para cada módulo, além de `bash -n` e parse YAML. `plan/apply` real exige Service Connections, backends, quotas e subscriptions AWS/Azure válidos; nenhum recurso cloud foi criado nesta tarefa. Validar primeiro em DEV e confirmar RBAC/propagação, DNS, Nginx/TLS, acesso a PostgreSQL, agent pool e rollback antes de liberar HML/PROD.

Este modelo cobre um runtime por cliente/ambiente, sem HA. O IP AWS é Elastic IP e o Azure é Public IP Standard estático. Não há auto scaling, LB, multi-AZ, CDN, criação de PostgreSQL gerenciado, DNS gerenciado ou TLS automatizado. O PostgreSQL de DEV/HML é um container com volume local à VM; não é serviço gerenciado nem alta disponibilidade. PRD permanece no RDS externo. Custos de VM, IPv4 público, registry, logs, tráfego e secrets devem ser estimados/monitorados por perfil.
