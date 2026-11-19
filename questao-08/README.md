# Questão 08 · Postmortem técnico de incidente em produção

**Framework escolhido:** R-I-S-E (Role · Input · Steps · Expectation)

**Modelo:** Claude Opus 5.5 (Anthropic, via claude.ai, esforço Médio). Escolhido pela capacidade de cruzar muitas fontes de dados numa análise única e de raciocinar sobre causas com incerteza explícita, o que um postmortem sob pressão exige.

## Cenário

Um incidente está em andamento no pico de tráfego, e o Doc Brown precisa de um postmortem técnico em 20 minutos para decidir entre **rollback do deploy v2.48.0** (que subiu ontem) e **scaling emergencial** (aumento de limits do RDS e do pool de conexões). Artefatos disponíveis: evento e changelog do deploy, métricas do Beacon dos últimos 30 minutos, trecho de log de um pod, estado da fila do Reactor e estado do cluster. A escolha do framework é livre entre os cinco do capítulo, com comparação explícita contra pelo menos dois candidatos.

## Prompt

```text
# ROLE
Você é um SRE sênior atuando como líder técnico de um incidente em
andamento, especialista em PostgreSQL (gestão de conexões), Kubernetes e
Argo CD. Você analisa evidências sob pressão, separa fato de hipótese e
segue a cultura de postmortem sem culpados (blameless).

# INPUT
Incidente em andamento no Chronos (API gateway da Hill Valley Tech) durante
pico de tráfego. O CTO, Doc Brown, precisa de um postmortem técnico em 20
minutos para decidir entre duas opções: rollback do deploy v2.48.0 (que subiu
ontem) ou scaling emergencial (aumento de limits do RDS e do pool de
conexões).

Evento do deploy anterior (ontem, 18:42 UTC):
Deploy chronos-api: v2.47.0 -> v2.48.0
Argo CD sync: 2026-04-23 18:42:11 UTC
Changelog:
- Adicionado endpoint POST /v2/transactions/batch
- Refatorado cliente do Ledger (pool de conexoes movido para nova biblioteca interna)
- Bump de psycopg 3.1.18 -> 3.2.0
- Reduzido timeout do Ledger de 5s para 2s

Métricas do Beacon nos últimos 30 minutos:
timestamp              p99_latency_ms  req_rate_s  err_rate_pct
2026-04-24 13:30 UTC   420             1200        0.2
2026-04-24 13:45 UTC   510             1450        0.3
2026-04-24 14:00 UTC   780             1780        0.8
2026-04-24 14:10 UTC   2400            2100        4.5
2026-04-24 14:15 UTC   5200            2400        8.2
2026-04-24 14:20 UTC   8100            2650        11.7

Trecho do log do pod chronos-api-79c4d8b9-xk2jp:
2026-04-24 14:19:48 [ERROR] [ledger-client] connection pool exhausted (max=20, active=20, waiting=147)
2026-04-24 14:19:49 [WARN]  [ledger-client] query timeout after 2000ms: SELECT ... FROM transactions WHERE ...
2026-04-24 14:19:49 [ERROR] [handler] POST /v2/transactions/batch failed: context deadline exceeded
2026-04-24 14:19:50 [ERROR] [ledger-client] connection reset by peer
2026-04-24 14:19:51 [WARN]  [circuit-breaker] ledger-client OPEN (threshold 50%, current 87%)
2026-04-24 14:19:52 [ERROR] [reactor] failed to publish message: chronos-api upstream error

Estado do Reactor (fila chronos-transactions):
- 50.127 mensagens acumuladas, crescendo a ~800/min.
- Consumer lag atual: 18 minutos e aumentando.

Estado do cluster:
- Chronos: 12/12 pods running (HPA no máximo).
- CPU médio dos pods: 62%.
- Memória média dos pods: 71%.
- Conexões ativas ao Ledger: 240/250 (limite do RDS).

# STEPS
Para produzir o postmortem, siga estes passos:
1. Monte a linha do tempo cruzando o deploy, a curva de tráfego, a latência
   e a taxa de erro. Explique por que o problema apareceu hoje no pico e não
   logo após o deploy de ontem.
2. Identifique o gargalo usando os números: relacione pods, tamanho do pool
   por pod, conexões ativas e limite do RDS, e compare com CPU e memória.
3. Trate cada item do changelog como uma hipótese de causa, com evidências a
   favor, evidências contra e nível de confiança.
4. Avalie as duas opções (rollback e scaling) quanto a: efeito esperado sobre
   o gargalo, tempo até surtir efeito, riscos (inclusive o que acontece com
   o limite de conexões do RDS se o pool ou o número de pods aumentar) e
   reversibilidade.
5. Recomende uma opção, com nível de confiança e as condições que mudariam a
   recomendação.
6. Considere o que acontece com a fila do Reactor quando o serviço voltar e
   como evitar uma segunda onda de carga no Ledger.
7. Separe explicitamente fatos (o que está nos dados) de hipóteses. Não
   invente dados; liste o que precisaria ser verificado e como.

# EXPECTATION
Um postmortem técnico em Markdown, legível em menos de 5 minutos, com a
decisão no topo:
1. Recomendação: a opção recomendada em até 3 linhas, com o nível de
   confiança.
2. Impacto até agora: números do incidente.
3. Linha do tempo: tabela com horário, evento e evidência.
4. Análise de causa: tabela com as colunas Hipótese, Evidências a favor,
   Evidências contra e Confiança.
5. Comparação das opções: tabela com os critérios do passo 4 lado a lado.
6. Plano de ação imediato: passos em ordem, com responsável sugerido e o
   critério de sucesso a verificar nos 15 minutos seguintes.
7. Riscos e pontos de atenção, incluindo a fila do Reactor.
8. Verificações pendentes e ações pós-incidente.
Sem culpar pessoas, sem texto fora dessa estrutura.
```

## Output

> O output foi mantido na íntegra. Os títulos foram rebaixados em dois níveis apenas para se encaixarem na hierarquia deste README.

### Postmortem técnico: Chronos API, degradação no pico de 2026-04-24

*Status: incidente em andamento. Documento preparado para decisão às ~14:40 UTC. Convenção: **[F]** = fato presente nos dados; **[H]** = hipótese a confirmar.*

#### 1. Recomendação

**Rollback para v2.47.0 via Argo CD, agora, com a fila do Reactor pausada antes e drenada de forma controlada depois.** O scaling emergencial não é viável: o RDS já está em 240/250 conexões, e mais pods ou pool maior batem no limite. Aumentar `max_connections` no RDS PostgreSQL exige reboot do Ledger no meio do incidente.
**Confiança: média-alta** de que o rollback é a melhor opção; **média** de que ele resolve sozinho, sem também conter a carga.

#### 2. Impacto até agora

| Indicador | Antes (13:30) | Agora (14:20) |
|---|---|---|
| Latência p99 **[F]** | 420 ms | 8.100 ms (19×) |
| Taxa de erro **[F]** | 0,2% | 11,7% (~310 req/s falhando a 2.650 req/s) |
| Requisições com erro desde 14:00 **[H]** | — | ~150 mil (estimativa por interpolação linear das amostras) |
| Fila `chronos-transactions` **[F]** | — | 50.127 mensagens, +~800/min, lag de 18 min e subindo |
| Conexões ao Ledger **[F]** | — | 240/250 (96% do limite do RDS) |
| Endpoint afetado nos logs **[F]** | — | `POST /v2/transactions/batch` falhando por deadline |

#### 3. Linha do tempo (UTC)

| Horário | Evento | Evidência |
|---|---|---|
| 04-23 18:42 | Deploy v2.47.0 → v2.48.0 (endpoint batch, nova lib de pool, psycopg 3.2.0, timeout do Ledger 5s → 2s) | Argo CD sync **[F]** |
| 04-23 18:42 → 04-24 13:30 | Sem degradação registrada. Tráfego provavelmente abaixo do pico **[H]** | Não há métricas desse período no material |
| 13:30 | Baseline: 1.200 req/s, p99 420 ms, erro 0,2% | Beacon **[F]** |
| 13:45 | Tráfego sobe (+21%); p99 510 ms, erro 0,3% | Beacon **[F]** |
| 14:00 | Primeira degradação: 1.780 req/s, p99 780 ms, erro 0,8% | Beacon **[F]** |
| ~14:02 | Fila do Reactor começa a acumular (18 min de lag às ~14:20) **[H]** | Estado do Reactor |
| 14:10 | **Ponto de ruptura:** p99 2.400 ms, **acima do novo timeout de 2s**; erro 4,5% | Beacon **[F]** |
| 14:15 | p99 5.200 ms, erro 8,2%; HPA já no máximo (12 pods) | Beacon e cluster **[F]** |
| 14:19:48 | Pool esgotado: 20/20 ativas, **147 requisições esperando** | Log do pod **[F]** |
| 14:19:49 | Query com timeout de 2.000 ms; batch falha por deadline | Log **[F]** |
| 14:19:50 | `connection reset by peer` no cliente do Ledger | Log **[F]** |
| 14:19:51 | Circuit breaker do Ledger **ABERTO** (87% de falha) | Log **[F]** |
| 14:20 | 2.650 req/s (2,2× o baseline), p99 8,1 s, erro 11,7% | Beacon **[F]** |

**Por que só hoje, e não logo após o deploy:** o deploy de ontem às 18:42 provavelmente pegou um período de tráfego baixo **[H]**. O defeito ficou latente até a carga dobrar. Com 2,2× o tráfego, as consultas ficam mais lentas, passam de 2s e começam a estourar o novo timeout. As conexões ficam ocupadas por mais tempo e as filas de espera crescem, até o pool esgotar e o circuit breaker abrir. É uma falha **sob carga**, não uma falha imediata da versão.

#### 4. Análise de causa

**O gargalo são as conexões com o Ledger, não a computação [F].** São 12 pods × pool de 20 = **240 conexões**, exatamente as 240/250 do RDS. CPU (62%) e memória (71%) estão abaixo da saturação, e o HPA (que escala por CPU) chegou a 12 pods, o que **somou** conexões em vez de aliviar. O sistema não está sem capacidade de processar: está esperando por conexão (147 na fila de um único pod).

| Hipótese | Evidências a favor | Evidências contra | Confiança |
|---|---|---|---|
| **H1. Nova lib de pool** mantém conexões ocupadas por mais tempo (não devolve ao pool após timeout ou erro, ou difere da antiga em tamanho e reuso) | Pool esgotado com 147 esperando [F]; `connection reset by peer` [F], compatível com conexões quebradas não descartadas [H]; mudou no mesmo deploy [F] | Não sabemos o tamanho do pool nem o uso de conexões na v2.47.0; com a mesma carga, a versão antiga também poderia esgotar | **Média** |
| **H2. Timeout reduzido de 5s para 2s** gera cascata sob carga (cancelamentos, retentativas, conexões em estado inconsistente) | A ruptura às 14:10 coincide com o p99 passando de 2.000 ms [F]; logs de timeout de 2.000 ms [F] | Timeout menor deveria **liberar** conexões mais cedo; sem retentativas, ele reduziria a pressão. Não sabemos se o cliente faz retry | **Média** |
| **H3. Endpoint batch** segura conexões por muito mais tempo que as rotas antigas | É a rota que aparece falhando no log [F]; operações em lote tendem a ter consultas maiores e mais lentas [H] | Não temos a participação do batch no tráfego nem duração das consultas por rota; o log mostra uma amostra de um pod | **Média-baixa** |
| **H4. psycopg 3.1.18 → 3.2.0** muda comportamento de conexão ou cancelamento | Mudança de versão minor no mesmo deploy [F] | Nenhum sinal específico nos logs aponta para o driver | **Baixa** |
| **H5. Só o pico de tráfego** (v2.48.0 inocente) | Tráfego 2,2× maior [F] | A ruptura é desproporcional: erro de 0,2% para 11,7% com 2,2× de tráfego, enquanto CPU e memória sobram; o pool esgotar com o mesmo número de pods sugere conexões presas, não falta de capacidade [H] | **Baixa-média** |
| **H6. O próprio Ledger (RDS) está lento** (CPU, locks, I/O) | `connection reset by peer` pode vir do servidor [F]; consultas passando de 2s [F] | Não temos métricas do RDS no material | **Indeterminada.** Verificar primeiro (seção 8) |

**Leitura combinada [H]:** o mais provável é a soma de H1, H2 e H3. A refatoração do pool e o timeout menor mudaram como as conexões são ocupadas e liberadas. O endpoint batch adicionou consultas pesadas. Sob o pico, essa combinação esgotou um orçamento de conexões que já estava no limite físico do RDS.

#### 5. Comparação das opções

| Critério | Rollback para v2.47.0 | Scaling emergencial (RDS + pool) |
|---|---|---|
| **Efeito sobre o gargalo** | Remove as quatro mudanças suspeitas de uma vez (H1 a H4). Se a causa estiver na v2.48.0, devolve o comportamento de conexões que suportava o tráfego anterior | **Piora ou não age.** Mais pool ou mais pods = mais de 250 conexões → `too many connections` imediato. Mais conexões também aumentam a contenção no banco. Se as conexões estão presas (H1), elas só ficam presas em maior número |
| **Tempo até surtir efeito** | ~5 a 10 min (sync do Argo CD + rollout de 12 pods) | Pool: minutos, mas inútil sem RDS. `max_connections` é parâmetro **estático** no RDS: exige **reboot** (minutos de indisponibilidade). Trocar a classe da instância: 10 a 30+ min com failover |
| **Riscos** | (a) Clientes que já usam `POST /v2/transactions/batch` recebem 404; (b) migração de banco incompatível na v2.48.0 (não sabemos se existe); (c) pods extras durante o rollout (maxSurge) somam conexões além de 250; (d) não resolve se a causa for H5 ou H6 | **Reboot do Ledger** derruba todas as transações durante o pico; parâmetros altos de conexão consomem memória do RDS; mudança grande e irreversível no meio do incidente |
| **Reversibilidade** | Alta: voltar à v2.48.0 é outro sync do Argo CD | Baixa: reboot já aconteceu; reverter também exige reboot |
| **Ataca a causa provável?** | Sim (H1 a H4) | Não, só trata o sintoma |

**Condições que mudariam a recomendação:**
- **Se a v2.48.0 tiver migração de banco incompatível com a v2.47.0**, o rollback de código não é seguro. Nesse caso, conter carga (pausar Reactor, rate limit no batch se houver flag) e escalar o `@chronos-core` para um hotfix.
- **Se o RDS estiver saturado (H6)** (CPU acima de 90%, locks, I/O no limite), nenhuma das duas opções resolve sozinha. A prioridade passa a ser reduzir carga e investigar as consultas.
- **Se a v2.47.0 já esgotava o pool** em picos parecidos no histórico, H5 ganha peso. O rollback ainda ajuda, mas a solução é capacidade com pooler (PgBouncer ou RDS Proxy), não `max_connections`.

#### 6. Plano de ação imediato

| # | Ação | Responsável sugerido | Critério de sucesso |
|---|---|---|---|
| 1 | Decisão registrada e comunicada em `#oncall-chronos` e no canal de status | Incident Commander (IC) | Decisão publicada com horário |
| 2 | **Gate de segurança (3 min):** confirmar se a v2.48.0 tem migração de banco e quem chama `/v2/transactions/batch` | `@chronos-core` | "Sem migração incompatível" confirmado → seguir; caso contrário, ver seção 5 |
| 3 | **Pausar ou reduzir os consumers do Reactor** da fila `chronos-transactions` | Dono do Reactor | Consumo próximo de zero; a fila continua crescendo, mas a carga sobre o Chronos e o Ledger cai |
| 4 | Em paralelo, coletar evidência rápida: `pg_stat_activity` por estado e aplicação, métricas do RDS (CPU, conexões, locks) | DBA ou dono do Ledger | Snapshot salvo no ticket do incidente |
| 5 | **Rollback:** desativar auto-sync do Argo CD e executar `argocd app rollback` para a revisão da v2.47.0 | SRE de plantão | Rollout concluído e app `Healthy` |
| 6 | Acompanhar as conexões durante o rollout (risco de surge acima de 250) | SRE de plantão | Conexões ≤ 250; se pods novos falharem por conexão, avançar o rollout pod a pod |
| 7 | **Verificação nos 15 min após o rollout** | IC | Erro < 1% (meta: voltar a 0,2–0,3%); p99 < 1.000 ms; conexões < 200 (80% do RDS); nenhum log de `pool exhausted`; circuit breaker **FECHADO** |
| 8 | **Drenar a fila de forma controlada:** retomar consumers com ~25% da concorrência e aumentar em degraus a cada 5 min | Dono do Reactor + IC | Em cada degrau: conexões < 200, p99 < 1.000 ms, erro < 1%; se algum limite estourar, voltar um degrau |
| 9 | Manter o auto-sync do Argo CD desativado até a correção ou revert da v2.48.0 estar na branch principal | `@chronos-core` | Responsável e prazo registrados no ticket |

**Se o critério do passo 7 falhar em 15 minutos:** o problema não está (só) na v2.48.0. Escalar para H6 ou H5 e priorizar redução de carga: rate limit, manter o Reactor pausado e acionar o DBA.

#### 7. Riscos e pontos de atenção

- **Segunda onda vinda do Reactor.** Há 50 mil mensagens acumuladas, crescendo ~800/min. Se os consumers drenarem na velocidade máxima após a recuperação, o Ledger recebe um pico maior que o original e o incidente se repete. A drenagem tem de ser em degraus (passo 8), com as conexões como métrica de controle. No ritmo de entrada atual, a fila **não diminui** enquanto o consumo for menor que ~800/min mais a vazão normal: o tempo de drenagem total precisa ser estimado com a vazão real.
- **Mensagens possivelmente perdidas ou duplicadas.** O log mostra `failed to publish message` [F]. Pode haver transações que nunca entraram na fila, e as retentativas podem gerar duplicatas. É preciso verificar idempotência e reconciliar com o Ledger.
- **Funcionalidade removida pelo rollback.** Clientes do endpoint batch passam a receber erro. É preciso comunicar os times consumidores.
- **Margem de conexões quase nula.** Mesmo na v2.47.0, 12 pods × pool já encostam no limite do RDS [H, depende do pool da v2.47]. Qualquer novo pico pode repetir o problema. O HPA, ao escalar por CPU, **aumenta** a pressão sobre o banco.
- **Auto-sync desativado** é um estado temporário perigoso: precisa de dono e prazo para não ficar esquecido.
- **Não culpar a mudança individual.** As quatro alterações juntas, num mesmo release, sem teste de carga no pico, é uma lacuna de **processo**, não de pessoas.

#### 8. Verificações pendentes e ações pós-incidente

**Verificações (durante e logo após o incidente):**

| O que verificar | Como | Por quê |
|---|---|---|
| Tamanho e comportamento do pool na v2.47.0 | Código e config da lib antiga vs nova | Confirma ou descarta H1 |
| Quem ocupa as 240 conexões e em que estado (`active`, `idle in transaction`, `idle`) | `pg_stat_activity` agrupado por `application_name` e `state` | Conexões ociosas presas confirmam H1; também revela se outros serviços usam o mesmo limite |
| Saúde do RDS (CPU, I/O, locks, consultas lentas) | CloudWatch/Performance Insights do RDS | Confirma ou descarta H6 |
| Picos históricos de tráfego na v2.47.0 e uso de conexões | Beacon, últimos 30 dias | Peso de H5 |
| Participação do batch no tráfego e duração das consultas por rota | Métricas por endpoint no Beacon | Peso de H3 |
| Se o cliente do Ledger faz retentativa após timeout | Código do cliente e logs | Peso de H2 |
| Origem do `connection reset by peer` | Logs do RDS (servidor encerrou?) vs rede | Distingue H1 de H6 |
| Mensagens com falha de publicação | Logs do Reactor e reconciliação com o Ledger | Integridade dos dados |
| Migrações de banco na v2.48.0 | Repositório `hvt/chronos-api` | Segurança do rollback |

**Ações pós-incidente:**

1. **Postmortem blameless** em até 5 dias úteis, com linha do tempo final e causa raiz confirmada.
2. **Orçamento de conexões:** regra de que `maxReplicas do HPA × pool por pod` (somando todos os serviços) fique ≤ 80% de `max_connections`, validada no CI.
3. **Pooler de conexões** (RDS Proxy ou PgBouncer) entre o Chronos e o Ledger, para desacoplar o número de pods do número de conexões no banco.
4. **Teste de carga a 2× o pico** antes de releases que mexem no acesso ao Ledger.
5. **Releases menores:** separar troca de lib de pool, bump de driver, mudança de timeout e endpoint novo em deploys distintos, com canário.
6. **Feature flag** para o endpoint batch e rate limit por rota, permitindo desligá-lo sem rollback.
7. **Alertas antecipados:** conexões no RDS acima de 80%, requisições esperando no pool acima de 0 por mais de 1 min, circuit breaker aberto.
8. **Backpressure no Reactor:** concorrência máxima de consumo configurável e controle de drenagem documentado em runbook.
9. **Rever o HPA** para não escalar além do que o orçamento de conexões suporta.


## Justificativa

### Por que R-I-S-E

O cenário tem três características que o R-I-S-E cobre melhor do que os outros frameworks do capítulo:

1. **Muitos dados heterogêneos que precisam ser cruzados** (changelog, série de métricas, logs, fila, cluster). O **Input** dá a eles um lugar próprio, separado das instruções, e o modelo consegue tratar tudo ali como evidência.
2. **Uma análise que exige um roteiro de raciocínio específico**: cruzar pods × pool × limite do RDS, tratar cada item do changelog como hipótese, avaliar as duas opções pelos mesmos critérios e pensar na fila do Reactor. Esse roteiro são os **Steps**. Sem ele, a tendência de um modelo é resumir os logs em vez de analisar os números.
3. **Uma saída orientada à decisão de um CTO em 20 minutos**, definida na **Expectation**: recomendação no topo, tabelas comparáveis e critérios de sucesso verificáveis.

O **Role** completa o quadro: um líder técnico de incidente, especialista em conexões PostgreSQL, que separa fato de hipótese e segue a cultura blameless.

### Como cada componente aparece no prompt e no output

| Componente | Onde está no prompt | Efeito visível no output |
|---|---|---|
| **Role** | Seção `# ROLE`: SRE sênior liderando o incidente, especialista em PostgreSQL, Kubernetes e Argo CD, blameless | Postura de líder de incidente: gate de segurança antes do rollback, responsáveis sugeridos por ação, e a seção de riscos atribui o problema a uma lacuna de processo, "não de pessoas". |
| **Input** | Seção `# INPUT`: o pedido do Doc Brown com as duas opções e todos os artefatos do enunciado, sem alteração | Todas as conclusões se apoiam nos dados fornecidos, marcadas como **[F]** quando vêm deles. |
| **Steps** | Seção `# STEPS`: 7 passos de raciocínio | Cada passo gerou uma parte da análise: a explicação de por que o problema só apareceu no pico (passo 1), a conta 12 × 20 = 240 conexões (passo 2), a tabela de hipóteses H1 a H6 (passo 3), a constatação de que `max_connections` exige reboot do RDS (passo 4), as condições que mudariam a recomendação (passo 5), a drenagem da fila em degraus (passo 6) e a convenção [F]/[H] (passo 7). |
| **Expectation** | Seção `# EXPECTATION`: 8 seções em ordem, decisão no topo, sem culpar pessoas e sem texto fora da estrutura | A estrutura foi seguida exatamente, com a recomendação e o nível de confiança logo no início. |

### Comparação com T-A-G

- **O que se ganharia:** o **Goal** ancoraria explicitamente todo o prompt na decisão "rollback ou scaling", reforçando o foco.
- **O que se perderia:** o T-A-G não tem um componente para os dados nem para o papel do especialista. Os artefatos ficariam misturados à Task e o roteiro de análise na Action, sem distinção entre o que é evidência e o que é instrução. Num prompt com seis blocos de dados, essa separação é o que permite exigir "separe fato de hipótese" de forma coerente. A postura de líder de incidente e a cultura blameless, que vêm do Role, também teriam que ser improvisadas dentro da Task.
- **Balanço:** a vantagem do T-A-G já está coberta, porque a Expectation do R-I-S-E pede "decisão no topo". O R-I-S-E fica com o foco do T-A-G e ainda ganha Input e Role.

### Comparação com C-A-R-E

- **O que se ganharia:** o **Example** permitiria fornecer um postmortem modelo da empresa, padronizando o formato com incidentes anteriores e facilitando a comparação histórica.
- **O que se perderia:** o cenário não traz nenhum postmortem de referência, e inventar um durante um incidente custaria tempo e poderia enviesar o modelo para as causas e conclusões do exemplo, justamente quando a análise precisa partir apenas dos dados. Além disso, a **Action** do C-A-R-E descreve o que fazer, mas não tem a vocação de roteiro de raciocínio dos Steps. Foram os Steps que levaram o modelo a fazer a conta pods × pool e a avaliar o reboot do RDS, que são o centro da recomendação.
- **Balanço:** o C-A-R-E seria a melhor escolha numa empresa com um template de postmortem consolidado e num momento sem pressão de tempo, como na versão final do postmortem, dias depois. Para a decisão em 20 minutos, o R-I-S-E é mais adequado.

### Frameworks descartados antes da comparação

O **B-A-B** serve para transformar um artefato de um estado para outro (como o manifest da questão 05), o que não é o caso aqui. O **R-T-F** não tem espaço para os dados nem para o roteiro de análise: tudo teria que caber na Task, e a qualidade dependeria do modelo inferir sozinho o que analisar.

## Observações sobre o output

- A análise técnica está correta e acerta os pontos centrais do cenário: o gargalo são as conexões (12 pods × 20 = 240 de 250), não CPU nem memória; o HPA somou conexões em vez de aliviar; o scaling é inviável porque `max_connections` no RDS PostgreSQL exige reboot; e a fila do Reactor pode causar uma segunda onda se for drenada de uma vez. A estimativa de cerca de 150 mil requisições com erro confere com a interpolação das métricas, e foi corretamente marcada como hipótese.
- A convenção [F]/[H], criada pelo modelo para atender ao passo 7, tornou visível o que é dado e o que é inferência, algo essencial para um CTO decidir sob pressão. O modelo também foi crítico com as próprias hipóteses, apontando, por exemplo, que um timeout menor deveria liberar conexões mais cedo, e não prendê-las.
- Pontos fracos: o documento ficou longo demais para "legível em menos de 5 minutos", com 8 seções e várias tabelas extensas; o horário "~14:40 UTC" do cabeçalho não está nos dados e não foi marcado como hipótese; e o plano assume que os consumers do Reactor geram carga sobre o Chronos e o Ledger, uma suposição razoável, mas não marcada como [H], já que o enunciado não diz em que direção o tráfego da fila flui.
- O que faria diferente: limitar a Expectation em tamanho (por exemplo, "no máximo uma página, com tabelas de até 5 linhas") e pedir uma versão de 5 linhas para ser lida na call, com o documento completo como anexo.
