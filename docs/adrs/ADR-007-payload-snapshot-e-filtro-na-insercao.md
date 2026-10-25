# ADR-007 — Payload renderizado (snapshot) e filtro de eventos na inserção da outbox

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão fechada após a saída de Marcos e Sofia)
- **Decisores:** Larissa, Diego, Bruno
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)

## Contexto

Duas perguntas de modelagem da outbox:

1. **O que a linha guarda?** Só `order_id` (e renderiza o payload no envio) ou o payload completo já renderizado ([09:51] Bruno)?
2. **Onde filtrar por status de interesse?** Cada endpoint escolhe os status que quer receber ([09:33] Marcos). Filtrar na inserção ou no envio ([09:34] Diego)?

Eventos podem ficar até ~15 h em retry ([ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)); nesse intervalo o pedido pode mudar de novo.

## Decisão

1. **Snapshot na inserção:** o payload JSON é montado dentro da transação de `changeStatus` e gravado na outbox. O evento reflete o estado do pedido **no momento da mudança** ([09:52] Larissa, Diego, Bruno: "decidido").
2. **Payload enxuto**: `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido como `total_cents`. **Sem itens**; o cliente consulta `GET /orders/:id` se precisar ([09:43] Diego, [09:44] Bruno).
3. **Filtro na inserção:** só se insere linha para endpoints ativos do customer que escutam o `to_status`. Se nenhum quer, nada é inserido ([09:34] Bruno, [09:34] Diego: "concordo").

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Guardar só `order_id` e renderizar no envio** | Um retry horas depois mandaria o estado atual com o `from/to` antigo, gerando evento incoerente ([09:52] Larissa: "caso esquisito"). |
| **Filtrar na hora do envio** | Grava linhas que nunca serão enviadas; desperdiça espaço e ciclos do worker ([09:34] Bruno). |
| **Payload com itens do pedido** | Infla o evento; o cliente pode buscar os detalhes sob demanda ([09:43] Diego). |

## Consequências

**Positivas**
- Evento imutável e auditável; retry e replay enviam exatamente o mesmo corpo, logo a mesma assinatura HMAC e o mesmo `event_id`.
- Outbox contém só trabalho real.
- Payload pequeno, muito abaixo do teto de 64 KB ([09:24] Diego).

**Negativas / trade-offs**
- A transação de `changeStatus` passa a consultar `webhook_endpoints` do customer e a serializar JSON.
- Alterar o filtro de eventos de um endpoint não afeta eventos já enfileirados.
- Cliente que precise de itens faz uma chamada extra a `GET /orders/:id`.
- **Trade-off explícito:** um pouco mais de trabalho dentro da transação em troca de eventos coerentes e outbox enxuta.
