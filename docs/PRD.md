# PRD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autor** | Andre Kunitz |
| **Status** | Para revisão |
| **Data** | 2026-10-25 |
| **Stakeholders** | Marcos (PM), Larissa (Tech Lead), Bruno, Diego, Sofia |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

## 1. Resumo e contexto da feature

O Order Management System passará a **notificar clientes B2B por webhook** sempre que o status de um pedido mudar, em menos de 10 segundos. A comunicação é apenas de saída: a plataforma envia, o cliente recebe (`[09:02] Marcos`). A feature é pedida formalmente por três clientes: Atlas Comercial, MaxDistribuição e Nova Cargo (`[09:00] Marcos`). Hoje o sistema não possui nenhum mecanismo de notificação externa; o ciclo de vida do pedido tem os status `PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED` e `CANCELLED` ([src/modules/orders/order.status.ts](../src/modules/orders/order.status.ts)).

## 2. Problema e motivação

- Os clientes consultam `GET /orders` periodicamente para descobrir mudanças, o que torna a integração **lenta e cara** para eles (`[09:00] Marcos`).
- A **Atlas sinalizou que pode migrar para o concorrente** se a funcionalidade não for entregue até o fim do trimestre; o prazo combinado é fim de novembro (`[09:00] Marcos`, `[09:45] Marcos`).
- Para os clientes, "tempo real" é qualquer coisa abaixo de 10 s, desde que não precisem atualizar manualmente (`[09:02] Marcos`).

## 3. Público-alvo e cenários de uso

**Públicos**

| Público | Interesse |
| --- | --- |
| Clientes B2B (Atlas, MaxDistribuição, Nova Cargo) | Receber mudanças de status dos próprios pedidos e consultar histórico de entregas |
| Usuários da plataforma que representam o cliente | Cadastrar e gerenciar webhooks via API autenticada com JWT (`[09:32] Marcos`) |
| Administradores (role `ADMIN`) | Reprocessar eventos que falharam definitivamente |
| Segurança (Sofia) | Garantir autenticidade, integridade e revisão do código sensível |

**Cenários**

1. *Cadastro:* um usuário cadastra a URL HTTPS do cliente, escolhe os status que quer receber e guarda a secret devolvida.
2. *Notificação:* o pedido do cliente passa para `SHIPPED`; em até ~2 s o endpoint recebe o evento assinado.
3. *Filtro:* o cliente quer saber "só quando vira `SHIPPED` e `DELIVERED`" (`[09:33] Marcos`), então não recebe outros status.
4. *Indisponibilidade:* o endpoint do cliente cai por 2 h de manutenção; a plataforma reenvia até ele voltar (`[09:16] Diego`).
5. *Falha permanente:* esgotadas as tentativas, um `ADMIN` reprocessa o evento.
6. *Rotação de secret:* o cliente solicita nova secret e migra seus sistemas em até 24 h (`[09:21] Sofia`).
7. *Diagnóstico:* o cliente consulta as últimas 100 entregas com resposta e tempo (`[09:34] Marcos`).

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta |
| --- | --- | --- | --- |
| O1 | Notificação em "tempo real" | Tempo entre o commit da mudança de status e a primeira tentativa de entrega | **< 10 s** (polling de 2 s no pior caso: `[09:10] Larissa`) |
| O2 | Nenhum evento perdido | Mudanças de status sem evento correspondente na outbox (para endpoints que o assinam) | **0** (atomicidade: `[09:40] Bruno`) |
| O3 | Recuperar indisponibilidades do cliente | Janela de retry até a DLQ | **≈ 14 h 36 min** (1m/5m/30m/2h/12h) |
| O4 | Cumprir o prazo comercial | Entrega em produção | **3 sprints**, até o fim de novembro (`[09:46] Larissa`) |
| O5 | Atender os clientes solicitantes | Clientes com webhook ativo | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) |

Redução do polling em `GET /orders` pelos clientes é um benefício esperado, mas **a reunião não definiu linha de base nem meta**; sugerimos medir após o lançamento ([RFC §5](RFC.md)).

## 5. Escopo

**Incluso**

- Cadastro, edição, remoção e listagem de webhooks por customer, com filtro de status.
- Geração de secret por endpoint e rotação com grace period de 24 h.
- Registro do evento junto da mudança de status e entrega assíncrona assinada.
- Retry com backoff, DLQ e replay manual por `ADMIN`.
- Histórico de entregas por webhook.

**Fora de escopo (descartado ou adiado na reunião)**

| Item | Situação | Fonte |
| --- | --- | --- |
| Email ao cliente quando o webhook falha repetidamente | Adiado para uma próxima fase, após medir o impacto | `[09:37] Larissa` |
| Rate limiting de envio por cliente | Não faz parte; observar e decidir depois | `[09:39] Larissa` |
| Dashboard/painel visual de webhooks | Fora; projeto separado do time de frontend | `[09:40] Larissa` |
| Arquivamento de eventos entregues (~30 dias) | Fora desta feature | `[09:08] Diego` |
| Garantia de ordering global / múltiplos workers | Limitação conhecida; ordem apenas por pedido e com um worker | `[09:13] Larissa` |
| Webhooks de entrada (cliente → plataforma) | Não; só saída | `[09:02] Marcos` |
| Itens do pedido no payload | Não enviados; cliente usa `GET /orders/:id` | `[09:43] Diego` |

## 6. Requisitos funcionais

| ID | Requisito | Fonte |
| --- | --- | --- |
| RF-01 | Cadastrar um webhook informando URL, `customerId` e lista de status desejados | `[09:31] Marcos`, `[09:32] Larissa` |
| RF-02 | A secret é gerada pela plataforma e devolvida na criação | `[09:31] Marcos` |
| RF-03 | Editar um webhook | `[09:33] Bruno` |
| RF-04 | Remover um webhook | `[09:33] Bruno` |
| RF-05 | Listar os webhooks de um customer | `[09:33] Bruno` |
| RF-06 | Filtrar eventos por status por webhook; sem assinante, nenhum evento é gerado | `[09:33] Marcos`, `[09:34] Bruno` |
| RF-07 | Registrar o evento de mudança de status na mesma transação da mudança | `[09:40] Bruno` |
| RF-08 | Enviar o evento por HTTP ao endpoint do cliente, de forma assíncrona | `[09:06] Diego` |
| RF-09 | Assinar cada envio com HMAC-SHA256 usando a secret do endpoint | `[09:22] Sofia` |
| RF-10 | Rotacionar a secret pela API, mantendo a anterior válida por 24 h | `[09:21] Sofia` |
| RF-11 | Reenviar com backoff 1m/5m/30m/2h/12h e mover para DLQ ao esgotar | `[09:17] Larissa` |
| RF-12 | Reprocessar manualmente um item da DLQ via endpoint exclusivo de `ADMIN`, registrando quem executou | `[09:35] Diego`, `[09:36] Sofia` |
| RF-13 | Identificar cada evento com `X-Event-Id` único para deduplicação | `[09:25] Diego` |
| RF-14 | Consultar o histórico das últimas 100 entregas (sucesso/falha, payload, resposta, tempo) | `[09:34] Marcos` |
| RF-15 | Enviar `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` | `[09:44] Diego`, `[09:44] Sofia` |

## 7. Requisitos não funcionais

| ID | Requisito | Fonte |
| --- | --- | --- |
| RNF-01 | Latência de notificação < 10 s; polling a cada 2 s | `[09:02] Marcos`, `[09:09] Diego` |
| RNF-02 | Entrega at-least-once; deduplicação do lado do cliente | `[09:24] Diego` |
| RNF-03 | Somente URLs HTTPS | `[09:23] Sofia` |
| RNF-04 | Payload máximo de 64 KB; acima disso é erro, sem truncar | `[09:24] Larissa` |
| RNF-05 | Timeout de 10 s por chamada ao cliente | `[09:42] Diego` |
| RNF-06 | Secret única por endpoint | `[09:21] Sofia` |
| RNF-07 | Entrega isolada da API: worker em processo separado | `[09:11] Diego` |
| RNF-08 | Ordem por pedido apenas com worker único (limitação documentada) | `[09:13] Larissa` |
| RNF-09 | Reuso dos padrões do projeto (`AppError`, Pino, módulos, erros `WEBHOOK_*`) | `[09:30] Larissa` |
| RNF-10 | Revisão de segurança antes do deploy (2 dias úteis) | `[09:46] Sofia` |

## 8. Decisões e trade-offs principais

Detalhe de cada uma nos ADRs; aqui só o resumo do ponto de vista de produto.

| Decisão | Ganho | Custo aceito | ADR |
| --- | --- | --- | --- |
| Outbox no MySQL | Sem perda de eventos, sem infra nova | Tabela cresce; arquivamento adiado | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Worker em polling de 2 s | Simples; atende a meta de 10 s | Até 2 s de latência extra | [ADR-005](adrs/ADR-005-worker-separado-em-polling.md) |
| Retry de ~14 h 36 min e DLQ | Cobre manutenções planejadas | Evento pode chegar horas depois ou exigir replay | [ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md) |
| HMAC-SHA256 com secret por endpoint | Autenticidade e raio de dano reduzido | Cliente precisa validar assinatura e gerir rotação | [ADR-003](adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md) |
| At-least-once | Simples, padrão de mercado | Cliente deduplica; duplicatas possíveis | [ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md) |
| Reuso dos padrões | Menor custo e risco | Herda limitações atuais | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| Evento em snapshot, na transação | Estado fiel e atomicidade | Transação de status mais pesada | [ADR-007](adrs/ADR-007-publicacao-transacional-e-payload-snapshot.md) |

## 9. Dependências

- **Banco MySQL existente** e Prisma, sem nova infraestrutura (`[09:07] Diego`).
- **Alteração em `OrderService.changeStatus`** do time de Pedidos ([src/modules/orders/order.service.ts](../src/modules/orders/order.service.ts)).
- **Autenticação JWT e roles** existentes ([src/middlewares/auth.middleware.ts](../src/middlewares/auth.middleware.ts)).
- **Portal de desenvolvedores / documentação para clientes**, a cargo de Marcos, incluindo a explicação destacada de at-least-once e dedup (`[09:26] Marcos`, `[09:40] Marcos`).
- **Revisão de segurança da Sofia**, com 2 dias úteis agendados antes do deploy (`[09:46] Sofia`, `[09:49] Sofia`).
- **Comunicação com os clientes** e confirmação de prazo por Marcos (`[09:47] Marcos`).
- **Sessão de revisão do design** com Larissa, Bruno e Diego antes de codar (`[09:50] Larissa`).

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Perder o prazo e a conta Atlas | Média | Alto | Escopo enxuto (sem email, dashboard ou rate limit); 3 sprints com revisão de segurança no final |
| Clientes não deduplicam e processam eventos em duplicidade | Alta | Médio | `X-Event-Id` e documentação destacada no portal |
| Falha em `changeStatus` por causa do módulo de webhooks | Baixa | Alto | Operação de insert simples e testes de transação |
| Vazamento de secret de um cliente | Média | Alto | Secret por endpoint, rotação com 24 h, revisão de segurança |
| Cliente fora do ar por mais de ~15 h perde a entrega automática | Baixa | Médio | DLQ preservada; replay manual por `ADMIN` |
| Falta de ordering global gera confusão em clientes | Baixa | Baixo | Limitação documentada; clientes só acompanham cada pedido (`[09:14] Marcos`) |
| Volume alto para um mesmo cliente (sem rate limit) | Média | Médio | Observar e decidir depois (`[09:39] Larissa`) |

## 11. Critérios de aceitação

- [ ] Um usuário autenticado cadastra um webhook HTTPS com lista de status e recebe a secret apenas na criação (RF-01, RF-02, RNF-03).
- [ ] Cadastro com `http://` é recusado com erro de validação.
- [ ] Ao mudar o status de um pedido para um status assinado, o endpoint recebe o evento em menos de 10 s (O1).
- [ ] Mudança para status não assinado não gera evento (RF-06).
- [ ] Cada entrega contém `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e a assinatura é verificável com a secret (RF-09, RF-13, RF-15).
- [ ] Endpoint indisponível recebe reenvios no cronograma definido e, ao esgotar, o evento fica na DLQ (RF-11).
- [ ] `ADMIN` reprocessa item da DLQ; `OPERATOR` recebe 403; o replay fica registrado em log (RF-12).
- [ ] Rotação de secret mantém a antiga válida por 24 h (RF-10).
- [ ] O cliente consulta as últimas 100 entregas de um webhook (RF-14).
- [ ] Se a gravação do evento falhar, o status do pedido não muda (RF-07, O2).
- [ ] Revisão de segurança concluída antes do deploy (RNF-10).

## 12. Estratégia de testes e validação

O projeto já usa Vitest com banco real ([tests/setup.ts](../tests/setup.ts), [tests/orders.test.ts](../tests/orders.test.ts)); os testes novos seguem o mesmo padrão. Detalhe técnico em [FDD §11](FDD.md#11-critérios-de-aceite-técnicos).

- **Integração de transação:** `changeStatus` grava a outbox atomicamente, e a falha da outbox reverte o status.
- **Filtro de eventos:** só gera linha para endpoints que assinam o status.
- **Worker:** com servidor HTTP falso, valida sucesso, timeout de 10 s, não-2xx, backoff e DLQ (relógio simulado).
- **Segurança:** assinatura HMAC verificada de ponta a ponta; rotação com grace period; URL `http` recusada; replay restrito a `ADMIN`.
- **Contrato:** campos do payload, headers e limite de 64 KB.
- **Ponta a ponta:** pedido criado, pago, enviado e entregue, com recebimento ordenado por pedido em condição normal.
- **Validação com clientes:** ensaio com um cliente-piloto (por exemplo, Atlas) antes da liberação geral (**[PROPOSTA]**).
- **Revisão de segurança** da Sofia como gate de deploy.
