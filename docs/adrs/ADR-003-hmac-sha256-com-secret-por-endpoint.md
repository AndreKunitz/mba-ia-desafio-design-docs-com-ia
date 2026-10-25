# ADR-003: Autenticação das entregas com HMAC-SHA256 e secret por endpoint

- **Status:** Aceito
- **Data:** reunião técnica (decisão em `[09:22] Sofia`)
- **Decisores:** Sofia (Segurança), Diego, Larissa, Bruno
- **Relacionados:** [ADR-008](ADR-008-contrato-de-entrega-payload-headers-e-limites.md), [FDD](../FDD.md)

## Contexto

Os eventos carregam dados de pedidos e saem para endpoints fora da infraestrutura da empresa. O cliente precisa validar que a requisição veio da plataforma e que o payload não foi adulterado (`[09:19] Sofia`). Já houve um cliente que vazou uma secret em log da própria aplicação (`[09:22] Diego`).

## Decisão

- Assinar o **corpo do request com HMAC-SHA256** e enviar o resultado no header `X-Signature` (`[09:20] Sofia`).
- **Cada endpoint de webhook tem uma secret única**, nunca uma secret global (`[09:21] Sofia`). A secret é gerada pela plataforma e devolvida na criação do webhook (`[09:31] Marcos`).
- A configuração do webhook guarda URL, secret, `customer_id` e estado ativo (`[09:21] Bruno`/`Sofia`).
- A secret é **rotacionável pela API**; na rotação, a secret antiga permanece válida por **24 horas** em paralelo, depois expira (`[09:21] Sofia`).
- A URL deve ser **HTTPS**; `http` é recusado por validação no schema Zod, o que segundo Sofia não é decisão arquitetural (`[09:23] Sofia`).
- Sofia revisa o código de geração de secret e de assinatura antes do deploy (reserva de 2 dias úteis, `[09:46] Sofia`).

## Alternativas Consideradas

1. **Secret global da plataforma.** Descartada: o vazamento de uma secret expõe todos os clientes (`[09:21] Sofia`).
2. **Sem assinatura, apenas TLS.** Plausível, mas não atende ao requisito de provar origem e integridade do payload (`[09:19] Sofia`).
3. **Outros algoritmos além de HMAC-SHA256.** Não discutidos; HMAC-SHA256 foi escolhido por ser "padrão de mercado", com bibliotecas disponíveis em qualquer cliente sério (`[09:20] Sofia`).

## Consequências

**Positivas**

- Raio de dano de um vazamento limitado a um endpoint.
- Rotação com grace period permite migração sem janela de indisponibilidade.
- Padrão amplamente suportado do lado do cliente.

**Negativas / trade-offs**

- Para assinar, a plataforma precisa guardar a secret de forma recuperável (não basta um hash); a proteção em repouso deve ser definida na revisão de segurança.
- Manter duas secrets válidas durante 24 h aumenta a complexidade do envio e da validação. A reunião não detalhou como a secret antiga é honrada em um envio *outbound*; o FDD traz uma proposta marcada como tal.
- O `X-Timestamp` é enviado separado (ver [ADR-008](ADR-008-contrato-de-entrega-payload-headers-e-limites.md)); como a assinatura cobre apenas o corpo, a detecção de replay pelo cliente é limitada. Ponto levado para a revisão de segurança.
