# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Responsável de produto** | Marcos (PM) |
| **Tech Lead** | Larissa |
| **Status** | Aprovado na reunião técnica; em detalhamento |
| **Prazo alvo** | Fim de novembro (pedido da Atlas) — estimativa de 3 sprints ([09:45]–[09:47]) |
| **Documentos** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

---

## 1. Resumo e contexto

O OMS passa a **avisar ativamente** os clientes B2B quando o status de um pedido deles muda, por meio de **webhooks HTTP** enviados para uma URL que o próprio cliente cadastra. Hoje o OMS não tem nenhum mecanismo de notificação externa; a única forma de um cliente saber de uma mudança é consultar a API repetidamente.

## 2. Problema e motivação

- Atlas Comercial, MaxDistribuição e Nova Cargo pediram formalmente notificação em tempo real ([09:00] Marcos).
- Hoje eles fazem polling em `GET /orders`, o que deixa a integração **lenta e cara** para eles ([09:00] Marcos).
- **Risco comercial:** a Atlas indicou que pode migrar para um concorrente se a solução não sair até o fim do trimestre ([09:00] Marcos).

## 3. Público-alvo e cenários de uso

| Persona | Necessidade |
|---|---|
| **Integrador do cliente B2B** (Atlas, MaxDistribuição, Nova Cargo) | Receber mudanças de status sem polling, validar a autenticidade, não processar evento duplicado |
| **Usuário do OMS que representa o cliente** | Cadastrar e manter endpoints via API autenticada do OMS ([09:32] Marcos) |
| **Administrador do OMS** | Reprocessar eventos que falharam definitivamente, com rastro de auditoria ([09:36] Sofia) |
| **Engenharia/operação** | Acompanhar entregas e falhas para diagnosticar problemas |

Cenários:
1. **Pedido despachado:** o pedido da Atlas vai de `PROCESSING` para `SHIPPED`; em segundos o sistema da Atlas recebe o evento e atualiza o rastreio ([09:12] Larissa).
2. **Assinar só o que interessa:** a MaxDistribuição quer saber apenas de `SHIPPED` e `DELIVERED` e não recebe os demais ([09:33] Marcos).
3. **Cliente em manutenção:** o endpoint da Nova Cargo fica 2 h fora do ar; os eventos são reenviados automaticamente quando ele volta ([09:16] Diego).
4. **Secret vazada:** o cliente rotaciona a secret pela API e tem 24 h para atualizar seus sistemas ([09:21] Sofia).
5. **Diagnóstico:** o cliente consulta as últimas 100 entregas para entender por que não recebeu algo ([09:34] Marcos).
6. **Falha permanente:** após esgotar as retentativas, um ADMIN reenvia o evento manualmente ([09:18] Diego).

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta |
|---|---|---|---|
| O-1 | Notificação percebida como "tempo real" | Latência entre mudança de status e primeira tentativa de entrega | **< 10 s** em 100 % das primeiras tentativas; ≤ 2 s de espera de polling no pior caso ([09:02] Marcos, [09:10] Larissa) |
| O-2 | Nenhuma mudança de status sem notificação | % de mudanças de status (para endpoints interessados) com evento registrado | **100 %** ([09:40] Bruno) |
| O-3 | Resiliência a indisponibilidade do cliente | Janela coberta por retry antes de falha permanente | **≈ 15 h** (1m/5m/30m/2h/12h) ([09:17] Diego) |
| O-4 | Entregar aos clientes que pediram | Clientes em produção recebendo webhooks | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) até o fim de novembro ([09:00], [09:45]) |

## 5. Escopo

### 5.1 Incluso
- Cadastro, listagem, edição e remoção de endpoints de webhook por customer ([09:31]–[09:33]).
- Escolha, por endpoint, dos status de pedido que geram notificação ([09:33]).
- Notificação a cada mudança de status de pedido ([09:40]).
- Assinatura de cada envio e rotação de secret ([09:20]–[09:22]).
- Reenvio automático em caso de falha e fila de falhas permanentes ([09:15]–[09:18]).
- Histórico de entregas por endpoint ([09:34]).
- Reprocessamento manual de falhas permanentes por administradores ([09:35]–[09:36]).

### 5.2 Fora de escopo

| Item | Situação | Fonte |
|---|---|---|
| **E-mail ao cliente quando o webhook falha repetidamente** | **Adiado** para a próxima fase, após medir impacto | [09:37] Larissa |
| **Dashboard visual / painel para o cliente** | **Descartado** desta fase; é projeto do time de frontend | [09:40] Larissa |
| **Rate limiting de envio por cliente** | **Adiado**: observar e decidir depois | [09:39] Diego, [09:39] Larissa |
| Webhooks de entrada (cliente → OMS) | Descartado: só saída | [09:02] Marcos |
| Garantia de ordem global entre pedidos | Não requerido pelos clientes | [09:14] Marcos |
| Arquivamento de eventos entregues | Fora desta feature | [09:08] Diego |
| Restrição de quais usuários gerenciam webhooks de qual cliente | Adiado ("mais pra frente endurecer") | [09:37] Sofia |

## 6. Requisitos funcionais

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | Cadastrar endpoint de webhook informando cliente, URL e status de interesse | [09:31] Marcos |
| RF-02 | A secret de assinatura é gerada pelo OMS e devolvida na criação | [09:31] Marcos |
| RF-03 | Listar os endpoints de um cliente | [09:33] Bruno |
| RF-04 | Editar um endpoint (URL, status de interesse, ativo/inativo) | [09:33] Bruno |
| RF-05 | Remover um endpoint | [09:33] Bruno |
| RF-06 | Notificar apenas os status escolhidos pelo endpoint | [09:33] Marcos, [09:34] Bruno |
| RF-07 | Enviar notificação a cada mudança de status de pedido, com dados básicos do pedido (sem itens) | [09:40] Bruno, [09:43] Diego |
| RF-08 | Assinar cada envio para que o cliente valide origem e integridade | [09:19]–[09:20] Sofia |
| RF-09 | Permitir ao cliente rotacionar a secret, com a antiga válida por 24 h | [09:21] Sofia |
| RF-10 | Reenviar automaticamente envios que falharam, em intervalos crescentes | [09:15], [09:17] Diego |
| RF-11 | Guardar em fila separada os envios que esgotaram as tentativas | [09:18] Diego |
| RF-12 | Administradores podem reprocessar manualmente um envio da fila de falhas, com registro de quem fez | [09:18] Diego, [09:36] Sofia |
| RF-13 | Consultar as últimas 100 entregas de um endpoint com sucesso/falha, payload, resposta e tempo de resposta | [09:34] Marcos |
| RF-14 | Cada evento carrega um identificador único para deduplicação | [09:25] Diego |
| RF-15 | Cada envio identifica o endpoint de destino, para clientes com vários cadastros | [09:44] Sofia |

## 7. Requisitos não funcionais

| ID | Requisito | Fonte |
|---|---|---|
| RNF-01 | Latência de notificação < 10 s | [09:02] Marcos |
| RNF-02 | A mudança de status nunca pode ser bloqueada ou desfeita por lentidão/falha do cliente | [09:04] Bruno |
| RNF-03 | Atomicidade: status mudou ⇔ evento registrado | [09:06] Diego, [09:40] Bruno |
| RNF-04 | Entrega at-least-once; o cliente deduplica | [09:24]–[09:26] |
| RNF-05 | URL obrigatoriamente `https` | [09:23] Sofia |
| RNF-06 | Payload máximo de 64 KB; acima disso o envio falha (não trunca) | [09:23] Sofia, [09:24] Diego/Larissa |
| RNF-07 | Timeout de 10 s por envio | [09:42] Diego |
| RNF-08 | Secret exclusiva por endpoint | [09:21] Sofia |
| RNF-09 | Sem infraestrutura nova (mesmo MySQL, mesma stack) | [09:07] Diego, [09:11] Diego |
| RNF-10 | Entrega roda isolada da API: reinício da API não interrompe entregas | [09:11] Diego |
| RNF-11 | Ordem de eventos garantida por pedido enquanto houver um único processo de entrega | [09:12]–[09:13] |
| RNF-12 | Revisão de segurança de pelo menos 2 dias úteis antes do deploy | [09:46] Sofia |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | ADR |
|---|---|---|
| Registrar o evento numa outbox no MySQL, na transação do pedido | Até 2 s de atraso e carga extra no banco, em troca de nunca perder evento e não subir infra | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Processo de entrega separado, consultando a cada 2 s | Um processo a mais para operar; não escala horizontalmente sem rever a ordem | [ADR-002](adrs/ADR-002-worker-processo-separado-polling.md) |
| 5 reenvios em ~15 h, depois fila de falhas com replay manual | Eventos que falham chegam muito atrasados; replay depende de um admin | [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md) |
| HMAC-SHA256, secret por endpoint, rotação com 24 h | Mais estado a gerenciar; cliente precisa implementar verificação | [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| At-least-once com identificador de evento | Cliente precisa deduplicar | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Seguir os padrões atuais do código | Código próprio para retry/fila em vez de biblioteca | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| Evento congelado no momento da mudança; só para status assinados | Mudar o filtro não afeta eventos já gerados | [ADR-007](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md) |

## 9. Dependências

| Dependência | Tipo | Fonte |
|---|---|---|
| Fluxo de mudança de status de pedidos (`src/modules/orders/order.service.ts`) | Técnica, interna | [09:40] Bruno |
| Banco MySQL e Prisma existentes | Técnica, interna | [09:07] Diego |
| Autenticação JWT e perfil ADMIN existentes (`src/middlewares/auth.middleware.ts`) | Técnica, interna | [09:36] Larissa |
| Revisão de segurança da Sofia antes do deploy | Processo | [09:46] Sofia |
| Documentação no portal do desenvolvedor (at-least-once, assinatura, integração via API) | Produto | [09:26], [09:40] Marcos |
| Clientes preparados para HTTPS, verificação HMAC e deduplicação | Externa | [09:23], [09:25] |
| Confirmação de prazo com a Atlas | Comercial | [09:47] Marcos |

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Atraso na entrega e perda da Atlas para o concorrente | Média | Alto | Estimativa de 3 sprints já com revisão de segurança; escopo enxuto (sem e-mail, painel ou rate limit) ([09:00], [09:46]) |
| Cliente processa evento duplicado por não deduplicar | Média | Médio | Identificador único por evento + destaque no portal do desenvolvedor ([09:26] Marcos) |
| Secret vazada por erro do cliente | Média | Alto | Secret por endpoint, rotação com 24 h de transição ([09:21]–[09:22]) |
| Processo de entrega parado sem ninguém notar | Baixa | Alto | Eventos ficam guardados; métrica de backlog e de idade do evento mais antigo ([FDD §9](FDD.md#9-observabilidade)) |
| Cliente recebe muitas notificações de uma vez | Média | Médio | Monitorar volume por cliente e decidir sobre rate limiting depois ([09:39]) |
| Cliente recebe eventos fora de ordem durante reenvios | Média | Baixo | Evento traz status anterior e novo; clientes não exigiram ordem global ([09:14] Marcos) |

## 11. Critérios de aceitação

- [ ] Um usuário autenticado cadastra um endpoint `https` para um cliente e recebe a secret na resposta.
- [ ] Cadastro com URL `http` é recusado.
- [ ] Mudar o status de um pedido para um status assinado gera uma notificação recebida pelo cliente em menos de 10 s.
- [ ] Mudar para um status não assinado não gera notificação.
- [ ] A notificação traz identificador do evento, identificador do endpoint, horário e assinatura verificável com a secret.
- [ ] Endpoint fora do ar recebe reenvios automáticos e, após esgotá-los, o evento aparece na fila de falhas.
- [ ] Apenas ADMIN consegue reprocessar um evento da fila de falhas, e a ação fica registrada com o usuário.
- [ ] O cliente consulta as últimas 100 entregas com status, payload, resposta e tempo.
- [ ] Após rotacionar a secret, a antiga continua aceita por 24 h.
- [ ] Nenhuma mudança de status de pedido fica mais lenta de forma perceptível nem falha por causa de um cliente fora do ar.

## 12. Estratégia de testes e validação

| Nível | O que valida | Como |
|---|---|---|
| Unitário | Montagem do payload, assinatura HMAC, cálculo do backoff, regra de `https`, limite de 64 KB | Vitest, funções puras do módulo |
| Integração (API) | CRUD de endpoints, rotação, histórico, replay com e sem ADMIN | Vitest + Supertest sobre `buildApp`, como em `tests/orders.test.ts` |
| Integração (transação) | Evento gravado junto com a mudança de status e desfeito no rollback | Teste sobre `PATCH /orders/:id/status` com falha forçada |
| Worker | Entrega, timeout, retry, DLQ, recuperação após crash | Servidor HTTP local de teste simulando 2xx, 500 e lentidão; relógio controlado |
| Ponta a ponta | Fluxo completo pedido → cliente | Ambiente local com `docker-compose.yml`, API e `npm run worker` |
| Segurança | HMAC, geração e rotação de secret, ausência de secret em logs | Revisão da Sofia, ≥ 2 dias úteis ([09:46]) |
| Validação com clientes | Integração real dos três clientes | Acompanhamento do PM após o deploy ([09:47] Marcos) |
