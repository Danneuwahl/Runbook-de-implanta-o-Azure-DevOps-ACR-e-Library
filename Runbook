# Runbook de implantação Lumiere — Azure DevOps, ACR e Library

**Estado:** em validação operacional. O build das imagens passou após corrigir a referência à Service Connection. O deploy ainda não está validado de ponta a ponta: houve falha na configuração do Nginx e a emissão do certificado Certbot segue sem causa raiz confirmada. **Não considerar PRD pronto até fechar os itens pendentes da seção 10.**

**Última atualização:** 10 de novembro de 2026. Este documento acompanha a pipeline principal Lumiere, o repositório compartilhado `devops-templates`, os grupos do Azure DevOps Library e o Azure Container Registry `lumiere.azurecr.io`.

## 1. Objetivo e escopo

A pipeline constrói e publica as imagens do frontend (`app`) e da API (`api`) no ACR. Para mudanças nas branches de ambiente, prepara a VM correspondente e atualiza somente o serviço alterado. A branch `main` representa PRD. Pull requests não acionam esta pipeline (`pr: none`).

O pipeline principal está no repositório `lumiere-pricing-app`, em `.pipelines/azure-pipelines.yml`. Os templates compartilhados estão no repositório `devops-templates`, sob `templates/` — não em `templates-test/`.

```text
lumiere-pricing-app/
├── .pipelines/
│   ├── azure-pipelines.yml
│   └── docker-compose.yml
├── client/
└── server/

devops-templates/
└── templates/
    ├── jobs/
    │   ├── ci/
    │   │   ├── detect-changes.yml
    │   │   └── build-and-push.yml
    │   └── vm/
    │       ├── prepare.yml
    │       └── deploy.yml
    └── steps/
        └── docker/
            ├── login.yml
            └── build-push.yml
```

A pipeline declara o repositório `devops-templates` com o alias `templates` e referencia, por exemplo, `templates/jobs/ci/detect-changes.yml@templates`.

## 2. Arquitetura e fluxo

1. **Selecionar o grupo Library:** `dev` carrega `lumiere-dev`, `hml` carrega `lumiere-hml` e `main` carrega `lumiere-prd`.
2. **Detectar alterações:** `detect-changes.yml` verifica `client` e `server` e publica as saídas `detect_app.changed` e `detect_api.changed`. Parâmetros manuais de força podem forçar detecção/build de cada componente.
3. **Build e push:** para cada componente alterado, a pipeline usa a Service Connection Docker e publica as tags `IMAGE_TAG` e `Build.BuildId` no ACR. O frontend recebe `VITE_API_URL` como build argument.
4. **Preparar VM:** `prepare.yml` sincroniza `.pipelines` para `deployPath` e gera o `.env` com as variáveis indicadas pelo consumidor. A pipeline usa `rsyncDelete: false` para não remover arquivos que só existam no host.
5. **Deploy seletivo:** `deploy.yml` valida a imagem, faz `docker compose pull` e `docker compose up` somente para `app` ou `api`, configura um vhost Nginx, tenta solicitar HTTPS com Certbot e executa o healthcheck.

O stage de CI usa `ubuntu-latest`. Os stages de deploy usam `pool.name: $(agentPool)`, resolvido pelo grupo Library da branch. **`agentPool` deve ser o nome literal do pool no Azure DevOps, não o nome do agente listado dentro dele.** Um pool pode conter agentes; um agente self-hosted se registra em um pool por vez. [Pools de agentes Azure Pipelines][2]

## 3. Branches, grupos, pools e agentes

| Ambiente | Branch | Grupo Library | Pool pretendido | Agente pretendido | Estado conhecido |
|---|---|---|---|---|---|
| DEV | `dev` | `lumiere-dev` | `Lumiere-DEV` (proposto; confirmar/criar no Azure DevOps) | `lumiere-pricing-vm-dev` | A captura anterior mostrava pool `Lumiere` e agente `lumiere-pricing-vm-dev` Online. Não confundir os nomes. |
| HML | `hml` | `lumiere-hml` | `Lumiere-HML` (proposto; confirmar/criar) | `lumiere-pricing-vm-hml` | A existência do pool/agente dedicado não foi confirmada. |
| PRD | `main` | `lumiere-prd` | `Lumiere-PRD` (proposto; confirmar/criar) | `lumiere-pricing-vm-prd` | A existência do pool/agente dedicado não foi confirmada. |

Os nomes de pools acima são uma proposta operacional, não valores confirmados no portal. Antes de configurar Library, crie os pools desejados e registre o agente de cada ambiente no pool correspondente. Se apenas o pool atual `Lumiere` existir, use `agentPool=Lumiere` temporariamente somente para o ambiente cuja VM/agente realmente está nesse pool. Não direcione HML ou PRD à VM DEV para contornar o erro.

Para três ambientes isolados, use hosts separados e normalmente um agente por host. Se os três agentes forem instalados na mesma VM, os nomes diferentes não fornecem isolamento de máquina, rede ou sistema operacional. Pools distintos controlam o agendamento no Azure DevOps, mas não substituem a separação dos hosts.

## 4. Matriz de variáveis do Library

Mantenha os três grupos: `lumiere-dev`, `lumiere-hml` e `lumiere-prd`. Cada execução carrega o grupo referente à branch; valores de um grupo não são automaticamente herdados pelos outros. Repita valores comuns nos grupos ou configure valores específicos por ambiente.

| Variável | DEV (`lumiere-dev`) | HML (`lumiere-hml`) | PRD (`lumiere-prd`) | Regras e validação |
|---|---|---|---|---|
| `agentPool` | Pool real do DEV (proposto `Lumiere-DEV`) | Pool real do HML (proposto `Lumiere-HML`) | Pool real do PRD (proposto `Lumiere-PRD`) | Nome do pool, nunca o nome do agente. Confirmar literal em Agent pools e autorizar pipeline. |
| `containerRegistryServiceConnection` | `sc-lumiere-acr` se esse for o nome salvo | Mesmo nome, se endpoint compartilhado | Mesmo nome, se endpoint compartilhado | Nome exato do endpoint Azure DevOps compatível com Docker@2. Não usar `lumiere.azurecr.io`. |
| `registryLoginServer` | `lumiere.azurecr.io` | `lumiere.azurecr.io` | `lumiere.azurecr.io` | Login server real, sem protocolo e sem barra final. |
| `repositoryApp` | `lumiere-pricing/app` | `lumiere-pricing/app` | `lumiere-pricing/app` | Nome do repositório de imagem, sem hostname e sem tag. |
| `repositoryApi` | `lumiere-pricing/api` | `lumiere-pricing/api` | `lumiere-pricing/api` | Nome do repositório de imagem, sem hostname e sem tag. |
| `IMAGE_TAG` | Convenção DEV (exemplo anterior: `dev`) | Convenção HML | Convenção PRD | Deve ser a mesma tag selecionada pelo Compose/deploy. Confirmar estratégia de imutabilidade e promoção. |
| `deployPath` | Exemplo visto: `/opt/apps/lumiere-pricing` | Confirmar caminho no host HML | Confirmar caminho no host PRD | Caminho absoluto local ao agente/VM; dedicado ao Lumiere e gravável pelo usuário de deploy. |
| `COMPOSE_PROJECT_NAME` | Nome estável DEV | Nome estável HML | Nome estável PRD | Único por instalação para evitar colisão com outros projetos Compose. |
| `ASPNETCORE_ENVIRONMENT` | Valor aceito pela API | Valor aceito pela API | Valor aceito pela API | Definir de acordo com a aplicação e ambiente. |
| `APP_PORT` | Porta host livre DEV | Porta host livre HML | Porta host livre PRD | Numérica, usada pelo Nginx como upstream; publicação em loopback `127.0.0.1`. |
| `API_PORT` | Porta host livre DEV | Porta host livre HML | Porta host livre PRD | Numérica, distinta de `APP_PORT` no mesmo host; upstream Nginx. |
| `APP_DOMAIN` | FQDN público DEV | FQDN público HML | FQDN público PRD | Hostname apenas: sem `https://`, porta ou path; deve resolver para o ponto de entrada dessa VM. |
| `API_DOMAIN` | FQDN público API DEV | FQDN público API HML | FQDN público API PRD | Mesma regra de hostname; DNS e portas de validação precisam alcançar o Nginx. |
| `VITE_API_URL` | URL pública da API DEV | URL pública da API HML | URL pública da API PRD | Build argument do frontend; URL completa conforme configuração da aplicação. |
| `CorsOrigins` | Origem frontend DEV | Origem(s) HML | Origem(s) PRD | Usar origens, não paths. Confirmar formato que a API espera. Um valor anterior mostrado foi `https://dev.lumierepricing.com.br`, sujeito a confirmação. |
| `ApplyMigrations` | `true`/`false` conforme política | `true`/`false` conforme política | Decisão explícita para PRD | Confirmar efeito na API e política operacional de migrations. |
| `ConnectionStrings__Default` | Connection string DEV | Connection string HML | Connection string PRD | **Secreto** no Library. Banco e credenciais apropriados a cada ambiente; nunca imprimir. |
| `Security__JwtSecret` | Chave JWT DEV | Chave JWT HML | Chave JWT PRD | **Secreto** no Library; segredo independente e forte por ambiente. |
| `certbotEmail` | E-mail operacional | E-mail operacional | E-mail operacional | Obrigatório se `configureHttps=true`. Não costuma ser secreto. |

### Parâmetros de template que não precisam ir ao Library

Os parâmetros com defaults podem permanecer no template, salvo necessidade de sobrescrevê-los: `rsyncDelete` (a pipeline atual passa `false`), `rsyncExcludes`, `composeAdditionalArgs` (default `--no-deps`), `configureNginx`/`configureHttps` (default `true`), `locationPath` (default `/`), `stripLocationPrefix` (default `false`), `healthcheckRetries` (24), `healthcheckDelaySeconds` (5) e `warmupSeconds` (15). O nome do usuário de deploy deve corresponder ao sistema: o `prepare.yml` enviado tem default `azureuser`; confirme se o agente roda como esse usuário e se `chown` usa grupo válido.

## 5. Recursos e permissões necessários

### 5.1 ACR e Service Connection

O ACR confirmado tem login server `lumiere.azurecr.io`. A Service Connection Azure DevOps tem nome distinto, atualmente indicado como `sc-lumiere-acr`. Na variável Library, use:

```text
containerRegistryServiceConnection = sc-lumiere-acr
registryLoginServer = lumiere.azurecr.io
```

A identidade da Service Connection precisa de permissão de push para CI e pull para deploy. Foi criado um service principal localmente pelo Azure CLI, com arquivo de credenciais protegido; não registrar client secret neste runbook nem em YAML. O pipeline de build foi observado concluindo três jobs após a correção do nome da Service Connection, o que indica que o bloqueio de referência anterior foi removido. A atribuição de papel deve ser confirmada no ACR e a Service Connection precisa continuar autorizada para a pipeline. Para RBAC tradicional do ACR, `AcrPush` permite push e pull; confirmar o modo RBAC/ABAC do registry antes de PRD. [Publicar no ACR com Azure Pipelines][4]

### 5.2 Library, templates e pools

- Autorize a pipeline em **Library > Pipeline permissions** para cada grupo de variáveis, em especial quando há segredos.
- Autorize a pipeline a consumir o repositório `devops-templates`; a referência atual aponta para `refs/heads/main`. Considere pin de versão protegida após homologar templates.
- Autorize a pipeline a usar cada agent pool em **Pool > Security/Pipeline permissions**. O agente estar `Online` não autoriza por si só o uso do pool pela pipeline. [Permissões de pools][3]
- A conta de registro do agente precisa de permissão de administração no pool no momento do registro; a identidade de execução precisa ter conectividade com ACR e acesso às operações locais de deploy.

### 5.3 VM de deploy

Os logs recentes identificam o host/agent Linux como `azureuser@LumiereDevHml`, com agent directory `/opt/agent`. O host que executa o job precisa ter Docker Engine, Docker Compose v2, `rsync`, `curl`, Nginx e Certbot com plugin Nginx. O agente precisa ter rede de saída para Azure DevOps/ACR e o Nginx deve receber conexões públicas conforme DNS e firewall.

Os templates usam `sudo` para criar diretório, ajustar ownership/permissões, gravar configuração Nginx, testar/recarregar Nginx e executar Certbot. Configure sudo para a conta do agente de forma controlada, sem prompts interativos durante o job. Não conceda sudo irrestrito sem avaliação de segurança. Confirme que os serviços app e api e o Certbot são executados no host correto.

Para Ubuntu/Debian, o procedimento base de instalação usado no runbook é:

```bash
sudo apt-get update
sudo apt-get install -y nginx snapd
sudo systemctl enable --now nginx
sudo snap install --classic certbot
sudo ln -sf /snap/bin/certbot /usr/local/bin/certbot
```

Valide os componentes antes de rodar a pipeline:

```bash
sudo nginx -t
sudo certbot --version
sudo certbot plugins
sudo systemctl status nginx --no-pager
```

Confirme que `certbot plugins` lista o plugin `nginx`. As instruções oficiais do Certbot para Nginx recomendam instalação Snap, comando `certbot --nginx` e teste de renovação `certbot renew --dry-run`. [Instruções do Certbot para Nginx][5]

## 6. Contrato do Docker Compose e roteamento

`prepare.yml` recebe `sourcePath: $(Build.SourcesDirectory)/.pipelines` e sincroniza o conteúdo para `deployPath`. Portanto, a pipeline usa `.pipelines/docker-compose.yml`, copiado como `$(deployPath)/docker-compose.yml`; não assume o Compose da raiz do repositório.

O Compose Lumiere esperado tem os serviços `app` e `api`, imagens formadas pelo ACR, pelos repositórios configurados no Library e por `IMAGE_TAG`, e bind local das portas. Modelo:

```yaml
services:
  app:
    image: ${registryLoginServer}/${repositoryApp}:${IMAGE_TAG}
    ports:
      - "127.0.0.1:${APP_PORT}:80"

  api:
    image: ${registryLoginServer}/${repositoryApi}:${IMAGE_TAG}
    ports:
      - "127.0.0.1:${API_PORT}:80"
    environment:
      ASPNETCORE_ENVIRONMENT: ${ASPNETCORE_ENVIRONMENT}
      ConnectionStrings__Default: ${ConnectionStrings__Default}
      CorsOrigins: ${CorsOrigins}
      ApplyMigrations: ${ApplyMigrations}
      Security__JwtSecret: ${Security__JwtSecret}
```

A porta `80` à direita é a porta interna do container. `APP_PORT` e `API_PORT` são portas publicadas no host e usadas pelo Nginx como upstream. Os Dockerfiles examinados anteriormente declaravam/escutavam na porta interna 80 para app e API. O bind `127.0.0.1:` evita expor diretamente as portas de aplicação em todas as interfaces, deixando o acesso passar pelo Nginx.

A pipeline envia `repositoryApp` e `repositoryApi` ao `.env`; o Compose precisa usar esses nomes dinâmicos e a mesma tag que o build publicou. O `.env` deve permanecer protegido com modo `600`. `rsyncDelete: false` reduz risco de apagar arquivos presentes somente no host, inclusive dados persistentes de Certbot, mas pode deixar arquivos antigos.

O Nginx e Certbot **não são serviços Compose** no desenho atual: `deploy.yml` tenta configurá-los diretamente no host do agente/deploy. Se o Nginx estiver em outra VM, container ou gateway, desabilite a configuração Nginx/HTTPS no deploy e faça a configuração no local que realmente termina o tráfego público.

## 7. DNS, Nginx e HTTPS

Para cada `APP_DOMAIN`/`API_DOMAIN`, confirme antes do deploy:

1. Os registros DNS públicos apontam para o IP público ou balanceador correto. Se existe AAAA, ele também precisa apontar para um endpoint IPv6 válido; um AAAA obsoleto pode causar validação indevida por IPv6.
2. As portas TCP 80 e 443 são permitidas nas regras de entrada Azure NSG/firewall e no firewall do sistema operacional. O Nginx precisa escutar em 80/443 e a solicitação externa precisa chegar ao host correto.
3. O arquivo do site contém `server_name` igual ao domínio solicitado pelo Certbot e o Nginx passa `nginx -t` antes do reload.
4. `certbotEmail` está definido e o Certbot dispõe do plugin Nginx.

O template `deploy.yml` escreve o vhost em `/etc/nginx/sites-available/<site>` e cria link em `/etc/nginx/sites-enabled/<site>`. Isso pressupõe que a configuração do Nginx da VM inclua esses diretórios. **Não crie `sites-available` cegamente** numa distribuição/layout que use `/etc/nginx/conf.d`; adapte o template ao layout incluído pelo `nginx.conf` ou use a configuração adequada do sistema.

O template anexado executa Certbot com `--nginx`, `--agree-tos`, `--non-interactive`, `--redirect`, e-mail e domínio. O passo estava marcado `continueOnError: true` no YAML inspecionado. O healthcheck seguinte, porém, escolhe HTTPS quando `configureHttps=true`; se não houver certificado emitido, essa etapa pode falhar. Além disso, `curl -k` não valida a cadeia de confiança e não é prova de TLS plenamente válido.

## 8. Como executar

### Primeiro bootstrap ou mudança apenas de configuração

A detecção acompanha `client` e `server`. Uma alteração somente em `.pipelines`, `docs` ou `plans` não marca automaticamente app/API como alterados. Para primeiro bootstrap ou mudança apenas de infraestrutura/configuração, execute manualmente na branch correta e habilite `forceLumierePricingApp` e/ou `forceLumierePricingApi`. Essas flags podem forçar também build/push da imagem; não são apenas um deploy sem build.

Comece por DEV. Antes da execução, confira Library, pool real, autorização do pool/variable group/service connection, DNS e pré-requisitos da VM.

### Execução normal

1. Faça push/merge em `dev`, `hml` ou `main`.
2. Em `CI / Detect`, confirme qual saída de alteração está `true`.
3. Em `Build_app` e/ou `Build_api`, confirme build, login e push das tags esperadas.
4. Confirme que o stage da branch corresponde ao ambiente e usa o pool correto.
5. Verifique `PrepareVM`, deploy do serviço alterado, configuração Nginx, etapa HTTPS, healthcheck e status dos containers.
6. Não publique valores do `.env`, connection strings, JWT secrets, PATs ou client secrets nos logs.

## 9. Troubleshooting real aplicado nesta implantação

Esta seção distingue correções observadas de hipóteses ainda pendentes. Não trate uma correção como concluída se não há log/saída confirmando o resultado.

| Sintoma observado | Diagnóstico e ação aplicada | Estado |
|---|---|---|
| Validação: `containerRegistry references service connection lumiere.azurecr.io which could not be found` (Build_app Docker1/Docker2) | A pipeline estava usando o login server como nome da Service Connection. Separam-se os valores: `containerRegistryServiceConnection=sc-lumiere-acr` e `registryLoginServer=lumiere.azurecr.io`. | **Superado operacionalmente:** depois da correção, a captura da execução mostrou o stage de build verde com 3 jobs concluídos. Manter os valores corretos nos três grupos Library e a autorização da conexão. |
| Build verde, mas Deploy DEV/HML/PROD exibidos como `Skipped due to conditions` | Os stages dependem da branch correspondente e de detecção de mudança em `client`/`server`. Build verde não significa deploy executado. Uma alteração somente em `.pipelines` pode deixar os stages ignorados. | **Comportamento esperado**, desde que branch/outputs expliquem o skip. Para bootstrap/config-only, usar os parâmetros force apropriados e confirmar a imagem que será construída. |
| `Could not find a pool with name lumiere-pricing-vm-dev` | O valor usado era o nome do agente, não do pool. A captura do Azure DevOps mostrava pool `Lumiere` e agente `lumiere-pricing-vm-dev` Online. | **Causa confirmada; correção final pendente de confirmação.** Configurar Library com o nome literal do pool realmente criado/selecionado, autorizar a pipeline, e planejar pools/agentes HML/PRD distintos. Nomes propostos `Lumiere-DEV`, `Lumiere-HML`, `Lumiere-PRD` ainda precisam ser criados e verificados. |
| `tee: /etc/nginx/sites-available/api: No such file or directory` no step `Configurar Nginx` | O deploy chegou ao host Linux (`/opt/agent`) e o caminho presumido pelo template não existia nesse host. O template espera `sites-available`/`sites-enabled`; não se deve presumir que qualquer instalação/layout Nginx os crie. | **Pendente de confirmação.** Nginx foi instalado/estava em execução posteriormente; conferir `nginx -t`, os includes do `nginx.conf` e o layout da distribuição. Ajustar template para `conf.d` se for o layout real. |
| Step `Configurar HTTPS` terminou com `Bash exited with code '1'`; `certbot certificates` mostrou `No certificates found` | Confirma que não há certificado emitido; o trecho de log enviado não contém o erro ACME causador. A saída `ss` mostrou Nginx escutando localmente em `0.0.0.0:80` e `[::]:80`, o que **não** prova acesso público pela internet. | **Pendente; causa raiz não confirmada.** Inspecionar `/var/log/letsencrypt/letsencrypt.log`, DNS A/AAAA, reachability externa das portas 80/443, `server_name`, `nginx -t` e plugin Certbot Nginx. |
| Logs mostram `/opt/agent` e prompt `azureuser@LumiereDevHml` | Os jobs de deploy atuais são executados num host Linux self-hosted. | **Confirmado pelos logs apresentados.** Instalar/configurar Nginx, Certbot e Docker no host alvo correto; verificar que não é outro host por ambiente e que o usuário real é compatível com `deployUser`. |

### Comandos de diagnóstico para o Certbot/Nginx

Executar no host que roda o deploy e que recebe o tráfego público, não no Microsoft-hosted agent do CI:

```bash
cat /etc/os-release
sudo nginx -t
sudo nginx -T 2>&1 | grep -E 'include|server_name|listen'
sudo ls -ld /etc/nginx /etc/nginx/sites-available /etc/nginx/sites-enabled /etc/nginx/conf.d 2>&1
sudo certbot --version
sudo certbot plugins
sudo certbot certificates
sudo ss -lntp | grep -E ':(80|443)\b' || true
sudo grep -n -B 12 -A 12 -Ei \
  'error|failed|unauthorized|timeout|invalid response|connection refused' \
  /var/log/letsencrypt/letsencrypt.log | tail -n 100
```

Validar DNS para cada domínio real:

```bash
getent ahosts SEU_DOMINIO
```

Também testar de uma máquina **fora** da VM se `http://SEU_DOMINIO/` alcança o vhost correto. A porta 80 estar aberta localmente só confirma listener local; verifique NSG, firewall e roteamento público. Não remova nem recrie `/etc/letsencrypt` para tentar resolver uma falha ainda não diagnosticada.

## 10. Pendências antes de homologação final/PRD

| Item | Evidência necessária para encerrar |
|---|---|
| Pools DEV/HML/PRD | Criar ou confirmar os nomes reais dos três pools; conferir o agente correto Online em cada um; mapear `agentPool` por Library; autorizar pipeline. O nome de agente não substitui pool. |
| Service Connection e ACR | Confirmar endpoint `sc-lumiere-acr` habilitado/autorizado em todos os grupos relevantes e acesso push/pull da identidade, sem expor o client secret. Confirmar modo RBAC/ABAC e papel correspondente. |
| Layout Nginx | Mostrar que `nginx -t` passa e que o include do Nginx carrega o local onde `deploy.yml` grava sites. Se usa `conf.d`, adaptar gravação e remover symlink para `sites-enabled`, mantendo o restante da lógica. |
| Certbot | Obter mensagem específica do log, validar DNS A/AAAA e acesso externo na porta 80, emitir certificado para o domínio e testar `certbot renew --dry-run`. Confirmar status TLS externo com certificado confiável. |
| Comportamento em erro HTTPS | Decidir se `continueOnError: true` é aceitável. Em PROD, recomendar falhar o deploy se a emissão/validação TLS for requisito. Rever healthcheck com `curl -k`, que não valida cadeia de confiança. |
| Healthchecks | A pipeline passa `/health` para app e API. Confirmar que ambos os endpoints existem no código implantado e retornam um código aceito. A inclusão de `/health` na API foi preparada como ajuste, mas o commit/push desse código ao repositório não foi confirmado nesta conversa. |
| Compose efetivamente implantado | Confirmar que o `.pipelines/docker-compose.yml` no repositório usa `${repositoryApp}`/`${repositoryApi}`, `registryLoginServer`, `IMAGE_TAG` e bind `127.0.0.1` e que está sendo sincronizado para `deployPath`. |
| Segredos e `.env` | Confirmar `chmod 600` depois da gravação do `.env`, valores sensíveis marcados como secret no Library e ausência de segredos em logs. |
| `deployUser` e sudo | Confirmar usuário de execução/ownership, comandos `sudo` sem prompt interativo e que a configuração sudo foi limitada segundo política de segurança. |
| Aplicação de migrations | Aprovar comportamento de `ApplyMigrations` por ambiente e plano de recuperação de dados antes de PRD. |

## 11. Rollback e recuperação

O build publica também a tag imutável `Build.BuildId`, enquanto a pipeline principal usa `IMAGE_TAG` para o deploy. O YAML analisado não implementa um fluxo de rollback dedicado. Para rollback, identificar o Build ID saudável e a imagem correspondente no ACR; executar procedimento controlado que selecione explicitamente a tag anterior e preserve ambiente, serviço, Compose e variáveis. Registrar aprovação e resultado. Reverter imagem não reverte migrations ou alterações persistentes de dados.

## Referências

[1] Grupos Library e autorização de variable groups: [Azure Pipelines variable groups](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/variable-groups?view=azure-devops).

[2] Pools de agentes, registro do agente em um único pool e seleção YAML: [Create and manage agent pools](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/pools-queues?view=azure-devops).

[3] Permissões e autorização de pipelines para agent pools: [Azure Pipelines permissions](https://learn.microsoft.com/en-us/azure/devops/pipelines/policies/permissions?view=azure-devops).

[4] Service Connection e publicação de imagens no ACR: [Publish to Azure Container Registry](https://learn.microsoft.com/en-us/azure/devops/pipelines/ecosystems/containers/publish-to-acr?view=azure-devops).

[5] Instalação/execução de Certbot com Nginx e teste de renovação: [Certbot instructions for Nginx on Ubuntu](https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal).

[6] Uso das tasks Docker e referência de `Docker@2`: [Docker@2 task reference](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/docker-v2?view=azure-pipelines).
