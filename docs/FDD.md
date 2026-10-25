# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Status** | Pronto para implementação (pendente revisão de segurança da Sofia nos itens HMAC/secret) |
| **Base** | [RFC](RFC.md) · [ADRs](adrs/README.md) · [PRD](PRD.md) · [Tracker](TRACKER.md) |
| **Módulo** | `src/modules/webhooks/` + entry point `src/worker.ts` |

> Convenção: `[hh:mm] Nome` aponta para a fala de origem em `TRANSCRICAO.md`. Itens marcados **(proposta de implementação)** são detalhes que a reunião não fixou e que este FDD escolhe para tornar o desenho acionável; todos derivam de uma decisão rastreável e estão listados no Tracker.

---

## 1. Contexto e motivação técnica

`OrderService.changeStatus` (`src/modules/orders/order.service.ts`) executa, dentro de `prisma.$transaction`, a validação da máquina de estados (`src/modules/orders/order.status.ts`), débito/reposição de estoque, `UPDATE orders` e `INSERT order_status_history`. Não existe nenhum ponto de extensão para efeitos colaterais externos.

Clientes B2B precisam saber dessas mudanças em menos de 10 s sem fazer polling em `GET /orders` ([09:00] Marcos, [09:02] Marcos). Uma chamada HTTP dentro da transação é inviável ([09:04] Bruno), então a entrega precisa ser **assíncrona, mas com registro atômico** do evento ([09:06] Diego).

## 2. Objetivos técnicos

| ID | Objetivo | Medida |
|---|---|---|
| OT-1 | Registrar o evento na mesma transação da mudança de status | 0 mudanças de status sem evento para endpoints interessados; rollback desfaz ambos ([09:40] Bruno) |
| OT-2 | Primeira tentativa de entrega em < 10 s após o commit | Polling de 2 s + timeout de 10 s ([09:02], [09:09], [09:42]) |
| OT-3 | Tolerar indisponibilidade do cliente por até ~15 h sem perder evento | Backoff 1m/5m/30m/2h/12h, depois DLQ ([09:17]) |
| OT-4 | Permitir ao cliente verificar origem e integridade | HMAC-SHA256 por endpoint ([09:22] Sofia) |
| OT-5 | Permitir dedup no cliente | `X-Event-Id` estável entre tentativas ([09:25] Diego) |
| OT-6 | Zero dependências novas de runtime | Reuso de Prisma, Pino, Zod, Express, `fetch` nativo do Node ≥ 20 ([09:29]–[09:30]) |

## 3. Escopo e exclusões

**Dentro**
- CRUD de endpoints de webhook por customer, com filtro por status ([09:31]–[09:33]).
- Geração e rotação de secret com grace period de 24 h ([09:21]).
- Publicação transacional em `changeStatus` ([09:40]–[09:41]).
- Worker separado com polling, retry, DLQ ([09:09]–[09:18]).
- Histórico de entregas (últimas 100) ([09:34] Marcos).
- Replay manual da DLQ por ADMIN, auditado ([09:35]–[09:36]).

**Fora** (com origem)

| Item | Situação | Fonte |
|---|---|---|
| Webhooks inbound (cliente → OMS) | Descartado | [09:02] Marcos |
| E-mail ao cliente quando o webhook falha | Adiado para próxima fase | [09:37] Larissa |
| Rate limiting de saída | Observar e decidir depois | [09:39] Larissa |
| Dashboard visual para o cliente | Projeto do time de frontend | [09:40] Larissa |
| Arquivamento de linhas entregues (~30 dias) | Fora desta feature | [09:08] Diego |
| Múltiplos workers / ordem global | Adiado; limitação conhecida | [09:13] Diego, [09:13] Larissa |
| Itens do pedido no payload | Descartado (payload enxuto) | [09:43] Diego |
| Evento na **criação** do pedido (`create`, status inicial `PENDING`) | Não discutido; a integração acordada é só em `changeStatus` ([09:40] Bruno). Registrado como fora desta entrega | [09:40] Bruno |

## 4. Modelo de dados

Novas entidades em `prisma/schema.prisma`, seguindo o padrão do arquivo (UUID `Char(36)`, `@@map` em snake_case, `createdAt/updatedAt`) ([09:51] Larissa).

```prisma
enum WebhookOutboxStatus {
  PENDING      // aguardando primeira tentativa ou replay
  PROCESSING   // reservado pelo worker no ciclo atual
  FAILED       // última tentativa falhou; aguardando next_attempt_at
  DELIVERED    // 2xx recebido
}

model WebhookEndpoint {
  id                       String        @id @default(uuid()) @db.Char(36)
  customerId               String        @db.Char(36)
  url                      String        @db.VarChar(2048)
  secret                   String        @db.VarChar(128)
  previousSecret           String?       @db.VarChar(128)
  previousSecretExpiresAt  DateTime?
  events                   Json          // OrderStatus[] que o endpoint escuta
  active                   Boolean       @default(true)
  createdAt                DateTime      @default(now())
  updatedAt                DateTime      @updatedAt

  customer   Customer             @relation(fields: [customerId], references: [id], onDelete: Cascade)
  outbox     WebhookOutbox[]
  deliveries WebhookDelivery[]
  deadLetter WebhookDeadLetter[]

  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id                String              @id @db.Char(36)          // = event_id (X-Event-Id)
  webhookEndpointId String              @db.Char(36)
  orderId           String              @db.Char(36)              // sem FK: o evento é snapshot
  eventType         String              @db.VarChar(64)           // "order.status_changed"
  payload           Json
  status            WebhookOutboxStatus @default(PENDING)
  attempts          Int                 @default(0)
  nextAttemptAt     DateTime            @default(now())
  lastError         String?             @db.VarChar(500)
  deliveredAt       DateTime?
  createdAt         DateTime            @default(now())
  updatedAt         DateTime            @updatedAt

  endpoint WebhookEndpoint @relation(fields: [webhookEndpointId], references: [id], onDelete: Cascade)

  @@index([status, createdAt])
  @@index([nextAttemptAt])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id                String   @id @default(uuid()) @db.Char(36)
  webhookEndpointId String   @db.Char(36)
  eventId           String   @db.Char(36)
  attempt           Int
  success           Boolean
  httpStatus        Int?
  durationMs        Int
  requestPayload    Json
  responseBody      String?  @db.Text   // truncado
  errorCode         String?  @db.VarChar(64)
  createdAt         DateTime @default(now())

  endpoint WebhookEndpoint @relation(fields: [webhookEndpointId], references: [id], onDelete: Cascade)

  @@index([webhookEndpointId, createdAt])
  @@index([eventId])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id                String    @id @default(uuid()) @db.Char(36)
  eventId           String    @db.Char(36)
  webhookEndpointId String    @db.Char(36)
  orderId           String    @db.Char(36)
  payload           Json
  reason            String    @db.VarChar(64)    // código WEBHOOK_*
  lastError         String?   @db.VarChar(500)
  attempts          Int
  failedAt          DateTime  @default(now())
  replayedAt        DateTime?
  replayedById      String?   @db.Char(36)

  endpoint WebhookEndpoint @relation(fields: [webhookEndpointId], references: [id], onDelete: Cascade)

  @@index([failedAt])
  @@index([eventId])
  @@map("webhook_dead_letter")
}
```

Notas de modelagem:
- `webhook_outbox.id` **é** o `event_id`: UUID gerado na inserção ([09:25] Diego). Uma mudança de status gera **uma linha por endpoint interessado**; cada linha tem seu `event_id` e seu `X-Webhook-Id`.
- Índices `status` + `created_at` conforme [09:08] Diego; `next_attempt_at` é necessário para o backoff **(proposta de implementação)**.
- `previousSecret*` implementa o grace period de 24 h ([09:21] Sofia).
- `onDelete: Cascade` a partir de `Customer` evita que o módulo crie um novo bloqueio de FK em `DELETE /customers/:id`. Remover um endpoint descarta sua outbox, DLQ e histórico **(proposta de implementação)**.
- Secrets em texto recuperável porque o HMAC precisa do valor original; proteção em repouso fica para a revisão de segurança ([RFC Q-7](RFC.md#5-questões-em-aberto)).

## 5. Fluxos detalhados

### 5.1 Cadastro de endpoint

1. `POST /api/v1/webhooks` com `customerId`, `url`, `events` ([09:31] Marcos). O `customerId` vem no body, **não** do JWT, porque o JWT identifica o usuário operador ([09:32] Bruno, [09:32] Larissa).
2. `validate({ body })` valida forma; o service valida URL absoluta com protocolo `https` (senão `WEBHOOK_INVALID_URL`) ([09:23] Sofia) e existência do customer.
3. Service gera a secret com `crypto.randomBytes(32).toString('hex')` **(proposta de implementação)** e persiste.
4. Resposta `201` com a secret **apenas nesta resposta e na rotação**; listagens não devolvem a secret (consequência de [09:21] Sofia, "se vaza uma, vaza tudo").

### 5.2 Criação do evento na outbox (dentro de `changeStatus`)

```ts
// src/modules/webhooks/webhook.publisher.ts
export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: Pick<Order, 'id' | 'orderNumber' | 'customerId' | 'totalCents'>,
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<number> {
  const endpoints = await tx.webhookEndpoint.findMany({
    where: { customerId: order.customerId, active: true },
    select: { id: true, events: true },
  });
  const targets = endpoints.filter((e) => (e.events as OrderStatus[]).includes(toStatus));
  if (targets.length === 0) return 0;                      // filtro na inserção

  const occurredAt = new Date().toISOString();
  await tx.webhookOutbox.createMany({
    data: targets.map((e) => {
      const eventId = randomUUID();
      return {
        id: eventId,
        webhookEndpointId: e.id,
        orderId: order.id,
        eventType: 'order.status_changed',
        payload: buildPayload(eventId, occurredAt, order, fromStatus, toStatus), // snapshot
      };
    }),
  });
  return targets.length;
}
```

Sequência na transação:

```
tx ─► findUnique(order) ─► canTransition ─► debit/replenish stock
   ─► update orders ─► create order_status_history
   ─► publishWebhookEvent(tx, order, from, to)    ◄── NOVO
   ─► findUnique(refreshed) ─► commit
```

- Qualquer erro em `publishWebhookEvent` propaga e faz **rollback** da mudança de status ([09:40] Bruno, [09:41] Diego).
- Payload montado e gravado na inserção ([09:52] Larissa); filtro pelos `events` do endpoint ([09:34] Bruno).
- Endpoints `active = false` não recebem evento ([09:21] Bruno).

### 5.3 Processamento pelo worker

`src/worker.ts` (bootstrap) → `WebhookProcessor.runForever()` em `src/modules/webhooks/webhook.processor.ts` ([09:28] Bruno).

```
startup:
  prisma = createPrismaClient()                                  // client próprio ([09:30])
  UPDATE webhook_outbox SET status='PENDING' WHERE status='PROCESSING'   // recupera crash
loop a cada WEBHOOK_POLL_INTERVAL_MS (2000):
  batch = SELECT * FROM webhook_outbox
          WHERE status IN ('PENDING','FAILED') AND next_attempt_at <= NOW(3)
          ORDER BY created_at ASC
          LIMIT WEBHOOK_BATCH_SIZE                                // batch pequeno ([09:08])
  UPDATE ... SET status='PROCESSING' WHERE id IN (batch)
  para cada evento, em ordem de created_at (sequencial):
    endpoint = findUnique(webhookEndpointId)
    se endpoint inexistente/inativo → descarta linha, log WEBHOOK_NOT_FOUND
    body = JSON.stringify(payload)
    se Buffer.byteLength(body) > 65536 → DLQ direto (WEBHOOK_PAYLOAD_TOO_LARGE), sem retry
    se secret ausente → DLQ direto (WEBHOOK_SECRET_REQUIRED)
    assinatura = HMAC_SHA256(secret, body) [+ previousSecret se dentro do grace]
    POST endpoint.url, timeout 10s
    grava webhook_deliveries (attempt, status, durationMs, payload, resposta truncada)
    2xx      → status=DELIVERED, deliveredAt=now
    falha    → §5.4
SIGINT/SIGTERM: termina o evento corrente, para o loop, prisma.$disconnect()
```

- Processamento **sequencial** dentro do batch preserva a ordem de `created_at` ([09:12] Diego).
- `X-Timestamp` é gerado no momento do envio; o `timestamp` do payload é o da mudança de status ([09:43]–[09:44] Diego).

### 5.4 Retry com backoff

| Falha nº (`attempts` após incremento) | Próxima tentativa em | Acumulado desde a 1ª falha |
|---|---|---|
| 1 | +1 min | 1 min |
| 2 | +5 min | 6 min |
| 3 | +30 min | 36 min |
| 4 | +2 h | 2h36 |
| 5 | +12 h | 14h36 |
| 6 | — → DLQ | — |

```ts
const BACKOFF_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000]; // [09:17] Diego
attempts += 1;
if (attempts > BACKOFF_MS.length) moveToDeadLetter(event, 'WEBHOOK_MAX_ATTEMPTS_EXCEEDED');
else update({ status: 'FAILED', attempts, nextAttemptAt: now + BACKOFF_MS[attempts - 1], lastError });
```

Interpretação "1 envio + 5 retentativas" justificada no [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md#nota-de-interpretação-registrada-na-revisão-do-documento).

### 5.5 DLQ e replay

**Mover para DLQ** (uma transação): `INSERT webhook_dead_letter (eventId, endpoint, orderId, payload, reason, lastError, attempts)` + `DELETE webhook_outbox WHERE id = eventId` ([09:18] Diego).

**Replay** — `POST /api/v1/admin/webhooks/dead-letter/:id/replay` (ADMIN):
1. Busca o registro da DLQ; inexistente → `WEBHOOK_DEAD_LETTER_NOT_FOUND`.
2. Já reprocessado (`replayedAt` não nulo) → `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` **(proposta de implementação, evita replay duplo)**.
3. Na mesma transação: `INSERT webhook_outbox` com **o mesmo `id = eventId`** e mesmo payload, `status = PENDING`, `attempts = 0`, `nextAttemptAt = now`; `UPDATE webhook_dead_letter SET replayedAt, replayedById` ([09:18] Diego: "recoloca na outbox como pendente").
4. `logger.info({ deadLetterId, eventId, userId }, 'webhook_dead_letter_replayed')` para auditoria ([09:36] Sofia).

### 5.6 Rotação de secret

`POST /api/v1/webhooks/:id/rotate-secret` **(path proposto; operação decidida em [09:21] Sofia)**:
1. `previousSecret = secret`, `previousSecretExpiresAt = now + 24h`, `secret = nova`.
2. Durante a janela, o worker envia `X-Signature: <hmac_nova>,<hmac_antiga>` e o cliente aceita se **qualquer uma** bater **(proposta, pendente revisão da Sofia — [RFC Q-7](RFC.md#5-questões-em-aberto))**.
3. Após a expiração, o worker ignora `previousSecret`; a próxima rotação o sobrescreve.

## 6. Contratos públicos

Todas as rotas ficam sob `/api/v1` (montagem em `src/app.ts`) e exigem `Authorization: Bearer <jwt>` (`authenticate`). Erros seguem o envelope do `errorMiddleware`: `{ "error": { "code", "message", "details?" } }`.

### 6.1 `POST /api/v1/webhooks` — cadastrar endpoint ([09:31] Marcos)

Request
```http
POST /api/v1/webhooks
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{
  "customerId": "3f1c2a9e-5b7d-4e21-9a0c-1d2e3f4a5b6c",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created`
```json
{
  "id": "8a7b6c5d-4e3f-4a2b-9c1d-0e9f8a7b6c5d",
  "customerId": "3f1c2a9e-5b7d-4e21-9a0c-1d2e3f4a5b6c",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "4b1f0e…c9a2",
  "createdAt": "2026-11-03T12:00:00.000Z",
  "updatedAt": "2026-11-03T12:00:00.000Z"
}
```

| Status | Quando |
|---|---|
| 201 | Criado; `secret` exibida só aqui (e na rotação) |
| 400 `VALIDATION_ERROR` | Body fora do schema (`events` vazio ou com status inexistente) |
| 400 `WEBHOOK_INVALID_URL` | URL inválida ou não-`https` |
| 401 `UNAUTHORIZED` | Sem JWT válido |
| 404 `WEBHOOK_CUSTOMER_NOT_FOUND` | `customerId` inexistente |

Schema (`src/modules/webhooks/webhook.schemas.ts`):
```ts
export const createWebhookSchema = z.object({
  customerId: z.string().uuid(),
  url: z.string().min(1).max(2048),
  events: z.array(z.nativeEnum(OrderStatus)).min(1),
});
export const httpsUrlSchema = z.string().url().refine((u) => new URL(u).protocol === 'https:');
```

### 6.2 `GET /api/v1/webhooks?customerId=<uuid>` — listar endpoints do customer ([09:33] Bruno)

Response `200 OK`
```json
{
  "data": [
    {
      "id": "8a7b6c5d-4e3f-4a2b-9c1d-0e9f8a7b6c5d",
      "customerId": "3f1c2a9e-5b7d-4e21-9a0c-1d2e3f4a5b6c",
      "url": "https://hooks.atlascomercial.com.br/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-11-03T12:00:00.000Z",
      "updatedAt": "2026-11-03T12:00:00.000Z"
    }
  ]
}
```
Status: `200`, `400 VALIDATION_ERROR` (sem `customerId`), `401`. A secret nunca é listada.

### 6.3 `PATCH /api/v1/webhooks/:id` — editar endpoint ([09:33] Bruno)

Request
```json
{ "url": "https://hooks.atlascomercial.com.br/oms/v2", "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```
Response `200 OK`: mesmo formato do item de 6.2.
Status: `200`, `400 VALIDATION_ERROR` (body vazio), `400 WEBHOOK_INVALID_URL`, `401`, `404 WEBHOOK_NOT_FOUND`. Mudanças **não** alteram eventos já enfileirados ([ADR-007](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md)).

### 6.4 `DELETE /api/v1/webhooks/:id` — remover endpoint ([09:33] Bruno)

Response `204 No Content`. Status: `204`, `401`, `404 WEBHOOK_NOT_FOUND`.

### 6.5 `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret ([09:21] Sofia)

Request: sem body.
Response `200 OK`
```json
{
  "id": "8a7b6c5d-4e3f-4a2b-9c1d-0e9f8a7b6c5d",
  "secret": "e02d…77f1",
  "previousSecretExpiresAt": "2026-11-04T12:00:00.000Z"
}
```
Status: `200`, `401`, `404 WEBHOOK_NOT_FOUND`.

### 6.6 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas ([09:34] Marcos)

Response `200 OK` (últimas 100, mais recentes primeiro)
```json
{
  "data": [
    {
      "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f",
      "eventId": "0b9c8d7e-6f5a-4b3c-2d1e-0f9e8d7c6b5a",
      "attempt": 2,
      "success": false,
      "httpStatus": 503,
      "durationMs": 412,
      "errorCode": "WEBHOOK_DELIVERY_HTTP_ERROR",
      "requestPayload": { "event_id": "0b9c8d7e-6f5a-4b3c-2d1e-0f9e8d7c6b5a", "event_type": "order.status_changed" },
      "responseBody": "Service Unavailable",
      "createdAt": "2026-11-03T12:06:02.000Z"
    }
  ]
}
```
Status: `200`, `401`, `404 WEBHOOK_NOT_FOUND`.

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay da DLQ ([09:18] Diego, [09:36] Sofia)

Request: sem body; `requireRole('ADMIN')`.
Response `202 Accepted`
```json
{
  "deadLetterId": "5e6f7a8b-9c0d-4e1f-8a2b-3c4d5e6f7a8b",
  "eventId": "0b9c8d7e-6f5a-4b3c-2d1e-0f9e8d7c6b5a",
  "status": "PENDING",
  "replayedAt": "2026-11-04T09:15:00.000Z",
  "replayedById": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d"
}
```
Status: `202` (reenfileirado; entrega assíncrona), `401`, `403 FORBIDDEN` (não-ADMIN, via `requireRole`), `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`, `409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`.

### 6.8 Contrato outbound — request do OMS para o cliente ([09:43]–[09:45])

```http
POST https://hooks.atlascomercial.com.br/oms
Content-Type: application/json
X-Event-Id: 0b9c8d7e-6f5a-4b3c-2d1e-0f9e8d7c6b5a
X-Webhook-Id: 8a7b6c5d-4e3f-4a2b-9c1d-0e9f8a7b6c5d
X-Timestamp: 2026-11-03T12:00:02.114Z
X-Signature: 9f2c1b…e4d0

{
  "event_id": "0b9c8d7e-6f5a-4b3c-2d1e-0f9e8d7c6b5a",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-03T12:00:00.871Z",
  "order_id": "d4c3b2a1-0f9e-4d8c-7b6a-5f4e3d2c1b0a",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "3f1c2a9e-5b7d-4e21-9a0c-1d2e3f4a5b6c",
  "total_cents": 1700000
}
```

| Header | Conteúdo | Fonte |
|---|---|---|
| `X-Event-Id` | UUID do evento, estável em retry e replay | [09:25] Diego |
| `X-Signature` | HMAC-SHA256 hex do corpo bruto com a secret do endpoint | [09:20], [09:22] Sofia |
| `X-Timestamp` | Instante do envio, para o cliente detectar replay attack | [09:44] Diego |
| `X-Webhook-Id` | ID do endpoint cadastrado | [09:44] Sofia |
| `Content-Type` | `application/json` | [09:44] Diego |

**Semântica para o cliente:** responder `2xx` em até 10 s significa "recebido"; qualquer outra coisa gera retry. O mesmo `X-Event-Id` pode chegar mais de uma vez ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)).

Verificação no cliente (exemplo para o portal do desenvolvedor, [09:26] Marcos):
```ts
const expected = createHmac('sha256', secret).update(rawBody).digest('hex');
const ok = req.header('X-Signature')!.split(',')
  .some((sig) => timingSafeEqual(Buffer.from(sig), Buffer.from(expected)));
```

## 7. Matriz de erros

Classes novas em `src/shared/errors/http-errors.ts` (ou `src/modules/webhooks/webhook.errors.ts` reexportado por `src/shared/errors/index.ts`), todas estendendo `AppError` ([09:28] Bruno, [09:29] Larissa).

| Código | HTTP | Onde | Gatilho | Comportamento |
|---|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | API / worker | Endpoint inexistente em PATCH/DELETE/rotate/deliveries; no worker, endpoint removido | API: erro ao cliente. Worker: descarta a linha e loga |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | API | `customerId` inexistente no POST | Erro ao cliente |
| `WEBHOOK_INVALID_URL` | 400 | API | URL malformada ou não-`https` ([09:23] Sofia) | Erro ao cliente; nada persiste |
| `WEBHOOK_SECRET_REQUIRED` | — (interno) | Worker | Endpoint sem secret no envio (citado por Bruno [09:28]) | DLQ direto, sem retry |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | — (interno) | Worker | Corpo > 64 KB ([09:24] Diego, Larissa) | DLQ direto, sem envio e sem retry ("erra, não trunca" — [09:23] Sofia) |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (interno) | Worker | Sem resposta em 10 s ([09:42] Diego) | Retry com backoff |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | — (interno) | Worker | Resposta não-2xx | Retry com backoff |
| `WEBHOOK_DELIVERY_NETWORK_ERROR` | — (interno) | Worker | DNS, TLS, conexão recusada | Retry com backoff |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | — (interno) | Worker | 6ª falha consecutiva | Move para DLQ ([09:15] Diego) |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | API admin | Replay de ID inexistente | Erro ao cliente |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | API admin | Replay repetido | Erro ao cliente |
| `VALIDATION_ERROR` (existente) | 400 | API | Falha de schema Zod via `validate` | Padrão atual |
| `UNAUTHORIZED` / `FORBIDDEN` (existentes) | 401 / 403 | API | JWT ausente / role ≠ ADMIN no replay | `authenticate` / `requireRole` |
| Erro na inserção da outbox | 500 `INTERNAL_SERVER_ERROR` | `changeStatus` | Falha de banco em `publishWebhookEvent` | **Rollback** da mudança de status ([09:40] Bruno) |

Códigos internos do worker não viram resposta HTTP: vão para `webhook_deliveries.errorCode`, `webhook_dead_letter.reason` e logs.

## 8. Estratégias de resiliência

| Mecanismo | Valor | Fonte |
|---|---|---|
| Timeout por envio | 10 s, `AbortSignal.timeout(10_000)` no `fetch` | [09:42] Diego |
| Retry | 5 retentativas, backoff 1m/5m/30m/2h/12h | [09:17] Diego / Larissa |
| Fallback final | DLQ em tabela própria + replay manual ADMIN | [09:18] Diego |
| Limite de payload | 64 KB, falha sem truncar | [09:24] Diego / Larissa |
| Isolamento | Worker em processo separado; API não depende dele | [09:11] Diego |
| Recuperação de crash | No startup, `PROCESSING` → `PENDING` (seguro com single-worker; pode gerar duplicata coberta por at-least-once) | [09:24] Diego, [09:12] Diego |
| Shutdown gracioso | `SIGINT`/`SIGTERM` como em `src/server.ts`: termina o evento corrente e desconecta o Prisma | padrão de `src/server.ts` |
| Atomicidade | Outbox na transação de `changeStatus` | [09:06] Diego |
| Fallback por e-mail | **Não** nesta fase | [09:37] Larissa |

Configuração nova em `src/config/env.ts` (com defaults, sem quebrar `.env` atual) **(proposta de implementação; valores de [09:09], [09:42], [09:24])**:

```ts
WEBHOOK_POLL_INTERVAL_MS: z.coerce.number().int().positive().default(2000),
WEBHOOK_HTTP_TIMEOUT_MS: z.coerce.number().int().positive().default(10000),
WEBHOOK_BATCH_SIZE: z.coerce.number().int().positive().default(10),
WEBHOOK_MAX_PAYLOAD_BYTES: z.coerce.number().int().positive().default(65536),
```

## 9. Observabilidade

O projeto não tem biblioteca de métricas nem de tracing; a reunião vetou infraestrutura nova ([09:07], [09:29]). A observabilidade usa **Pino estruturado** e **consultas às tabelas do módulo**.

### 9.1 Métricas

Emitidas como campos de log (`msg` fixo, agregáveis por qualquer coletor de logs) e calculáveis por SQL.

| Métrica | Tipo | Origem | Uso |
|---|---|---|---|
| `webhook_events_enqueued_total` | contador | log `webhook_event_enqueued` | Volume por customer/status |
| `webhook_deliveries_total{result}` | contador | `webhook_deliveries.success` | Taxa de sucesso |
| `webhook_delivery_duration_ms` | histograma | `webhook_deliveries.durationMs` | Clientes lentos (perto de 10 s) |
| `webhook_end_to_end_latency_ms` | histograma | `deliveredAt − createdAt` | Meta < 10 s ([09:02]) |
| `webhook_outbox_pending` | gauge | `COUNT(*)` PENDING/FAILED, logado a cada ciclo | Backlog |
| `webhook_outbox_oldest_pending_age_s` | gauge | `NOW() − MIN(created_at)` dos pendentes | **Worker parado** |
| `webhook_dead_letter_total` | contador | `webhook_dead_letter` | Falhas permanentes; subsídio para a decisão de e-mail ([09:37]) |
| `webhook_events_per_customer_per_min` | gauge | outbox por `customer_id` | Subsídio para a decisão de rate limiting ([09:39]) |

Alertas sugeridos: idade do pendente mais antigo > 60 s (worker parado) e crescimento de `webhook_dead_letter_total`.

### 9.2 Logs (Pino)

| `msg` | Campos | Nível |
|---|---|---|
| `webhook_event_enqueued` | `eventId, webhookEndpointId, orderId, fromStatus, toStatus` | info |
| `webhook_delivery_succeeded` | `eventId, webhookEndpointId, attempt, httpStatus, durationMs` | info |
| `webhook_delivery_failed` | `eventId, webhookEndpointId, attempt, errorCode, httpStatus?, nextAttemptAt` | warn |
| `webhook_moved_to_dead_letter` | `eventId, webhookEndpointId, reason, attempts` | error |
| `webhook_dead_letter_replayed` | `deadLetterId, eventId, userId` | info (auditoria, [09:36] Sofia) |
| `webhook_worker_cycle` | `batchSize, pending, oldestPendingAgeS, cycleMs` | debug/info |
| `worker_started` / `shutdown_initiated` | `pollIntervalMs` / `signal` | info |

`'*.secret'` e `'*.previousSecret'` entram em `redactPaths` de `src/shared/logger/index.ts` (incidente de secret vazada em log, [09:22] Diego).

### 9.3 Tracing

Sem APM no projeto, o rastreio é por **IDs de correlação** propagados em logs, tabelas e headers:
- `eventId` liga `webhook_event_enqueued` → linhas de `webhook_deliveries` → `webhook_dead_letter` → `webhook_dead_letter_replayed`, e chega ao cliente em `X-Event-Id`, o que permite cruzar com os logs dele.
- `orderId` liga o evento ao `order_status_history` e ao `http_request` do `PATCH /orders/:id/status`.
- `requestId` (já gerado por `src/middlewares/request-logger.middleware.ts`) é registrado no log `webhook_event_enqueued` quando disponível no controller de pedidos.
- Adoção de tracing distribuído (ex.: OpenTelemetry) fica fora desta fase por exigir dependência nova.

## 10. Integração com o sistema existente

| # | Arquivo real | Mudança |
|---|---|---|
| 1 | `src/modules/orders/order.service.ts` | Em `changeStatus`, após `tx.orderStatusHistory.create(...)` e antes do `findUnique` final, chamar `await publishWebhookEvent(tx, order, from, to)`. Mesma transação: erro propaga e faz rollback ([09:40]–[09:41]). Construtor do `OrderService` **não muda** (função pura, sem injeção de repository). `create()` não é alterado. |
| 2 | `src/shared/errors/http-errors.ts` + `src/shared/errors/index.ts` | Novas classes `WebhookNotFoundError` (404), `WebhookCustomerNotFoundError` (404), `WebhookInvalidUrlError` (400), `WebhookDeadLetterNotFoundError` (404), `WebhookDeadLetterAlreadyReplayedError` (409, via `ConflictError`), todas sobre `AppError` com prefixo `WEBHOOK_` ([09:28]–[09:29]). `NotFoundError` não é reutilizado diretamente porque fixa o código `NOT_FOUND`. |
| 3 | `src/middlewares/error.middleware.ts` | **Sem alteração**: o ramo `err instanceof AppError` já serializa os novos erros ([09:29] Bruno). |
| 4 | `src/middlewares/auth.middleware.ts` | Reuso: `authenticate` em todo o router de webhooks; `requireRole('ADMIN')` no replay ([09:36] Larissa), como já feito em `src/modules/users/user.routes.ts`. |
| 5 | `src/middlewares/validate.middleware.ts` | Reuso para body/query/params dos novos schemas Zod. |
| 6 | `src/app.ts` | Em `buildControllers`, instanciar `WebhookRepository`, `WebhookService`, `WebhookController`, `WebhookAdminController` e devolvê-los em `Controllers`. |
| 7 | `src/routes/index.ts` | Adicionar `webhooks` e `webhooksAdmin` ao tipo `Controllers`; montar `router.use('/webhooks', buildWebhookRouter(...))` e `router.use('/admin/webhooks', buildWebhookAdminRouter(...))`. |
| 8 | `src/server.ts` | Referência para `src/worker.ts`: mesmo padrão de `bootstrap()`, `logger.fatal` em falha e `shutdown` em `SIGINT`/`SIGTERM`. O arquivo em si não muda. |
| 9 | `src/config/database.ts` | O worker chama `createPrismaClient()` para ter instância própria ([09:30] Bruno). |
| 10 | `src/config/env.ts` | Quatro variáveis `WEBHOOK_*` com default (§8). |
| 11 | `src/shared/logger/index.ts` | Adicionar `'*.secret'` e `'*.previousSecret'` a `redactPaths`. |
| 12 | `prisma/schema.prisma` | Enum `WebhookOutboxStatus`, quatro models (§4), relação `webhookEndpoints WebhookEndpoint[]` em `Customer`; nova migration em `prisma/migrations/`. |
| 13 | `package.json` | Scripts `"worker": "tsx --env-file=.env src/worker.ts"` e `"worker:start": "node --env-file=.env dist/worker.js"`, espelhando `dev`/`start` ([09:11] Larissa). |
| 14 | `tests/setup.ts` | Limpar `webhookDelivery`, `webhookDeadLetter`, `webhookOutbox`, `webhookEndpoint` **antes** de `customer.deleteMany()`, senão a FK nova quebra o `beforeEach` atual. |
| 15 | `tests/helpers/factories.ts` | Nova factory `createTestWebhookEndpoint()` reutilizando `createTestCustomer()`. |

Estrutura nova:
```
src/
├── worker.ts                              # entry point do worker
└── modules/webhooks/
    ├── webhook.controller.ts
    ├── webhook.admin.controller.ts
    ├── webhook.service.ts
    ├── webhook.repository.ts
    ├── webhook.routes.ts                  # buildWebhookRouter, buildWebhookAdminRouter
    ├── webhook.schemas.ts
    ├── webhook.errors.ts
    ├── webhook.publisher.ts               # publishWebhookEvent(tx, …)
    ├── webhook.signer.ts                  # HMAC-SHA256
    └── webhook.processor.ts               # loop, envio, retry, DLQ
```

## 11. Dependências e compatibilidade

| Item | Situação |
|---|---|
| Node.js | `>=20` (`package.json`): `fetch`, `AbortSignal.timeout`, `crypto.randomUUID`, `createHmac` nativos. Nenhuma dependência nova |
| MySQL 8 + Prisma 5.22 | Colunas `Json` suportadas; migration aditiva (só tabelas novas + relação) |
| Contratos existentes | Nenhum endpoint atual muda de forma ou status code. `PATCH /orders/:id/status` só fica um pouco mais lento (1 SELECT + N INSERT) |
| Deploy | Novo processo `npm run worker` em produção ao lado da API; **uma instância** ([09:12] Diego) |
| Rollback | Desligar o worker para a entrega sem afetar pedidos; eventos acumulam e são entregues ao religar |
| Segurança | Revisão da Sofia (≥ 2 dias úteis) antes do deploy ([09:46]) |
| Portal do desenvolvedor | Marcos documenta at-least-once, headers e verificação HMAC ([09:26], [09:40]) |

## 12. Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| Worker parado sem ninguém perceber | Média | Alto | Gauge de idade do pendente mais antigo + alerta; eventos não se perdem (outbox) |
| Ordem invertida durante retry (evento N+1 entregue antes do N que está em backoff) | Média | Médio | Payload traz `from_status`/`to_status` e `timestamp`; documentar no portal; ordem só é garantida na primeira tentativa e com single-worker ([09:13]) |
| Duas instâncias do worker por engano | Baixa | Médio | Runbook de deploy com uma réplica; duplicatas cobertas por `X-Event-Id` |
| Vazamento de secret em log | Média | Alto | `redact` no Pino, secret só nas respostas de criação/rotação, rotação com grace ([09:22]) |
| `changeStatus` mais lento por inserts na outbox | Baixa | Médio | Índice `customer_id, active`; filtro na inserção; payload enxuto |
| Cliente recebe rajadas (muitos pedidos mudando juntos) | Média | Médio | Métrica de eventos/min por customer para a decisão de rate limiting ([09:39]) |
| Crescimento da outbox sem arquivamento | Alta (no longo prazo) | Baixo | Índices em `status`/`created_at`; arquivamento planejado como próxima feature ([09:08]) |
| Prazo de 3 sprints estourar | Média | Alto | Fatiamento: modelagem → worker/retry → CRUD/deliveries → integração/E2E → revisão de segurança ([09:46]) |

## 13. Critérios de aceite técnicos

- [ ] `PATCH /api/v1/orders/:id/status` com endpoint ativo escutando o `toStatus` cria exatamente 1 linha em `webhook_outbox` por endpoint, com `payload` snapshot.
- [ ] Falha forçada em `publishWebhookEvent` deixa `orders.status`, `order_status_history` e estoque inalterados (rollback).
- [ ] Nenhuma linha é criada quando nenhum endpoint ativo do customer escuta o `toStatus`.
- [ ] Worker roda via `npm run worker`, em processo próprio, e entrega em < 10 s após o commit em ambiente local.
- [ ] Request outbound contém `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`, e `X-Signature` confere com HMAC-SHA256 do corpo bruto.
- [ ] Endpoint que responde 500 recebe retentativas em 1m/5m/30m/2h/12h (testável com relógio simulado) e o evento termina em `webhook_dead_letter` com `WEBHOOK_MAX_ATTEMPTS_EXCEEDED`.
- [ ] Endpoint que demora > 10 s gera falha `WEBHOOK_DELIVERY_TIMEOUT`.
- [ ] Payload > 64 KB vai para DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE` sem nenhum envio.
- [ ] Replay por OPERATOR retorna `403`; por ADMIN retorna `202`, reenfileira com o **mesmo** `event_id` e gera log `webhook_dead_letter_replayed` com `userId`.
- [ ] Cadastro com `http://` retorna `400 WEBHOOK_INVALID_URL`.
- [ ] Após rotação, ambas as secrets validam por 24 h; depois, só a nova.
- [ ] `GET /webhooks/:id/deliveries` retorna no máximo 100 itens, mais recentes primeiro.
- [ ] Secret não aparece em listagens nem em logs.
- [ ] `npm test`, `npm run lint` e `npm run build` verdes; suíte existente de `tests/orders.test.ts` continua passando.
