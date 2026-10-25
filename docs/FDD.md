# FDD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autor** | Andre Kunitz |
| **Status** | Rascunho para revisão |
| **Data** | 2026-10-25 |
| **Documentos relacionados** | [PRD](PRD.md) · [RFC](RFC.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

> **Convenção:** itens com a marca **[PROPOSTA]** não foram decididos na reunião; são escolhas de implementação necessárias para tornar o desenho acionável e devem ser validadas na revisão. Tudo o mais tem origem na transcrição (`[hh:mm] Nome`) ou no código (caminho de arquivo). Mapeamento completo em [TRACKER.md](TRACKER.md).

## 1. Contexto e motivação técnica

`OrderService.changeStatus` ([src/modules/orders/order.service.ts](../src/modules/orders/order.service.ts)) executa, em uma `prisma.$transaction`, `tx.order.update`, `tx.orderStatusHistory.create` e a movimentação de estoque (`debitStock`/`replenishStock`). Não há eventos, filas ou webhooks no sistema. Três clientes B2B precisam ser avisados em menos de 10 s a cada mudança de status (`[09:00] Marcos`, `[09:02] Marcos`). A chamada HTTP não pode ocorrer dentro da transação (`[09:04] Bruno`), e o evento não pode se perder se a transação commitar (`[09:41] Diego`). Racional das escolhas: [ADR-001](adrs/ADR-001-outbox-no-mysql.md) e [ADR-007](adrs/ADR-007-publicacao-transacional-e-payload-snapshot.md).

## 2. Objetivos técnicos

- **OT-1.** Registrar o evento atomicamente com a mudança de status (zero estado "status mudou e evento não saiu").
- **OT-2.** Entregar em menos de 10 s no caso nominal (polling de 2 s + timeout de 10 s por chamada).
- **OT-3.** Garantia at-least-once com `X-Event-Id` ([ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md)).
- **OT-4.** Autenticar cada entrega com HMAC-SHA256, secret por endpoint, rotação com 24 h ([ADR-003](adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md)).
- **OT-5.** Retry com backoff 1m/5m/30m/2h/12h e DLQ com replay ADMIN ([ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md)).
- **OT-6.** Não introduzir infraestrutura nova nem divergir dos padrões do projeto ([ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)).

## 3. Escopo e exclusões

**Dentro:** tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_dead_letter` e histórico de entregas; módulo `src/modules/webhooks`; worker `src/worker.ts`; endpoints de CRUD, rotação de secret, histórico de entregas e replay de DLQ; integração em `changeStatus`.

**Fora (explicitamente descartado ou adiado):**

| Item | Situação | Fonte |
| --- | --- | --- |
| Email ao cliente quando o webhook falha | Adiado para próxima fase | `[09:37] Larissa` |
| Rate limiting de saída | Observar e decidir depois | `[09:39] Larissa` |
| Dashboard visual para o cliente | Fora; projeto separado do frontend | `[09:40] Larissa` |
| Arquivamento de linhas entregues (~30 dias) | Fora desta feature | `[09:08] Diego` |
| Múltiplos workers / ordering global | Limitação conhecida | `[09:13] Larissa` |
| Webhooks de entrada (cliente → plataforma) | Fora; só outbound | `[09:02] Marcos` |
| Exactly-once, Redis Streams, entrega síncrona | Descartados ([RFC §4](RFC.md)) | `[09:25] Diego`, `[09:07] Diego`, `[09:06] Diego` |

## 4. Fluxos detalhados

### 4.1 Modelo de dados

Todas as chaves primárias são UUID `Char(36)`, como no restante de [prisma/schema.prisma](../prisma/schema.prisma) (`[09:51] Larissa`). Nomes de `webhook_outbox` e `webhook_dead_letter` vêm da reunião (`[09:06] Diego`, `[09:18] Diego`); demais nomes e colunas são **[PROPOSTA]**.

| Tabela | Colunas principais |
| --- | --- |
| `webhook_endpoints` | `id`, `customer_id` (FK `customers`), `url`, `secret`, `previous_secret`, `previous_secret_expires_at`, `events` (JSON, lista de `OrderStatus`), `active`, `created_at`, `updated_at` |
| `webhook_outbox` | `id` (= `event_id`), `webhook_id` (FK), `order_id`, `event_type`, `payload` (JSON, snapshot), `status` (`PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`), `attempts`, `next_attempt_at`, `last_error`, `created_at`, `updated_at`. Índices em `status` e `created_at` (`[09:08] Diego`) e composto `(status, next_attempt_at)` **[PROPOSTA]** |
| `webhook_dead_letter` | `id`, `outbox_id`, `webhook_id`, `payload`, `failure_reason`, `attempts`, `failed_at`, `replayed_at`, `replayed_by` (cf. `[09:18] Diego`, `[09:36] Sofia`) |
| `webhook_deliveries` **[PROPOSTA]** | `id`, `outbox_id`, `webhook_id`, `attempt`, `success`, `status_code`, `response_body` (truncado), `duration_ms`, `error_code`, `created_at`. Atende `GET /webhooks/:id/deliveries` (`[09:34] Marcos`) |

**Decisão de modelagem:** há **uma linha de outbox por par (evento, webhook)**, pois URL e secret são por endpoint. O `X-Event-Id` é o `id` dessa linha, gerado na inserção (`[09:25] Diego`). **[PROPOSTA]** (consequência de `[09:21] Sofia`).

Estados da outbox **[PROPOSTA de mapeamento]** sobre os quatro estados citados em `[09:08] Diego`: `PENDING` (aguardando, inclusive reagendado para retry via `next_attempt_at`), `PROCESSING` (reservado pelo worker), `DELIVERED` (sucesso) e `FAILED` (terminal, enviado à DLQ).

### 4.2 Criação do evento na outbox

1. `PATCH /api/v1/orders/:id/status` chega ao `OrderController.changeStatus`, que chama `OrderService.changeStatus`.
2. Dentro da `$transaction` existente, após `tx.order.update` e `tx.orderStatusHistory.create`, o service chama `publishWebhookEvent(tx, order, from, to)` (`[09:41] Bruno`).
3. A função busca em `webhook_endpoints` os registros `active = true` do `order.customerId` cujo `events` contém `to`. **Se nenhum, retorna sem inserir** (`[09:34] Bruno`).
4. Para cada endpoint, renderiza o payload (snapshot, [§5.3](#53-payload-e-headers-de-entrega)) e insere uma linha `PENDING` com `next_attempt_at = now()`.
5. Qualquer erro propaga e causa **rollback de toda a transação** (`[09:40] Bruno`).
6. Commit. O evento só é visível ao worker após o commit.

```
OrderService.changeStatus
  └─ $transaction
       ├─ order.update / orderStatusHistory.create / (stock)
       ├─ publishWebhookEvent(tx, …)  ──▶ INSERT webhook_outbox (PENDING)
       └─ commit | rollback (tudo ou nada)
```

**Payload acima de 64 KB:** a reunião define "erra, não envia" (`[09:23] Sofia`, `[09:24] Larissa`), mas um erro dentro de `changeStatus` bloquearia a mudança de status. **[PROPOSTA]** validar o tamanho no `publishWebhookEvent`; se exceder, registrar a linha diretamente na DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem envio e sem abortar a transação. Pendente de confirmação ([RFC §5](RFC.md)).

### 4.3 Processamento pelo worker

Entry-point `src/worker.ts`, script `npm run worker`, processo separado (`[09:11] Larissa`, `[09:11] Diego`), com `PrismaClient` próprio (`[09:30] Bruno`). Lógica em `src/modules/webhooks/webhook.processor.ts` (`[09:28] Bruno`).

Loop a cada **2 s** (`[09:09] Diego`):

1. Selecionar lote pequeno de eventos `PENDING` com `next_attempt_at <= now()`, `ORDER BY created_at ASC` (`[09:08] Diego`, `[09:12] Diego`). Tamanho do lote: **[PROPOSTA]** 10, configurável.
2. Marcar o lote como `PROCESSING`.
3. Para cada evento: carregar o endpoint; se `active = false` ou removido, tratar como falha permanente (DLQ com `WEBHOOK_NOT_FOUND`) **[PROPOSTA]**.
4. Montar headers, assinar o corpo e executar `POST` com **timeout de 10 s** (`[09:42] Diego`).
5. Registrar a tentativa em `webhook_deliveries`.
6. Resposta `2xx`: marcar `DELIVERED`. Qualquer outra coisa (não-2xx, timeout, erro de rede): fluxo de retry ([§4.4](#44-retry-com-backoff)). O critério "2xx = sucesso" é **[PROPOSTA]**.
7. Dormir até o próximo ciclo. Em `SIGINT`/`SIGTERM`, concluir o evento em curso e desconectar o Prisma, como faz [src/server.ts](../src/server.ts).

**Recuperação de crash [PROPOSTA]:** ao iniciar, e periodicamente, eventos `PROCESSING` há mais de 60 s voltam a `PENDING`. Pode causar reenvio, aceito pelo at-least-once.

### 4.4 Retry com backoff

Intervalos após cada falha (`[09:17] Diego`, `[09:17] Larissa`):

| Falha nº | Reenvio após | Acumulado desde a 1ª falha |
| --- | --- | --- |
| 1 | 1 min | 1 min |
| 2 | 5 min | 6 min |
| 3 | 30 min | 36 min |
| 4 | 2 h | 2 h 36 min |
| 5 | 12 h | 14 h 36 min |
| 6 | — (DLQ) | — |

Em cada falha: incrementar `attempts`, gravar `last_error`, voltar a `PENDING` com `next_attempt_at = now() + intervalo[attempts-1]`. A sexta falha (1 envio + 5 reenvios) esgota a política e leva à DLQ. Esta leitura de "5 tentativas" é a que fecha com os números da reunião; confirmação pendente ([RFC §5](RFC.md)).

**Limitação de ordering:** o `ORDER BY created_at` só preserva a ordem por `order_id` sem falhas. Se o evento A de um pedido falha e é reagendado, o evento B seguinte do mesmo pedido pode ser entregue antes dele. Isto é consequência do desenho aceito em `[09:12] Diego`/`[09:13] Larissa` (ordering "implícita") e deve constar da documentação para o cliente.

### 4.5 DLQ e replay

**Entrada na DLQ:** após esgotar o retry, em transação: inserir em `webhook_dead_letter` (payload, motivo, timestamp), marcar a outbox como `FAILED` (`[09:17] Larissa`, `[09:18] Diego`).

**Replay:** `POST /api/v1/admin/webhooks/dead-letter/:id/replay` ([§5.7](#57-replay-de-dlq-admin)):

1. `authenticate` + `requireRole('ADMIN')` (`[09:36] Larissa`).
2. Carrega a entrada da DLQ (`WEBHOOK_DEAD_LETTER_NOT_FOUND` se não existir).
3. Recoloca o evento na outbox como `PENDING`, `attempts = 0`, preservando o `event_id` (`[09:18] Diego`; preservar o id é **[PROPOSTA]**, coerente com a dedup de [ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md)).
4. Registra `replayed_by`/`replayed_at` e emite log Pino com o usuário (`[09:36] Sofia`).

## 5. Contratos públicos

Prefixo `/api/v1`, como em [src/app.ts](../src/app.ts). Autenticação por `Authorization: Bearer <JWT>`. Erros no formato do error middleware: `{ "error": { "code", "message", "details?" } }`. Paths e métodos citados na reunião: `POST/PATCH/DELETE/GET` de configuração (`[09:31] Marcos`, `[09:33] Bruno`), `GET /webhooks/:id/deliveries` (`[09:34] Marcos`), `POST /admin/webhooks/dead-letter/:id/replay` (`[09:35] Diego`). O prefixo `/api/v1` e o caminho de rotação são **[PROPOSTA]**.

### 5.1 `POST /webhooks`: cadastrar webhook

Qualquer role autenticada (`[09:37] Sofia`). `customerId` vai no body e **não** vem do JWT (`[09:32] Larissa`).

```http
POST /api/v1/webhooks
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "customerId": "5b0f6b0e-3c0e-4b51-9a0e-6f3f4b1c2a10",
  "url": "https://api.atlas.example.com/hooks/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

`201 Created` (a secret só é devolvida aqui e na rotação: `[09:31] Marcos`):

```json
{
  "id": "0c9d3f1e-8a4b-4a39-8b64-2d6a9b7d1e55",
  "customerId": "5b0f6b0e-3c0e-4b51-9a0e-6f3f4b1c2a10",
  "url": "https://api.atlas.example.com/hooks/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_3f1c…(gerada pela plataforma)",
  "createdAt": "2026-10-25T14:00:00.000Z"
}
```

| Status | Código | Quando |
| --- | --- | --- |
| 201 | — | Criado |
| 400 | `VALIDATION_ERROR` / `WEBHOOK_INVALID_URL` | Schema inválido; URL não `https` (`[09:23] Sofia`) |
| 401 | `UNAUTHORIZED` | JWT ausente/inválido |
| 404 | `WEBHOOK_CUSTOMER_NOT_FOUND` | `customerId` inexistente |

### 5.2 `GET /webhooks?customerId=`: listar webhooks de um customer

```http
GET /api/v1/webhooks?customerId=5b0f6b0e-3c0e-4b51-9a0e-6f3f4b1c2a10&page=1&pageSize=20
```

`200 OK`, formato paginado de [src/shared/http/response.ts](../src/shared/http/response.ts). A secret **não** é retornada.

```json
{
  "data": [
    {
      "id": "0c9d3f1e-8a4b-4a39-8b64-2d6a9b7d1e55",
      "customerId": "5b0f6b0e-3c0e-4b51-9a0e-6f3f4b1c2a10",
      "url": "https://api.atlas.example.com/hooks/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Status: `200`, `400 VALIDATION_ERROR`, `401 UNAUTHORIZED`.

### 5.3 Payload e headers de entrega

Contrato de saída ([ADR-008](adrs/ADR-008-contrato-de-entrega-payload-headers-e-limites.md); `[09:43] Diego`, `[09:44] Diego`, `[09:44] Sofia`). Nomes em `snake_case` como ditados na reunião:

```http
POST https://api.atlas.example.com/hooks/oms
Content-Type: application/json
X-Event-Id: 7d0e1c4a-52c1-4c53-9d35-0b6f3b8e9a01
X-Signature: sha256=9f2c…
X-Timestamp: 2026-10-25T14:05:02.114Z
X-Webhook-Id: 0c9d3f1e-8a4b-4a39-8b64-2d6a9b7d1e55

{
  "event_id": "7d0e1c4a-52c1-4c53-9d35-0b6f3b8e9a01",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-25T14:05:01.870Z",
  "order_id": "b2a1c9d4-1e6f-4f0a-92c7-3a5d6e7f8091",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "5b0f6b0e-3c0e-4b51-9a0e-6f3f4b1c2a10",
  "total_cents": 125000
}
```

Regras: sem `items`; corpo ≤ 64 KB; `X-Signature` = HMAC-SHA256 hex do **corpo bruto** com a secret do endpoint. O prefixo `sha256=` e a codificação hex são **[PROPOSTA]**. Resposta esperada do cliente: `2xx` em até 10 s.

**Rotação [PROPOSTA]:** durante os 24 h de grace, `X-Signature` carrega duas assinaturas separadas por vírgula (`sha256=<nova>,sha256=<antiga>`), para que o cliente valide com qualquer uma. A reunião define o grace period, mas não como ele se aplica ao envio (`[09:21] Sofia`); validar com Sofia.

### 5.4 `PATCH /webhooks/:id`: editar webhook

```http
PATCH /api/v1/webhooks/0c9d3f1e-8a4b-4a39-8b64-2d6a9b7d1e55
{ "url": "https://api.atlas.example.com/hooks/v2", "events": ["PAID", "SHIPPED"], "active": true }
```

`200 OK` devolve o recurso (sem `secret`). Campos opcionais: `url`, `events`, `active`. Status: `400 VALIDATION_ERROR`/`WEBHOOK_INVALID_URL`, `401`, `404 WEBHOOK_NOT_FOUND`.

### 5.5 `DELETE /webhooks/:id`: remover webhook

`204 No Content` sem corpo. Status: `401`, `404 WEBHOOK_NOT_FOUND`. Eventos já na outbox para este endpoint seguem para a DLQ com `WEBHOOK_NOT_FOUND` **[PROPOSTA]**.

### 5.6 `GET /webhooks/:id/deliveries`: histórico de entregas

Últimas 100 entregas: sucesso/falha, payload, resposta e tempo de resposta (`[09:34] Marcos`).

```json
{
  "data": [
    {
      "id": "e1a2b3c4-0000-4000-8000-000000000001",
      "eventId": "7d0e1c4a-52c1-4c53-9d35-0b6f3b8e9a01",
      "attempt": 1,
      "success": true,
      "statusCode": 200,
      "responseBody": "{\"ok\":true}",
      "durationMs": 184,
      "payload": { "event_type": "order.status_changed", "to_status": "SHIPPED" },
      "createdAt": "2026-10-25T14:05:02.300Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

Status: `200`, `401`, `404 WEBHOOK_NOT_FOUND`.

### 5.7 Replay de DLQ (ADMIN)

```http
POST /api/v1/admin/webhooks/dead-letter/3e8f0a52-7c7d-4a0e-b1c1-9d2e4f6a8b10/replay
Authorization: Bearer <jwt-ADMIN>
```

`202 Accepted` (o evento volta à outbox; a entrega é assíncrona; o código 202 é **[PROPOSTA]**):

```json
{ "eventId": "7d0e1c4a-52c1-4c53-9d35-0b6f3b8e9a01", "status": "PENDING", "replayedBy": "a1b2…" }
```

Status: `202`, `401 UNAUTHORIZED`, `403 FORBIDDEN` (role `OPERATOR`), `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`.

### 5.8 `POST /webhooks/:id/rotate-secret`: rotacionar secret

A reunião exige um endpoint de rotação pela API (`[09:21] Sofia`), sem definir o caminho (**[PROPOSTA]**). Gera nova secret, mantém a anterior por 24 h.

`200 OK`:

```json
{ "id": "0c9d3f1e-8a4b-4a39-8b64-2d6a9b7d1e55", "secret": "whsec_a91d…", "previousSecretExpiresAt": "2026-10-26T14:10:00.000Z" }
```

Status: `200`, `401`, `404 WEBHOOK_NOT_FOUND`.

## 6. Matriz de erros

Padrão `AppError` com prefixo `WEBHOOK_` (`[09:28] Bruno`, `[09:29] Larissa`). Classes novas em `src/modules/webhooks/` estendem as de [src/shared/errors/http-errors.ts](../src/shared/errors/http-errors.ts). Os nomes `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` foram citados na reunião; os demais são **[PROPOSTA]**.

| Código | HTTP | Classe base | Quando | Superfície |
| --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `AppError` (404) | Webhook inexistente | API / worker |
| `WEBHOOK_INVALID_URL` | 400 | `ValidationError` | URL não `https` ou malformada | API |
| `WEBHOOK_SECRET_REQUIRED` | 500 (interno) | `AppError` | Endpoint sem secret ao assinar | worker |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `NotFoundError` | `customerId` inexistente no cadastro | API |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `NotFoundError` | Entrada de DLQ inexistente no replay | API |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | n/a | `AppError` | Payload > 64 KB; vai direto à DLQ | publish / worker |
| `WEBHOOK_DELIVERY_TIMEOUT` | n/a | n/a | Sem resposta em 10 s; entra em retry | worker |
| `WEBHOOK_DELIVERY_FAILED` | n/a | n/a | Resposta não-2xx ou erro de rede; entra em retry | worker |
| `WEBHOOK_RETRIES_EXHAUSTED` | n/a | n/a | Reenvios esgotados; motivo na DLQ | worker |

Os códigos "n/a" não saem na API HTTP; são gravados em `last_error`/`failure_reason`/`error_code` e em logs. Erros genéricos (`VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`) são os já existentes e tratados pelo [error.middleware.ts](../src/middlewares/error.middleware.ts) sem mudança (`[09:29] Bruno`).

## 7. Estratégias de resiliência

| Aspecto | Estratégia | Fonte |
| --- | --- | --- |
| Timeout | 10 s por chamada HTTP; estouro = falha com retry | `[09:42] Diego` |
| Retry | 1 envio + 5 reenvios (1m/5m/30m/2h/12h) | `[09:17] Larissa` |
| Fallback | DLQ em `webhook_dead_letter` + replay manual ADMIN | `[09:18] Diego` |
| Fallback do cliente | Sem email/alerta nesta fase | `[09:37] Larissa` |
| Isolamento | Cliente lento/offline não afeta a API nem outros pedidos (worker separado, outbox) | `[09:04] Bruno`, `[09:11] Diego` |
| Idempotência | `X-Event-Id` estável entre reenvios e replay | `[09:25] Diego` |
| Crash do worker | Reserva `PROCESSING` com recuperação por tempo limite | **[PROPOSTA]** |
| Cliente lento | Com worker único, uma chamada pode ocupar até 10 s; aceito na primeira fase | análise |
| Redirects | Não seguir redirects automaticamente | **[PROPOSTA]** |

## 8. Observabilidade

O projeto hoje só tem logs (Pino); não há métricas nem tracing ([src/shared/logger/index.ts](../src/shared/logger/index.ts)). A reunião cita apenas "Pino, nada novo" (`[09:29] Bruno`). O que segue em métricas e tracing é **[PROPOSTA]**.

**Logs (Pino, obrigatório).** Eventos estruturados no estilo de `server_started`/`shutdown_initiated`: `webhook_event_enqueued`, `webhook_delivery_succeeded`, `webhook_delivery_failed` (com `eventId`, `webhookId`, `attempt`, `statusCode`, `durationMs`, `errorCode`), `webhook_event_dead_lettered`, `webhook_replay_requested` (com o `userId` do ADMIN: `[09:36] Sofia`), `worker_started`/`worker_stopped`. **Nunca** logar `secret` nem o valor completo de `X-Signature`: adicionar `*.secret` e `*.previous_secret` aos `redactPaths` do logger.

**Métricas** (exposição em formato a definir, pois não há biblioteca de métricas no projeto):

| Métrica | Tipo | Para quê |
| --- | --- | --- |
| `webhook_outbox_pending` | gauge | Backlog; alerta se crescer |
| `webhook_event_lag_seconds` | histograma | `now − created_at` na entrega; meta p95 < 10 s |
| `webhook_delivery_attempts_total{result}` | counter | Sucesso/falha/timeout |
| `webhook_delivery_duration_seconds` | histograma | Latência dos clientes |
| `webhook_dead_letter_total` | counter/gauge | Falhas permanentes |

**Tracing.** Propagar o `requestId` (`req.id`, já declarado em [auth.middleware.ts](../src/middlewares/auth.middleware.ts)) para uma coluna `correlation_id` na outbox, para correlacionar a requisição de `changeStatus` com os logs do worker. Adoção de OpenTelemetry é questão em aberto, fora do escopo mínimo.

## 9. Integração com o sistema existente

| Arquivo | Como o módulo de webhooks se integra |
| --- | --- |
| [src/modules/orders/order.service.ts](../src/modules/orders/order.service.ts) | `changeStatus` chama `publishWebhookEvent(tx, order, from, to)` dentro do `$transaction`, depois de `tx.orderStatusHistory.create` e antes do `findUnique` final. Erro → rollback (`[09:40] Bruno`). `create` não é alterado (ver [RFC §5](RFC.md)). |
| [src/modules/orders/order.status.ts](../src/modules/orders/order.status.ts) | Fonte da lista de status válidos do filtro `events` (`OrderStatus`); `canTransition` continua a única autoridade sobre transições — o webhook só reage a transições já aceitas. |
| [src/shared/errors/app-error.ts](../src/shared/errors/app-error.ts) e [http-errors.ts](../src/shared/errors/http-errors.ts) | Erros `WEBHOOK_*` estendem `AppError`/`ValidationError`/`NotFoundError`, seguindo o precedente de `InsufficientStockError` e `InvalidStatusTransitionError`. Exportar em [index.ts](../src/shared/errors/index.ts). |
| [src/middlewares/error.middleware.ts](../src/middlewares/error.middleware.ts) | Sem mudanças: já serializa `AppError`, `ZodError` e erros Prisma (`[09:29] Bruno`). |
| [src/middlewares/auth.middleware.ts](../src/middlewares/auth.middleware.ts) | `authenticate` em todas as rotas; `requireRole('ADMIN')` apenas no replay (`[09:36] Larissa`). Já usado em [user.routes.ts](../src/modules/users/user.routes.ts). |
| [src/middlewares/validate.middleware.ts](../src/middlewares/validate.middleware.ts) | `validate({ body, params, query })` com schemas Zod do módulo, incluindo `https` obrigatório (`[09:23] Sofia`). |
| [src/routes/index.ts](../src/routes/index.ts) | Adicionar `webhooks: WebhookController` em `Controllers` e `router.use('/webhooks', …)` e a rota admin. |
| [src/app.ts](../src/app.ts) | `buildControllers` instancia `WebhookRepository`, `WebhookService`, `WebhookController` com o mesmo `prisma`, no padrão dos demais módulos. |
| [src/server.ts](../src/server.ts) | Modelo para o novo `src/worker.ts`: bootstrap, `logger.info`, handlers de `SIGINT`/`SIGTERM`, `prisma.$disconnect()`. API e worker permanecem processos distintos. |
| [src/config/database.ts](../src/config/database.ts) | O worker chama `createPrismaClient()`/importa `prisma` no seu processo: instância própria, mesma `DATABASE_URL` (`[09:30] Bruno`). |
| [src/config/env.ts](../src/config/env.ts) | Novas variáveis opcionais **[PROPOSTA]**: `WEBHOOK_POLL_INTERVAL_MS` (2000), `WEBHOOK_BATCH_SIZE`, `WEBHOOK_TIMEOUT_MS` (10000). |
| [src/shared/logger/index.ts](../src/shared/logger/index.ts) | Mesmo Pino no worker; ampliar `redactPaths` com secrets. |
| [prisma/schema.prisma](../prisma/schema.prisma) | Novos models com `@id @default(uuid()) @db.Char(36)` e `@@map` snake_case, via nova migration em `prisma/migrations/`. |
| [package.json](../package.json) | Script `"worker": "tsx watch --env-file=.env src/worker.ts"` (dev) e equivalente com `node dist/worker.js` (prod). |
| [tests/](../tests/) | Novos testes Vitest seguindo [tests/orders.test.ts](../tests/orders.test.ts) e [tests/helpers/factories.ts](../tests/helpers/factories.ts); `tests/setup.ts` precisará limpar as novas tabelas. |

Estrutura do módulo (`[09:27] Bruno`, `[09:28] Bruno`): `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, `webhook.processor.ts` (ou `webhook.worker.ts`), mais `publishWebhookEvent`.

> Esta entrega é documental: nenhum arquivo acima foi alterado.

## 10. Dependências e compatibilidade

- **Infra:** MySQL 8 existente ([docker-compose.yml](../docker-compose.yml)); nenhuma infraestrutura nova (`[09:07] Diego`).
- **Bibliotecas:** `crypto` nativo do Node (HMAC), `fetch` nativo do Node ≥ 20 (conforme `engines` do [package.json](../package.json)) com `AbortSignal.timeout` **[PROPOSTA]**; `uuid`, `zod`, `pino` já instalados.
- **Compatibilidade:** mudança aditiva. Nova migration; respostas existentes de `/orders` não mudam. `changeStatus` mantém a mesma assinatura e resposta.
- **Pessoas:** revisão de segurança da Sofia, com 2 dias úteis reservados antes do deploy (`[09:46] Sofia`).
- **Prazo:** 3 sprints, fim de novembro para a Atlas (`[09:45] Marcos`, `[09:46] Larissa`).

## 11. Critérios de aceite técnicos

1. Mudança de status gera a linha de outbox na mesma transação; falha forçada na inserção reverte o status (teste de integração).
2. Se nenhum webhook do customer assina o status, nenhuma linha é criada.
3. Entrega nominal em < 10 s a partir do commit (polling 2 s + chamada).
4. Corpo assinado com HMAC-SHA256 verificável com a secret do endpoint; secret distinta entre endpoints.
5. Após rotação, ambas as secrets são aceitas por 24 h e só a nova depois.
6. Falhas seguem o cronograma 1m/5m/30m/2h/12h e terminam em `webhook_dead_letter`.
7. Replay exige `ADMIN` (`OPERATOR` recebe 403), preserva `X-Event-Id` e registra o usuário em log.
8. URL `http://` recusada com `WEBHOOK_INVALID_URL`; payload > 64 KB nunca é enviado.
9. Todos os headers de [§5.3](#53-payload-e-headers-de-entrega) presentes; payload sem `items`.
10. `npm run worker` sobe processo independente; derrubar a API não interrompe entregas.
11. Todos os erros do módulo têm prefixo `WEBHOOK_` e saem no formato do error middleware.
12. Nenhum log contém secret.

## 12. Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- |
| Insert na outbox degrada `changeStatus` | Baixa | Alto | Índices; consulta de endpoints por `customer_id`; teste de carga |
| Worker único é SPOF e gargalo (timeout de 10 s) | Média | Alto | Supervisão, métrica de backlog, caminho de evolução (partição por `order_id`/lock) |
| Quebra de ordem após retry | Média | Médio | Documentar; cliente usa `timestamp`/`to_status` |
| Vazamento de secret | Média | Alto | Secret por endpoint, rotação 24 h, redaction, revisão da Sofia |
| Secret armazenada em claro | Média | Alto | Definir proteção em repouso na revisão de segurança |
| Crescimento da outbox | Média | Médio | Arquivamento a tratar em feature futura (`[09:08] Diego`) |
| Duplicatas mal tratadas pelo cliente | Alta | Médio | `X-Event-Id` + documentação destacada (`[09:26] Marcos`) |
| Autorização por customer ausente | Média | Alto | Questão aberta na [RFC](RFC.md); endurecer após revisão (`[09:37] Sofia`) |
