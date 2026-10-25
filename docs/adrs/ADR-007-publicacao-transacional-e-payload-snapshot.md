# ADR-007: Publicação transacional via `publishWebhookEvent(tx, …)` com payload em snapshot

- **Status:** Aceito
- **Data:** reunião técnica (decisões em `[09:41] Diego` e `[09:52] Bruno`)
- **Decisores:** Bruno, Diego, Larissa
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md), [FDD](../FDD.md)

## Contexto

O ponto crítico da integração é `OrderService.changeStatus` ([src/modules/orders/order.service.ts](../../src/modules/orders/order.service.ts)). Hoje a transação faz `tx.order.update`, `tx.orderStatusHistory.create` e movimentação de estoque. Se a inserção do evento ficasse fora dessa transação, perderia a garantia do outbox (`[09:41] Diego`). Também era preciso decidir o que a linha da outbox guarda: o payload pronto ou só o `order_id`.

## Decisão

1. **Inserir o evento dentro da transação de `changeStatus`.** Se a inserção falhar, ocorre rollback: "não pode ter caso de status mudar e evento não sair" (`[09:40] Bruno`).
2. Expor uma função **`publishWebhookEvent(tx, order, fromStatus, toStatus)`** que recebe o `tx` da transação corrente, em vez de injetar um repository inteiro no `OrderService` (`[09:41] Bruno`, `[09:41] Diego`: "função pura recebendo o tx").
3. **Filtrar na inserção:** a função só insere se algum webhook ativo do customer assina aquele status; caso contrário, nenhuma linha é criada (`[09:34] Bruno`).
4. **Payload renderizado na inserção (snapshot):** o evento reflete o estado do pedido no momento da mudança, ainda que o pedido mude depois (`[09:52] Larissa`, `[09:52] Diego`).

## Alternativas Consideradas

1. **Injetar `WebhookRepository` no `OrderService`.** Descartada em favor da função recebendo `tx`: menos acoplamento do módulo de pedidos (`[09:41] Diego`).
2. **Filtrar apenas no envio.** Descartada: grava linhas inúteis na outbox (`[09:34] Bruno`).
3. **Guardar só o `order_id` e renderizar o payload no envio.** Descartada: o payload poderia refletir um estado posterior ao da mudança, "caso esquisito" (`[09:52] Larissa`).

## Consequências

**Positivas**

- Atomicidade total entre mudança de status e evento.
- Eventos fiéis ao momento da transição e imunes a edições posteriores.
- Menor volume na outbox graças ao filtro na inserção.

**Negativas / trade-offs**

- A transação de `changeStatus` fica mais pesada (consulta de webhooks assinantes + insert), e uma falha no módulo de webhooks passa a bloquear a mudança de status.
- O payload gravado ocupa mais espaço que uma simples referência.
- O `OrderService` ganha uma dependência de importação do módulo de webhooks (a função), exigindo cuidado para evitar ciclos.
