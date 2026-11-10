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

### Credenciais e segredos

- As credenciais de provisionamento AWS/Azure ficam em **Service Connections** com menor privilégio, preferindo federação/identidade gerenciada a chaves estáticas.
- Connection string PostgreSQL e `Security__JwtSecret` ficam no Secrets Manager ou Key Vault do cliente/ambiente; a VM recebe acesso por IAM Role/Managed Identity.
- Não gravar segredos no YAML, Compose versionado, Terraform state não protegido ou logs.
- O Variable Group deve armazenar configuração e identificadores, não substituir o cofre para segredo de runtime.

### Reuso sem editar o pipeline

O seletor de perfil não deve depender de uma lista de nomes de clientes codificada no YAML. A configuração deve permitir selecionar um Variable Group/perfil cadastrado no Azure DevOps e carregar suas variáveis em tempo de compilação de forma segura. Como inputs de Service Connection em Azure Pipelines podem exigir resolução em tempo de template, a seleção de conexão por provider deve ser testada com o projeto Azure DevOps real; não assumir que qualquer macro `$(SERVICE_CONNECTION)` é aceita por todas as tasks.

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

## 8. Critérios de aceite para o modelo

- Uma única definição de pipeline atende os providers AWS e Azure e reutiliza o Compose da aplicação.
- Uma nova empresa é configurada por perfil/Variable Group, Service Connection, segredo externo e parâmetros de rede/domínio — sem alterar scripts de aplicação ou copiar um pipeline inteiro.
- O fluxo cria/atualiza uma VM Linux por ambiente, obtém imagens do registry adequado e sobe exatamente os containers `app` e `api`.
- PostgreSQL permanece externo e a aplicação consegue alcançá-lo sem expor o banco publicamente.
- Produção exige aprovação; nenhuma execução de pipeline apaga recursos automaticamente.
- Imagens são rastreáveis por cliente/ambiente/commit e existe procedimento de rollback testado.
- Os arquivos de infraestrutura e pipeline são validados estaticamente e o primeiro teste real ocorre em DEV, sem usar credenciais ou recursos de produção.
