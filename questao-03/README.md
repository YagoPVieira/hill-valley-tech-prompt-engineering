# Questão 03 · Relatório de redução de custos cloud

**Framework:** T-A-G (Task · Action · Goal)

**Modelo:** Claude Opus 5.5 (Anthropic, via claude.ai, esforço Médio). Escolhido por ser forte em análise de dados com cálculos encadeados e em escrita executiva, as duas coisas que o relatório para a CEO exige.

## Cenário

Goldie (CEO) apresentou à diretoria a meta de **15% de redução no custo cloud até o fim do trimestre, sem degradar SLA**. Doc Brown passou ao time o breakdown de custos AWS do último mês, em CSV. O relatório para a Goldie precisa trazer as oportunidades de economia priorizadas por impacto, quanto cada uma representa em percentual da conta total, o esforço de implementação (baixo, médio, alto) e os riscos ou pré-requisitos de cada uma.

## Prompt

```text
# TASK
Analise o breakdown de custos AWS do último mês da Hill Valley Tech, abaixo em
CSV, e produza um relatório de oportunidades de redução de custo cloud para a
CEO, Goldie Wilson.

servico,categoria,custo_mensal_usd,uso_medio_pct,observacao
EC2 reservada,compute,4200,72,contrato de 1 ano
EC2 on-demand,compute,8200,45,workloads variaveis
EKS,compute,6700,58,3 clusters
RDS PostgreSQL,databases,8200,62,multi-AZ
ElastiCache Redis,databases,2100,40,cluster de producao
S3 Standard,storage,3100,,5 buckets principais
EBS gp3,storage,1600,68,volumes de producao
CloudWatch Logs,observability,2800,,retencao de 90 dias
CloudWatch Metrics,observability,900,,
Data Transfer Out,network,1900,,trafego entre regioes
NAT Gateway,network,1200,,3 gateways ativos
Lambda,compute,900,30,~12M invocacoes/mes

# ACTION
1. Calcule o custo mensal total e o valor em USD que corresponde a 15% dele.
2. Para cada serviço, identifique oportunidades concretas de economia usando
   os dados do CSV (uso médio, observação e tipo de serviço).
3. Para cada oportunidade, estime a economia mensal em USD (use estimativas
   conservadoras) e o percentual que ela representa sobre a conta total.
4. Classifique o esforço de implementação:
   - Baixo: ajuste de configuração, sem mudança de arquitetura, feito em dias.
   - Médio: mudança de infraestrutura que exige testes, feita em semanas.
   - Alto: mudança de arquitetura ou de contrato, exige planejamento.
5. Liste os riscos e pré-requisitos de cada oportunidade, deixando explícito
   qualquer impacto possível em SLA.
6. Priorize as oportunidades por impacto (economia em USD) e, em caso de
   empate, pelo menor esforço.
7. Some a economia das oportunidades e compare com a meta de 15%.
8. Quando faltar um dado no CSV (ex.: uso_medio_pct vazio), não invente:
   declare a premissa usada e indique o que precisaria ser medido.
9. Entregue o relatório nesta estrutura:
   a) Resumo executivo em até 5 linhas: custo total, meta em USD, economia
      estimada e se a meta é alcançável.
   b) Tabela Markdown com as colunas: Prioridade, Oportunidade, Serviço,
      Economia (USD/mês), % da conta, Esforço, Riscos e pré-requisitos.
   c) Plano de execução em fases ao longo do trimestre, começando pelos
      ganhos rápidos.
   d) Premissas adotadas e dados que precisam ser coletados.

# GOAL
Dar à Goldie uma base clara para decidir como atingir a meta de reduzir em
pelo menos 15% o custo cloud até o fim do trimestre, sem degradar o SLA. O
relatório deve ser escrito em linguagem de negócio, ser realista (preferir
subestimar a economia do que prometer o que não se cumpre) e deixar claro
quais iniciativas, juntas, garantem a meta.
```

## Output

### Relatório de redução de custos AWS: Hill Valley Tech

**Para:** Goldie Wilson, CEO
**Base:** fatura AWS do último mês

#### a) Resumo executivo

O custo mensal total é de **US$ 41.800**, e a meta de 15% corresponde a **US$ 6.270/mês**. As 11 iniciativas com estimativa confiável somam **US$ 6.700/mês (16,0%)**, com potencial adicional de US$ 200 em transferência de dados. **A meta é alcançável, mas a margem é estreita (cerca de US$ 430)**: ela só é atingida executando as 8 primeiras iniciativas da tabela. Nenhuma iniciativa reduz redundância (Multi-AZ, NAT por zona), por isso o SLA fica preservado. Os ganhos são de *run-rate* mensal ao fim do trimestre. A economia acumulada dentro do próprio trimestre será menor, porque as iniciativas entram em vigor ao longo dele.

#### b) Oportunidades priorizadas

| Prioridade | Oportunidade | Serviço | Economia (USD/mês) | % da conta | Esforço | Riscos e pré-requisitos |
|---|---|---|---|---|---|---|
| 1 | Comprar instâncias reservadas de 1 ano (sem pagamento antecipado) para o banco, mantendo Multi-AZ | RDS PostgreSQL | 1.800 | 4,3% | Alto | Compromisso contratual de 12 meses. Exige confirmar que o tipo e o tamanho da instância não vão mudar no período. O uso de 62% não justifica reduzir a instância sem dados de pico. **Sem impacto em SLA** (Multi-AZ mantido). |
| 2 | Redimensionar instâncias on-demand subutilizadas (uso médio de 45%) | EC2 on-demand | 1.200 | 2,9% | Médio | Precisa medir picos de CPU e memória por instância (p95, 30 dias). **Risco de SLA:** se o tamanho for subdimensionado, há degradação em picos. Mitigação: reduzir um tamanho por vez, com testes de carga e rollback pronto. |
| 3 | Ajustar autoscaling de nós, consolidar *node groups* e usar Spot em ambientes não produtivos | EKS | 1.000 | 2,4% | Médio | Pré-requisito: *requests/limits* corretos nos pods e PodDisruptionBudgets. **Risco de SLA:** Spot só fora de produção; em produção, apenas o ajuste de escala. |
| 4 | Contratar Compute Savings Plan de 1 ano para a base estável restante, **após** o redimensionamento | EC2 on-demand | 700 | 1,7% | Alto | Compromisso de 12 meses. Deve cobrir só o consumo mínimo constante, para não pagar por capacidade ociosa. Depende da iniciativa 2. Sem impacto em SLA. |
| 5 | Reduzir o tipo de nó do cluster (uso médio de 40%) | ElastiCache Redis | 500 | 1,2% | Médio | Precisa medir pico de memória, *evictions* e conexões. **Risco de SLA:** a falta de memória gera *evictions* e aumento de latência. Manter réplica e fazer a troca em janela de manutenção. |
| 6 | Aplicar políticas de ciclo de vida e Intelligent-Tiering nos 5 buckets | S3 Standard | 450 | 1,1% | Baixo | Falta dado de padrão de acesso por bucket. Risco de custo de recuperação se dados "frios" forem lidos com frequência. Sem impacto em SLA se Glacier não for usado em dados operacionais. |
| 7 | Reduzir retenção de 90 para 30 dias, arquivar o histórico no S3 e cortar logs de debug em produção | CloudWatch Logs | 400 | 1,0% | Baixo | Verificar exigências de auditoria e compliance antes de reduzir a retenção. Sem impacto em SLA. Perda de investigação rápida de incidentes antigos, mitigada pelo arquivo no S3. |
| 8 | Criar VPC Endpoints (S3, ECR, demais serviços AWS) para tirar tráfego dos NAT Gateways | NAT Gateway | 300 | 0,7% | Baixo | Manter os 3 gateways, um por zona, porque removê-los reduziria a resiliência. Pré-requisito: medir quanto do tráfego do NAT vai para serviços AWS. Validar regras de rede. |
| 9 | Ajustar memória e timeout das funções com AWS Lambda Power Tuning | Lambda | 150 | 0,4% | Baixo | Testar latência após o ajuste. Risco baixo, com rollback imediato. |
| 10 | Remover volumes desanexados e snapshots antigos | EBS gp3 | 100 | 0,2% | Baixo | Confirmar com os donos antes de apagar. Manter snapshots exigidos por política de backup. |
| 11 | Eliminar métricas customizadas e alarmes sem uso | CloudWatch Metrics | 100 | 0,2% | Baixo | Não remover métricas ligadas a alarmes de SLA. Precisa de inventário de métricas por uso. |
| Potencial | Reduzir tráfego entre regiões (cache, réplicas locais, compressão) | Data Transfer Out | 200* | 0,5% | Alto | *Não contado na meta. A origem do tráfego é desconhecida e pode exigir mudança de arquitetura. **Risco de SLA** se envolver réplicas ou failover entre regiões. |
| — | Nenhuma ação imediata (uso de 72%, contrato vigente) | EC2 reservada | 0 | 0,0% | — | Reavaliar na renovação: migrar para Savings Plan, que dá mais flexibilidade. |

**Total contado na meta: US$ 6.700/mês (16,0%)**, contra a meta de US$ 6.270 (15%). Com o potencial de Data Transfer, o total vai a US$ 6.900 (16,5%).

**Iniciativas que, juntas, garantem a meta:** as prioridades 1 a 8 somam US$ 6.350 (15,2%). As prioridades 9 a 11 (US$ 350) formam a margem de segurança. Se a iniciativa 1 ou a 2 atrasar ou entregar menos, a meta não fecha. Por isso, as duas devem ser acompanhadas semanalmente.

#### c) Plano de execução no trimestre

**Fase 1, semanas 1 a 3 (ganhos rápidos, esforço baixo):** iniciativas 6, 7, 8, 9, 10 e 11. Juntas somam cerca de US$ 1.500/mês e não mudam a arquitetura. Nessa mesma fase começa a coleta de métricas de pico (EC2, Redis, RDS), que é pré-requisito das fases seguintes.

**Fase 2, semanas 3 a 8 (otimização com testes):** iniciativas 2, 3 e 5. Todas passam por homologação, testes de carga e rollback planejado, aplicadas primeiro fora de produção. Somam cerca de US$ 2.700/mês.

**Fase 3, semanas 6 a 10 (compromissos contratuais):** iniciativa 1, com o RDS reservado, aprovada assim que a estabilidade do banco estiver confirmada. Em seguida, a iniciativa 4: o Savings Plan só é contratado depois que o redimensionamento da fase 2 estabilizar, para não comprometer capacidade que seria cortada. Somam cerca de US$ 2.500/mês.

**Semanas 10 a 13 (consolidação):** validar a economia real na fatura via Cost Explorer, investigar Data Transfer Out como iniciativa extra e revisar os alertas de orçamento.

#### d) Premissas e dados a coletar

As premissas adotadas foram as seguintes. Descontos de reserva e Savings Plan foram estimados abaixo das tabelas públicas da AWS: 25% em vez de 30 a 40% para o RDS, e cerca de 25% sobre parte do EC2. A economia de redimensionamento foi limitada a uma fração da ociosidade observada, porque uso médio não revela picos. Multi-AZ, os 3 NAT Gateways e a capacidade de produção são tratados como requisitos de SLA e não foram reduzidos.

Para S3, CloudWatch Logs, CloudWatch Metrics, Data Transfer e NAT, o CSV não traz uso médio. As estimativas são percentuais conservadores sobre o custo, e não medições.

Os dados que precisam ser coletados são estes:

* **EC2:** picos de CPU e memória (p95/p99) por instância em 30 dias.
* **Redis:** memória máxima, *evictions* e conexões.
* **S3:** padrão de acesso por bucket, via S3 Storage Lens.
* **CloudWatch Logs:** divisão entre custo de ingestão e de armazenamento, e exigências legais de retenção.
* **NAT Gateway:** volume de tráfego por destino, via VPC Flow Logs.
* **Data Transfer:** origem e destino do tráfego entre regiões.
* **EBS:** inventário de volumes desanexados e snapshots.
* **CloudWatch Metrics:** lista de métricas customizadas sem consulta.
* **Lambda:** distribuição de duração e memória por função.

Se quiser, posso transformar este relatório em um documento editável para compartilhar com a Goldie ou exportar em Word ou PDF.

## Justificativa

A **Task** (seção `# TASK`) define o que fazer e para quem: analisar o CSV, que vem completo no prompt, e produzir um relatório de redução de custos para a CEO. A **Action** (seção `# ACTION`) detalha os passos em ordem: calcular a meta em USD, estimar economia e percentual, classificar esforço com critérios objetivos, apontar riscos de SLA, priorizar por impacto, tratar dados ausentes sem inventar e seguir a estrutura do relatório. O **Goal** (seção `# GOAL`) ancora tudo na meta de 15% sem degradar SLA e no público executivo, o que orientou o tom e a postura conservadora das estimativas.

## Observações sobre o output

- Os cálculos conferem: total de US$ 41.800, meta de US$ 6.270, soma das iniciativas de US$ 6.700 (16,0%) e subtotal das prioridades 1 a 8 de US$ 6.350 (15,2%). Os subtotais de cada fase do plano também batem.
- O Goal teve efeito visível: o modelo preservou explicitamente Multi-AZ e os 3 NAT Gateways por causa do SLA, deixou Data Transfer fora da conta da meta por falta de dados e alertou que a economia é de *run-rate* (o acumulado dentro do trimestre será menor), um ponto que a Goldie precisaria saber antes de levar à diretoria.
- Pontos fracos: a margem sobre a meta é de só US$ 430 e depende de estimativas, não de medições, já que 5 dos 12 serviços não têm uso médio no CSV. Além disso, o modelo terminou com uma oferta de exportar o relatório, texto fora da estrutura pedida.
- O que faria diferente: incluir no Action "classifique a confiança de cada estimativa (alta, média, baixa)", para a Goldie enxergar quais números são medidos e quais são suposições, e terminar o prompt com "entregue apenas o relatório".
