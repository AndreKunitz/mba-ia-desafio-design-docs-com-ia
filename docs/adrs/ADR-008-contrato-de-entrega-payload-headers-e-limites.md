# ADR-008: Contrato de entrega: payload enxuto, headers padronizados e limites

- **Status:** Aceito
- **Data:** reunião técnica (decisões entre `[09:23]` e `[09:45]`)
- **Decisores:** Diego, Sofia, Bruno, Larissa, Marcos
- **Relacionados:** [ADR-003](ADR-003-hmac-sha256-com-secret-por-endpoint.md), [ADR-004](ADR-004-at-least-once-com-x-event-id.md), [FDD, seção "Contratos públicos"](../FDD.md#5-contratos-públicos)

## Contexto

Os clientes precisam integrar com um formato previsível. A reunião fechou o formato do payload, os headers enviados, o limite de tamanho e o timeout. Embora detalhes secundários, eles definem o contrato público e afetam todos os clientes.

## Decisão

**Payload** (JSON, enxuto), com os campos: `event_id`, `event_type` (ex.: `"order.status_changed"`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido, como `total_cents`. **Não inclui `items`**; quem precisar de detalhes consulta `GET /orders/:id` (`[09:43] Diego`).

**Headers** (`[09:44] Diego`, `[09:44] Sofia`, `[09:45] Diego`):

| Header | Conteúdo |
| --- | --- |
| `X-Event-Id` | UUID do evento (dedup, [ADR-004](ADR-004-at-least-once-com-x-event-id.md)) |
| `X-Signature` | HMAC-SHA256 do corpo ([ADR-003](ADR-003-hmac-sha256-com-secret-por-endpoint.md)) |
| `X-Timestamp` | Timestamp do envio, permite ao cliente detectar replay attack |
| `X-Webhook-Id` | ID do cadastro de webhook que originou o envio |
| `Content-Type` | `application/json` |

**Limites:**

- **Payload máximo de 64 KB**; acima disso, **erro, sem truncar** (`[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa`). Tratado como requisito não funcional, não decisão arquitetural.
- **Timeout HTTP de 10 s** por chamada; estouro vira falha e entra em retry (`[09:42] Diego`).
- **Somente HTTPS** nas URLs cadastradas, via schema Zod (`[09:23] Sofia`).

## Alternativas Consideradas

1. **Incluir `items` no payload.** Descartada para não inflar o evento (`[09:43] Diego`, `[09:44] Bruno`).
2. **Truncar payloads grandes.** Descartada: "se chegou nesse tamanho, tem algo errado" (`[09:23] Sofia`).
3. **Sem `X-Webhook-Id`.** Descartada: clientes com vários cadastros não saberiam qual deles originou o envio (`[09:44] Sofia`).

## Consequências

**Positivas**

- Contrato pequeno, estável e fácil de documentar no portal de desenvolvedores (`[09:40] Marcos`).
- Payload enxuto reduz custo de rede e risco de estouro do limite.

**Negativas / trade-offs**

- O cliente que precisa de itens faz uma segunda chamada à API.
- O limite de 64 KB é, na prática, um teto de segurança. Como o evento inclui poucos campos, a rejeição indica bug, e não uso normal.
- Timeout de 10 s significa que um cliente lento ocupa o worker até 10 s por evento (worker único).
