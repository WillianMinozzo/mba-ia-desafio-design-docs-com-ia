# ADR-006 — Reuso dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa, Bruno, Diego, Sofia
- **Relacionados:** [ADR-002](ADR-002-worker-processo-separado-polling.md), [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md)

## Contexto

O OMS já tem convenções consolidadas ([09:27] Bruno):

| Padrão | Onde está no código |
|---|---|
| Módulo por domínio com controller, service, repository, routes, schemas | `src/modules/orders/`, `src/modules/customers/`, `src/modules/products/` |
| Hierarquia de erros com `statusCode` + `errorCode` | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` (`InvalidStatusTransitionError`, `InsufficientStockError`) |
| Tratamento centralizado de `AppError`, `ZodError` e erros Prisma | `src/middlewares/error.middleware.ts` |
| Logger Pino com redaction | `src/shared/logger/index.ts` |
| Validação Zod por middleware | `src/middlewares/validate.middleware.ts` |
| Autenticação JWT e autorização por role | `authenticate` e `requireRole` em `src/middlewares/auth.middleware.ts` |
| IDs UUID em todas as tabelas | `prisma/schema.prisma` (`@default(uuid()) @db.Char(36)`) |
| Composição manual de dependências e rotas | `src/app.ts` (`buildControllers`), `src/routes/index.ts` (`buildApiRouter`) |

A dúvida era se o módulo de webhooks introduziria estruturas próprias (biblioteca de fila, logger, formato de erro).

## Decisão

**Reuso máximo do que já existe** ([09:30] Larissa):

1. Módulo `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, mais `webhook.processor.ts` (lógica do worker) ([09:27] Bruno, [09:28] Bruno).
2. Novas classes de erro estendendo `AppError` (via `NotFoundError`, `ValidationError`, `UnprocessableEntityError`, `ConflictError`), com **prefixo `WEBHOOK_`** em todos os códigos: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` etc. ([09:28] Bruno, [09:29] Larissa).
3. **Nenhuma mudança** no `errorMiddleware`: ele já trata `AppError` ([09:29] Bruno).
4. **Pino** como único logger, no API e no worker ([09:29] Bruno).
5. Replay da DLQ protegido com o **`requireRole('ADMIN')` existente** ([09:36] Larissa).
6. IDs **UUID** nas tabelas novas, seguindo o resto do projeto ([09:51] Larissa).
7. Worker com `PrismaClient` próprio criado pelo `createPrismaClient()` de `src/config/database.ts` ([09:30] Bruno).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Estrutura própria para o módulo** (ex.: biblioteca de jobs, logger dedicado, envelope de erro próprio) | Duplica infraestrutura, quebra a consistência das respostas de erro e aumenta o custo de revisão; a reunião foi explícita em "não vamos botar nada novo" ([09:29] Bruno). |
| **Injetar o repository de webhooks inteiro no `OrderService`** | Acopla pedidos ao módulo de webhooks; preferiu-se uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o client da transação ([09:41] Bruno, [09:41] Diego). |
| **ID auto-incremental na outbox** | Fugiria do padrão UUID do projeto ([09:51] Diego pergunta, [09:51] Larissa decide UUID). |

## Consequências

**Positivas**
- Erros do módulo saem no mesmo envelope `{ error: { code, message, details } }` sem tocar no middleware.
- Onboarding trivial: quem conhece `orders` conhece `webhooks`.
- Revisão de código mais rápida; nenhuma dependência nova a auditar.

**Negativas / trade-offs**
- O padrão atual não tem DI container; `src/app.ts` e `src/routes/index.ts` precisam ser editados à mão para registrar o módulo.
- `AuthUser.role` só tem `ADMIN | OPERATOR`; não existe vínculo usuário → customer. Qualquer usuário autenticado gerencia webhooks de qualquer customer nesta fase ([09:37] Sofia: "mais pra frente a gente pode endurecer").
- Sem biblioteca de jobs, o loop, o backoff e a recuperação de linhas presas são código próprio, com testes próprios.
- **Trade-off explícito:** consistência e zero dependências em troca de escrever à mão o que uma lib de filas daria pronto.
