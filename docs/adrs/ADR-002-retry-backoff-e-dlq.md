# ADR-002: Política de retry com backoff exponencial e DLQ em tabela separada

- **Status:** Aceito
- **Data:** reunião técnica (decisões em `[09:17] Larissa`)
- **Decisores:** Larissa, Diego, Bruno, Marcos
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-005](ADR-005-worker-separado-em-polling.md), [ADR-004](ADR-004-at-least-once-com-x-event-id.md), [FDD](../FDD.md)

## Contexto

Clientes podem estar indisponíveis (inclusive em manutenção planejada de duas horas, `[09:16] Diego`). O worker precisa decidir quando tentar novamente e quando desistir sem deixar eventos pendurados para sempre. Falhas definitivas precisam ficar guardadas para diagnóstico e reprocessamento.

## Decisão

- **Backoff exponencial com intervalos fixos de 1 min, 5 min, 30 min, 2 h e 12 h** (`[09:17] Diego`, confirmado em `[09:17] Larissa`). A soma dos intervalos é de 14 h 36 min, o que a reunião chamou de "quase 15 horas".
- Esgotados os reenvios, o evento é considerado falha permanente e **movido para a DLQ**.
- A **DLQ é uma tabela separada, `webhook_dead_letter`**, com payload, motivo da falha e timestamp (`[09:18] Diego`).
- O **reprocessamento é manual** pelo endpoint `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. Exige role `ADMIN` e registra em log quem executou o replay (`[09:36] Sofia`, `[09:36] Larissa`).
- **Timeout de 10 s** por chamada HTTP; estouro conta como falha e entra no fluxo de retry (`[09:42] Diego`).

> **Ponto de atenção (ver RFC, questões em aberto):** a reunião diz "5 tentativas", mas lista 5 intervalos. Este ADR adota a leitura consistente com os números: 1 envio inicial + 5 reenvios nos intervalos acima. A confirmação é pedida na revisão do RFC.

## Alternativas Consideradas

1. **Retry indefinido com backoff.** Descartada: o evento fica pendurado para sempre se o cliente sumir (`[09:15] Diego`).
2. **3 tentativas.** Descartada: cobriria só ~30 minutos e mataria o evento numa indisponibilidade de manhã ou numa manutenção de 2 h (`[09:16] Diego`).
3. **DLQ como status `failed` na própria outbox.** Descartada: polui a leitura da outbox principal e perde a função de evidência para debug e reprocessamento (`[09:18] Diego`).

## Consequências

**Positivas**

- Janela de ~14 h 36 min cobre indisponibilidades planejadas e a maioria dos incidentes.
- Outbox principal fica enxuta; a DLQ vira evidência auditável.
- Replay manual evita reenvios automáticos de eventos que já se mostraram problemáticos.

**Negativas / trade-offs**

- Um evento pode chegar ao cliente até ~15 h depois da mudança, ou nunca (DLQ), até alguém acionar o replay.
- O replay manual depende de um operador `ADMIN`; sem email ou alerta ao cliente (fora do escopo, `[09:37] Larissa`), a falha pode passar despercebida.
- Reenvios e replay geram duplicatas, tratadas por [ADR-004](ADR-004-at-least-once-com-x-event-id.md).
