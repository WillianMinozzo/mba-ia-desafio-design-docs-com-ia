# ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period de 24 h

- **Status:** Aceito (sujeito à revisão de segurança da Sofia antes do deploy)
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Sofia (Segurança), Larissa, Bruno, Diego
- **Relacionados:** [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Contexto

Os webhooks são **outbound** apenas: saem do OMS para os clientes; não há recebimento ([09:02] Marcos, [09:03] Sofia). Eles carregam dados de pedidos para fora da infraestrutura. O cliente precisa validar que (a) a requisição veio do OMS e (b) o payload não foi adulterado ([09:19] Sofia). Já houve cliente que vazou secret em log de aplicação ([09:22] Diego).

## Decisão

1. **Assinatura HMAC-SHA256 sobre o corpo bruto do request**, enviada no header `X-Signature` ([09:20] Sofia, [09:22] Sofia).
2. **Secret única por endpoint de webhook**, nunca global ([09:21] Sofia). A secret é **gerada pelo OMS** e devolvida na criação do endpoint ([09:31] Marcos).
3. A configuração armazena `url + secret + customer_id + active` ([09:21] Bruno, confirmado por Sofia).
4. **Rotação via API:** o cliente pede nova secret; a antiga continua válida por **24 h** em paralelo e depois é descartada ([09:21] Sofia, [09:22] Sofia).
5. **TLS obrigatório:** URL precisa ser `https`; `http` é recusado por validação Zod no schema, não é decisão arquitetural ([09:23] Sofia).
6. Código de segurança (HMAC e geração de secret) passa por **revisão da Sofia com pelo menos 2 dias úteis** antes do deploy ([09:46] Sofia).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Secret global da plataforma** | Vazou uma, vaza tudo: compromete todos os clientes de uma vez ([09:21] Sofia). |
| **Rotação com corte imediato da secret antiga** | Cliente não teria tempo de migrar seus sistemas; motivou o grace period de 24 h ([09:21] Sofia). |
| **Sem assinatura, confiando apenas em TLS** (plausível) | TLS protege o canal, mas não prova ao cliente a origem do request numa URL pública; a reunião exigiu prova de origem e integridade ([09:19] Sofia). |

## Consequências

**Positivas**
- Padrão de mercado; qualquer cliente tem biblioteca para HMAC-SHA256 ([09:20] Sofia).
- Vazamento de uma secret afeta um único endpoint.
- Rotação sem downtime para o cliente.

**Negativas / trade-offs**
- O OMS precisa guardar a secret de forma recuperável (HMAC exige o valor original; hash não serve). Criptografia em repouso não foi discutida: fica para a revisão de segurança.
- Durante as 24 h de grace period o worker precisa considerar duas secrets válidas. O mecanismo exato de envio (ver FDD) é proposta pendente de validação da Sofia.
- A secret passa a ser dado sensível a ser escondido dos logs (`redactPaths` em `src/shared/logger/index.ts`).
- **Trade-off explícito:** mais estado e mais lógica (duas secrets, expiração) em troca de isolamento de vazamentos e rotação sem quebra.
