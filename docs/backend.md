# Guia completo: novo desenvolvedor backend – onboarding e performance

**Objetivo:** Ajudar um novo desenvolvedor a entrar no projeto, entender a arquitetura do backend e contribuir para a **performance** e a **manutenção** das APIs, Lambdas e bancos de dados do ecossistema Budd.

**Público:** Desenvolvedores que vão trabalhar em Lambda, DynamoDB, API Gateway, Amplify (budduser, buddadmin, buddattendant) ou payment-backend.

---

## Índice

1. [Primeiros passos](#1-primeiros-passos)
2. [Mapa do backend](#2-mapa-do-backend)
3. [Convenções e regras obrigatórias](#3-convenções-e-regras-obrigatórias)
4. [Performance: onde atuar](#4-performance-onde-atuar)
5. [DynamoDB e performance](#5-dynamodb-e-performance)
6. [Lambdas e performance](#6-lambdas-e-performance)
7. [Streams, triggers e consistência](#7-streams-triggers-e-consistência)
8. [Monitoramento e métricas](#8-monitoramento-e-métricas)
9. [Checklist antes de mudar o backend](#9-checklist-antes-de-mudar-o-backend)
10. [Referências rápidas](#10-referências-rápidas)

---

## 1. Primeiros passos

### 1.1 O que ler primeiro

| Ordem | Documento | Conteúdo |
|-------|-----------|----------|
| 1 | **docs/INFRAESTRUTURA_BACKEND_BUDD.md** | Visão geral: projetos, tabelas DynamoDB, APIs, Lambdas, scripts, secrets. |
| 2 | **docs/ONBOARDING_DESENVOLVEDOR_BACKEND_BUDD.md** | Acesso AWS (IAM, MFA, CLI), deploy, convenções de não duplicar. |
| 3 | **payment-backend/CONVENCOES.md** | Regra: consultar inventário antes de criar qualquer recurso; não duplicar. |
| 4 | **.cursor/rules/verificar-antes-de-duplicar.mdc** | Regra do projeto: verificar inventário antes de executar mudanças. |

### 1.2 Configuração local

- **Região AWS:** `us-east-1`.
- **CLI:** `aws configure` (ou IAM Identity Center, conforme orientação do time).
- **Verificar acesso:**  
  `aws sts get-caller-identity`  
  `aws dynamodb list-tables --region us-east-1`

### 1.3 Deploy (não crie novos scripts de deploy)

- **payment-backend:** cada Lambda tem seu `deploy.sh` (ex.: `processEfiBankPayment/deploy.sh prod`). Ver `payment-backend/scripts/README.md`.
- **Amplify (budduser, buddadmin, buddattendant):** usar o fluxo existente (`amplify push` na pasta do app). Não criar um segundo deploy para o mesmo serviço.

---

## 2. Mapa do backend

### 2.1 Onde está cada coisa

| Área | Pasta / recurso | Responsabilidade |
|------|------------------|------------------|
| **Pagamentos PIX (Efí/Woovi)** | `payment-backend/` | processEfiBankPayment, efiPixWebhook, solicitarSaque, getBarWooviBalance, listarHistoricoSaques. Deploy por pasta (`deploy.sh`). |
| **App consumidor** | `budduser/kankOpenblack/amplify/backend/` | API GraphQL (budduserparty), Lambdas (ingressoPayment, GenerateQrCode, GetQrCodesLambda, BuddIA, etc.). Deploy: `amplify push` em kankOpenblack. |
| **App gestor** | `buddadmin/kankAtendente/amplify/backend/` | API GraphQL (budd), Lambdas (addProduct, addProductToBar, syncBarData, ValidateQrCode, mesaHandler, buddChatOrchestrator, etc.). Deploy: `amplify push` em kankAtendente. |
| **App atendente** | `buddattendant/budd_atendente/amplify/backend/` | Validação QR, vínculo atendente–bar, etc. Compartilha ou referencia tabelas do gestor. |
| **Dashboard** | `dashbudd/` | Next.js; chama APIs (saldo Woovi, saque, etc.) e acessa DynamoDB via `lib/aws-config.ts`. |

### 2.2 Tabelas DynamoDB que impactam performance

- **ProcessedQrCode-dev:** stream habilitado → dispara processEfiBankPayment (e/ou processPaymentOnQrValidation). Evite Scan em produção; use Query por índice quando existir.
- **Order-…-prod:** stream → GenerateQrCode. GSI `barId-index`.
- **Product-…-prod:** stream → addProductToBar. Hoje **sem GSI** por `establishmentId`; consultas por estabelecimento usam Scan (ver seção 5).
- **BarOwner-…-prod:** stream → syncBarData. GSI `cognitoId-index`.
- **PayoutRequest-dev:** usada por ValidateQrCode e processEfiBankPayment. GSI `qrCodeId-index`; opcional `idEnvio-index`.

Inventário completo: **docs/INFRAESTRUTURA_BACKEND_BUDD.md** e **payment-backend/CHECAGEM_COMPLETA.md**.

---

## 3. Convenções e regras obrigatórias

### 3.1 Antes de criar qualquer recurso

1. **Consultar o inventário**  
   - Lambdas: nomes em INFRAESTRUTURA_BACKEND_BUDD.md e CHECAGEM_COMPLETA.md.  
   - API Gateway: IDs e rotas já documentados.  
   - Tabelas DynamoDB: listadas nos mesmos docs.  
   - Scripts: `payment-backend/scripts/` e `scripts/README.md`.

2. **Não duplicar**  
   - Não criar uma nova Lambda com a mesma função de uma existente.  
   - Não criar uma nova rota de API para o mesmo propósito.  
   - Não criar uma nova tabela com o mesmo uso.  
   Preferir **atualizar** o recurso existente.

3. **Deploy**  
   - Usar o script de deploy **existente** do serviço (ex.: `processEfiBankPayment/deploy.sh`, `efiPixWebhook/deploy.sh`).  
   - Não criar um segundo script de deploy para o mesmo serviço.

### 3.2 Onde verificar

- **payment-backend:** `CHECAGEM_COMPLETA.md` (inventário) e `CONVENCOES.md`.
- **buddadmin:** `DOCUMENTACAO_COMPLETA_BACKEND.md`, análise de Lambdas e tabelas.
- **budduser:** documentação de fluxos (QR, Order, ingressoPayment) na pasta `budduser/` (arquivos `COMO_*.md`, `TROUBLESHOOTING_*.md`).

---

## 4. Performance: onde atuar

### 4.1 Pontos que mais impactam performance

| Área | Problema comum | Ação recomendada |
|------|-----------------|-------------------|
| **DynamoDB** | Uso de **Scan** para buscar por atributo que não é chave. | Criar **GSI** com esse atributo como partition key (ou partition + sort) e usar **Query**. |
| **DynamoDB** | Itens muito grandes (muitos atributos ou listas grandes). | Reduzir tamanho dos itens; mover metadados pesados para outra tabela ou para S3. |
| **Lambda** | **Rate limiting em memória** (Map). Em serverless, o estado não persiste entre invocações. | Migrar para DynamoDB (tabela de rate limit com TTL) ou ElastiCache/Redis. |
| **Lambda** | **Timeout** ou **memória** insuficiente em picos. | Ajustar timeout e memory no CloudFormation/template da função; considerar reserved concurrency se necessário. |
| **Lambda** | **Cold start** em funções com muitas dependências. | Reduzir tamanho do bundle; usar Lambda Layers para dependências comuns; considerar Provisioned Concurrency apenas se métricas justificarem. |
| **Streams** | **Event Source Mapping** desabilitado ou com erro (ex.: role não assumida). | Verificar IAM (trust policy e políticas de Stream); reabilitar ou recriar o mapping. |
| **APIs** | Chamadas síncronas em cadeia (uma Lambda chama outra e espera). | Preferir fluxos assíncronos (Stream → Lambda) ou enfileiramento (SQS). |

### 4.2 Anti-padrões já documentados no projeto

- **addProduct:** verificação de produto duplicado por **Scan** (establishmentId + nome). Ver ANALISE_LAMBDAS_PRODUCT.md e ANALISE_ADD_PRODUCT_TO_BAR.md.  
  **Melhoria:** GSI `establishmentId-nome-index` e usar Query.

- **addProductToBar** e **addProduct:** **rate limit em memória**. Ver ANALISE_LAMBDAS_PRODUCT.md.  
  **Melhoria:** rate limit em DynamoDB (ou Redis) com TTL.

- **syncBarData** e **addProductToBar:** Event Source Mapping às vezes fica **Disabled** por erro de role. Ver STATUS_SYNC_BAR_DATA.md, DIAGNOSTICO_ADD_PRODUCT_TO_BAR.md.  
  **Melhoria:** garantir políticas de Stream na role e reabilitar mapping após deploy.

- **Validações inconsistentes** entre Lambdas (ex.: limite de preço diferente em addProduct e addProductToBar).  
  **Melhoria:** módulo compartilhado de validação ou constantes centralizadas.

---

## 5. DynamoDB e performance

### 5.1 Regras de ouro

1. **Preferir Query em vez de Scan**  
   - Query usa partition key (e opcionalmente sort key e GSI).  
   - Scan percorre a tabela inteira; custo e latência crescem com o tamanho da tabela.

2. **Criar GSI quando precisar buscar por outro atributo**  
   - Ex.: buscar produtos por `establishmentId` → GSI com `establishmentId` como partition key.  
   - Ex.: buscar PayoutRequest por `idEnvio` → GSI `idEnvio-index` (já documentado como opcional em payment-backend).

3. **Evitar itens muito grandes**  
   - DynamoDB tem limite de 400 KB por item.  
   - Listas ou mapas muito grandes aumentam custo de leitura/escrita e tempo de resposta.

4. **Usar nomes de tabela via variável de ambiente**  
   - Facilita ambientes (dev/prod) e evita hardcode de sufixos Amplify.

### 5.2 Onde estão os schemas e índices

- **payment-backend:** scripts `create-payout-request-table.sh`, `create-withdrawal-request-table.sh`; CHECAGEM_COMPLETA.md.
- **buddadmin:** DOCUMENTACAO_COMPLETA_BACKEND.md, DESCRICAO_TABELA_PRODUCT.md.
- **docs/INFRAESTRUTURA_BACKEND_BUDD.md:** lista de tabelas e GSIs conhecidos.

### 5.3 Comandos úteis

```bash
# Ver estrutura e índices de uma tabela
aws dynamodb describe-table --table-name ProcessedQrCode-dev --region us-east-1 --query 'Table.{TableName:TableName,KeySchema:KeySchema,GSI:GlobalSecondaryIndexes[*].IndexName}'

# Ver se o stream está habilitado (importante para Lambdas acionadas por stream)
aws dynamodb describe-table --table-name ProcessedQrCode-dev --region us-east-1 --query 'Table.StreamSpecification'
```

---

## 6. Lambdas e performance

### 6.1 Configuração (memory e timeout)

- Ajustes típicos: **memory** (ex.: 256 MB → 512 MB reduz cold start e tempo de execução em funções com bastante lógica); **timeout** (evitar valor excessivo; 25–60 s costuma ser suficiente para a maioria das funções).
- Documentação do budduser (DOCUMENTACAO_COMPLETA_DEPLOY) traz sugestões por Lambda (ex.: ingressoPayment 1024 MB, 30 s). Use como referência; mantenha consistência entre ambientes.

### 6.2 Cold start

- Reduzir tamanho do pacote (excluir devDependencies, usar apenas dependências necessárias).  
- Dependências comuns: considerar **Lambda Layers**.  
- **Provisioned Concurrency** só se houver SLA de latência e orçamento; monitorar custo.

### 6.3 Rate limiting

- **Problema:** várias Lambdas usam `Map` em memória para rate limit. Em ambiente serverless isso não é confiável (estado não persiste entre invocações).  
- **Solução:** tabela DynamoDB com chave (ex.: `userId` ou `barId`) + contador + TTL para janela de tempo; ou ElastiCache/Redis se o projeto já usar.

### 6.4 Idempotência e retry

- Lambdas acionadas por **DynamoDB Stream** podem receber o mesmo evento mais de uma vez.  
- Implementar **idempotência** (ex.: checar `paymentStatus` antes de processar pagamento; usar `X-Idempotency-Key` em chamadas a APIs externas).  
- Documentação: payment-backend e buddadmin (RATE_LIMIT_E_IDEMPOTENCIA.md, DETALHES_TECNICOS_PAGAMENTO_AUTOMATICO.md).

### 6.5 Logs e erros

- Usar **logs estruturados** (ex.: JSON com `requestId`, `action`, `durationMs`).  
- **Não logar** dados sensíveis (chave PIX, CPF, token) em texto claro; mascarar em produção.  
- Em caso de falha, registrar contexto suficiente para debug (IDs, status, código de erro).

---

## 7. Streams, triggers e consistência

### 7.1 Fluxos importantes

- **Order (INSERT)** → DynamoDB Stream → **GenerateQrCode** → escreve em QrCode-prod.  
- **Product (INSERT)** → DynamoDB Stream → **addProductToBar** → atualiza Bar.produtos.  
- **BarOwner (INSERT/MODIFY)** → DynamoDB Stream → **syncBarData** → atualiza Bar.  
- **ProcessedQrCode-dev (INSERT)** → DynamoDB Stream → **processEfiBankPayment** (e/ou processPaymentOnQrValidation).

### 7.2 O que verificar se “não está acontecendo nada”

1. **Stream habilitado** na tabela de origem:  
   `aws dynamodb describe-table --table-name <TABELA> --query 'Table.StreamSpecification'`

2. **Event Source Mapping** existe e está **Enabled**:  
   `aws lambda list-event-source-mappings --function-name <NOME_DA_LAMBDA>`

3. **LastProcessingResult:** se estiver com mensagem de erro (ex.: "Lambda failed to assume your function execution role"), corrigir **IAM** (trust policy da role e políticas de acesso ao Stream).

4. **Permissões da Lambda** para o Stream:  
   `dynamodb:DescribeStream`, `GetRecords`, `GetShardIterator`, `ListStreams` no ARN do stream.

Documentação de troubleshooting: budduser (COMO_QRCODES_SAO_INSERIDOS.md, TROUBLESHOOTING_COMPRA_FALHA.md, ANALISE_PROBLEMA_GENERATEQRCODE.md); buddadmin (PROBLEMA_SYNC_BAR_DATA.md, CORRECAO_SYNC_BAR_DATA.md, DIAGNOSTICO_ADD_PRODUCT_TO_BAR.md).

---

## 8. Monitoramento e métricas

### 8.1 CloudWatch Logs

- **Grupos:** `/aws/lambda/<NomeDaFunção>-prod` (ou `-dev`).  
- **Filtrar:** por `requestId`, `ERROR`, `paymentStatus`, etc.  
- **Comando:**  
  `aws logs tail /aws/lambda/processEfiBankPayment-prod --follow --region us-east-1`

### 8.2 Métricas úteis (CloudWatch)

- **Invocations**, **Duration**, **Errors**, **Throttles** por Lambda.  
- **ConsumedReadCapacityUnits** / **ConsumedWriteCapacityUnits** por tabela DynamoDB (se usar provisioned capacity).  
- **UserErrors** e **SystemErrors** em DynamoDB.

### 8.3 O que observar para performance

- **Duration** alto: otimizar código, aumentar memory ou revisar chamadas externas (API, DynamoDB).  
- **Throttles** ou **ConcurrentExecutions** no limite: revisar reserved concurrency e carga.  
- **Errors** em alta: logs, Dead Letter Queue (se configurada), e idempotência/retry.

### 8.4 Documentação de deploy e verificação

- **budduser:** DOCUMENTACAO_COMPLETA_DEPLOY.md (comandos de verificação, checklist).  
- **payment-backend:** scripts em `scripts/` (ex.: verify-setup.sh); CHECAGEM_COMPLETA.md.

---

## 9. Checklist antes de mudar o backend

Use este checklist antes de criar recurso, alterar Lambda ou fazer deploy:

- [ ] Li **docs/INFRAESTRUTURA_BACKEND_BUDD.md** e sei onde está a Lambda/tabela/API que vou alterar.
- [ ] Consultei **payment-backend/CHECAGEM_COMPLETA.md** (se for payment-backend) e **CONVENCOES.md**.
- [ ] **Não estou duplicando** uma Lambda, rota, tabela ou script que já existe.
- [ ] Se for adicionar consulta por novo atributo no DynamoDB: avaliei **Query com GSI** em vez de Scan.
- [ ] Se for alterar Lambda acionada por stream: confirmei **Event Source Mapping** e **permissões de Stream** na role.
- [ ] Se for rate limit: considerei **DynamoDB ou Redis** em vez de Map em memória.
- [ ] Mantive **idempotência** e **retry** em mente (principalmente em funções de pagamento e stream).
- [ ] Usei o **script de deploy existente** do serviço (não criei um novo).
- [ ] Após o deploy, **verifiquei** logs e métricas (CloudWatch) e, se aplicável, Event Source Mapping.

---

## 10. Referências rápidas

### 10.1 Documentação por projeto

| Projeto | Documentos principais |
|--------|------------------------|
| **payment-backend** | CHECAGEM_COMPLETA.md, CONVENCOES.md, scripts/README.md, LEIA-ME-PRIMEIRO.md |
| **buddadmin** | DOCUMENTACAO_COMPLETA_BACKEND.md, ANALISE_LAMBDAS_PRODUCT.md, ANALISE_ADD_PRODUCT_TO_BAR.md, PROBLEMA_SYNC_BAR_DATA.md, ARQUITETURA_BACKEND_PAGAMENTO.md, DETALHES_TECNICOS_PAGAMENTO_AUTOMATICO.md |
| **budduser** | COMO_QRCODES_SAO_INSERIDOS.md, TROUBLESHOOTING_COMPRA_FALHA.md, ANALISE_PROBLEMA_GENERATEQRCODE.md, DOCUMENTACAO_COMPLETA_DEPLOY.md |
| **buddattendant** | DOCUMENTACAO_COMPLETA_CADASTRO_ATENDENTE.md (em buddadmin), docs de tabelas e PDV |
| **Geral** | docs/INFRAESTRUTURA_BACKEND_BUDD.md, docs/ONBOARDING_DESENVOLVEDOR_BACKEND_BUDD.md |

### 10.2 Comandos AWS úteis

```bash
# Identidade e região
aws sts get-caller-identity
aws configure get region

# DynamoDB: listar tabelas, descrever tabela, stream
aws dynamodb list-tables --region us-east-1
aws dynamodb describe-table --table-name ProcessedQrCode-dev --region us-east-1 --query 'Table.{Stream:StreamSpecification,GSI:GlobalSecondaryIndexes[*].IndexName}'

# Lambda: listar funções, event source mappings
aws lambda list-functions --region us-east-1 --query 'Functions[*].FunctionName'
aws lambda list-event-source-mappings --function-name processEfiBankPayment-prod --region us-east-1

# Logs
aws logs tail /aws/lambda/processEfiBankPayment-prod --follow --region us-east-1
```

### 10.3 Região e conta

- **Região:** `us-east-1`.  
- **Conta:** ver ARNs nos documentos (ex.: 459032456812).  
- **Secrets:** efi-bank-credentials-prod, woovi-credentials-prod (ler via IAM; não compartilhar valor por canal inseguro).

---

**Última atualização:** Março 2026.  
Para dúvidas sobre acesso e permissões, use **docs/ONBOARDING_DESENVOLVEDOR_BACKEND_BUDD.md**. Para inventário detalhado de infraestrutura, use **docs/INFRAESTRUTURA_BACKEND_BUDD.md** e **payment-backend/CHECAGEM_COMPLETA.md**.