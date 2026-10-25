# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autor** | Larissa (Tech Lead), responsável por abrir o design doc ([09:50]). Redação: Will, com apoio de IA |
| **Status** | Em revisão |
| **Data** | 2026-10-25 (reunião de origem: quinta-feira, 09:00–09:53) |
| **Revisores** | Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança), Marcos (Produto) |
| **Documentos** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

---

## 1. TL;DR

Propomos notificar clientes B2B, via **webhooks HTTP outbound**, sempre que um pedido deles muda de status. A mudança de status grava o evento numa **tabela outbox no MySQL, na mesma transação**; um **worker em processo separado** lê a outbox a cada **2 s** e faz o POST para a URL `https` do cliente, **assinado com HMAC-SHA256** (secret por endpoint). Falhas são retentadas com **backoff 1m/5m/30m/2h/12h** e, esgotadas, vão para uma **DLQ** com replay manual por ADMIN. A entrega é **at-least-once**; o cliente deduplica por `X-Event-Id`. Tudo reaproveita os padrões atuais do projeto. Estimativa: **3 sprints** com revisão de segurança incluída.

## 2. Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo fazem polling em `GET /orders` para descobrir mudanças de status, o que deixa a integração lenta e cara para eles. A Atlas sinalizou possível migração para o concorrente se a solução não sair até o fim do trimestre ([09:00] Marcos). Para os clientes, "tempo real" é **abaixo de 10 s** ([09:02] Marcos).

No OMS, a mudança de status ocorre em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de uma transação que já atualiza `orders`, grava `order_status_history` e mexe em estoque ([09:04] Bruno). O sistema não tem nenhum mecanismo de eventos, filas ou notificação externa.

**Restrições de desenho:**
- Nenhuma chamada HTTP pode entrar nessa transação, e uma falha no cliente não pode desfazer a mudança de status ([09:04] Bruno).
- Se o status mudou, o evento não pode se perder ([09:40] Bruno).
- Time pequeno: sem infraestrutura nova ([09:07] Diego).

## 3. Proposta técnica

```
PATCH /orders/:id/status
        │
        ▼
┌──────────────── transação MySQL (changeStatus) ────────────────┐
│ update orders · insert order_status_history · estoque          │
│ publishWebhookEvent(tx, …) → insert webhook_outbox (snapshot)  │
└────────────────────────────────────────────────────────────────┘
        │ commit
        ▼
  webhook_outbox ◄── poll 2s ── worker (src/worker.ts, processo próprio)
                                   │  POST https://cliente  (HMAC, X-Event-Id)
                                   ├─ 2xx ─────────► entregue + webhook_deliveries
                                   ├─ falha ───────► agenda retry (1m…12h)
                                   └─ esgotou ─────► webhook_dead_letter ─► replay ADMIN
```

Componentes:

1. **Configuração de endpoints** (`webhook_endpoints`): CRUD autenticado onde o cliente cadastra URL `https` e a lista de status que quer ouvir; o OMS gera a secret e permite rotacioná-la ([09:31]–[09:33], [09:21]). → [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
2. **Publicação transacional**: `changeStatus` chama uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` que, para cada endpoint ativo interessado no novo status, grava o payload já renderizado na outbox ([09:41], [09:34], [09:52]). → [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-007](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md)
3. **Worker de entrega**: processo Node separado, mesmo banco, polling 2 s, single-instance, timeout HTTP de 10 s ([09:09]–[09:11], [09:42]). → [ADR-002](adrs/ADR-002-worker-processo-separado-polling.md)
4. **Resiliência**: backoff exponencial em 5 degraus, DLQ em tabela própria, replay manual restrito a ADMIN e auditado ([09:17]–[09:18], [09:36]). → [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md)
5. **Contrato de entrega**: JSON enxuto + headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`; semântica at-least-once ([09:43]–[09:45], [09:24]). → [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)
6. **Histórico de entregas** consultável pelo cliente: últimas 100 tentativas com resultado, payload, resposta e tempo ([09:34] Marcos).
7. **Encaixe no código**: novo módulo `src/modules/webhooks/` no padrão dos demais, erros `WEBHOOK_*` sobre `AppError`, Pino, `errorMiddleware` e `requireRole` existentes ([09:27]–[09:30]). → [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)

Contratos, modelo de dados, fluxos e matriz de erros estão no [FDD](FDD.md).

## 4. Alternativas consideradas

| # | Alternativa | Trade-off que levou ao descarte | Fonte |
|---|---|---|---|
| ALT-1 | **Disparo HTTP síncrono em `changeStatus`** | Acopla a latência de cada cliente à transação de status; cliente lento trava outros pedidos e cliente fora do ar exigiria rollback de uma mudança válida. | [09:04] Bruno, [09:06] Diego |
| ALT-2 | **Redis Streams / Redis Cluster como fila** | Entrega reatividade, mas obriga a subir e operar infra nova; overengineering para um time pequeno, e ainda não garante atomicidade com a transação MySQL. | [09:07] Larissa, [09:07] Diego |
| ALT-3 | **Trigger do banco para acordar o worker** | MySQL não tem `NOTIFY/LISTEN`; trigger não avisa processo externo sem gambiarra. Polling de 2 s já cumpre o teto de 10 s. | [09:09] Bruno, [09:09] Diego |
| ALT-4 | **Worker dentro do processo da API** | Restart/deploy da API interromperia entregas. | [09:11] Diego |
| ALT-5 | **Retry indefinido** / **3 tentativas** | Indefinido deixa evento pendurado para sempre; 3 tentativas cobrem só ~30 min, menos que manutenções reais de 2 h. | [09:15]–[09:16] Diego |
| ALT-6 | **Secret global** | Um vazamento comprometeria todos os clientes. | [09:21] Sofia |
| ALT-7 | **Exactly-once** | Exigiria coordenação dos dois lados; complexidade desproporcional. | [09:25] Diego |
| ALT-8 | **Renderizar payload no envio** | Retry horas depois mandaria estado atual com `from/to` antigo. | [09:52] Larissa |

## 5. Questões em aberto

| # | Questão | Situação na reunião | Dono sugerido |
|---|---|---|---|
| Q-1 | **Rate limiting de saída por cliente** (ex.: 50 mudanças em 1 min = 50 POSTs) | Fora do escopo; "observar e decidir depois". O FDD expõe métricas para essa decisão. | Diego — [09:38]–[09:39] |
| Q-2 | **Escalar para múltiplos workers** mantendo ordem por pedido (partição por `order_id` ou lock pessimista) | Adiado: "problema do futuro"; registrado como limitação conhecida. | Diego — [09:13] |
| Q-3 | **Endurecer autorização do CRUD** (hoje qualquer usuário autenticado gerencia webhooks de qualquer customer; não há vínculo usuário→customer no JWT) | "Por enquanto sim. Mais pra frente a gente pode endurecer." | Sofia — [09:37], [09:32] |
| Q-4 | **Notificar o cliente por e-mail** quando o webhook falha repetidamente | Adiado para próxima fase, após medir impacto. | Marcos — [09:37] |
| Q-5 | **Arquivamento** de linhas entregues da outbox (~30 dias) | Fora do escopo desta feature. | Diego — [09:08] |
| Q-6 | **Contagem de tentativas**: "5 tentativas" vs. 5 intervalos após a primeira falha (≈14h36). O FDD adota 1 envio + 5 retentativas. | Ambiguidade identificada na revisão do texto; confirmar com Diego. | Diego — [09:15], [09:17] |
| Q-7 | **Mecânica do grace period de 24 h** no envio (o FDD propõe enviar as duas assinaturas) e **armazenamento da secret em repouso** | Decidido o comportamento, não o mecanismo; cabe na revisão de segurança. | Sofia — [09:21], [09:46] |

## 6. Impacto e riscos

**Impacto no sistema existente**
- `changeStatus` ganha leitura de `webhook_endpoints` e N inserts na outbox dentro da transação.
- Três tabelas novas no MySQL (`webhook_endpoints`, `webhook_outbox`, `webhook_dead_letter`) mais o log de entregas `webhook_deliveries`.
- Um processo novo para operar (`npm run worker`).
- Nenhuma mudança em `errorMiddleware` nem no contrato dos endpoints atuais.

**Riscos principais** (detalhe e mitigação no [FDD §12](FDD.md#12-riscos-e-mitigação) e no [PRD](PRD.md#10-riscos-e-mitigação))
- Worker parado → eventos acumulam sem alarme. Mitigação: métrica de idade do pendente mais antigo.
- Ordem quebrada durante retries ou com mais de um worker. Mitigação: `from_status`/`to_status` no payload, single-worker e documentação.
- Vazamento de secret em log. Mitigação: secret por endpoint, rotação e `redact` no Pino.
- Prazo: Atlas quer até o fim de novembro; estimativa de 3 sprints com 2 dias úteis de revisão da Sofia ([09:45]–[09:47]).

## 7. Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker em processo separado com polling de 2 s](adrs/ADR-002-worker-processo-separado-polling.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](adrs/ADR-005-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes](adrs/ADR-006-reuso-dos-padroes-existentes.md)
- [ADR-007 — Payload snapshot e filtro na inserção](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md)
