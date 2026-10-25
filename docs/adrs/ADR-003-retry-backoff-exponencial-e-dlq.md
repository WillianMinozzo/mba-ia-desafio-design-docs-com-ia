# ADR-003 — Retry com backoff exponencial (5 retentativas) e DLQ em tabela separada

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa, Diego, Bruno, Marcos, Sofia
- **Relacionados:** [ADR-002](ADR-002-worker-processo-separado-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Contexto

Endpoints de clientes ficam indisponíveis. Já houve cliente com duas horas de indisponibilidade em manutenção planejada ([09:16] Diego). É preciso decidir quantas vezes tentar, com que intervalo, e o que fazer com um evento que nunca foi entregue.

## Decisão

1. **Backoff exponencial com teto de 5 retentativas**, intervalos fixos: **1 min → 5 min → 30 min → 2 h → 12 h** ([09:17] Diego, [09:17] Larissa: "decidido").
2. **Falha de uma tentativa** = timeout de 10 s ([09:42] Diego), erro de rede ou resposta não-2xx.
3. Esgotadas as retentativas, o evento é **movido para a tabela `webhook_dead_letter`**, com payload, motivo da falha e timestamp ([09:18] Diego). A linha sai da `webhook_outbox`, mantendo a leitura da outbox principal limpa.
4. **Reprocessamento manual** via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego).
5. O replay **exige role `ADMIN`** e registra quem executou, para auditoria ([09:36] Sofia, [09:36] Larissa).

### Nota de interpretação (registrada na revisão do documento)

Na reunião aparecem "5 tentativas" ([09:15] Diego, [09:48] Larissa) e cinco intervalos, com "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego). A soma 1+5+30+120+720 min = **14h36**, que só fecha se os cinco intervalos vierem **depois** da tentativa inicial. Este ADR adota: **1 tentativa inicial + 5 retentativas = 6 envios no máximo**. Se o time entender "5 tentativas no total", basta remover o último degrau (12 h); a estrutura não muda. Ponto marcado para confirmação no [RFC](../RFC.md#5-questões-em-aberto).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Retry indefinido com backoff** | Evento fica pendurado para sempre se o cliente sumiu ([09:15] Diego). |
| **3 tentativas (mais agressivo)** | Cobriria ~30 min; uma indisponibilidade matinal ou manutenção de 2 h perderia eventos ([09:16] Bruno propõe, [09:16] Diego descarta). |
| **Marcar como `failed` na própria outbox em vez de DLQ separada** | Polui a leitura da outbox principal e mistura eventos vivos com evidência de debug ([09:17] Larissa coloca a opção, [09:18] Diego argumenta pela tabela separada). |
| **Reprocessamento automático da DLQ** | Não discutido como opção; a reunião fechou replay **manual** e restrito a ADMIN ([09:18], [09:36]). |

## Consequências

**Positivas**
- Janela de ~15 h cobre manutenções planejadas e quedas longas ([09:17] Marcos: "aceitável").
- DLQ explícita serve de evidência para debug e de fila para replay controlado.
- Replay auditado e restrito.

**Negativas / trade-offs**
- Evento em retry não bloqueia eventos posteriores do mesmo pedido: durante uma janela de retry o cliente pode receber `SHIPPED` antes de um `PROCESSING` que falhou. A garantia de ordem do [ADR-002](ADR-002-worker-processo-separado-polling.md) vale para a primeira tentativa, não atravessa retries (ver Riscos no FDD).
- Replay é manual: alguém precisa observar a DLQ. Notificação por e-mail ao cliente foi adiada para fase futura ([09:37] Larissa).
- Um evento entregue com atraso de até ~15 h pode chegar com dado já obsoleto; mitigado pelo snapshot + consulta a `GET /orders/:id` ([ADR-007](ADR-007-payload-snapshot-e-filtro-na-insercao.md)).
- **Trade-off explícito:** cobertura de indisponibilidades longas em troca de latência alta para eventos que falham e de esforço operacional no replay.
