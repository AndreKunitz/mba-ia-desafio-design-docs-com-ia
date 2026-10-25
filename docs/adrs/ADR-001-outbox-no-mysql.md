# ADR-001: Padrão Outbox no MySQL para publicação de eventos de webhook

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão em `[09:08] Larissa`)
- **Decisores:** Larissa (Tech Lead), Diego, Bruno, Sofia, Marcos
- **Relacionados:** [ADR-005](ADR-005-worker-separado-em-polling.md), [ADR-007](ADR-007-publicacao-transacional-e-payload-snapshot.md), [RFC](../RFC.md), [FDD](../FDD.md)

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) querem ser notificados quando o status de um pedido muda, hoje consultando `GET /orders` em polling (`[09:00] Marcos`). A mudança de status acontece em `OrderService.changeStatus` ([src/modules/orders/order.service.ts](../../src/modules/orders/order.service.ts)), dentro de uma única `prisma.$transaction` que atualiza `orders`, insere em `order_status_history` e movimenta estoque.

Fazer a chamada HTTP dentro dessa transação travaria a mudança de status de outros pedidos quando um cliente estivesse lento e obrigaria a decidir entre rollback ou perda do evento quando o cliente estivesse fora do ar (`[09:04] Bruno`). O time é pequeno e não opera outra infraestrutura além do MySQL (`[09:07] Diego`).

## Decisão

Adotar o **padrão Outbox no MySQL existente**: na mesma transação SQL que atualiza `orders` e `order_status_history`, inserir uma linha na tabela `webhook_outbox` com o evento. Um worker separado ([ADR-005](ADR-005-worker-separado-em-polling.md)) lê a tabela e faz as chamadas HTTP.

Características acordadas:

- A tabela tem índice no campo de status (`pendente`, `processando`, `falhou`, `entregue`) e em `created_at`; o worker lê apenas pendentes, em lote pequeno (`[09:08] Diego`).
- Chave primária em UUID, seguindo o restante do projeto (`[09:51] Larissa`; ver `@default(uuid()) @db.Char(36)` em [prisma/schema.prisma](../../prisma/schema.prisma)).
- Se a inserção na outbox falhar, a transação inteira sofre rollback (`[09:40] Bruno`).
- Arquivamento de linhas entregues após ~30 dias está **fora do escopo** desta feature (`[09:08] Diego`).

## Alternativas Consideradas

1. **Chamada HTTP síncrona dentro de `changeStatus`.** Descartada: acopla a latência do cliente à transação de pedidos e não tem resposta boa para cliente offline (`[09:04] Bruno`, `[09:06] Diego`).
2. **Redis Streams ou fila equivalente.** Descartada: exige subir e operar infraestrutura nova, "overengineering" para um time pequeno (`[09:07] Larissa`, `[09:07] Diego`).
3. **Publicar após o commit (fora da transação).** Descartada: se o processo cair entre o commit e a publicação, o status muda e o evento se perde (`[09:40] Bruno`, `[09:41] Diego`).

## Consequências

**Positivas**

- Atomicidade: se a transação commitou, o evento existe; se deu rollback, o evento some junto (`[09:06] Diego`).
- Nenhuma infraestrutura nova; reutiliza MySQL, Prisma e migrations.
- Isola a API da disponibilidade dos clientes.

**Negativas / trade-offs**

- A tabela cresce e passa a competir por I/O com o banco transacional; o arquivamento fica como dívida declarada.
- Latência mínima imposta pelo polling (ver [ADR-005](ADR-005-worker-separado-em-polling.md)).
- Cada mudança de status passa a ter uma escrita extra na transação mais pesada do sistema.
