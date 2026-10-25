# ADR-004: Garantia de entrega at-least-once com X-Event-Id para deduplicação

- **Status:** Aceito
- **Data:** reunião técnica (decisão em `[09:26] Larissa`)
- **Decisores:** Diego, Larissa, Sofia, Marcos
- **Relacionados:** [ADR-002](ADR-002-retry-backoff-e-dlq.md), [ADR-008](ADR-008-contrato-de-entrega-payload-headers-e-limites.md), [PRD](../PRD.md)

## Contexto

Com retry ([ADR-002](ADR-002-retry-backoff-e-dlq.md)) e replay de DLQ, o mesmo evento pode ser enviado mais de uma vez, por exemplo quando o cliente processa mas a resposta se perde ou estoura os 10 s. É preciso definir qual garantia de entrega a plataforma oferece.

## Decisão

- Garantir **at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e deve estar preparado (`[09:24] Diego`).
- Cada evento recebe um **UUID gerado quando entra na outbox**, enviado no header **`X-Event-Id`**. O cliente deduplica por esse identificador (`[09:25] Diego`).
- O mesmo `event_id` é reutilizado em todos os reenvios e no replay.
- A responsabilidade de deduplicar é do cliente. O PM documentará isso de forma destacada no portal de desenvolvedores (`[09:26] Marcos`).
- **Ordering:** só há ordem implícita por `order_id` enquanto houver um único worker, pela ordenação por `created_at` da outbox. Não é garantia global e é uma limitação conhecida (`[09:12] Diego`, `[09:13] Larissa`). Se o worker escalar, será preciso particionar por `order_id` ou usar lock pessimista (`[09:13] Diego`).

## Alternativas Consideradas

1. **Exactly-once.** Descartada: exigiria coordenação dos dois lados e é muito mais complexa; at-least-once com `event_id` "resolve 99% dos casos" (`[09:25] Diego`).
2. **At-most-once (sem retry).** Plausível, mas incompatível com o objetivo de não perder notificações quando o cliente está fora do ar.

## Consequências

**Positivas**

- Modelo simples, previsível e igual ao de Stripe e GitHub (`[09:25] Diego`).
- Habilita retry e replay sem risco de perda.

**Negativas / trade-offs**

- Transfere ao cliente a responsabilidade de dedup (`[09:25] Sofia`); clientes mal implementados podem processar duas vezes.
- Sem garantia de ordem global nem com múltiplos workers; o cliente pode precisar usar `timestamp` e `to_status` para reconciliar. Marcos informou que os clientes nunca pediram ordem global (`[09:14] Marcos`).
