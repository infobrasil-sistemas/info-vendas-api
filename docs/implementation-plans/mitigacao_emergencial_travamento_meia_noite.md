# Mitigação Emergencial: Travamento Silencioso às 00h (UTC/BRT)

Este documento registra a descoberta, ação de mitigação e próximos passos referentes ao problema crônico de *silent freeze* (erro 504) que afetava a API de produção pontualmente por volta da meia-noite.

## 1. Descoberta e Diagnóstico

Durante a investigação profunda sobre os congelamentos (que resultavam em retornos `504 Gateway Timeout` constantes e falta de logs de requisições processadas durante a madrugada), foi identificada a seguinte correlação:

- **O Sintoma:** A API parava de processar requisições exatamente após as `00:00:00 BRT` (Horário de Brasília).
- **A Causa Raiz Suspeita:** No arquivo `src/modules/status/status.service.ts`, existia uma rotina agendada (Cron Job) programada para rodar diariamente:
  ```typescript
  @Cron(CronExpression.EVERY_DAY_AT_3AM)
  async cleanup() { ... }
  ```
- **O Problema de Fuso Horário:** Como a imagem Docker da API (`node:22-alpine`) roda, por padrão, no fuso horário **UTC**, `03:00 AM UTC` corresponde a exatamente **00:00 (Meia-noite) BRT**.
- **Impacto no Banco de Dados:** Essa rotina invoca a exclusão (`deleteMany`) de milhares de registros antigos. Com o passar dos meses desde a sua criação (Maio/2026), o crescimento da tabela e possíveis contenções de locks no PostgreSQL/Prisma começaram a enfileirar e travar o *Event Loop* do Node.js, bloqueando qualquer nova requisição e causando o "congelamento" irreversível da API (uma vez que não havia reinicialização automática para *unhealthy* contêineres).

## 2. Ação de Mitigação Emergencial

Para estancar imediatamente o "sangramento" da API (evitar quedas em produção nesta e nas próximas noites), a seguinte medida emergencial foi aplicada:

**Ação:** O agendamento da rotina de limpeza foi **desativado via comentário no código**. 
**Arquivo Alterado:** `src/modules/status/status.service.ts`

```typescript
  /**
   * [MITIGAÇÃO EMERGENCIAL] Cron desativado temporariamente pois estava
   * causando travamento silencioso da API às 00h (03:00 UTC).
   * @Cron(CronExpression.EVERY_DAY_AT_3AM)
   */
  async cleanup() { ... }
```

Com esta alteração em vigor no próximo *deploy*, a API não mais sofrerá o gatilho de travamento de meia-noite gerado pela contenção de locks do PostgreSQL nessa operação. 

*Nota:* O ambiente também recebeu em paralelo uma infraestrutura de `autoheal` via Docker Compose para reiniciar agressivamente a API em caso de travamentos residuais.

## 3. Próximos Passos (Investigação Aprofundada)

A tabela de status (`status_logs`) começará a acumular registros indeterminadamente enquanto a rotina estiver desativada. Embora a tabela seja de baixo volume individual, uma solução adequada deverá ser implementada no futuro. 

As seguintes abordagens devem ser avaliadas e resolvidas na próxima janela de manutenção:

- **Estratégia de Exclusão em Lotes (Batching):** Refatorar a exclusão para apagar em lotes limitados (`LIMIT 500`) com *sleeps* iterativos, ao invés de um único comando `deleteMany` pesadíssimo.
- **Normalização de Fuso Horário (TZ):** Adicionar a variável de ambiente `TZ=America/Sao_Paulo` ao Docker, para que `CronExpression.EVERY_DAY_AT_3AM` de fato rode de madrugada no Brasil, fora de pico comercial.
- **Análise do Prisma Query Engine:** Investigar por que um único bloqueio no banco PostgreSQL resultou no travamento total das outras instâncias e do pool (possível *thread starvation*).
