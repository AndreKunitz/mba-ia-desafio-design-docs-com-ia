# RFC: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autor** | Andre Kunitz |
| **Status** | Em revisão |
| **Data** | 2026-10-25 |
| **Revisores** | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Segurança) |
| **Origem** | Reunião técnica de quinta-feira, 09:00 (`TRANSCRICAO.md`) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

## 1. Resumo executivo (TL;DR)

Propomos notificar clientes B2B por webhooks HTTP de saída sempre que o status de um pedido mudar. O evento é gravado numa tabela **outbox** do MySQL, **na mesma transação** da mudança de status. Um **worker em processo separado**, com polling a cada 2 s, entrega o evento assinado com **HMAC-SHA256** (secret por endpoint). Falhas seguem **5 reenvios com backoff (1m/5m/30m/2h/12h)** e depois vão para uma **DLQ** com replay manual por ADMIN. A garantia é **at-least-once**, com `X-Event-Id` para deduplicação. Tudo reaproveita os padrões do código atual. Prazo estimado: 3 sprints, incluindo a revisão de segurança.

## 2. Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo pediram para ser avisados quando o status de seus pedidos muda. Hoje consultam `GET /orders` periodicamente, o que deixa a integração lenta e cara; a Atlas indicou que pode migrar para um concorrente se a entrega não ocorrer até o fim do trimestre (`[09:00] Marcos`). Para eles, "tempo real" significa menos de 10 segundos (`[09:02] Marcos`). A comunicação é só de saída: a plataforma envia, os clientes não enviam webhooks para ela (`[09:02] Marcos`).

O sistema atual não possui eventos, filas nem webhooks. A mudança de status ocorre em `OrderService.changeStatus` ([src/modules/orders/order.service.ts](../src/modules/orders/order.service.ts)), numa única transação que atualiza o pedido, grava o histórico e movimenta estoque. Essa transação não pode depender da disponibilidade de terceiros (`[09:04] Bruno`).

## 3. Proposta técnica

Visão geral (detalhes de implementação no [FDD](FDD.md)):

```
changeStatus ──(mesma transação)──▶ orders + order_status_history + webhook_outbox
                                                    │
                          worker (src/worker.ts, polling 2s)
                                                    │  POST assinado (HMAC-SHA256)
                                                    ▼
                                          endpoint HTTPS do cliente
                          falha ──▶ backoff 1m/5m/30m/2h/12h ──▶ webhook_dead_letter ──▶ replay (ADMIN)
```

Pontos principais:

1. **Outbox transacional** ([ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-007](adrs/ADR-007-publicacao-transacional-e-payload-snapshot.md)): `publishWebhookEvent(tx, order, from, to)` insere o evento já renderizado (snapshot), filtrando por status assinado.
2. **Worker separado em polling de 2 s** ([ADR-005](adrs/ADR-005-worker-separado-em-polling.md)): novo entry-point `src/worker.ts`, mesmo banco, `PrismaClient` próprio. Um único worker.
3. **Retry e DLQ** ([ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md)): timeout de 10 s por chamada; esgotados os reenvios, o evento vai para `webhook_dead_letter`, com replay manual via `POST /admin/webhooks/dead-letter/:id/replay` (role ADMIN).
4. **Segurança** ([ADR-003](adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md)): HMAC-SHA256 sobre o corpo, secret única por endpoint, rotação com 24 h de grace period, URL somente HTTPS.
5. **Semântica de entrega** ([ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md), [ADR-008](adrs/ADR-008-contrato-de-entrega-payload-headers-e-limites.md)): at-least-once, `X-Event-Id` para dedup, payload enxuto de até 64 KB.
6. **API de configuração**: CRUD de webhooks por customer com filtro de status, histórico de entregas por webhook, rotação de secret e replay de DLQ.
7. **Reuso do projeto** ([ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)): módulo `src/modules/webhooks`, `AppError`, Pino, error middleware, `requireRole`, schemas Zod e erros `WEBHOOK_*`.

## 4. Alternativas consideradas

| Alternativa | Por que foi descartada (trade-off) | Fonte |
| --- | --- | --- |
| **HTTP síncrono dentro de `changeStatus`** | Simples, mas um cliente lento trava mudanças de status de outros pedidos, e cliente offline força escolher entre rollback ou perda do evento. | `[09:04] Bruno`, `[09:06] Diego` |
| **Redis Streams / fila externa** | Bom desacoplamento, mas exige nova infra para um time pequeno ("overengineering"). Outbox no MySQL existente basta. | `[09:07] Larissa`, `[09:07] Diego` |
| **Trigger / listener no banco para acordar o worker** | Seria mais reativo, mas o MySQL não tem `LISTEN/NOTIFY` e a trigger só executa SQL; polling de 2 s já atende a meta de 10 s. | `[09:09] Bruno`, `[09:09] Diego` |
| **Retry indefinido** | Cobre qualquer indisponibilidade, mas deixa o evento pendurado para sempre se o cliente sumir. | `[09:15] Diego` |
| **3 tentativas** | Mais agressivo, mas cobriria só ~30 min e mataria eventos em manutenção planejada de 2 h. | `[09:16] Bruno`, `[09:16] Diego` |
| **Exactly-once** | Ofereceria menos duplicatas, mas exige coordenação nos dois lados e é bem mais complexo; at-least-once com `event_id` é o padrão de mercado. | `[09:25] Diego` |
| **Falha permanente como `failed` na própria outbox** | Evita uma tabela, mas polui a leitura da outbox e perde a evidência para reprocessamento. | `[09:18] Diego` |

## 5. Questões em aberto

Itens levantados na reunião e **não decididos ou adiados**, mais pontos que a análise do código e da transcrição deixou sem resposta. O primeiro bloco vem da reunião; o segundo reúne perguntas desta revisão.

**Da reunião**

1. **Rate limiting de saída.** Se um cliente tiver 50 pedidos mudando em um minuto, hoje receberá 50 chamadas. Decisão: "observar e decidir depois" (`[09:38] Diego`, `[09:39] Larissa`).
2. **Alerta ao cliente por email** quando o webhook falha repetidamente. Fora desta fase; reavaliar após medir o impacto (`[09:37] Marcos`, `[09:37] Larissa`).
3. **Escala do worker e ordering.** Com mais de um worker a ordem por `order_id` se perde; opções citadas: particionar por `order_id` ou lock pessimista. "Problema do futuro" (`[09:12] Diego`, `[09:13] Diego`).
4. **Arquivamento da outbox** (~30 dias) fora do escopo desta feature (`[09:08] Diego`).
5. **Endurecer permissões do CRUD** de configuração, hoje aberto a qualquer role autenticada (`[09:37] Sofia`).

**Para esta revisão (lacunas não cobertas na reunião)**

6. **Contagem de tentativas.** A reunião diz "5 tentativas" e lista 5 intervalos. Propomos 1 envio + 5 reenvios (14 h 36 min no total). Confirmar (`[09:15] Diego`, `[09:17] Diego`).
7. **Autorização por customer.** O `customer_id` vem no body/path e não do JWT, e qualquer role autenticada pode cadastrar (`[09:32] Larissa`, `[09:37] Sofia`). Um usuário poderia configurar webhook de outro customer. Precisa de uma regra de vínculo usuário-customer?
8. **Secret antiga em envio outbound.** Como "válida por 24 h" se traduz na assinatura enviada (assinar com as duas?). Proposta no FDD, para validação de Sofia (`[09:21] Sofia`).
9. **Armazenamento da secret** e **assinatura sem o `X-Timestamp`**, que limita a proteção contra replay. Para a revisão de segurança (`[09:46] Sofia`).
10. **Evento na criação do pedido** (`PENDING`): a reunião só trata mudanças via `changeStatus`; criação ficou fora do desenho.
11. **Payload > 64 KB.** Sofia pede "erro" (`[09:23] Sofia`), mas lançar erro dentro de `changeStatus` bloquearia a mudança de status. O FDD propõe enviar direto à DLQ sem abortar a transação. Confirmar.

## 6. Impacto e riscos

**Impacto**

- `OrderService.changeStatus` ganha uma chamada dentro da transação existente; `routes/index.ts` e `app.ts` registram o novo módulo; novo entry-point `src/worker.ts` e script `npm run worker`; novas tabelas via migration Prisma.
- Operação: novo processo a implantar e monitorar.
- Clientes: passam a deduplicar por `X-Event-Id` e a validar a assinatura (documentação a cargo de Marcos, `[09:26] Marcos`).

**Riscos**

| Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- |
| Falha na outbox bloqueia mudança de status | Baixa | Alto | Insert simples e indexado; testes de integração da transação. |
| Worker único parado atrasa todas as entregas | Média | Alto | Supervisão do processo e log/alerta de backlog (proposta no FDD). |
| Vazamento de secret de cliente | Média | Alto | Secret por endpoint, rotação com grace period, redação em logs. |
| Cliente processa duplicata | Alta | Médio | `X-Event-Id` e documentação em destaque no portal. |
| Prazo de 3 sprints comprimido pela revisão de segurança | Média | Médio | Reservar 2 dias úteis de Sofia no fim do cronograma (`[09:46] Sofia`). |

## 7. Decisões relacionadas

[ADR-001 Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md) ·
[ADR-002 Retry, backoff e DLQ](adrs/ADR-002-retry-backoff-e-dlq.md) ·
[ADR-003 HMAC-SHA256 por endpoint](adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md) ·
[ADR-004 At-least-once e X-Event-Id](adrs/ADR-004-at-least-once-com-x-event-id.md) ·
[ADR-005 Worker separado em polling](adrs/ADR-005-worker-separado-em-polling.md) ·
[ADR-006 Reuso dos padrões](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) ·
[ADR-007 Publicação transacional e snapshot](adrs/ADR-007-publicacao-transacional-e-payload-snapshot.md) ·
[ADR-008 Contrato de entrega](adrs/ADR-008-contrato-de-entrega-payload-headers-e-limites.md)
