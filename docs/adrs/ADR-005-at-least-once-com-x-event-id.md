# ADR-005 — Garantia de entrega at-least-once com `X-Event-Id` para deduplicação

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Diego, Larissa, Sofia, Marcos, Bruno
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)

## Contexto

Com outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) e retry ([ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)), existem cenários em que o OMS não sabe se o cliente processou um envio: timeout depois de o cliente já ter recebido, queda do worker entre o POST e a marcação como entregue, replay manual da DLQ. É preciso escolher a semântica de entrega e como o cliente lida com duplicatas.

## Decisão

1. **Semântica at-least-once:** o cliente pode receber o mesmo evento mais de uma vez e deve estar preparado ([09:24] Diego).
2. Cada evento carrega um **UUID gerado no momento em que entra na outbox**, enviado no header **`X-Event-Id`** e no campo `event_id` do payload ([09:25] Diego, [09:43] Diego).
3. O `event_id` é **estável** entre retentativas e no replay da DLQ, para que a deduplicação do lado do cliente funcione.
4. O PM documenta essa responsabilidade com destaque no portal de desenvolvedor ([09:26] Marcos).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Exactly-once** | Exigiria coordenação dos dois lados (protocolo de confirmação, estado compartilhado); muito mais complexo ([09:25] Diego). |
| **At-most-once** (plausível) | Incompatível com a decisão de retry: perderia eventos sempre que o cliente estivesse fora do ar ([09:14]–[09:17]). |

## Consequências

**Positivas**
- Mesmo padrão de Stripe e GitHub ([09:25] Diego); clientes B2B conhecem o modelo.
- Permite retry e replay agressivos sem risco de perda.
- O OMS não precisa de protocolo de confirmação.

**Negativas / trade-offs**
- Responsabilidade de dedup recai sobre o cliente ([09:25] Sofia). Cliente que ignorar o `X-Event-Id` pode processar duas vezes.
- `X-Event-Id` não protege contra replay attack de terceiros; para isso existe o `X-Timestamp` opcional para o cliente verificar ([09:44] Diego).
- **Trade-off explícito:** simplicidade do lado do OMS em troca de exigir idempotência do consumidor.
