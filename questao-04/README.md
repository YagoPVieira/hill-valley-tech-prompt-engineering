# Questão 04 · Relatório mensal de transações do Ledger

**Framework:** T-A-G (Task · Action · Goal)

**Modelo:** Claude Opus 5.5 (Anthropic, via claude.ai, esforço Médio). Escolhido por ser forte em SQL e em raciocinar sobre casos de borda (fuso horário, precisão numérica, uso de índices), que são onde uma query como esta costuma errar em silêncio.

## Cenário

Jennifer (PM) está fechando a apresentação para a Goldie sobre o crescimento de transações nos últimos 6 meses por categoria. Ela precisa dos números consolidados, mas não escreve SQL. Regras do enunciado: só entram transações com `status = 'completed'`; `amount_cents` deve aparecer em reais com 2 casas decimais; recorte dos últimos 6 meses corridos a partir de 2026-04-24; agrupamento por mês (`YYYY-MM`) e categoria, com quantidade e volume total; ordenação por mês e depois categoria.

## Prompt

````text
# TASK
Escreva uma query SQL para PostgreSQL que gere o relatório mensal de transações
do Ledger, o data warehouse da Hill Valley Tech. O relatório será usado pela PM,
Jennifer Parker, que não escreve SQL, numa apresentação para a CEO sobre o
crescimento de transações por categoria.

Esquema relevante:

CREATE TABLE transactions (
  id              BIGSERIAL PRIMARY KEY,
  customer_id     BIGINT NOT NULL REFERENCES customers(id),
  category        VARCHAR(32) NOT NULL,
  amount_cents    BIGINT NOT NULL,
  status          VARCHAR(16) NOT NULL,
  payment_method  VARCHAR(16),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at    TIMESTAMPTZ
);
CREATE INDEX idx_transactions_created_at ON transactions(created_at);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_transactions_category ON transactions(category);

CREATE TABLE customers (
  id          BIGSERIAL PRIMARY KEY,
  segment     VARCHAR(16) NOT NULL,
  country     CHAR(2) NOT NULL,
  signup_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

Categorias em produção hoje: subscription, one_time, refund e credit_adjustment.

# ACTION
1. Considere apenas transações com status = 'completed'.
2. Use como recorte os últimos 6 meses corridos a partir de 2026-04-24
   (de 2025-10-24 até o fim de 2026-04-24), com a data fixa no código para
   o resultado ser reproduzível. Use created_at como data de referência.
3. Converta as datas para o fuso America/Sao_Paulo antes de agrupar por mês,
   para que transações perto da virada do mês caiam no mês correto.
4. Agrupe por mês (formato texto 'YYYY-MM') e por categoria.
5. Traga duas métricas por linha: quantidade de transações e volume total em
   reais, convertendo amount_cents para reais com exatamente 2 casas decimais,
   sem divisão inteira.
6. Ordene por mês crescente e depois por categoria crescente.
7. Escreva o filtro de data de forma que o índice em created_at possa ser
   usado (sem aplicar funções sobre a coluna no WHERE).
8. Use apenas as tabelas necessárias; não faça joins sem necessidade.
9. Use nomes de coluna em português e autoexplicativos no resultado
   (ex.: mes, categoria, qtd_transacoes, volume_reais).
10. Responda nesta estrutura:
    a) Um bloco ```sql``` com a query completa e comentada.
    b) Uma explicação curta de cada decisão tomada (recorte, fuso, conversão
       de valores, uso de índice).
    c) Um exemplo ilustrativo do formato do resultado, com 3 linhas
       inventadas e identificadas como fictícias.
    d) Os pontos de atenção que a Jennifer precisa saber ao interpretar os
       números (ex.: meses parciais nas pontas do recorte, sinal dos valores
       de refund e credit_adjustment).

# GOAL
Entregar à Jennifer números consolidados corretos e fáceis de levar para a
apresentação, sem que ela precise entender SQL. A query deve ser correta antes
de tudo, reproduzível e eficiente num banco de produção com histórico completo,
e os pontos de atenção devem evitar que o crescimento seja interpretado de
forma errada na frente da CEO.
````

## Output

### a) Query

```sql
-- Relatório mensal de transações concluídas do Ledger, por categoria
-- Recorte fixo: 24/10/2025 00:00 até o fim de 24/04/2026, no horário de Brasília
SELECT
    -- Mês no fuso de São Paulo, para transações perto da virada do mês caírem no mês certo
    to_char(t.created_at AT TIME ZONE 'America/Sao_Paulo', 'YYYY-MM') AS mes,
    t.category                                                         AS categoria,
    COUNT(*)                                                           AS qtd_transacoes,
    -- Centavos para reais: divisão em numeric (sem truncamento) e arredondamento para 2 casas
    ROUND(SUM(t.amount_cents)::numeric / 100, 2)                       AS volume_reais
FROM transactions AS t                -- só a tabela necessária, sem join com customers
WHERE t.status = 'completed'          -- apenas transações concluídas
  -- Limites são constantes convertidas do horário de São Paulo para timestamptz.
  -- A coluna fica "nua" no WHERE, então o índice em created_at pode ser usado.
  AND t.created_at >= (TIMESTAMP '2025-10-24 00:00:00' AT TIME ZONE 'America/Sao_Paulo')
  AND t.created_at <  (TIMESTAMP '2026-04-25 00:00:00' AT TIME ZONE 'America/Sao_Paulo')  -- fim exclusivo: inclui todo o dia 24/04
GROUP BY mes, categoria
ORDER BY mes ASC, categoria ASC;
```

### b) Decisões tomadas

**Recorte.** As datas estão fixas no código, e por isso a query devolve o mesmo resultado em qualquer dia que for executada. O fim é escrito como "antes de 25/04 00:00" em vez de "até 24/04 23:59:59". Esse formato inclui o dia 24 inteiro, até a última fração de segundo, sem risco de perder transações.

**Fuso.** `created_at` é guardado como `TIMESTAMPTZ`, ou seja, um instante absoluto. A conversão com `AT TIME ZONE 'America/Sao_Paulo'` é aplicada em dois lugares. Nos limites do filtro, ela garante que "24/10/2025" significa meia-noite em Brasília, e não em UTC. No agrupamento, ela faz uma transação às 22h de 31/01 em Brasília (01h de 01/02 em UTC) contar em janeiro.

**Conversão de valores.** `amount_cents` é inteiro, e dividir inteiro por inteiro no PostgreSQL trunca o resultado. Por isso a soma é convertida para `numeric` antes da divisão por 100. Somar primeiro e dividir depois mantém a precisão exata, e `ROUND(..., 2)` garante as duas casas decimais.

**Índice.** O `WHERE` compara a coluna `created_at` sem nenhuma função aplicada sobre ela, contra valores constantes. Assim, o PostgreSQL pode usar `idx_transactions_created_at` para ler só o intervalo de 6 meses, e não o histórico inteiro. O índice em `status` sozinho ajuda pouco, porque a maioria das linhas tende a ser `completed`. Se essa query virar rotina, um índice composto `(created_at, status)`, ou um índice parcial `ON transactions(created_at) WHERE status = 'completed'`, melhora ainda mais a performance.

**Sem join.** Segmento e país do cliente não fazem parte do relatório, então a tabela `customers` não é consultada.

### c) Exemplo do formato do resultado

**Dados fictícios, apenas para ilustrar o formato:**

| mes | categoria | qtd_transacoes | volume_reais |
|---|---|---|---|
| 2025-10 | one_time | 1.250 | 87.430,50 |
| 2025-10 | refund | 42 | -3.120,00 |
| 2025-11 | subscription | 8.910 | 401.000,75 |

### d) Pontos de atenção para a apresentação

**Os meses das pontas são parciais.** Outubro/2025 cobre só 8 dias (24 a 31), e abril/2026 cobre só 24 dias. Compará-los com meses cheios vai mostrar uma "alta" ou "queda" que não existe. Para falar de crescimento, use novembro/2025 a março/2026, ou apresente média diária nos meses parciais.

**Refund e credit_adjustment não são receita.** Antes da apresentação, confirme com a engenharia se esses valores são gravados como negativos ou positivos. Se forem positivos, somar todas as categorias vai inflar o total. Em qualquer caso, apresente essas duas categorias separadas do crescimento de vendas. Um aumento de refund é um sinal de alerta, não de crescimento.

**O critério é "concluídas" com data de criação.** Transações criadas no período mas ainda pendentes ou falhas não aparecem. Uma transação criada em 30/03 e concluída em 02/04 conta em março. Se a query for rodada logo após o fim do período, os últimos dias podem subir um pouco depois, à medida que transações pendentes forem concluídas.

**Quantidade e volume contam histórias diferentes.** Uma categoria pode crescer em número de transações e cair em volume, ou o contrário, por mudança de ticket médio. Vale mostrar as duas métricas lado a lado.

**Todos os valores estão em reais.** Isso parte da premissa de que `amount_cents` está sempre em BRL, já que o esquema não tem coluna de moeda. Como há clientes com `country` diferente de BR, vale confirmar essa premissa antes de levar os números à CEO.

**Categorias novas.** Hoje são 4 categorias. Se surgir outra no período, ela aparecerá como linha nova automaticamente, e não deve ser confundida com crescimento das existentes.

## Justificativa

A **Task** (seção `# TASK`) define o entregável, uma query PostgreSQL para o relatório mensal, e traz o esquema completo e o contexto de uso pela PM. A **Action** (seção `# ACTION`) transforma cada regra do enunciado num passo verificável (status, recorte, agrupamento, métricas, ordenação) e acrescenta passos contra erros silenciosos: fuso horário, divisão inteira de centavos, uso do índice e joins desnecessários. O **Goal** (seção `# GOAL`) ancora a resposta na necessidade da Jennifer, números corretos e interpretáveis diante da CEO, o que justifica os pontos de atenção sobre meses parciais e valores de refund.

## Observações sobre o output

- A query está correta e cumpre todos os passos: o filtro deixa `created_at` sem função (o índice pode ser usado), os limites são convertidos do horário de Brasília, o fim exclusivo (`< 2026-04-25`) inclui o dia 24 inteiro e a soma vira `numeric` antes da divisão. O modelo seguiu a estrutura pedida, sem texto fora dela.
- O Goal teve efeito claro nos pontos de atenção: o modelo alertou que outubro e abril são parciais e distorcem o crescimento, que refund não é receita e que transações pendentes podem mudar os últimos dias do recorte.
- Pontos fracos: o exemplo fictício usa formatação brasileira (`87.430,50`), mas o PostgreSQL devolve `87430.50`, o que pode confundir quem for comparar; o modelo afirmou que "há clientes com `country` diferente de BR", algo que o enunciado não diz; e não avisou que meses ou categorias sem nenhuma transação simplesmente não aparecem no resultado.
- O que faria diferente: pedir que o exemplo use exatamente o formato de saída do PostgreSQL e incluir no Action "não afirme nada sobre os dados que não esteja no esquema ou no enunciado".
