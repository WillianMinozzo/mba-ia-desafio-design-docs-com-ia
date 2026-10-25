# ADR-001 — Padrão Outbox no MySQL existente

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-002](ADR-002-worker-processo-separado-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-payload-snapshot-e-filtro-na-insercao.md)

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) precisam ser notificados quando o status de seus pedidos muda ([09:00] Marcos). A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que já atualiza `orders`, insere em `order_status_history` e debita/repõe `stock_quantity` dos produtos ([09:04] Bruno).

Duas forças se opõem:

1. **Consistência:** se o status mudou, o evento precisa existir; se a transação fez rollback, o evento não pode existir ([09:06] Diego, [09:40] Bruno).
2. **Isolamento:** uma chamada HTTP a um cliente lento ou fora do ar não pode travar a transação de status nem forçar rollback dela ([09:04] Bruno).

O sistema hoje não tem filas, eventos ou broker; a única infraestrutura persistente é o MySQL 8 do `docker-compose.yml`.

## Decisão

Adotar o **padrão Transactional Outbox no MySQL existente**:

- Criar a tabela `webhook_outbox` no mesmo banco (`prisma/schema.prisma`).
- Dentro da **mesma transação** de `changeStatus`, inserir uma linha por evento a ser entregue. Falha nessa inserção causa rollback de toda a mudança de status ([09:40] Bruno).
- A entrega HTTP é feita fora da transação, por um worker que lê a outbox ([ADR-002](ADR-002-worker-processo-separado-polling.md)).
- A tabela tem índices em `status` (pendente, processando, falhou, entregue) e `created_at` ([09:08] Diego).
- Arquivamento de linhas entregues (≈30 dias) fica **fora do escopo** desta feature ([09:08] Diego).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Disparo HTTP síncrono dentro de `changeStatus`** | Um cliente lento trava a mudança de status de outros pedidos; cliente fora do ar forçaria rollback de uma mudança de negócio válida ([09:04] Bruno, [09:06] Diego: "fora de questão"). |
| **Redis Streams / Redis Cluster** | Exige subir e operar infra nova; para um time pequeno é overengineering ([09:07] Larissa, [09:07] Diego). Também não resolve a atomicidade com a transação MySQL sem um mecanismo adicional. |

## Consequências

**Positivas**
- Atomicidade garantida pelo próprio banco: não existe "status mudou e evento sumiu" nem "evento de mudança que não aconteceu".
- Zero infraestrutura nova; mesmo Prisma, mesmo `DATABASE_URL`.
- A outbox serve como trilha consultável do que foi emitido.

**Negativas / trade-offs**
- A transação de `changeStatus`, já pesada, ganha um `INSERT` (ou N, um por endpoint interessado) a mais.
- O MySQL passa a acumular carga de fila (escrita + polling). Aceito porque o volume esperado é baixo e o arquivamento futuro limita o crescimento.
- Latência de entrega deixa de ser imediata e passa a depender do polling ([ADR-002](ADR-002-worker-processo-separado-polling.md)).
- **Trade-off explícito:** troca-se latência e uma pequena carga extra no MySQL por consistência forte e simplicidade operacional.
