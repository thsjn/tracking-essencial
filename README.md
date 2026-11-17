# Tracking Essencial

Skill para o **Claude Code** que transforma o assistente num engenheiro de dados de rastreamento: Meta Pixel e Conversions API com deduplicação, GA4 e Measurement Protocol, Google Tag Manager, data layer, server-side, atribuição e LGPD.

Gratuita, em português, sem nenhum dado de cliente: todo ID nos exemplos é placeholder.

## O que ela faz

Quando você pede algo como "instala o pixel com CAPI neste formulário", "por que o Gerenciador não bate com a loja" ou "monta o dataLayer deste checkout", o Claude passa a seguir as regras de quem implementa tracking em produção:

- `event_id` único gerado uma vez e enviado igual ao Pixel e à CAPI
- dados pessoais normalizados e com hash SHA-256 no servidor, nunca no navegador
- pixel disparado só depois da confirmação do servidor em formulários
- `client_id` e `session_id` do GA4 lidos dos cookies certos para o Measurement Protocol
- filtro de robôs e de tráfego interno antes de qualquer envio
- checklist de QA, de LGPD e de go-live

## Conteúdo (20 seções)

1. Filosofia e princípios
2. Taxonomia de eventos
3. Meta Pixel e Conversions API (deduplicação, normalização, ITP, erros comuns, payload completo)
4. Google Analytics 4 e Measurement Protocol
5. Google Tag Manager (data layer, convenções, SPA, conversões otimizadas do Google Ads)
6. Server-side: quando usar Pixel, CAPI ou sGTM
7. Atribuição e UTM
8. Identity resolution
9. Schema canônico de data layer
10. Pipelines de dados e webhooks
11. BigQuery e SQL analítico do GA4
12. LGPD, GDPR e Consent Mode v2
13. Debugging e QA
14. Google Ads
15. TikTok Events API
16. Validação de schema com Zod
17. Impacto dos scripts no Core Web Vitals
18. Monitoramento e alertas
19. Variáveis de ambiente
20. Boas práticas universais

## Instalação

```bash
# Claude Code (pasta de skills do usuário)
git clone https://github.com/thsjn/tracking-essencial ~/.claude/skills/tracking-essencial
```

No Windows, a pasta é `C:\Users\<voce>\.claude\skills\tracking-essencial`. Reinicie o Claude Code. A skill é acionada sozinha quando o pedido fala de pixel, CAPI, GA4, GTM, dataLayer, UTM ou conversão.

## Versão completa

A versão completa tem mais **31 seções**, todas tiradas de implementações reais em produção:

- hospedar o GTM Server-Side no Cloud Run e sair da Stape sem derrubar o tracking
- o incidente do custom loader e o runbook de migração em 9 fases
- auditoria de container herdado: os defeitos que se repetem
- a corrida do cookie que zera `_fbp` e `_fbc` no primeiro acesso
- conversão offline do Google Ads para checkout de terceiro (Hotmart, Kiwify) pela Data Manager API
- Safari apagando o `gclid` e como recuperar
- editar o GTM pela API, com versão e publicação
- tracking server-side sem sGTM, com custo quase zero
- teste A/B medido no servidor, com P(B>A) e amostra planejada

Quer a versão completa? Me chame no LinkedIn: https://www.linkedin.com/in/thiagonogueira-tracking

## Licença

MIT. Use, adapte e compartilhe.
