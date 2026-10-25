# ADR-002 — Worker em processo separado, com polling de 2 segundos

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa, Diego, Bruno, Marcos
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)

## Contexto

Com a outbox ([ADR-001](ADR-001-outbox-no-mysql.md)), alguém precisa ler os eventos pendentes e entregá-los. Restrições levantadas:

- O requisito de latência dos clientes é **abaixo de 10 segundos** ([09:02] Marcos).
- O MySQL não tem mecanismo nativo de notificação a processo externo como o `NOTIFY/LISTEN` do Postgres; trigger só executa SQL ([09:09] Diego).
- Se o worker rodar dentro da API, um restart da API derruba o worker ([09:11] Diego).
- O projeto já tem um entry point HTTP em `src/server.ts` e um factory de Prisma em `src/config/database.ts`.

## Decisão

1. **Processo separado:** novo entry point `src/worker.ts`, executado por um script `npm run worker`, independente de `src/server.ts` ([09:11] Larissa, [09:11] Diego).
2. **Mesma stack e mesmo banco:** o worker usa o mesmo `DATABASE_URL`, mas cria **sua própria instância** de `PrismaClient`, porque o client é por processo ([09:11] Bruno, [09:30] Bruno).
3. **Polling em loop a cada 2 segundos**, buscando os eventos pendentes mais antigos em **batch pequeno** ([09:08] Diego, [09:09] Diego).
4. **Single-worker:** uma única instância em execução. Ordenação por `created_at`, o que dá ordem por `order_id` enquanto houver um único worker ([09:12] Diego, [09:13] Larissa).
5. A lógica de processamento vive no módulo, em `src/modules/webhooks/webhook.processor.ts`; `src/worker.ts` só faz bootstrap e shutdown ([09:28] Bruno).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Trigger de banco para ser reativo** | MySQL não notifica processo externo; seria preciso improvisar (escrever arquivo, chamar endpoint) ([09:09] Bruno pergunta, [09:09] Diego descarta). |
| **Worker dentro do processo da API** | Restart da API mata o worker; acopla ciclo de vida de entrega HTTP ao servidor ([09:11] Diego). |
| **Múltiplos workers paralelos** | Perde a garantia de ordem por pedido; exigiria particionar por `order_id` ou lock pessimista. Adiado ("problema do futuro") ([09:13] Diego). |

## Consequências

**Positivas**
- Latência máxima de detecção de ~2s, bem abaixo do teto de 10s acordado ([09:10] Larissa: "aceitamos").
- Restart/deploy da API não interrompe entregas.
- Nenhuma dependência nova: mesmo Prisma, mesmo logger.

**Negativas / trade-offs**
- Polling gera consultas constantes no MySQL mesmo sem eventos.
- Latência mínima de até 2s no pior caso (aceita na reunião).
- **Limitação conhecida:** não há garantia de ordem global, apenas por `order_id` e apenas enquanto for single-worker ([09:13] Larissa). Os clientes não pediram ordem global ([09:14] Marcos).
- Mais um processo para operar e monitorar em produção.
- **Trade-off explícito:** simplicidade e zero infra nova em troca de latência de até 2s e escalabilidade horizontal limitada.
