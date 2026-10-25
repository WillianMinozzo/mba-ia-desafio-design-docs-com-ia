# Tracker de Rastreabilidade

Cada linha liga um item dos documentos à sua origem. **Fonte** = `TRANSCRICAO` (localização `[hh:mm] Nome` em `TRANSCRICAO.md`) ou `CODIGO` (caminho real do repositório). Quando um item tem mais de uma origem, a coluna traz a fala principal; falas complementares aparecem no próprio documento.

## PRD — `docs/PRD.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B pediram notificação em tempo real | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Problema | Clientes fazem polling em GET /orders; integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Risco comercial | Atlas pode migrar ao concorrente se não sair até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-PER-01 | docs/PRD.md | Público-alvo | Usuários do OMS que representam o cliente cadastram via API autenticada | TRANSCRICAO | [09:32] Marcos |
| PRD-PER-02 | docs/PRD.md | Público-alvo | Administrador reprocessa falhas com auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-OBJ-01 | docs/PRD.md | Métrica | Latência < 10 s na primeira tentativa | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Métrica | 100 % das mudanças de status com evento registrado | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-03 | docs/PRD.md | Métrica | Janela de retry ≈ 15 h | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-04 | docs/PRD.md | Métrica | 3 clientes em produção até fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | E-mail ao cliente em falhas repetidas (adiado) | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Dashboard visual (projeto do frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Rate limiting de saída (observar) | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Webhooks inbound | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Ordem global não requerida | TRANSCRICAO | [09:14] Marcos |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Arquivamento de eventos entregues | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-07 | docs/PRD.md | Fora de escopo | Restrição de permissões do CRUD adiada | TRANSCRICAO | [09:37] Sofia |
| PRD-RF-01 | docs/PRD.md | Requisito Funcional | Cadastrar endpoint (cliente, URL, status) | TRANSCRICAO | [09:31] Marcos |
| PRD-RF-02 | docs/PRD.md | Requisito Funcional | Secret gerada pelo OMS e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-RF-03 | docs/PRD.md | Requisito Funcional | Listar endpoints do cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-04 | docs/PRD.md | Requisito Funcional | Editar endpoint | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-05 | docs/PRD.md | Requisito Funcional | Remover endpoint | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-06 | docs/PRD.md | Requisito Funcional | Filtro por status de interesse | TRANSCRICAO | [09:33] Marcos |
| PRD-RF-07 | docs/PRD.md | Requisito Funcional | Notificar a cada mudança de status, sem itens | TRANSCRICAO | [09:43] Diego |
| PRD-RF-08 | docs/PRD.md | Requisito Funcional | Assinar cada envio | TRANSCRICAO | [09:20] Sofia |
| PRD-RF-09 | docs/PRD.md | Requisito Funcional | Rotação de secret com 24 h de convivência | TRANSCRICAO | [09:21] Sofia |
| PRD-RF-10 | docs/PRD.md | Requisito Funcional | Reenvio automático com intervalos crescentes | TRANSCRICAO | [09:15] Diego |
| PRD-RF-11 | docs/PRD.md | Requisito Funcional | Fila separada de falhas permanentes | TRANSCRICAO | [09:18] Diego |
| PRD-RF-12 | docs/PRD.md | Requisito Funcional | Replay manual por ADMIN com auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-RF-13 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-RF-14 | docs/PRD.md | Requisito Funcional | Identificador único por evento | TRANSCRICAO | [09:25] Diego |
| PRD-RF-15 | docs/PRD.md | Requisito Funcional | Identificar o endpoint de destino no envio | TRANSCRICAO | [09:44] Sofia |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência < 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Status não bloqueia nem faz rollback por causa do cliente | TRANSCRICAO | [09:04] Bruno |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Atomicidade status ⇔ evento | TRANSCRICAO | [09:06] Diego |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | At-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | URL https obrigatória | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Payload máx. 64 KB, falha sem truncar | TRANSCRICAO | [09:24] Larissa |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Timeout 10 s | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-08 | docs/PRD.md | Requisito Não Funcional | Secret por endpoint | TRANSCRICAO | [09:21] Sofia |
| PRD-RNF-09 | docs/PRD.md | Restrição | Sem infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| PRD-RNF-10 | docs/PRD.md | Requisito Não Funcional | Entrega isolada da API | TRANSCRICAO | [09:11] Diego |
| PRD-RNF-11 | docs/PRD.md | Restrição | Ordem por pedido só com processo único | TRANSCRICAO | [09:13] Larissa |
| PRD-RNF-12 | docs/PRD.md | Restrição | Revisão de segurança ≥ 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-01 | docs/PRD.md | Dependência | Fluxo de mudança de status existente | CODIGO | src/modules/orders/order.service.ts |
| PRD-DEP-02 | docs/PRD.md | Dependência | JWT e role ADMIN existentes | CODIGO | src/middlewares/auth.middleware.ts |
| PRD-DEP-03 | docs/PRD.md | Dependência | Portal do desenvolvedor documenta integração | TRANSCRICAO | [09:40] Marcos |
| PRD-DEP-04 | docs/PRD.md | Dependência | Confirmação de prazo com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-RISK-01 | docs/PRD.md | Risco | Atraso e perda da Atlas; estimativa 3 sprints | TRANSCRICAO | [09:46] Larissa |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente não deduplica | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-03 | docs/PRD.md | Risco | Secret vazada pelo cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Rajada de notificações | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-05 | docs/PRD.md | Risco | Eventos fora de ordem | TRANSCRICAO | [09:12] Diego |
| PRD-TST-01 | docs/PRD.md | Validação | Testes de API no padrão Vitest + Supertest | CODIGO | tests/orders.test.ts |
| PRD-TST-02 | docs/PRD.md | Validação | Ambiente ponta a ponta com MySQL local | CODIGO | docker-compose.yml |

## RFC — `docs/RFC.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| RFC-META-01 | docs/RFC.md | Metadado | Larissa abre o design doc; revisores = participantes | TRANSCRICAO | [09:50] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | Transação de status já pesada (orders, history, estoque) | TRANSCRICAO | [09:04] Bruno |
| RFC-CTX-02 | docs/RFC.md | Contexto | changeStatus em prisma.$transaction | CODIGO | src/modules/orders/order.service.ts |
| RFC-PROP-01 | docs/RFC.md | Decisão | Publicação via publishWebhookEvent(tx, …) | TRANSCRICAO | [09:41] Bruno |
| RFC-PROP-02 | docs/RFC.md | Decisão | Resumo das decisões da reunião | TRANSCRICAO | [09:48] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo síncrono no service | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / Cluster | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger de banco | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Worker no processo da API | TRANSCRICAO | [09:11] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Retry indefinido / 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Secret global | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-07 | docs/RFC.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-08 | docs/RFC.md | Alternativa descartada | Renderizar payload no envio | TRANSCRICAO | [09:52] Larissa |
| RFC-Q-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída | TRANSCRICAO | [09:39] Diego |
| RFC-Q-02 | docs/RFC.md | Questão em aberto | Múltiplos workers e ordem | TRANSCRICAO | [09:13] Diego |
| RFC-Q-03 | docs/RFC.md | Questão em aberto | Endurecer autorização do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-Q-04 | docs/RFC.md | Questão em aberto | E-mail em falhas | TRANSCRICAO | [09:37] Larissa |
| RFC-Q-05 | docs/RFC.md | Questão em aberto | Arquivamento da outbox | TRANSCRICAO | [09:08] Diego |
| RFC-Q-06 | docs/RFC.md | Questão em aberto | "5 tentativas" vs. 5 intervalos (≈14h36) | TRANSCRICAO | [09:17] Diego |
| RFC-Q-07 | docs/RFC.md | Questão em aberto | Mecânica do grace period e secret em repouso | TRANSCRICAO | [09:46] Sofia |
| RFC-IMP-01 | docs/RFC.md | Impacto | Um processo novo para operar | TRANSCRICAO | [09:11] Larissa |
| RFC-IMP-02 | docs/RFC.md | Impacto | Prazo fim de novembro, 3 sprints | TRANSCRICAO | [09:47] Larissa |

## ADRs — `docs/adrs/`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL na mesma transação | TRANSCRICAO | [09:08] Larissa |
| ADR-001-CTX | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | Explicação do padrão outbox e atomicidade | TRANSCRICAO | [09:06] Diego |
| ADR-001-IDX | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Índices em status e created_at | TRANSCRICAO | [09:08] Diego |
| ADR-001-ALT | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa | Redis Streams como fila | TRANSCRICAO | [09:07] Larissa |
| ADR-002 | docs/adrs/ADR-002-worker-processo-separado-polling.md | Decisão | Worker em polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-PROC | docs/adrs/ADR-002-worker-processo-separado-polling.md | Decisão | src/worker.ts + npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-002-PRISMA | docs/adrs/ADR-002-worker-processo-separado-polling.md | Decisão | PrismaClient próprio, mesmo banco | TRANSCRICAO | [09:30] Bruno |
| ADR-002-ORD | docs/adrs/ADR-002-worker-processo-separado-polling.md | Trade-off | Ordem só por order_id e single-worker | TRANSCRICAO | [09:13] Larissa |
| ADR-002-SRV | docs/adrs/ADR-002-worker-processo-separado-polling.md | Padrão existente | Entry point HTTP de referência | CODIGO | src/server.ts |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | 5 retentativas 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-DLQ | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | DLQ em tabela webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| ADR-003-ALT1 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Alternativa | 3 tentativas | TRANSCRICAO | [09:16] Bruno |
| ADR-003-ALT2 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Alternativa | Marcar failed na própria outbox | TRANSCRICAO | [09:17] Larissa |
| ADR-003-CTX | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Contexto | Cliente já ficou 2 h fora em manutenção | TRANSCRICAO | [09:16] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, grace 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-ALG | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Algoritmo SHA-256 | TRANSCRICAO | [09:20] Sofia |
| ADR-004-CFG | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Config = url + secret + customer_id + ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-004-TLS | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Restrição | https obrigatório via Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-004-LOG | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Consequência | Redaction de secret no logger | CODIGO | src/shared/logger/index.ts |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-005-UUID | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | UUID gerado na entrada da outbox | TRANSCRICAO | [09:25] Diego |
| ADR-005-ALT | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Alternativa | Exactly-once | TRANSCRICAO | [09:25] Diego |
| ADR-005-DOC | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Consequência | Destaque no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso máximo dos padrões existentes | TRANSCRICAO | [09:30] Larissa |
| ADR-006-MOD | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Módulo src/modules/webhooks no padrão | TRANSCRICAO | [09:27] Bruno |
| ADR-006-ERR | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Erros com prefixo WEBHOOK_ | TRANSCRICAO | [09:29] Larissa |
| ADR-006-ERRCODE | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Padrão existente | AppError com statusCode/errorCode | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-ERRCLS | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Padrão existente | InvalidStatusTransitionError, InsufficientStockError | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-MW | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Padrão existente | errorMiddleware trata AppError/Zod/Prisma | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-ROLE | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso de requireRole no replay | TRANSCRICAO | [09:36] Larissa |
| ADR-006-UUID | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | IDs UUID, padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-006-SCHEMA | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Padrão existente | UUID Char(36) em todos os models | CODIGO | prisma/schema.prisma |
| ADR-006-ALT | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa | Injetar repository inteiro no OrderService | TRANSCRICAO | [09:41] Diego |
| ADR-007 | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Decisão | Payload snapshot na inserção | TRANSCRICAO | [09:52] Bruno |
| ADR-007-FILTRO | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Decisão | Filtro de eventos na inserção | TRANSCRICAO | [09:34] Bruno |
| ADR-007-PAYLOAD | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Decisão | Campos do payload, sem itens | TRANSCRICAO | [09:43] Diego |

## FDD — `docs/FDD.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-OT-01 | docs/FDD.md | Objetivo técnico | Evento na mesma transação | TRANSCRICAO | [09:40] Bruno |
| FDD-OT-02 | docs/FDD.md | Objetivo técnico | Primeira tentativa < 10 s | TRANSCRICAO | [09:09] Diego |
| FDD-OT-06 | docs/FDD.md | Restrição | Zero dependências novas | TRANSCRICAO | [09:29] Bruno |
| FDD-ESC-01 | docs/FDD.md | Exclusão | Evento na criação do pedido não emitido (só changeStatus) | TRANSCRICAO | [09:40] Bruno |
| FDD-DADOS-01 | docs/FDD.md | Modelo de dados | webhook_endpoints com url, secret, customer, ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-02 | docs/FDD.md | Modelo de dados | Status da outbox: pendente/processando/falhou/entregue | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-03 | docs/FDD.md | Modelo de dados | id da outbox = event_id (UUID) | TRANSCRICAO | [09:25] Diego |
| FDD-DADOS-04 | docs/FDD.md | Modelo de dados | webhook_dead_letter com payload, motivo, timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-05 | docs/FDD.md | Modelo de dados | webhook_deliveries para histórico | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-06 | docs/FDD.md | Modelo de dados | Convenções de model (Char(36), @@map) | CODIGO | prisma/schema.prisma |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | customerId no body, não no JWT | TRANSCRICAO | [09:32] Larissa |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | publishWebhookEvent dentro da transação, rollback em falha | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Worker lê pendentes mais antigos em batch | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Processamento sequencial por created_at | TRANSCRICAO | [09:12] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Retry 1m/5m/30m/2h/12h e DLQ | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Replay recoloca na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-07 | docs/FDD.md | Fluxo | Log de auditoria do replay | TRANSCRICAO | [09:36] Sofia |
| FDD-FLUXO-08 | docs/FDD.md | Fluxo | Rotação com secret anterior por 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/webhooks?customerId | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Request outbound: payload JSON | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | Prefixo /api/v1 e envelope de erro | CODIGO | src/app.ts |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (64 KB, erra) | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT (10 s) | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_MAX_ATTEMPTS_EXCEEDED → DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-05 | docs/FDD.md | Erro | 403 no replay sem ADMIN | TRANSCRICAO | [09:36] Sofia |
| FDD-ERR-06 | docs/FDD.md | Erro | Falha de outbox → rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Isolamento por processo | TRANSCRICAO | [09:11] Diego |
| FDD-RES-03 | docs/FDD.md | Resiliência | Shutdown gracioso no padrão existente | CODIGO | src/server.ts |
| FDD-RES-04 | docs/FDD.md | Resiliência | Sem fallback por e-mail nesta fase | TRANSCRICAO | [09:37] Larissa |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs via Pino, sem lib nova | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Métricas para decidir rate limit | TRANSCRICAO | [09:39] Diego |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Redaction de secret | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | requestId do request-logger para correlação | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus chama publishWebhookEvent | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Novas classes de erro | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Exportação das classes novas | CODIGO | src/shared/errors/index.ts |
| FDD-INT-04 | docs/FDD.md | Integração | errorMiddleware sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | authenticate + requireRole('ADMIN') | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Reuso do validate | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Registro em buildControllers | CODIGO | src/app.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Montagem das rotas /webhooks e /admin/webhooks | CODIGO | src/routes/index.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Worker usa createPrismaClient() | CODIGO | src/config/database.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Variáveis WEBHOOK_* | CODIGO | src/config/env.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Scripts worker no package.json | CODIGO | package.json |
| FDD-INT-12 | docs/FDD.md | Integração | Limpeza das tabelas novas antes de customer | CODIGO | tests/setup.ts |
| FDD-INT-13 | docs/FDD.md | Integração | Factory de endpoint | CODIGO | tests/helpers/factories.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Exemplo de requireRole('ADMIN') em rota | CODIGO | src/modules/users/user.routes.ts |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node ≥ 20 com fetch nativo | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | Uma instância de worker em produção | TRANSCRICAO | [09:12] Diego |
| FDD-DEP-03 | docs/FDD.md | Dependência | Revisão de segurança antes do deploy | TRANSCRICAO | [09:49] Sofia |
| FDD-RISK-01 | docs/FDD.md | Risco | Ordem invertida durante retry | TRANSCRICAO | [09:13] Larissa |
| FDD-RISK-02 | docs/FDD.md | Risco | Crescimento da outbox sem arquivamento | TRANSCRICAO | [09:07] Bruno |
| FDD-RISK-03 | docs/FDD.md | Risco | Prazo de 3 sprints e fatiamento | TRANSCRICAO | [09:46] Larissa |
| FDD-CA-01 | docs/FDD.md | Critério de aceite | Suíte existente continua verde | CODIGO | tests/orders.test.ts |
| FDD-CA-02 | docs/FDD.md | Critério de aceite | Máquina de estados inalterada | CODIGO | src/modules/orders/order.status.ts |

## Propostas de implementação (sem fala literal, derivadas de decisão)

Itens que o FDD marca como **(proposta de implementação)**. Não são requisitos inventados: concretizam uma decisão rastreável e ficam visíveis para revisão do time.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PROP-01 | docs/FDD.md | Derivado | Coluna next_attempt_at para agendar backoff | TRANSCRICAO | [09:17] Larissa |
| PROP-02 | docs/FDD.md | Derivado | Path /webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| PROP-03 | docs/FDD.md | Derivado | X-Signature com duas assinaturas durante o grace | TRANSCRICAO | [09:21] Sofia |
| PROP-04 | docs/FDD.md | Derivado | Recuperação PROCESSING→PENDING no startup | TRANSCRICAO | [09:24] Diego |
| PROP-05 | docs/FDD.md | Derivado | Bloqueio de replay duplo (409) | TRANSCRICAO | [09:18] Diego |
| PROP-06 | docs/FDD.md | Derivado | Batch size default 10 | TRANSCRICAO | [09:08] Diego |
| PROP-07 | docs/FDD.md | Derivado | Interpretação 1 envio + 5 retentativas | TRANSCRICAO | [09:17] Diego |
| PROP-08 | docs/FDD.md | Derivado | onDelete Cascade a partir de Customer | CODIGO | src/modules/customers/customer.service.ts |
