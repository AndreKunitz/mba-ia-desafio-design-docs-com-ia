# ADR-005: Worker em processo separado com polling de 2 segundos

- **Status:** Aceito
- **Data:** reunião técnica (decisão em `[09:10] Larissa`)
- **Decisores:** Diego, Larissa, Bruno, Marcos
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-retry-backoff-e-dlq.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md), [FDD](../FDD.md)

## Contexto

Os clientes consideram "tempo real" qualquer latência abaixo de 10 segundos (`[09:02] Marcos`). Alguém precisa ler a outbox e executar as chamadas HTTP. O MySQL não possui mecanismo nativo equivalente a `LISTEN/NOTIFY` do Postgres, e triggers só executam SQL, sem avisar um processo externo (`[09:09] Diego`).

## Decisão

- O worker faz **polling a cada 2 segundos**, buscando os eventos pendentes mais antigos em lote pequeno, processando e marcando (`[09:09] Diego`). A latência mínima no pior caso é de 2 s, aceita pelo PM (`[09:10] Larissa`, `[09:10] Marcos`).
- O worker roda como **processo separado da API**, para que reiniciar a API não derrube o worker (`[09:11] Diego`).
- Nova entry-point **`src/worker.ts`**, no mesmo molde de [src/server.ts](../../src/server.ts), com o script **`npm run worker`** (`[09:11] Larissa`). A lógica de processamento fica no módulo de webhooks, em `webhook.worker.ts` ou `webhook.processor.ts` (`[09:28] Bruno`).
- Mesmo banco e mesma `DATABASE_URL`, mas **instância própria de `PrismaClient`**, pois o cliente é por processo (`[09:30] Bruno`; ver [src/config/database.ts](../../src/config/database.ts)).
- Inicialmente **um único worker**, o que dá a ordenação por `order_id` descrita em [ADR-004](ADR-004-at-least-once-com-x-event-id.md).

## Alternativas Consideradas

1. **Worker dentro da instância da API.** Descartada: reiniciar a API perde o worker (`[09:11] Diego`).
2. **Trigger no banco para acordar o worker.** Descartada: o MySQL não notifica processo externo; seria preciso improvisar escrita em arquivo ou chamada a endpoint (`[09:09] Diego`).
3. **Broker externo com consumo reativo (Redis Streams).** Descartada junto com o [ADR-001](ADR-001-outbox-no-mysql.md) por custo de infraestrutura.

## Consequências

**Positivas**

- Latência de ~2 s, bem abaixo da meta de 10 s.
- Falha ou deploy da API não interrompe entregas, e vice-versa.
- Reutiliza stack, Prisma e logger existentes.

**Negativas / trade-offs**

- Consultas periódicas ao banco mesmo sem eventos.
- Um worker único é um ponto único de falha e limita o throughput; escalar exige tratar ordering e concorrência (lock pessimista ou partição, `[09:13] Diego`).
- Um novo processo para deploy, monitorar e encerrar (shutdown gracioso, como `server.ts` faz hoje).
