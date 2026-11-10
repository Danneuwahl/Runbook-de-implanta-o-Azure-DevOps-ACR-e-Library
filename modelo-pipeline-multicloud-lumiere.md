# Modelo de pipeline reutilizável multi-cloud — Lumiere Pricing

**Decisão de baseline:** reproduzir o runtime observado no pacote: uma VM Linux por ambiente; frontend e API em containers Docker Compose; Nginx/TLS no host; PostgreSQL externo. Este modelo **não promete alta disponibilidade** e não cria banco de dados.

## 1. Objetivo operacional

Uma mesma base de código, infraestrutura como código e pipeline deve atender a diferentes empresas clientes sem editar YAML por implantação. Um administrador configura uma vez o perfil do cliente — Variable Group, segredo externo, Service Connection e limites de rede — e o operador escolhe o perfil/ambiente e executa **Run pipeline**.

O pipeline deve falhar antes de qualquer alteração quando perfil, credencial, provider ou valores obrigatórios forem inválidos. Implantação e rollback usam tags de imagem imutáveis. Não há etapa de destruição automática de infraestrutura.

## 2. Baseline comum e adaptadores de provedor

A arquitetura de execução é a mesma; somente a infraestrutura, registry, identidade e comandos de controle variam por provedor.

| Capacidade | AWS | Azure |
|---|---|---|
| VM Linux única por ambiente | EC2 | Azure Linux VM |
| Rede/Firewall | VPC, subnet e Security Group | VNet, subnet e NSG |
| Registry de imagens | ECR | ACR |
| Nginx/TLS do host | Nginx + Certbot no Linux | Nginx + Certbot no Linux |
| Identidade da VM para pull de imagens | Instance Profile / IAM Role | Managed Identity + RBAC no ACR |
| Segredos de runtime | AWS Secrets Manager | Azure Key Vault |
| PostgreSQL | Externo à implantação; connection string no segredo | Externo à implantação; connection string no segredo |
| Deploy | Docker Compose com a mesma definição da aplicação | Docker Compose com a mesma definição da aplicação |

A pipeline comum não deve conhecer comandos específicos de ECR/ACR, EC2/Azure VM ou Secrets Manager/Key Vault. Esses detalhes ficam em adaptadores reutilizáveis por provider, com uma interface comum de entradas e saídas.

## 3. Fluxo lógico da pipeline

1. **Selecionar perfil e ambiente:** o operador informa o perfil do cliente e escolhe `dev`, `hml` ou `prod`. O ambiente define o Variable Group e as políticas de aprovação.
2. **Validar configuração:** conferir `CLOUD_PROVIDER` (`aws` ou `azure`), Service Connection, identidade e permissões, imagem/registry, hostname, portas, caminho de deploy, agente, referência do segredo e conectividade esperada. Não imprimir segredos.
3. **Planejar infraestrutura:** executar `terraform plan` com estado remoto isolado por `cliente/provider/ambiente`; publicar o plano como artefato. Nunca aplicar plano de produção sem o gate definido.
4. **Provisionar/atualizar a VM:** `terraform apply` idempotente cria ou atualiza VM, rede mínima, regras de entrada e identidade de máquina para acesso ao registry/secret store. Não provisiona nem apaga PostgreSQL externo.
5. **Build e publicação:** compilar frontend e API uma vez por execução, marcar com Build ID/commit e publicar no registry do provider selecionado. Evitar tag imutável compartilhada entre clientes.
6. **Deploy comum:** transferir somente o Compose e executar `docker compose pull` / `up -d` no destino. Configuração da API é fornecida pelo secret store/identidade da VM; segredos não devem ir para o repositório ou para o comando visível no log.
7. **Verificar e reverter:** validar frontend (`/health`) e rota da API pelo domínio; registrar a versão apenas após sucesso. Em falha, restaurar o Build ID anterior se a imagem ainda estiver retida no registry.
8. **Diagnóstico:** publicar status e logs sanitizados; nunca despejar connection strings, JWT secrets ou configurações completas resolvidas do Compose.

O run normal deve reconstruir apenas recursos declarados e realizar deploy; mudanças destrutivas de rede, disco ou identidade precisam ser identificadas no plano e bloqueadas por aprovação explícita.

## 4. Perfil do cliente e variáveis

Usar um Variable Group por cliente e ambiente, por exemplo `LUMIERE-PRICING-<CLIENTE>-DEV`, `...-HML` e `...-PROD`. Para a instalação atual podem ser mantidos os nomes existentes até a migração.

### Seleção e identificação

- `CLIENT_CODE`: identificador imutável do cliente, usado em tags e nomes de recursos.
- `CLOUD_PROVIDER`: `aws` ou `azure`.
- `ENVIRONMENT`: derivado da seleção/branch, não digitado livremente no deploy.
- `CLOUD_REGION`: região do provider.
- `RESOURCE_PREFIX`: prefixo único, minúsculo e sem dados pessoais.
- `TERRAFORM_STATE_KEY`: chave lógica isolada por cliente/provider/ambiente.

### Infraestrutura de runtime

- tamanho/tipo de VM;
- CIDR da rede e CIDRs administrativos autorizados;
- `DEPLOY_PATH`, `COMPOSE_PROJECT_NAME`, `APP_PORT`, `API_PORT`;
- domínio do frontend, domínio/base da API e contato do Certbot;
- nomes do registry e dos repositórios da API/frontend;
- nome do pool e do agente ou método equivalente de execução remota;
- referência de segredo de runtime por provider.

### Mapeamento de variáveis não secretas

Os valores abaixo pertencem ao Variable Group do perfil/ambiente, salvo quando identificados como saídas calculadas pelo pipeline. Os nomes são o contrato lógico entre o perfil, os templates comuns e os adaptadores de cloud; cada adaptador mapeia o contrato para recursos AWS ou Azure.

| Variável | Origem/classe | Finalidade e regra |
|---|---|---|
| `CLIENT_CODE` | Library, não secreta | Código estável do cliente; usar em nomes/tags, nunca dado pessoal. |
| `CLOUD_PROVIDER` | Library ou parâmetro restrito | Somente `aws` ou `azure`; seleciona o adaptador. Não aceitar texto livre sem validação. |
| `ENVIRONMENT` | Branch/parâmetro controlado | `dev`, `hml` ou `prod`; deve concordar com o Variable Group e com as aprovações. |
| `CLOUD_REGION` | Library, não secreta | Região AWS/Azure conforme provider selecionado. |
| `RESOURCE_PREFIX` | Library, não secreta | Prefixo de nomes, validado para as restrições do provider. |
| `TERRAFORM_STATE_KEY` | Library, não secreta | Identificador lógico exclusivo por cliente/provider/ambiente; o backend e suas credenciais são configuração protegida separada. |
| `VM_SIZE` | Library, não secreta | Tipo/tamanho de uma VM Linux; valor permitido deve ser validado na região. |
| `NETWORK_CIDR`, `SUBNET_CIDR`, `ADMIN_CIDRS` | Library, não secreta | Rede da VM e origens administrativas permitidas; não abrir administração para `0.0.0.0/0`. |
| `APP_DOMAIN`, `API_DOMAIN`, `CERTBOT_EMAIL` | Library, não secreta | DNS/TLS e roteamento; o DNS deve apontar para o destino antes da emissão HTTP-01. |
| `DEPLOY_PATH`, `COMPOSE_PROJECT_NAME` | Library, não secreta | Caminho e nome do projeto Compose; validar caminho absoluto e impedir diretórios protegidos do host. |
| `APP_PORT`, `API_PORT` | Library, não secreta | Portas locais dos dois containers; manter compatibilidade com Nginx/Compose. |
| `VITE_API_URL` | Library, não secreta, build-time | URL/base consumida pelo bundle estático do frontend. Como é compilada na imagem, a imagem pode ficar vinculada ao ambiente; definir estratégia de imagem por ambiente ou configuração runtime antes de promover o mesmo digest. |
| `IMAGE_REPOSITORY_APP`, `IMAGE_REPOSITORY_API` | Library, não secreta | Repositórios lógicos; hostname/login do registry são resolvidos pelo adaptador. |
| `DB_SECRET_REF`, `JWT_SECRET_REF` | Library, identificadores não secretos | Nome/ARN do segredo no cofre do cliente. Não armazenar os valores do segredo nesses campos. |
| `DEPLOY_TRANSPORT` | Library, enum validado | `agent` ou mecanismo remoto aprovado (`ssm`/Run Command). Define como o Compose chega à VM; não habilitar SSH público por conveniência. |
| `IMAGE_TAG` | Saída calculada | Build ID + commit, imutável e rastreável por cliente/ambiente; rollback usa tag anterior ainda retida. |
| `REGISTRY_LOGIN_SERVER` | Saída/configuração derivada | Endereço do ECR/ACR associado ao perfil. Não é credencial. |

### Configuração específica do provider

| Provider | Variáveis/identificadores por perfil | Observações |
|---|---|---|
| AWS | `AWS_ACCOUNT_ID`, `AWS_REGION`, `AWS_VPC_CIDR`, `AWS_SUBNET_CIDR`, `AWS_AZ`, `AWS_EC2_INSTANCE_TYPE`, `AWS_ECR_REGISTRY`, `AWS_ECR_REPOSITORY_APP`, `AWS_ECR_REPOSITORY_API`, `AWS_INSTANCE_PROFILE`, `AWS_DB_SECRET_ARN`, `AWS_JWT_SECRET_ARN`, `AWS_STATE_BUCKET`, `AWS_STATE_LOCK_TABLE` | Não incluir `AWS_ACCESS_KEY_ID` nem `AWS_SECRET_ACCESS_KEY`. Região/account/ARNs são metadados; os segredos da aplicação ficam no Secrets Manager. Backend remoto de Terraform precisa de controle de acesso e isolamento do state por instalação. |
| Azure | `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `AZURE_RESOURCE_GROUP`, `AZURE_LOCATION`, `AZURE_VNET_CIDR`, `AZURE_SUBNET_CIDR`, `AZURE_VM_SIZE`, `AZURE_ACR_NAME`, `AZURE_ACR_LOGIN_SERVER`, `AZURE_KEY_VAULT_NAME`, `AZURE_VM_MANAGED_IDENTITY_ID`, `AZURE_DB_SECRET_NAME`, `AZURE_JWT_SECRET_NAME`, `AZURE_STATE_STORAGE_ACCOUNT`, `AZURE_STATE_CONTAINER` | IDs/nome do tenant, subscription, resource group, vault e identidades não são credenciais. Não guardar client secret, senha de ACR ou chave de storage no YAML/Library; preferir WIF/identidade e permissões de dados mínimas. |

Se algum valor for diferente por ambiente, mantê-lo no Variable Group daquele ambiente. Valores globais realmente compartilhados podem ficar em um grupo comum somente de configuração não secreta; evitar grupos com nomes ambíguos ou que combinem dados de vários clientes.

### Credenciais e segredos

- As credenciais de provisionamento AWS/Azure ficam em **Service Connections** com menor privilégio, preferindo federação/identidade gerenciada a chaves estáticas.
- Connection string PostgreSQL e `Security__JwtSecret` ficam no Secrets Manager ou Key Vault do cliente/ambiente; a VM recebe acesso por IAM Role/Managed Identity. Referências/ARNs são variáveis comuns; os valores secretos nunca são variáveis de seleção.
- Não gravar segredos no YAML, Compose versionado, Terraform state não protegido ou logs.
- O Variable Group deve armazenar configuração e identificadores, não substituir o cofre para segredo de runtime.

O contrato de configuração da aplicação continua sendo `ConnectionStrings__Default` e `Security__JwtSecret`. No AWS, o bootstrap/adapter lê os segredos referenciados por `AWS_DB_SECRET_ARN` e `AWS_JWT_SECRET_ARN` via identidade EC2 e os injeta no processo/container sem logá-los. No Azure, o bootstrap/adapter faz o mesmo com `AZURE_DB_SECRET_NAME` e `AZURE_JWT_SECRET_NAME` usando Managed Identity. Os nomes dos segredos devem ser individuais; evitar um JSON completo como único valor secreto. Esse bootstrap/consumo de cofre é trabalho de infraestrutura a implementar e validar: não foi identificado no código atual um provider nativo de AWS Secrets Manager ou Azure Key Vault.

**Estado atual a migrar:** o `.pipelines/azure-pipelines.yml` fornecido inclui `ConnectionStrings__Default` e `Security__JwtSecret` na lista `envVars` enviada ao host. Na implementação do modelo, substituir esses valores por referências de cofre e autenticação da VM; então remover os valores de segredo dos Variable Groups e dos arquivos de ambiente de pipeline. Se houver uma etapa transitória em que o pipeline lê o cofre, mapear o segredo explicitamente como variável de ambiente do task, não como argumento de linha de comando, não escrever no log e reduzir o escopo de autorização.

### Service Connections e identidades — configuração única por instalação

Service Connections são endpoints protegidos do Azure DevOps, não variáveis secretas comuns. Um administrador de plataforma cria e testa as conexões uma vez durante o onboarding do cliente; o desenvolvedor não recebe access keys para editar. Autorizar cada pipeline explicitamente, em vez de marcar **Grant access permission to all pipelines**. A conexão só deve alcançar a conta/subscription, região e recursos daquele cliente/ambiente.

| Uso | AWS | Azure | Permissões e limites recomendados |
|---|---|---|---|
| Provisionamento/atualização IaC | Service Connection do tipo **AWS** do AWS Toolkit, com **Use OIDC**, assumindo uma role IAM dedicada. Ex.: `sc-lumiere-<cliente>-<ambiente>-aws-iac`. | **Azure Resource Manager Service Connection** com **Workload Identity Federation**, escopada ao Resource Group dedicado quando possível. Ex.: `sc-lumiere-<cliente>-<ambiente>-azure-iac`. | Somente ações para criar/atualizar os recursos previstos e identidade de VM; evitar `AdministratorAccess`, `Owner` ou escopo de conta/subscription sem necessidade. Separar o bootstrap que cria IAM/RBAC/identidades da role cotidiana, pois criar atribuições de acesso normalmente exige privilégio adicional. |
| Build e push das imagens | AWS Toolkit + conexão OIDC que assume role de publicação no ECR. Permissões mínimas de autenticação e push nos dois repositórios previstos; pode ser a mesma role de IaC somente se a análise de menor privilégio justificar. | Preferir login ao ACR pelo contexto autenticado da ARM connection com WIF e role `AcrPush` no registry; se o template usar `Docker@2`, pode ser necessário um endpoint Docker Registry/ACR autorizado e configurado sem senha duradoura. | A identidade do build publica; a identidade da VM tem apenas leitura. Não reutilizar o segredo de runtime da aplicação como credencial de registry. |
| Deploy remoto do Compose (quando não houver agente na VM) | Adapter via AWS Systems Manager: role pipeline OIDC com `ssm:SendCommand`/consulta restrita ao documento/instância-alvo; EC2 com instance role e SSM Agent autorizado. | Adapter via Azure Run Command/ARM usando a conexão WIF com permissão apenas sobre a VM-alvo; alternativamente agente self-hosted dedicado. | O transporte ainda é uma decisão de implementação do projeto. SSH só se necessário, com chave protegida/rotacionada e rede restrita; nunca abrir SSH para toda a Internet. |
| Segredos da aplicação em runtime | Sem Service Connection do pipeline: role no EC2 Instance Profile, vinculada à instância, lê somente os ARNs de segredos necessários. | Sem Service Connection do pipeline: System-assigned ou User-assigned Managed Identity da VM recebe `Key Vault Secrets User` no vault/escopo necessário. | AWS: `secretsmanager:GetSecretValue` somente nos segredos específicos e `kms:Decrypt` apenas se uma chave KMS gerenciada pelo cliente exigir. Azure: RBAC de leitura de segredos no vault, sem permissão de administração. |
| Leitura de segredo pelo próprio pipeline (opcional, evitar para runtime) | AWS OIDC Service Connection com `GetSecretValue` limitado ao segredo necessário, apenas em task que realmente precisa do valor. | ARM Service Connection + Variable Group vinculado ao Key Vault, com permissão `Get`/`List` ou RBAC equivalente. | Só usar quando a pipeline precisa do valor antes da VM. A integração do Variable Group mapeia nomes, busca valores a cada execução e não atualiza automaticamente a lista quando novos segredos são adicionados/removidos. Key Vault RBAC atrás de private endpoint não é suportado pela integração direta do Variable Group do Azure DevOps; nesse caso prefira identidade da VM ou desenho de rede compatível. |

**Federação AWS:** instalar a extensão oficial AWS Toolkit para Azure DevOps, criar provider OIDC AWS para o issuer da organização (`vstoken.dev.azure.com/<organization-guid>`) e trust policy com `aud=api://AzureADTokenExchange` e `sub=sc://<organização>/<projeto>/<service-connection>`. Essa condição limita a role à conexão aprovada. Não usar access/secret keys estáticas, mesmo que a extensão ainda aceite esse modo.

**Federação Azure:** criar uma ARM Service Connection com Workload Identity Federation (a Microsoft recomenda WIF em novas conexões), no escopo do Resource Group/subscription necessário. A identidade federada é a identidade do pipeline; é distinta da Managed Identity atribuída à VM. Escopos de dados (Key Vault/ACR) e control plane (VM/rede) devem ser concedidos separadamente quando possível.

### Seleção do perfil e limites do Azure DevOps

O operador deve escolher um perfil/ambiente previamente cadastrado e autorizado; não deve digitar account ID, ARN, Subscription ID, role ou connection string na execução. Nome de Variable Group e valores de Service Connection têm fases de avaliação/permissões diferentes: o grupo deve ser carregado em tempo de template e os endpoints devem ser autorizados como recursos protegidos do pipeline. Não presumir que uma macro de Library como `$(AWS_SERVICE_CONNECTION)` será aceita em todo campo de Service Connection.

Para atender de verdade ao requisito “cliente novo sem alterar YAML”, antes de produção executar um PoC no projeto Azure DevOps real: (1) receber perfil e ambiente como parâmetros validados; (2) resolver o Variable Group no tempo de template; (3) selecionar apenas Service Connections autorizadas ao pipeline; (4) rejeitar conexão de outro cliente/ambiente; (5) testar o erro de perfil ausente/inválido. A documentação pública do Azure Pipelines lista parâmetros tipados de template e recomenda usar parâmetros para escolhas controladas, mas não lista um tipo `serviceConnection` na tabela de tipos de parâmetro; portanto a seleção dinâmica de endpoints não é tratada aqui como recurso garantido.

Se o PoC não comprovar seleção dinâmica segura, o fallback suportável é uma definição fina por cliente/ambiente que apenas passa parâmetros e Service Connection estáticos para os templates centrais (sem copiar lógica de build/deploy), ou uma conexão por account/subscription compartilhada com escopo controlado e cadastro externo de perfis. Isso preserva reutilização e reduz YAML duplicado, mas deve ser escolhido conscientemente; não prometer seleção livre de qualquer conexão por Variable Group.

### Reuso sem editar o pipeline

Depois do PoC e do onboarding administrativo, a rotina do desenvolvedor deve ficar restrita a revisar/alterar variáveis não secretas do perfil ou atualizar o segredo no cofre AWS/Azure e executar **Run pipeline**. O cadastro inicial de Service Connections, identidades IAM/RBAC, cofre, rede, grupo e autorizações pertence à equipe de plataforma e não é repetido pelo desenvolvedor a cada release.

## 5. Limites explícitos deste baseline

- O banco é **externo**. A equipe responsável pelo cliente deve criar o PostgreSQL, fornecer a connection string no cofre e liberar a conexão de rede a partir da VM.
- Não há provisionamento de RDS ou Azure Database for PostgreSQL neste primeiro baseline.
- O código observado persiste anexos como bytes em `budgets.budget_files` no PostgreSQL; não foi localizado adapter S3/Blob. Portanto S3/Blob não será provisionado como dependência funcional até existir suporte no código.
- A infraestrutura será uma VM única por ambiente; indisponibilidade dessa VM interrompe o sistema. O baseline não inclui VMSS/Auto Scaling, balanceador redundante, failover ou SLA.
- DNS e certificado precisam ter uma estratégia definida por instalação. O host usa Nginx/TLS e encaminha as portas locais do Compose; DNS deve apontar ao endereço esperado antes da validação HTTP-01 do Certbot.
- A pipeline não deve declarar sucesso de restore/migration: o código atual não aplica migrations em Production. Mudanças de schema ficam fora do deploy rotineiro até existir uma etapa de migração controlada.

## 6. Estrutura de repositório recomendada

```text
.pipelines/
  azure-pipelines.yml                 # orquestra perfil/provider/ambiente
  docker-compose.yml                  # contrato comum dos dois containers
  templates/
    validate-profile.yml
    terraform-plan-apply.yml
    build-push.yml
    deploy-compose.yml
    providers/aws/                     # login, registry, IaC e controle AWS
    providers/azure/                   # login, registry, IaC e controle Azure
infra/
  terraform/
    modules/aws-vm/                    # rede mínima, EC2, identidade e outputs
    modules/azure-vm/                  # rede mínima, VM, identidade e outputs
    profiles/                          # configuração não secreta por instalação
```

Os módulos AWS/Azure devem apresentar outputs compatíveis: identificador/endereço da VM, registry, nome do agente/método de deploy, domínio/portas e referência do secret store. A pipeline comum consome esse contrato; ela não deve duplicar o workflow para cada cloud.

## 7. Lacunas observadas no pacote antes de implementar

1. Não há Terraform/Bicep/CloudFormation nem script de criação de VM no ZIP.
2. `.pipelines/azure-pipelines.yml` referencia templates do repositório externo `devops-templates`, cujo conteúdo não foi fornecido; o pipeline atual também não é provider-agnostic.
3. A arquitetura documentada em `docs/arquitetura.md` é uma proposta Azure mais ampla e marcada para validação de Infra; o baseline aprovado aqui deliberadamente não adota VMSS, storage de objetos nem banco gerenciado.
4. As Variable Groups, Service Connections, pools/agents, contas cloud, zonas DNS, rede e cofre são configurações externas ao ZIP e não podem ser testadas neste workspace.
5. A VM precisa ter um caminho de deploy autenticado que não abra SSH ao mundo. O método precisa ser escolhido entre agente self-hosted no host ou comandos remotos seguros do provider; os dois providers devem cumprir os mesmos requisitos operacionais.
6. O pipeline atual encaminha valores de connection string e JWT via `envVars`; a migração para acesso ao secret store por identidade da VM ainda não está implementada nem foi testada.
7. Os pré-requisitos reais de criação de VPC/EC2/IAM/ECR/Secrets Manager e VNet/VM/Managed Identity/ACR/Key Vault, incluindo WIF e roles com mínimo privilégio, precisam ser validados por cliente e por tenant antes do primeiro `apply`.

## 8. Critérios de aceite para o modelo

- Uma única definição de pipeline atende os providers AWS e Azure e reutiliza o Compose da aplicação.
- Uma nova empresa é configurada por perfil/Variable Group, Service Connection, segredo externo e parâmetros de rede/domínio — sem alterar scripts de aplicação ou copiar um pipeline inteiro.
- O fluxo cria/atualiza uma VM Linux por ambiente, obtém imagens do registry adequado e sobe exatamente os containers `app` e `api`.
- PostgreSQL permanece externo e a aplicação consegue alcançá-lo sem expor o banco publicamente.
- Produção exige aprovação; nenhuma execução de pipeline apaga recursos automaticamente.
- Imagens são rastreáveis por cliente/ambiente/commit e existe procedimento de rollback testado.
- Os arquivos de infraestrutura e pipeline são validados estaticamente e o primeiro teste real ocorre em DEV, sem usar credenciais ou recursos de produção.

## 9. Referências oficiais consultadas

- Microsoft Learn — [Azure Resource Manager Service Connection e Workload Identity Federation](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure?view=azure-devops).
- Microsoft Learn — [Service Connections, uso em YAML e autorização por pipeline](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints?view=azure-devops).
- Microsoft Learn — [Variable Group ligado ao Azure Key Vault](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/link-variable-groups-to-key-vaults?view=azure-devops).
- Microsoft Learn — [Secrets em Azure Pipelines: proteção, acesso e logging](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/secrets?view=azure-devops) e [variáveis secretas](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/set-secret-variables?view=azure-devops).
- Microsoft Learn — [Parâmetros de template e tipos suportados](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/template-parameters?view=azure-devops) e [segurança de inputs/variáveis de execução](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/inputs?view=azure-devops).
- AWS — [AWS Toolkit for Azure DevOps: OIDC, Service Connection AWS e permissões](https://marketplace.visualstudio.com/items?itemName=AmazonWebServices.aws-vsts-tools); [federação Azure DevOps → AWS com OIDC](https://aws.amazon.com/blogs/modernizing-with-aws/how-to-federate-into-aws-from-azure-devops-using-openid-connect/).
- AWS IAM — [Instance Profiles para roles em EC2](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html) e AWS Secrets Manager — [políticas de identidade para limitar leitura de segredos](https://docs.aws.amazon.com/secretsmanager/latest/userguide/auth-and-access_iam-policies.html).
- Microsoft Learn — [Autenticação no Azure Key Vault, incluindo Managed Identity](https://learn.microsoft.com/en-us/azure/key-vault/general/authentication).
