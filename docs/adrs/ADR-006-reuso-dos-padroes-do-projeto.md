# ADR-006: Reuso dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceito
- **Data:** reunião técnica (decisão em `[09:30] Larissa`)
- **Decisores:** Larissa, Bruno, Diego
- **Relacionados:** [ADR-005](ADR-005-worker-separado-em-polling.md), [ADR-007](ADR-007-publicacao-transacional-e-payload-snapshot.md), [FDD, seção "Integração com o sistema existente"](../FDD.md#9-integração-com-o-sistema-existente)

## Contexto

A base de código segue convenções claras: cada domínio é um módulo em `src/modules/<nome>` com controller, service, repository, routes e schemas (`[09:27] Bruno`). Os erros são `AppError` com `errorCode` e subclasses específicas, tratados por um middleware central. O logger é Pino em todo o projeto. A feature não deve introduzir um novo estilo de código.

## Decisão

O módulo de webhooks segue o mesmo padrão dos demais, reutilizando o máximo possível:

| Padrão existente | Como será reutilizado | Arquivo de referência |
| --- | --- | --- |
| Estrutura modular | Nova pasta `src/modules/webhooks` com controller, service, repository, routes e schemas | [src/modules/orders/](../../src/modules/orders/) |
| `AppError` e subclasses | Erros do módulo estendem `AppError`/`ConflictError`/etc. com códigos `WEBHOOK_*` | [src/shared/errors/](../../src/shared/errors/) |
| Error middleware central | Já trata `AppError`, Zod e Prisma; nenhuma mudança necessária | [src/middlewares/error.middleware.ts](../../src/middlewares/error.middleware.ts) |
| Pino | Único logger, inclusive no worker | [src/shared/logger/index.ts](../../src/shared/logger/index.ts) |
| Validação Zod | Schemas por módulo e middleware `validate` | [src/middlewares/validate.middleware.ts](../../src/middlewares/validate.middleware.ts) |
| Autenticação e roles | `authenticate` no CRUD; `requireRole('ADMIN')` no replay | [src/middlewares/auth.middleware.ts](../../src/middlewares/auth.middleware.ts) |
| Registro de rotas | Novo router em `buildApiRouter` | [src/routes/index.ts](../../src/routes/index.ts) |
| IDs | UUID em todas as tabelas novas | [prisma/schema.prisma](../../prisma/schema.prisma) |

Prefixo **`WEBHOOK_`** para todos os códigos de erro do módulo, ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` (`[09:28] Bruno`, `[09:29] Larissa`).

## Alternativas Consideradas

1. **Módulo isolado com convenções próprias** (novo logger, tratamento de erro próprio). Descartada: duplica infraestrutura e fragmenta a observabilidade (`[09:29] Bruno`).
2. **Serviço independente/microserviço de webhooks.** Plausível para o futuro; descartada por ora porque a reunião decidiu manter "mesma stack, mesmo banco" (`[09:11] Diego`).

## Consequências

**Positivas**

- Curva de aprendizado e revisão menores; o middleware de erro funciona sem alterações.
- Logs e erros homogêneos entre API e worker.

**Negativas / trade-offs**

- O módulo herda limitações dos padrões atuais. O logger e o `env` hoje carregam `service: 'order-management-api'` e leem `env.ts`, então o worker compartilha essa configuração.
- Acoplamento ao monólito: o módulo evolui junto com o deploy da API e do schema do banco.
