# Da Reunião ao Documento — Design Docs do Sistema de Webhooks

> Enunciado original: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia)

## Sobre o desafio

Uma empresa que opera um Order Management System (Node.js + TypeScript, Express, Prisma, MySQL) decidiu, numa call de ~55 minutos, construir um **Sistema de Webhooks de Notificação de Pedidos**: avisar clientes B2B quando o status de um pedido muda, em vez de deixá-los fazendo polling. A única documentação da decisão era a transcrição (`TRANSCRICAO.md`).

A tarefa foi transformar essa transcrição, cruzada com o código real do repositório, num pacote de design docs acionável (PRD, RFC, FDD, 7 ADRs e um tracker de rastreabilidade), usando IA como ferramenta principal e garantindo que **nada** fosse inventado: cada requisito, decisão e restrição aponta para uma fala `[hh:mm] Nome` ou para um arquivo do código. O código da aplicação não foi alterado.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude (Anthropic) — modo agente com shell e leitura de arquivos** | Leitura integral da transcrição e do código, classificação das falas, redação de todos os documentos, revisão crítica |
| **Script Python de verificação** (escrito com o Claude) | Checagem automática: todo `[hh:mm] Nome` citado existe na transcrição com aquele falante, todo caminho de código citado existe no repositório, % de linhas do tracker por fonte e IDs duplicados |
| **Revisão humana** | Leitura de cada documento contra a transcrição; ajuste de prompts quando a saída veio genérica ou errada |

## Workflow adotado

Segui a ordem sugerida no enunciado, com uma etapa de inventário antes de tudo:

1. **Contextualização** — o agente leu `TRANSCRICAO.md` inteira e os arquivos que importam para a feature: `prisma/schema.prisma`, `src/modules/orders/*`, `src/shared/errors/*`, `src/middlewares/*`, `src/app.ts`, `src/routes/index.ts`, `src/server.ts`, `src/config/*`, `src/shared/logger/index.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`, `package.json`.
2. **Inventário classificado** — cada fala relevante foi classificada como *decisão fechada*, *requisito*, *descartado*, *adiado*, *em aberto* ou *detalhe técnico secundário*. Esse inventário virou o esqueleto do tracker.
3. **ADRs** (7) — as 6 decisões principais mais uma secundária (payload snapshot + filtro na inserção), que tinha alternativas reais discutidas.
4. **RFC** — consolidação em nível de arquitetura, com 8 alternativas descartadas e 7 questões em aberto, linkando os ADRs.
5. **FDD** — modelo de dados Prisma, fluxos, 7 endpoints + contrato outbound, matriz `WEBHOOK_*`, resiliência, observabilidade e a seção "Integração com o sistema existente" com 15 arquivos reais.
6. **PRD** — por último entre os grandes, consolidando em linguagem de produto.
7. **Tracker** — montado varrendo os documentos prontos.
8. **Verificação** — script de checagem + revisão item a item contra os critérios de aceite.

Regra fixa em todos os prompts: **cada afirmação precisa de `[hh:mm] Nome` ou caminho de arquivo; o que não tiver fonte é removido ou marcado como "proposta de implementação"**.

## Prompts customizados

**1. Inventário e filtragem do que entra e do que não entra**

```text
Você é tech lead revisando a transcrição TRANSCRICAO.md de uma reunião sobre webhooks
de notificação de pedidos num OMS (Node/TS, Prisma, MySQL). Leia a transcrição inteira.

Para CADA fala que contenha conteúdo técnico ou de produto, classifique em exatamente uma:
DECISAO_FECHADA | REQUISITO_FUNCIONAL | REQUISITO_NAO_FUNCIONAL | RESTRICAO |
DESCARTADO | ADIADO | EM_ABERTO | DETALHE_SECUNDARIO.

Saída: tabela com colunas [timestamp] | falante | classe | resumo em 1 linha | evidência
(trecho curto da fala).

Regras:
- "Fica pra próxima fase", "observar", "problema do futuro" = ADIADO ou EM_ABERTO, nunca requisito.
- Uma proposta só é DECISAO_FECHADA se alguém fechar ("decidido", "anotado", "concordo")
  ou se estiver no resumo final da Larissa ([09:48]).
- Se duas falas se contradizem ou são ambíguas (ex.: contagem de tentativas), aponte.
- Não complete lacunas com boas práticas de mercado. Se não está na fala, não está na tabela.
```

**2. ADR ancorado em transcrição e código**

```text
Escreva o ADR-00N no formato MADR (Status, Contexto, Decisão, Alternativas Consideradas,
Consequências) para a decisão: <decisão>.

Use APENAS:
- falas do inventário com estes timestamps: <lista>;
- arquivos reais do repositório que você leu (cite o caminho exato).

Obrigatório:
- Toda frase de Contexto e Decisão termina com a fonte entre parênteses: ([hh:mm] Nome)
  ou `caminho/do/arquivo.ts`.
- Alternativas: só as discutidas na reunião; se nenhuma, no máximo 1 "plausível", rotulada assim.
- Consequências com positivas, negativas e uma linha "Trade-off explícito".
- Não descreva endpoints, payloads nem SQL: isso é do FDD.
- Se algo da decisão não foi fechado na reunião, escreva "não decidido" e não preencha.
```

**3. Seção "Integração com o sistema existente" do FDD**

```text
Com base nos arquivos que você leu, liste cada arquivo real que o módulo de webhooks
toca ou reutiliza. Para cada um: caminho exato, o que existe hoje (função/classe pelo
nome real) e a mudança proposta, citando a fala que justifica.
Inclua efeitos colaterais que a reunião NÃO discutiu mas o código revela
(ex.: FKs novas quebrando limpezas de teste, middleware que mapearia erros de outra forma).
Proibido citar arquivo que não exista; arquivos novos vão numa lista separada "a criar".
```

**4. Revisão crítica (rodada depois de cada documento)**

```text
Aja como revisor cético. Para o documento <X>:
1. Liste toda afirmação sem fonte ou com fonte que não diz aquilo.
2. Liste qualquer item DESCARTADO/ADIADO do inventário que apareça como requisito.
3. Liste conteúdo duplicado de outro documento no nível errado
   (RFC com detalhe de FDD, PRD com detalhe técnico).
4. Aponte contradições internas (números, nomes de códigos de erro, status).
Não reescreva; só liste os problemas com linha e correção sugerida.
```

## Iterações e ajustes

Foram **4 ciclos principais** (inventário → ADRs → RFC/FDD/PRD → revisão + verificação automática). Os ajustes concretos:

1. **Contagem de tentativas ambígua.** A primeira versão do ADR-003 tratava "5 tentativas" como 5 envios no total. Na revisão, a conta dos intervalos (1m+5m+30m+2h+12h = 14h36, "quase 15 horas entre a primeira falha e a última tentativa", [09:17] Diego) só fecha com **1 envio + 5 retentativas**. Adotei essa leitura, deixei a nota de interpretação no ADR-003 e abri a questão Q-6 no RFC em vez de esconder a ambiguidade.
2. **Ordem de entrega × retry.** O texto inicial dizia "ordem garantida por pedido" sem ressalva. Cruzando [09:12] (ordem por `created_at`) com o backoff de [09:17], um evento em retry não bloqueia o seguinte do mesmo pedido, então a ordem quebra durante retentativas. Virou consequência negativa no ADR-003 e risco no FDD e no PRD.
3. **`WEBHOOK_INVALID_URL` que nunca seria emitido.** A primeira versão colocava a regra `https` no schema Zod do body. Lendo `src/middlewares/validate.middleware.ts`, toda falha de schema vira `VALIDATION_ERROR`, então o código `WEBHOOK_INVALID_URL` pedido pelo Bruno ([09:28]) seria código morto. Ajuste: o body só valida forma e o service valida URL + `https` com um schema Zod exportado, lançando `WebhookInvalidUrlError`. Continua sendo "validação no schema Zod" ([09:23] Sofia), sem alterar o middleware.
4. **Erro de "customer não encontrado".** A IA reutilizou `NotFoundError('Customer')`, que fixa o código `NOT_FOUND` em `src/shared/errors/http-errors.ts`, contrariando o prefixo `WEBHOOK_` para tudo do módulo ([09:29] Larissa). Trocado por `WEBHOOK_CUSTOMER_NOT_FOUND`.
5. **Efeito colateral nos testes existentes.** A reunião não falou disso, mas `tests/setup.ts` apaga `customers` no `beforeEach`; com as FKs novas isso quebraria a suíte. Entrou na tabela de integração do FDD.
6. **Métrica errada no PRD.** "< 10 s em todos os envios" contradizia o próprio retry de até 12 h. Corrigido para "< 10 s em 100 % das **primeiras** tentativas".
7. **Detalhes que a reunião não fixou** (path do endpoint de rotação, coluna `next_attempt_at`, formato de `X-Signature` no grace period, tamanho do batch, bloqueio de replay duplo). Em vez de apresentá-los como decididos, ficaram marcados no FDD como **(proposta de implementação)** e numa seção própria do tracker, cada um apontando para a decisão de onde deriva.
8. **Verificação automática** pegou um link de âncora quebrado no ADR-003 para as questões em aberto do RFC; corrigido.

Script usado na verificação final:

```python
import re, os, glob
T = open('TRANSCRICAO.md').read()
falas = set(re.findall(r'^\[(\d\d:\d\d)\] (\w+):', T, re.M))
docs = ['docs/PRD.md', 'docs/RFC.md', 'docs/FDD.md', 'docs/TRACKER.md', *glob.glob('docs/adrs/ADR-*.md')]
for d in docs:
    s = open(d).read()
    for t, n in re.findall(r'\[(\d\d:\d\d)\] ([A-Z][a-z]+)', s):
        assert (t, n) in falas, f'{d}: [{t}] {n} não existe na transcrição'
    for p in re.findall(r'((?:src|prisma|tests)/[\w./-]+\.(?:ts|prisma))', s):
        if not p.startswith('src/modules/webhooks/') and p != 'src/worker.ts':
            assert os.path.exists(p), f'{d}: {p} não existe'
```

Resultado final: 0 timestamps inválidos, 0 caminhos inexistentes; tracker com 185 linhas, 82 % com fonte `TRANSCRICAO` e 33 com fonte `CODIGO`.

## Como navegar a entrega

Ordem sugerida de leitura:

| # | Arquivo | Para quê |
|---|---|---|
| 1 | [`docs/PRD.md`](docs/PRD.md) | Por que e o quê: problema, escopo, fora de escopo, métricas |
| 2 | [`docs/RFC.md`](docs/RFC.md) | Proposta técnica, alternativas descartadas, questões em aberto |
| 3 | [`docs/adrs/`](docs/adrs/README.md) | Uma decisão por arquivo (ADR-001 a ADR-007) |
| 4 | [`docs/FDD.md`](docs/FDD.md) | Como construir: dados, fluxos, contratos, erros, integração com o código |
| 5 | [`docs/TRACKER.md`](docs/TRACKER.md) | Origem de cada item na transcrição ou no código |
| — | [`TRANSCRICAO.md`](TRANSCRICAO.md) | Fonte primária (inalterada) |

```
.
├── README.md
├── TRANSCRICAO.md
└── docs/
    ├── PRD.md
    ├── RFC.md
    ├── FDD.md
    ├── TRACKER.md
    └── adrs/
        ├── README.md
        ├── ADR-001-outbox-no-mysql.md
        ├── ADR-002-worker-processo-separado-polling.md
        ├── ADR-003-retry-backoff-exponencial-e-dlq.md
        ├── ADR-004-hmac-sha256-secret-por-endpoint.md
        ├── ADR-005-at-least-once-com-x-event-id.md
        ├── ADR-006-reuso-dos-padroes-existentes.md
        └── ADR-007-payload-snapshot-e-filtro-na-insercao.md
```
