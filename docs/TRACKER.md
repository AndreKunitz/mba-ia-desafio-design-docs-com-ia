# Tracker de Rastreabilidade

Cada linha liga um item dos documentos à sua origem na transcrição (`TRANSCRICAO.md`, formato `[hh:mm] Nome`) ou no código (caminho de arquivo). Itens marcados **[PROPOSTA]** nos documentos (escolhas de implementação sem decisão na reunião) não têm linha de origem aqui e estão listados na seção final.

| **ID** | **Documento** | **Tipo** | **Conteúdo (resumo)** | **Fonte** | **Localização** |
|--------|---------------|----------|-----------------------|-----------|-----------------|
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Outbox no MySQL, evento na mesma transação do changeStatus | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa descartada | HTTP síncrono no service de orders | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa descartada | Redis Streams exige infraestrutura extra | TRANSCRICAO | [09:07] Larissa |
| ADR-001-ALT-03 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa descartada | Publicar fora da transação perde a garantia | TRANSCRICAO | [09:41] Diego |
| ADR-001-IDX | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Índices em status e created_at; worker lê só pendentes em lote pequeno | TRANSCRICAO | [09:08] Diego |
| ADR-001-UUID | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Chave primária UUID, como o resto do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-001-COD | `docs/adrs/ADR-001-outbox-no-mysql.md` | Restrição | Transação atual de changeStatus (order, history, estoque) | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-001-SCH | `docs/adrs/ADR-001-outbox-no-mysql.md` | Restrição | Padrão de ids uuid Char(36) no schema | CODIGO | `prisma/schema.prisma` |
| ADR-002 | `docs/adrs/ADR-002-retry-backoff-e-dlq.md` | Decisão | Backoff exponencial 1m/5m/30m/2h/12h e DLQ | TRANSCRICAO | [09:17] Larissa |
| ADR-002-ALT-01 | `docs/adrs/ADR-002-retry-backoff-e-dlq.md` | Alternativa descartada | Retry indefinido deixa evento pendurado | TRANSCRICAO | [09:15] Diego |
| ADR-002-ALT-02 | `docs/adrs/ADR-002-retry-backoff-e-dlq.md` | Alternativa descartada | 3 tentativas cobrem pouco tempo | TRANSCRICAO | [09:16] Diego |
| ADR-002-ALT-03 | `docs/adrs/ADR-002-retry-backoff-e-dlq.md` | Alternativa descartada | DLQ como status na própria outbox | TRANSCRICAO | [09:18] Diego |
| ADR-002-DLQ | `docs/adrs/ADR-002-retry-backoff-e-dlq.md` | Decisão | DLQ em tabela separada webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| ADR-002-REPLAY | `docs/adrs/ADR-002-retry-backoff-e-dlq.md` | Decisão | Replay manual via endpoint admin, ADMIN e log de quem executou | TRANSCRICAO | [09:36] Sofia |
| ADR-002-TMO | `docs/adrs/ADR-002-retry-backoff-e-dlq.md` | Restrição | Timeout de 10 s por chamada | TRANSCRICAO | [09:42] Diego |
| ADR-002-OBS | `docs/adrs/ADR-002-retry-backoff-e-dlq.md` | Trade-off | Ambiguidade 5 tentativas vs 5 intervalos (1+5 reenvios) | TRANSCRICAO | [09:15] Diego |
| ADR-003 | `docs/adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md` | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-003-ALG | `docs/adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md` | Decisão | Algoritmo SHA-256, padrão de mercado | TRANSCRICAO | [09:20] Sofia |
| ADR-003-ALT-01 | `docs/adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md` | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| ADR-003-HTTPS | `docs/adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md` | Restrição | URL somente https, validação via Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-003-ROT | `docs/adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md` | Decisão | Rotação via API; antiga válida 24 h | TRANSCRICAO | [09:21] Sofia |
| ADR-003-REV | `docs/adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md` | Restrição | Revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| ADR-003-LEAK | `docs/adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md` | Restrição | Histórico de secret vazada em log de cliente | TRANSCRICAO | [09:22] Diego |
| ADR-004 | `docs/adrs/ADR-004-at-least-once-com-x-event-id.md` | Decisão | At-least-once com X-Event-Id para dedup | TRANSCRICAO | [09:26] Larissa |
| ADR-004-ID | `docs/adrs/ADR-004-at-least-once-com-x-event-id.md` | Decisão | UUID gerado na entrada da outbox vai em X-Event-Id | TRANSCRICAO | [09:25] Diego |
| ADR-004-ALT-01 | `docs/adrs/ADR-004-at-least-once-com-x-event-id.md` | Alternativa descartada | Exactly-once exige coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| ADR-004-DOC | `docs/adrs/ADR-004-at-least-once-com-x-event-id.md` | Restrição | Dedup é responsabilidade do cliente; PM documenta no portal | TRANSCRICAO | [09:26] Marcos |
| ADR-004-ORD | `docs/adrs/ADR-004-at-least-once-com-x-event-id.md` | Restrição | Ordering só por order_id e com worker único | TRANSCRICAO | [09:13] Larissa |
| ADR-004-ORD2 | `docs/adrs/ADR-004-at-least-once-com-x-event-id.md` | Trade-off | Clientes não pediram ordering global | TRANSCRICAO | [09:14] Marcos |
| ADR-005 | `docs/adrs/ADR-005-worker-separado-em-polling.md` | Decisão | Worker em polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| ADR-005-PROC | `docs/adrs/ADR-005-worker-separado-em-polling.md` | Decisão | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| ADR-005-ENTRY | `docs/adrs/ADR-005-worker-separado-em-polling.md` | Decisão | Nova entry-point src/worker.ts e npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-005-ALT-01 | `docs/adrs/ADR-005-worker-separado-em-polling.md` | Alternativa descartada | Trigger de banco não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| ADR-005-PRISMA | `docs/adrs/ADR-005-worker-separado-em-polling.md` | Decisão | PrismaClient próprio no worker, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-005-LAT | `docs/adrs/ADR-005-worker-separado-em-polling.md` | Requisito Não Funcional | Latência < 10 s é "tempo real" | TRANSCRICAO | [09:02] Marcos |
| ADR-005-SRV | `docs/adrs/ADR-005-worker-separado-em-polling.md` | Restrição | Padrão de bootstrap e shutdown do server.ts | CODIGO | `src/server.ts` |
| ADR-005-DB | `docs/adrs/ADR-005-worker-separado-em-polling.md` | Restrição | PrismaClient por processo em config/database | CODIGO | `src/config/database.ts` |
| ADR-006 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | Reuso máximo de AppError, Pino, error middleware e módulos | TRANSCRICAO | [09:30] Larissa |
| ADR-006-MOD | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | Módulo src/modules/webhooks com controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-ERR | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | Erros com prefixo WEBHOOK_ | TRANSCRICAO | [09:29] Larissa |
| ADR-006-LOG | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | Pino e error middleware sem mudanças | TRANSCRICAO | [09:29] Bruno |
| ADR-006-COD1 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Restrição | Hierarquia AppError e subclasses | CODIGO | `src/shared/errors/http-errors.ts` |
| ADR-006-COD2 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Restrição | Error middleware trata AppError, Zod e Prisma | CODIGO | `src/middlewares/error.middleware.ts` |
| ADR-006-COD3 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Restrição | authenticate e requireRole | CODIGO | `src/middlewares/auth.middleware.ts` |
| ADR-006-COD4 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Restrição | Logger Pino centralizado | CODIGO | `src/shared/logger/index.ts` |
| ADR-006-COD5 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Restrição | Registro de routers por módulo | CODIGO | `src/routes/index.ts` |
| ADR-007 | `docs/adrs/ADR-007-publicacao-transacional-e-payload-snapshot.md` | Decisão | Evento inserido dentro da transação de changeStatus | TRANSCRICAO | [09:40] Bruno |
| ADR-007-FN | `docs/adrs/ADR-007-publicacao-transacional-e-payload-snapshot.md` | Decisão | publishWebhookEvent(tx, order, from, to) em vez de injetar repository | TRANSCRICAO | [09:41] Diego |
| ADR-007-FILT | `docs/adrs/ADR-007-publicacao-transacional-e-payload-snapshot.md` | Decisão | Filtro de status na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| ADR-007-SNAP | `docs/adrs/ADR-007-publicacao-transacional-e-payload-snapshot.md` | Decisão | Payload renderizado (snapshot) na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007-COD | `docs/adrs/ADR-007-publicacao-transacional-e-payload-snapshot.md` | Restrição | changeStatus usa prisma.$transaction com TxClient | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-008 | `docs/adrs/ADR-008-contrato-de-entrega-payload-headers-e-limites.md` | Decisão | Payload enxuto sem items, headers e limites | TRANSCRICAO | [09:43] Diego |
| ADR-008-HDR | `docs/adrs/ADR-008-contrato-de-entrega-payload-headers-e-limites.md` | Decisão | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| ADR-008-WID | `docs/adrs/ADR-008-contrato-de-entrega-payload-headers-e-limites.md` | Decisão | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| ADR-008-SZ | `docs/adrs/ADR-008-contrato-de-entrega-payload-headers-e-limites.md` | Requisito Não Funcional | Limite de 64 KB, erro sem truncar | TRANSCRICAO | [09:24] Larissa |
| ADR-008-ALT-01 | `docs/adrs/ADR-008-contrato-de-entrega-payload-headers-e-limites.md` | Alternativa descartada | Truncar payload grande | TRANSCRICAO | [09:23] Sofia |
| RFC-CTX-01 | `docs/RFC.md` | Contexto | Pedido formal de Atlas, MaxDistribuição e Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | `docs/RFC.md` | Restrição | Webhook apenas outbound | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-03 | `docs/RFC.md` | Restrição | Sem eventos/filas/webhooks no sistema atual; changeStatus em transação única | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa descartada | HTTP síncrono em changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa descartada | Redis Streams / fila externa | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | `docs/RFC.md` | Alternativa descartada | Trigger/listener no banco | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | `docs/RFC.md` | Alternativa descartada | Retry indefinido | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-05 | `docs/RFC.md` | Alternativa descartada | 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-06 | `docs/RFC.md` | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-07 | `docs/RFC.md` | Alternativa descartada | Falha permanente como status na outbox | TRANSCRICAO | [09:18] Diego |
| RFC-OPEN-01 | `docs/RFC.md` | Questão em aberto | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-02 | `docs/RFC.md` | Questão em aberto | Alerta por email adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| RFC-OPEN-03 | `docs/RFC.md` | Questão em aberto | Escala do worker: partição por order_id ou lock pessimista | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-04 | `docs/RFC.md` | Questão em aberto | Arquivamento da outbox (~30 dias) fora do escopo | TRANSCRICAO | [09:08] Diego |
| RFC-OPEN-05 | `docs/RFC.md` | Questão em aberto | Endurecer permissões do CRUD de webhooks | TRANSCRICAO | [09:37] Sofia |
| RFC-OPEN-06 | `docs/RFC.md` | Questão em aberto | Contagem de tentativas (5 tentativas vs 5 intervalos) | TRANSCRICAO | [09:17] Diego |
| RFC-OPEN-07 | `docs/RFC.md` | Questão em aberto | customer_id vem do body/path, não do JWT; autorização por customer | TRANSCRICAO | [09:32] Larissa |
| RFC-OPEN-08 | `docs/RFC.md` | Questão em aberto | Secret antiga no envio durante o grace period | TRANSCRICAO | [09:21] Sofia |
| RFC-OPEN-09 | `docs/RFC.md` | Questão em aberto | Armazenamento da secret e X-Timestamp fora da assinatura | TRANSCRICAO | [09:46] Sofia |
| RFC-OPEN-10 | `docs/RFC.md` | Questão em aberto | Evento na criação do pedido (create não passa por changeStatus) | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-OPEN-11 | `docs/RFC.md` | Questão em aberto | Payload > 64 KB versus rollback de changeStatus | TRANSCRICAO | [09:23] Sofia |
| RFC-RSK-01 | `docs/RFC.md` | Risco | Cronograma de 3 sprints e revisão de segurança de 2 dias | TRANSCRICAO | [09:46] Sofia |
| RFC-RSK-02 | `docs/RFC.md` | Risco | Cliente processa duplicata; documentar no portal | TRANSCRICAO | [09:26] Marcos |
| PRD-CTX | `docs/PRD.md` | Contexto | Clientes consultam GET /orders por polling; risco de migração da Atlas | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-01 | `docs/PRD.md` | Objetivo | Notificação em menos de 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | `docs/PRD.md` | Objetivo | Zero evento perdido por atomicidade da transação | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-03 | `docs/PRD.md` | Objetivo | Janela de retry ≈ 14 h 36 min | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-04 | `docs/PRD.md` | Objetivo | Entrega em 3 sprints até fim de novembro | TRANSCRICAO | [09:46] Larissa |
| PRD-OBJ-05 | `docs/PRD.md` | Objetivo | Atender os 3 clientes solicitantes | TRANSCRICAO | [09:00] Marcos |
| PRD-ST | `docs/PRD.md` | Restrição | Status do pedido PENDING→…→DELIVERED/CANCELLED | CODIGO | `src/modules/orders/order.status.ts` |
| PRD-OUT-01 | `docs/PRD.md` | Fora de escopo | Email ao cliente | TRANSCRICAO | [09:37] Larissa |
| PRD-OUT-02 | `docs/PRD.md` | Fora de escopo | Rate limiting de saída | TRANSCRICAO | [09:39] Larissa |
| PRD-OUT-03 | `docs/PRD.md` | Fora de escopo | Dashboard visual | TRANSCRICAO | [09:40] Larissa |
| PRD-OUT-04 | `docs/PRD.md` | Fora de escopo | Arquivamento de eventos entregues | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-05 | `docs/PRD.md` | Fora de escopo | Ordering global / múltiplos workers | TRANSCRICAO | [09:13] Larissa |
| PRD-OUT-06 | `docs/PRD.md` | Fora de escopo | Itens do pedido no payload | TRANSCRICAO | [09:43] Diego |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Cadastrar webhook com URL, customerId e status | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Filtro de eventos por status | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Evento registrado na mesma transação da mudança | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Envio HTTP assíncrono ao cliente | TRANSCRICAO | [09:06] Diego |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Assinatura HMAC-SHA256 em cada envio | TRANSCRICAO | [09:22] Sofia |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | Rotação de secret com grace de 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-11 | `docs/PRD.md` | Requisito Funcional | Retry com backoff e DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-12 | `docs/PRD.md` | Requisito Funcional | Replay de DLQ por ADMIN com registro do executor | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-13 | `docs/PRD.md` | Requisito Funcional | X-Event-Id para deduplicação | TRANSCRICAO | [09:25] Diego |
| PRD-FR-14 | `docs/PRD.md` | Requisito Funcional | Histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-15 | `docs/PRD.md` | Requisito Funcional | Headers X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | Polling a cada 2 s | TRANSCRICAO | [09:09] Diego |
| PRD-NFR-02 | `docs/PRD.md` | Requisito Não Funcional | Garantia at-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-03 | `docs/PRD.md` | Requisito Não Funcional | Somente HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-04 | `docs/PRD.md` | Requisito Não Funcional | Payload máximo de 64 KB | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | `docs/PRD.md` | Requisito Não Funcional | Timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-06 | `docs/PRD.md` | Requisito Não Funcional | Secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-07 | `docs/PRD.md` | Requisito Não Funcional | Worker em processo separado | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-08 | `docs/PRD.md` | Requisito Não Funcional | Revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-01 | `docs/PRD.md` | Dependência | Portal de desenvolvedores documentado pelo PM | TRANSCRICAO | [09:40] Marcos |
| PRD-DEP-02 | `docs/PRD.md` | Dependência | JWT e roles existentes | CODIGO | `src/middlewares/auth.middleware.ts` |
| PRD-DEP-03 | `docs/PRD.md` | Dependência | Sessão de revisão do design antes de codar | TRANSCRICAO | [09:50] Larissa |
| PRD-RSK-01 | `docs/PRD.md` | Risco | Perda do prazo e da conta Atlas | TRANSCRICAO | [09:00] Marcos |
| PRD-RSK-02 | `docs/PRD.md` | Risco | Falta de ordering global confunde clientes | TRANSCRICAO | [09:14] Marcos |
| PRD-TST | `docs/PRD.md` | Estratégia de teste | Testes Vitest com banco real e factories | CODIGO | `tests/orders.test.ts` |
| FDD-MOD-01 | `docs/FDD.md` | Decisão | Tabelas webhook_outbox/webhook_dead_letter | TRANSCRICAO | [09:06] Diego |
| FDD-MOD-02 | `docs/FDD.md` | Decisão | Estados pendente/processando/falhou/entregue | TRANSCRICAO | [09:08] Diego |
| FDD-MOD-03 | `docs/FDD.md` | Decisão | Config do webhook: url, secret, customer_id, ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-MOD-04 | `docs/FDD.md` | Decisão | Dead letter com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-01 | `docs/FDD.md` | Fluxo | Inserção na outbox dentro da transação com rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-FLX-02 | `docs/FDD.md` | Fluxo | Worker lê pendentes mais antigos por created_at em lote pequeno | TRANSCRICAO | [09:12] Diego |
| FDD-FLX-03 | `docs/FDD.md` | Fluxo | Backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-FLX-04 | `docs/FDD.md` | Fluxo | DLQ e replay recolocando na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-05 | `docs/FDD.md` | Fluxo | Shutdown gracioso do worker nos moldes do server | CODIGO | `src/server.ts` |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | POST /webhooks (cadastro) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | GET /webhooks por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato | GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | `docs/FDD.md` | Contrato | POST /admin/webhooks/dead-letter/:id/replay (ADMIN) | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-07 | `docs/FDD.md` | Contrato | Endpoint de rotação de secret (caminho proposto) | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-08 | `docs/FDD.md` | Contrato | Payload e headers de entrega | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-09 | `docs/FDD.md` | Restrição | Formato de resposta paginada reutilizado | CODIGO | `src/shared/http/response.ts` |
| FDD-CONTRATO-10 | `docs/FDD.md` | Restrição | Rotas sob /api/v1 | CODIGO | `src/app.ts` |
| FDD-AUTH-01 | `docs/FDD.md` | Decisão | CRUD com qualquer role autenticada | TRANSCRICAO | [09:37] Sofia |
| FDD-AUTH-02 | `docs/FDD.md` | Decisão | Replay exige ADMIN via requireRole | TRANSCRICAO | [09:36] Larissa |
| FDD-AUTH-03 | `docs/FDD.md` | Restrição | customer_id no body/path e não no JWT | TRANSCRICAO | [09:32] Larissa |
| FDD-ERR-01 | `docs/FDD.md` | Restrição | Códigos WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | `docs/FDD.md` | Restrição | Prefixo WEBHOOK_ em todos os códigos | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-03 | `docs/FDD.md` | Restrição | Padrão de classes de erro com errorCode | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-ERR-04 | `docs/FDD.md` | Restrição | Payload > 64 KB é erro | TRANSCRICAO | [09:23] Sofia |
| FDD-RES-01 | `docs/FDD.md` | Resiliência | Timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | `docs/FDD.md` | Resiliência | Idempotência por X-Event-Id | TRANSCRICAO | [09:25] Diego |
| FDD-RES-03 | `docs/FDD.md` | Resiliência | Isolamento da API por worker separado | TRANSCRICAO | [09:11] Diego |
| FDD-OBS-01 | `docs/FDD.md` | Observabilidade | Logger Pino único, nada novo; log de quem fez replay | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | `docs/FDD.md` | Observabilidade | Auditoria do replay | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-03 | `docs/FDD.md` | Observabilidade | Redaction de campos sensíveis no logger | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-01 | `docs/FDD.md` | Integração | Estender changeStatus com publishWebhookEvent(tx, …) | TRANSCRICAO | [09:41] Bruno |
| FDD-INT-02 | `docs/FDD.md` | Integração | Ponto de extensão em changeStatus | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração | Transições válidas definem os status do filtro | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração | requireRole já usado em user.routes | CODIGO | `src/modules/users/user.routes.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração | Wiring de controllers em buildControllers | CODIGO | `src/app.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração | Registro de router em buildApiRouter | CODIGO | `src/routes/index.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração | Middleware validate com Zod | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-INT-08 | `docs/FDD.md` | Integração | Validação de env com Zod | CODIGO | `src/config/env.ts` |
| FDD-INT-09 | `docs/FDD.md` | Integração | Models Prisma com uuid Char(36) | CODIGO | `prisma/schema.prisma` |
| FDD-INT-10 | `docs/FDD.md` | Integração | Scripts npm (dev/start) a espelhar para worker | CODIGO | `package.json` |
| FDD-PRZ | `docs/FDD.md` | Restrição | Prazo de 3 sprints e 2 dias de revisão da Sofia | TRANSCRICAO | [09:46] Larissa |
| FDD-OT-01 | `docs/FDD.md` | Objetivo | Entrega < 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-ORD | `docs/FDD.md` | Restrição | Ordering implícita por order_id enquanto worker único | TRANSCRICAO | [09:12] Diego |
| FDD-OUT-01 | `docs/FDD.md` | Fora de escopo | Email, rate limit, dashboard, arquivamento | TRANSCRICAO | [09:37] Larissa |

## Itens sem origem direta (propostas de implementação)

Registrados nos documentos como **[PROPOSTA]**; precisam de validação na revisão do RFC e não constituem decisão da reunião:

- Nomes `webhook_endpoints`, `webhook_deliveries` e colunas auxiliares (`next_attempt_at`, `attempts`, `correlation_id`).
- Uma linha de outbox por par (evento, webhook); mapeamento de estados da outbox.
- Prefixo `/api/v1` nas rotas admin, caminho `POST /webhooks/:id/rotate-secret`, status `202` no replay.
- Critério de sucesso `2xx`; formato `sha256=<hex>` de `X-Signature`; assinatura dupla durante o grace period.
- Tamanho de lote, recuperação de eventos `PROCESSING`, não seguir redirects.
- Tratamento de payload > 64 KB via DLQ sem abortar `changeStatus`.
- Métricas, `correlation_id` e variáveis de ambiente `WEBHOOK_*` do worker.
- Piloto com cliente antes do lançamento geral.
