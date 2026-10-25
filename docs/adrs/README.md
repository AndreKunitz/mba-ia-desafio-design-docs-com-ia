# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) da feature de **Sistema de Webhooks de Notificação de Pedidos**.
Cada decisão arquitetural fica em um arquivo `ADR-NNN-titulo-em-kebab-case.md`, no formato MADR (Status, Contexto, Decisão, Alternativas Consideradas, Consequências).

| ADR | Decisão | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL | Aceito |
| [ADR-002](ADR-002-retry-backoff-e-dlq.md) | Retry com backoff exponencial e DLQ em tabela separada | Aceito |
| [ADR-003](ADR-003-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação | Aceito |
| [ADR-004](ADR-004-at-least-once-com-x-event-id.md) | Entrega at-least-once com `X-Event-Id` | Aceito |
| [ADR-005](ADR-005-worker-separado-em-polling.md) | Worker em processo separado com polling de 2 s | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões existentes do projeto | Aceito |
| [ADR-007](ADR-007-publicacao-transacional-e-payload-snapshot.md) | Publicação transacional e payload em snapshot | Aceito |
| [ADR-008](ADR-008-contrato-de-entrega-payload-headers-e-limites.md) | Contrato de entrega: payload, headers e limites | Aceito |
