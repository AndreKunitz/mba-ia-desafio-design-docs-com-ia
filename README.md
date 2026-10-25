# Da Reunião ao Documento: Design Docs do Sistema de Webhooks de Notificação de Pedidos

> O enunciado original do desafio está no repositório base: https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia (também preservado no histórico git deste repositório).

## Sobre o desafio

O repositório contém um Order Management System (Node.js, TypeScript, Express, Prisma e MySQL) sem nenhum mecanismo de notificação externa. Uma reunião técnica de ~55 minutos, registrada em [TRANSCRICAO.md](TRANSCRICAO.md), decidiu construir um sistema de webhooks de saída para avisar clientes B2B sobre mudanças de status de pedidos, mas nada além da transcrição foi documentado.

A minha tarefa foi transformar essa transcrição, cruzada com o código existente, em um pacote de design docs (PRD, RFC, FDD, ADRs e Tracker) acionável o bastante para o time começar a implementar, sem inventar requisitos e sem alterar o código da aplicação. A parte difícil não é escrever, é separar o que foi **decidido**, o que foi **descartado ou adiado** e o que apenas foi **mencionado**, e manter cada afirmação rastreável.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code** (extensão VS Code, modelo Claude Sonnet 5.5) | Ferramenta única de produção: exploração do código e da transcrição, planejamento, redação dos documentos, scripts de validação e commits. |
| **Scripts Python gerados pelo Claude** | Geração do `TRACKER.md` a partir de uma lista estruturada e validação automática: timestamps existentes na transcrição, caminhos de arquivo existentes, links internos resolvendo. |

## Workflow adotado

1. **Contextualização.** Pedi à IA que lesse o enunciado, a transcrição e o código (`order.service.ts`, `order.status.ts`, erros, middlewares, rotas, `server.ts`, schema Prisma) e **propusesse um plano antes de escrever qualquer coisa**. Só segui depois de aprovar o plano e responder às perguntas em aberto (idioma, local dos arquivos, commits, autoria).
2. **ADRs primeiro** (8 arquivos): as decisões formam o esqueleto dos demais documentos.
3. **RFC**, consolidando a proposta sobre os ADRs, com alternativas descartadas e questões em aberto.
4. **FDD**, com fluxos, contratos, matriz `WEBHOOK_*`, observabilidade e a seção "Integração com o sistema existente".
5. **PRD**, por último entre os grandes documentos, como consolidação de alto nível.
6. **Tracker**, gerado por script com validação automática.
7. **README** (este arquivo) e revisão final contra a checklist de critérios de aceite.

Tudo foi feito em uma branch separada (`docs/design-docs-webhooks`), com um commit por documento, para abrir PR contra a `main` do fork. Nenhum arquivo de `src/`, `prisma/` ou `tests/` foi alterado.

**Regra de ouro para a IA:** tudo que não tem origem na transcrição ou no código precisa ser marcado como **[PROPOSTA]** e listado ao fim do tracker, em vez de entrar como se fosse decisão.

## Prompts customizados

**Prompt 1: contextualização com plano antes de executar**

```text
Este é um exercício do meu MBA de Engenharia com IA que requer que eu gere artefatos
utilizando IA, seguindo um passo a passo bem definido que está transcrito em INSTRUCTIONS.md.

Primeiramente, entenda o que é este projeto, vasculhando o codebase e os docs disponíveis.

Após um entendimento do projeto, gere um plano de como você fará o entendimento do que
deve ser feito no exercício e confirme comigo antes de prosseguir.
```

**Prompt 2: execução dirigida, com guard-rails contra alucinação**

```text
Siga o plano aprovado, na ordem do enunciado: ADRs, RFC, FDD, PRD, Tracker, README.
Regras:
- Escreva tudo em português (pt-BR), direto em docs/, sem criar artifacts.
- Toda afirmação deve ter origem em [hh:mm] Nome da TRANSCRICAO ou em um caminho de arquivo real.
- O que NÃO entra (email, rate limiting, dashboard, arquivamento, ordering global) deve
  aparecer só como "fora de escopo" ou "questão em aberto", nunca como requisito.
- Se precisar decidir algo que a reunião não decidiu, marque como [PROPOSTA] e leve para o RFC.
- Não altere src/, prisma/ nem tests/. Um commit por documento, em branch separada.
- Valide por script antes de concluir: timestamps, caminhos de arquivo e links.
```

## Iterações e ajustes

Foram **três ciclos de geração e correção**, mais enxutos do que os 3 a 5 sugeridos no enunciado, porque o plano prévio já tinha mapeado as ambiguidades. Os principais ajustes:

1. **Contagem de tentativas.** A reunião diz "5 tentativas" mas lista 5 intervalos (1m/5m/30m/2h/12h) e fala em "quase 15 horas" (a soma real é 14 h 36 min). Em vez de escolher em silêncio, adotei 1 envio + 5 reenvios, que é a leitura coerente com os números, registrei a ambiguidade no ADR-002 e abri uma questão no RFC.
2. **Erro de 64 KB versus atomicidade.** Ao escrever o FDD percebi que "erra se passar de 64 KB" (Sofia) conflita com "rollback se a outbox falhar" (Bruno): um erro dentro de `changeStatus` bloquearia a mudança de status. Voltei ao RFC, acrescentei a questão 11 e deixei a proposta (DLQ direta) marcada como **[PROPOSTA]** no FDD.
3. **Retry versus ordering.** O FDD inicial descrevia ordering por `created_at` sem ressalvas. Ao detalhar o retry, ficou claro que um evento reagendado pode ser ultrapassado pelo seguinte do mesmo pedido. Passei a documentar isso como limitação, em vez de repetir a garantia da reunião.
4. **Rotação de secret.** A reunião diz que a secret antiga "fica válida por 24 h", mas não como isso se traduz num envio de saída. Não inventei a regra: virou questão em aberto no RFC e proposta no FDD.
5. **Nome dos ADRs.** O `docs/adrs/README.md` original sugeria `0001-…`; o enunciado exige `ADR-NNN-…`. Reescrevi o índice.
6. **Tracker.** Para respeitar "sem inventar", separei no fim do tracker os itens **[PROPOSTA]** (sem origem na reunião). A validação por script confirmou que todos os timestamps existem na transcrição e todos os caminhos existem no repositório. Resultado: 167 linhas, ~83% com fonte `TRANSCRICAO` e 29 com fonte `CODIGO`.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [docs/PRD.md](docs/PRD.md): o quê e por quê (produto).
2. [docs/RFC.md](docs/RFC.md): a proposta técnica, alternativas e questões em aberto, para revisão.
3. [docs/adrs/](docs/adrs/README.md): cada decisão isolada.
   - [ADR-001 Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
   - [ADR-002 Retry, backoff e DLQ](docs/adrs/ADR-002-retry-backoff-e-dlq.md)
   - [ADR-003 HMAC-SHA256 por endpoint](docs/adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md)
   - [ADR-004 At-least-once e X-Event-Id](docs/adrs/ADR-004-at-least-once-com-x-event-id.md)
   - [ADR-005 Worker separado em polling](docs/adrs/ADR-005-worker-separado-em-polling.md)
   - [ADR-006 Reuso dos padrões do projeto](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
   - [ADR-007 Publicação transacional e snapshot](docs/adrs/ADR-007-publicacao-transacional-e-payload-snapshot.md)
   - [ADR-008 Contrato de entrega](docs/adrs/ADR-008-contrato-de-entrega-payload-headers-e-limites.md)
4. [docs/FDD.md](docs/FDD.md): como construir, em detalhe (fluxos, contratos, erros, integração).
5. [docs/TRACKER.md](docs/TRACKER.md): origem de cada item.

A transcrição de origem é [TRANSCRICAO.md](TRANSCRICAO.md) (não alterada).
