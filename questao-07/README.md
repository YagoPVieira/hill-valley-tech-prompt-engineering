# Questão 07 · Runbook para alerta recorrente

**Framework:** R-I-S-E (Role · Input · Steps · Expectation)

**Modelo:** Claude Opus 5.5 (Anthropic, via claude.ai, esforço Médio). Escolhido por manter a coerência em documentos procedurais longos, com passos que se referenciam entre si, e pelo raciocínio operacional sobre Kubernetes e Argo CD.

## Cenário

O Beacon dispara em média 4 vezes por semana o alerta `[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)`. O plantonista leva de 30 a 40 minutos para resolver porque não há procedimento documentado. Lorraine quer um runbook que qualquer plantonista siga de ponta a ponta, cobrindo diagnóstico com comandos específicos, verificação esperada de cada passo, critérios objetivos de escalação para o `@chronos-core` e critério de encerramento. Ambiente: EKS (namespace `production`, 6 réplicas, HPA por CPU), deploy via Argo CD, dependências Ledger e Reactor, ferramentas `kubectl`, `aws cli` e `argocd cli`.

## Prompt

```text
# ROLE
Você é um SRE sênior com ampla experiência em Kubernetes (EKS), Argo CD e
resposta a incidentes, especialista em escrever runbooks que plantonistas
sem conhecimento prévio do sistema conseguem seguir sob pressão.

# INPUT
Alerta recorrente, disparado em média 4 vezes por semana pelo Beacon
(observabilidade) no canal de plantão:
[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)

Hoje, quem assume o plantão gasta de 30 a 40 minutos para resolver, e o
tempo varia muito porque não existe procedimento documentado. A líder de
SRE, Lorraine Baines, quer um runbook que qualquer plantonista consiga
seguir de ponta a ponta sem depender de quem conhece o sistema.

Ambiente:
- Chronos (API gateway e plataforma core) roda no EKS, namespace production,
  6 réplicas com HPA configurado (min 4, max 12, CPU target 70%).
- Deploy via Argo CD a partir do repositório hvt/chronos-api.
- Dependências diretas: Ledger (PostgreSQL) e Reactor (filas SQS).
- Observabilidade: métricas expostas em /metrics, logs centralizados no
  Beacon, dashboards em Grafana.
- Ferramentas disponíveis para o plantão: kubectl, aws cli, argocd cli.
- Canal de plantão: #oncall-chronos no Slack.
- Time sênior de escalação: @chronos-core (SLA de resposta: 15 minutos em
  horário comercial, 30 fora).

# STEPS
Para construir o runbook, siga estes passos:
1. Levante as causas prováveis de uso alto de memória nesse ambiente
   (vazamento de memória, pico de tráfego, deploy recente, lentidão das
   dependências acumulando requisições) e desenhe o diagnóstico para
   diferenciá-las.
2. Considere que o HPA escala por CPU, não por memória: deixe claro no
   runbook o que isso significa para o plantonista.
3. Ordene o diagnóstico do menos para o mais invasivo: primeiro comandos
   somente leitura, depois mitigação. Antes de qualquer reinício, inclua um
   passo para coletar evidências que permitam investigar a causa raiz.
4. Garanta que qualquer rollback seja feito pelo Argo CD, já que mudanças
   manuais via kubectl podem ser revertidas pelo sync automático.
5. Defina critérios objetivos (com números, tempos ou estados verificáveis)
   para escalar ao @chronos-core e para encerrar o incidente.
6. Como o alerta é recorrente, inclua o que deve ser feito após o incidente
   para atacar a causa raiz.
7. Onde faltar informação (nomes de dashboards, labels exatos, nome da
   aplicação no Argo CD), use placeholders entre <> e não invente valores.

# EXPECTATION
Entregue um runbook em Markdown, pronto para ser publicado, com:
- Cabeçalho: nome do alerta, severidade, serviço, dono, tempo-alvo de
  resolução e data da última revisão.
- Pré-requisitos de acesso (contexto do kubectl, login no argocd e aws).
- Passos numerados. Cada passo deve ter: objetivo, comando exato pronto para
  copiar e colar, verificação esperada (o que o plantonista deve ver se
  estiver tudo certo) e próximo passo em caso de resultado diferente.
- Ações que alteram produção marcadas com ⚠️.
- Seção "Quando escalar" com os critérios objetivos e um modelo de mensagem
  para o Slack.
- Seção "Critério de encerramento", com as condições verificáveis.
- Seção "Pós-incidente", com o registro e as ações de causa raiz.
O runbook deve ser seguível por alguém que nunca trabalhou no Chronos e
reduzir o tempo de resolução de 30-40 minutos para menos de 15 minutos nos
casos que não exigem escalação.
```

## Output

> O output foi mantido na íntegra. Os títulos do runbook foram rebaixados em dois níveis apenas para se encaixarem na hierarquia deste README.

### Runbook: Chronos API, uso alto de memória

| Campo | Valor |
|---|---|
| **Alerta** | `[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)` |
| **Origem** | Beacon (observabilidade) → `#oncall-chronos` |
| **Severidade** | Critical |
| **Serviço** | Chronos API (API gateway e plataforma core) |
| **Ambiente** | EKS, namespace `production` |
| **Dono** | SRE (Lorraine Baines); escalação técnica: `@chronos-core` |
| **Tempo-alvo de resolução** | Menos de 15 minutos nos casos sem escalação |
| **Última revisão** | 2026-11-19 |

---

#### Antes de começar: o que você precisa saber

O Chronos é a porta de entrada de todo o tráfego da empresa. Se ele cair, tudo cai. Por isso, **nunca reinicie todos os pods ao mesmo tempo**.

**O HPA não vai resolver este alerta sozinho.** O HPA (autoscaler) do Chronos escala por **CPU** (alvo de 70%), não por memória. Os pods podem estar com 95% de memória e o HPA não criar nenhuma réplica nova. E mesmo que crie, réplicas novas **não resolvem vazamento de memória**: elas apenas atrasam o problema.

**Não altere o Deployment ou o HPA com `kubectl edit`, `kubectl scale` ou `kubectl patch`.** O Argo CD sincroniza o cluster com o repositório `hvt/chronos-api` e desfaz mudanças manuais. Toda mudança de configuração ou rollback passa pelo Argo CD ou por PR no repositório.

**Este alerta é recorrente (cerca de 4 vezes por semana).** Isso indica um problema de causa raiz. Coletar evidências **antes** de reiniciar pods (Passo 6) não é opcional: sem elas, ninguém consegue corrigir o problema de verdade.

**Ações que alteram produção estão marcadas com ⚠️.** Todo o resto é somente leitura e pode ser executado sem risco.

---

#### Pré-requisitos de acesso

Execute uma vez no início do plantão, ou ao receber o alerta.

```bash
# 1. Login na AWS (perfil de plantão)
aws sso login --profile <perfil-aws-plantao>
export AWS_PROFILE=<perfil-aws-plantao>

# 2. Configurar o kubectl para o cluster de produção
aws eks update-kubeconfig --name <nome-do-cluster-eks> --region <regiao> --alias hvt-production
kubectl config use-context hvt-production

# 3. Login no Argo CD
argocd login <endereco-do-argocd> --sso

# 4. Variáveis usadas em todos os comandos deste runbook
export NS=production
export SELECTOR='<label-selector-dos-pods, ex.: app.kubernetes.io/name=chronos-api>'
export DEPLOY='<nome-do-deployment>'
export ARGO_APP='<nome-da-app-no-argocd>'
export EVIDENCE_BUCKET='<bucket-s3-de-evidencias>'
```

**Verificação esperada:**

```bash
kubectl get pods -n $NS -l $SELECTOR
argocd app get $ARGO_APP --refresh | head -20
```

Você deve ver de 4 a 12 pods `Running` e a aplicação com `Sync Status: Synced` e `Health Status: Healthy` (ou `Degraded`, se houver pods com problema).

**Se falhar:** erros de autenticação ou permissão (`Forbidden`, `Unauthorized`) devem ser resolvidos antes de continuar. Se não conseguir em 5 minutos, **escale** (veja "Quando escalar"), informando que não tem acesso.

---

#### Passo 0: avisar no canal (30 segundos)

**Objetivo:** deixar claro que alguém assumiu o alerta e evitar trabalho duplicado.

Poste em `#oncall-chronos`:

```
:eyes: Assumi o alerta de memória do Chronos. Seguindo o runbook. Próxima atualização em 10 min.
```

Anote o **horário de início**. Você vai precisar dele para os critérios de escalação.

---

#### Passo 1: medir o tamanho do problema

**Objetivo:** saber quantos pods estão afetados e quão perto do limite eles estão.

```bash
# Limite de memória configurado por pod
kubectl get deploy $DEPLOY -n $NS \
  -o jsonpath='{.spec.template.spec.containers[*].resources.limits.memory}{"\n"}'

# Consumo atual, do maior para o menor, com idade dos pods
kubectl top pods -n $NS -l $SELECTOR --sort-by=memory
kubectl get pods -n $NS -l $SELECTOR -o wide --sort-by=.status.startTime
```

**Como ler:** divida o consumo de cada pod pelo limite. Por exemplo, com `1Gi` de limite e `900Mi` de consumo, o pod está em cerca de 88%.

**Verificação esperada:** a maioria dos pods abaixo de 85%, e apenas alguns acima.

**Próximo passo:**
- **Qualquer pod acima de 95%**, ou **mais da metade dos pods acima de 85%**: risco iminente de OOMKill. Avise no canal, siga para os Passos 2 a 6 **rapidamente** (no máximo 5 minutos no total) e já prepare a escalação.
- Caso contrário, siga para o Passo 2.

---

#### Passo 2: verificar se já há pods morrendo por falta de memória

**Objetivo:** saber se o problema já está causando reinícios (OOMKilled).

```bash
kubectl get pods -n $NS -l $SELECTOR \
  -o custom-columns='POD:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount,ULTIMO_MOTIVO:.status.containerStatuses[0].lastState.terminated.reason,QUANDO:.status.containerStatuses[0].lastState.terminated.finishedAt'

kubectl get events -n $NS --field-selector reason=OOMKilling --sort-by=.lastTimestamp | tail -10
```

**Verificação esperada:** `RESTARTS` igual a 0 (ou sem aumento recente) e nenhum `OOMKilled` nos últimos 30 minutos.

**Próximo passo:**
- **2 ou mais OOMKilled nos últimos 15 minutos:** **escale agora**, em paralelo, e continue o runbook.
- Caso contrário, siga para o Passo 3.

---

#### Passo 3: verificar se houve deploy recente

**Objetivo:** descobrir se uma versão nova introduziu o consumo de memória.

```bash
argocd app history $ARGO_APP | tail -5
argocd app get $ARGO_APP | grep -E 'Sync Status|Health Status|Target|Revision'
```

**Verificação esperada:** o último deploy foi há mais de 24 horas.

**Próximo passo:**
- **Deploy nas últimas 24 horas**, com memória que começou a subir depois dele (confira no dashboard `<dashboard-grafana-chronos-memoria>`): a causa provável é **deploy recente**. Anote o `ID` da revisão anterior e siga para o Passo 6 (evidências) e depois o **Passo 7C** (rollback).
- Sem deploy recente: siga para o Passo 4.

---

#### Passo 4: verificar pico de tráfego e lentidão das dependências

**Objetivo:** diferenciar pico de tráfego de dependência lenta, que acumula requisições presas na memória.

**4.1 Tráfego.** No Grafana, abra `<dashboard-grafana-chronos-trafego>` e compare as requisições por segundo da última hora com o mesmo horário da semana anterior.

**4.2 Ledger (PostgreSQL).** Procure erros de timeout ou conexão nos logs:

```bash
kubectl logs -n $NS -l $SELECTOR --since=15m --max-log-requests=12 \
  | grep -ciE 'timeout|too many connections|connection refused|pool exhausted'
```

Confira também a latência do banco em `<dashboard-grafana-ledger>`.

**4.3 Reactor (filas SQS).** Veja o backlog da fila:

```bash
aws sqs get-queue-attributes \
  --queue-url <url-da-fila-reactor> \
  --attribute-names ApproximateNumberOfMessages ApproximateAgeOfOldestMessage
```

**Verificação esperada:** tráfego dentro de ±30% do mesmo horário da semana anterior; menos de `<limiar-de-erros>` ocorrências de timeout em 15 minutos; backlog e idade da mensagem mais antiga dentro do normal (`<valores-normais-sqs>`).

**Próximo passo:** siga para o Passo 5 e use a tabela de decisão.

---

#### Passo 5: identificar a causa provável

**Objetivo:** escolher a mitigação certa. Reiniciar pods não resolve todas as causas.

| O que você viu | Causa provável | Mitigação |
|---|---|---|
| Deploy nas últimas 24 horas e memória subiu depois dele (Passo 3) | **Deploy recente** | Passo 6, depois **7C** |
| Timeouts do Ledger, latência alta do banco ou backlog do SQS anormal (Passo 4) | **Dependência lenta** | Passo 6, depois **7D** (não reiniciar) |
| Tráfego mais de 30% acima do normal e memória alta em todos os pods de forma parecida | **Pico de tráfego** | Passo 6, depois **7B** |
| Tráfego normal, dependências normais, **pods mais antigos usando mais memória** que os mais novos (compare a idade do Passo 1 com o consumo) | **Vazamento de memória** (o caso mais comum neste alerta) | Passo 6, depois **7A** |
| Nada se encaixa | Indeterminada | Passo 6 e **escale** |

---

#### Passo 6: coletar evidências (obrigatório antes de qualquer reinício)

**Objetivo:** guardar os dados que permitem investigar a causa raiz. Leva cerca de 2 minutos.

```bash
TS=$(date -u +%Y%m%dT%H%M%SZ)
DIR=chronos-mem-$TS && mkdir -p $DIR

kubectl top pods -n $NS -l $SELECTOR --sort-by=memory > $DIR/top.txt
kubectl get pods -n $NS -l $SELECTOR -o wide > $DIR/pods.txt
kubectl describe pods -n $NS -l $SELECTOR > $DIR/describe.txt
kubectl get events -n $NS --sort-by=.lastTimestamp > $DIR/events.txt
argocd app history $ARGO_APP > $DIR/argocd-history.txt

# Pod com MAIS memória: logs, métricas e dump de memória
TOP_POD=$(kubectl top pods -n $NS -l $SELECTOR --sort-by=memory --no-headers | head -1 | awk '{print $1}')
echo $TOP_POD > $DIR/top-pod.txt
kubectl logs -n $NS $TOP_POD --since=1h > $DIR/logs-$TOP_POD.txt
kubectl logs -n $NS $TOP_POD --previous > $DIR/logs-previous-$TOP_POD.txt 2>/dev/null
kubectl exec -n $NS $TOP_POD -- <comando-para-ler-/metrics> > $DIR/metrics-$TOP_POD.txt
kubectl exec -n $NS $TOP_POD -- <comando-de-heap-dump-da-linguagem-do-chronos> > $DIR/heap-$TOP_POD.out

# Enviar para o bucket de evidências
tar czf $DIR.tar.gz $DIR
aws s3 cp $DIR.tar.gz s3://$EVIDENCE_BUCKET/chronos/$DIR.tar.gz
```

**Verificação esperada:** o `aws s3 cp` termina com `upload: ...` sem erro.

**Próximo passo:**
- Se o comando de heap dump falhar ou não estiver definido, **não bloqueie a mitigação**: registre "heap dump não coletado" no canal e siga em frente.
- Se o upload falhar, mantenha o arquivo `.tar.gz` local e anexe-o no ticket depois.
- Siga para a mitigação indicada no Passo 5.

---

#### Passo 7: mitigar

##### 7A ⚠️ Vazamento de memória: reiniciar os pods afetados, um por vez

**Objetivo:** liberar memória sem derrubar o tráfego. Apagar um pod não muda o Deployment, então o Argo CD não interfere: o Kubernetes recria o pod automaticamente.

Reinicie **apenas os pods acima de 85%**, começando pelo de maior consumo, **um por vez**:

```bash
# ⚠️ Altera produção: apaga UM pod
kubectl delete pod -n $NS <nome-do-pod>

# Espere o substituto ficar pronto (READY 1/1) antes do próximo
kubectl get pods -n $NS -l $SELECTOR -w
```

**Regras de segurança:** nunca apague o próximo pod antes de o novo estar `1/1 Running` há pelo menos 30 segundos. **Nunca** apague mais de 1 pod ao mesmo tempo. **Não** use `kubectl rollout restart`: ele altera o Deployment e pode entrar em conflito com o auto-sync do Argo CD.

**Verificação esperada:** os pods novos sobem em até `<tempo-de-boot-esperado>` e ficam bem abaixo de 85% de memória.

**Próximo passo:** siga para o Passo 8. Se um pod novo não ficar pronto em 5 minutos, **pare e escale**.

##### 7B ⚠️ Pico de tráfego: aumentar o mínimo de réplicas pelo Git

**Objetivo:** distribuir a carga em mais pods, já que o HPA não reage à memória.

Não use `kubectl scale`: o HPA e o Argo CD desfazem a mudança. Abra um PR em `hvt/chronos-api` alterando o `minReplicas` do HPA em `<caminho-do-manifesto-do-hpa>`, de 4 para 8, com o título `[INCIDENTE] Aumento temporário de minReplicas do Chronos`. Depois do merge:

```bash
# ⚠️ Altera produção: aplica a mudança do Git
argocd app sync $ARGO_APP
argocd app wait $ARGO_APP --health --timeout 600
```

**Verificação esperada:** `kubectl get hpa -n $NS` mostra `MINPODS 8`, e a memória por pod cai em 10 minutos.

**Próximo passo:** se não houver quem aprove o PR em 10 minutos, ou se a memória continuar subindo mesmo com mais réplicas (o que indica vazamento, e não tráfego), **escale**. Se 2 pods ou mais estiverem acima de 95% enquanto isso, aplique a 7A neles.

##### 7C ⚠️ Deploy recente: rollback pelo Argo CD

**Objetivo:** voltar para a última versão estável.

O Argo CD **não permite rollback com auto-sync ligado**, e o auto-sync desfaria o rollback. Por isso o auto-sync é desativado primeiro, e isso **precisa** ser registrado no canal.

```bash
# Anunciar no canal ANTES:
# ":warning: Fazendo rollback do Chronos para a revisão <ID> e desativando auto-sync do Argo CD temporariamente."

# ⚠️ 1. Desativar auto-sync
argocd app set $ARGO_APP --sync-policy none

# ⚠️ 2. Rollback para a revisão anterior ao deploy suspeito (ID do Passo 3)
argocd app rollback $ARGO_APP <ID-da-revisao-anterior>

# 3. Acompanhar
argocd app wait $ARGO_APP --health --timeout 600
```

**Verificação esperada:** `Health Status: Healthy`, todos os pods novos `Running` e memória estável.

**Próximo passo:** siga para o Passo 8. **Não reative o auto-sync** enquanto o código problemático estiver na branch principal, senão a versão ruim volta. Abra um PR de revert em `hvt/chronos-api` e avise o `@chronos-core`, que reativa o auto-sync depois do revert. Se o rollback falhar, **escale imediatamente**.

##### 7D Dependência lenta: não reiniciar, escalar

**Objetivo:** atacar a origem. Reiniciar o Chronos não resolve: os pods novos vão encher de requisições presas da mesma forma.

1. **Escale** ao `@chronos-core` com a evidência do Passo 4.
2. Avise os donos da dependência afetada: `<canal-ou-time-do-ledger>` ou `<canal-ou-time-do-reactor>`.
3. Enquanto isso, se algum pod passar de 95% de memória, aplique a **7A apenas nesse pod**, para evitar o OOMKill.

---

#### Passo 8: confirmar a recuperação

**Objetivo:** garantir que a mitigação funcionou antes de encerrar.

```bash
watch -n 30 "kubectl top pods -n $NS -l $SELECTOR --sort-by=memory"
```

**Verificação esperada:** após 15 minutos, todos os pods abaixo de 75% do limite, sem tendência de alta, e sem novos reinícios no Passo 2.

**Próximo passo:** se tudo estiver certo, vá para o "Critério de encerramento". Se a memória voltar a passar de 85% em menos de 15 minutos, a mitigação não resolveu: **escale**.

---

#### Quando escalar

Acione o `@chronos-core` em `#oncall-chronos` assim que **qualquer** condição abaixo for verdadeira. Não espere tentar tudo. O SLA de resposta é de 15 minutos em horário comercial e 30 fora dele, então escalar cedo é melhor do que tarde.

| Condição | Como verificar |
|---|---|
| 2 ou mais pods `OOMKilled` em 15 minutos | Passo 2 |
| Mais da metade dos pods acima de 95% | Passo 1 |
| Taxa de erros 5xx acima de `<limiar-5xx, ex.: 1%>` por mais de 5 minutos | `<dashboard-grafana-chronos-erros>` |
| Causa provável é dependência lenta (7D) ou indeterminada | Passo 5 |
| Rollback executado (7C) ou falhou | Passo 7C |
| Memória volta acima de 85% em menos de 15 minutos após a mitigação | Passo 8 |
| Um pod novo não fica pronto em 5 minutos | Passo 7A |
| **15 minutos desde o início sem melhora** | Horário anotado no Passo 0 |
| Falta de acesso ou dúvida sobre qualquer ação marcada com ⚠️ | — |

**Modelo de mensagem para o Slack:**

```
:rotating_light: @chronos-core preciso de escalação: memória alta no Chronos API

• Início: <HH:MM> | Agora: <HH:MM>
• Pods afetados: <X> de <Y> acima de 85% (maior: <pod> em <N>%)
• OOMKilled nos últimos 15 min: <quantidade>
• Erros 5xx: <valor ou "normais">
• Causa provável: <vazamento | pico de tráfego | deploy recente | dependência lenta | indeterminada>
• Último deploy: <revisão e horário, via argocd app history>
• Ações já feitas: <ex.: reiniciei 2 pods, rollback para revisão X>
• Evidências: s3://<bucket>/chronos/<arquivo>.tar.gz
• Motivo da escalação: <critério da tabela>
```

---

#### Critério de encerramento

O incidente só pode ser encerrado quando **todas** as condições forem verdadeiras:

- [ ] Todos os pods abaixo de 75% do limite de memória por **15 minutos seguidos**, sem tendência de alta.
- [ ] Alerta resolvido no Beacon (status `RESOLVED`).
- [ ] Nenhum reinício ou `OOMKilled` novo nos últimos 15 minutos (Passo 2).
- [ ] Taxa de erros 5xx dentro do normal em `<dashboard-grafana-chronos-erros>`.
- [ ] `argocd app get $ARGO_APP` mostra `Healthy`. Se houve rollback, o auto-sync continua desativado **e** isso está registrado no canal, com um responsável no `@chronos-core`.
- [ ] Evidências do Passo 6 salvas (no S3 ou no ticket).
- [ ] Mensagem de encerramento postada em `#oncall-chronos`:

```
:white_check_mark: Alerta de memória do Chronos resolvido.
• Duração: <início> – <fim> (<N> min)
• Causa provável: <...>
• Ação: <...>
• Evidências: s3://<bucket>/chronos/<arquivo>.tar.gz
• Ticket de causa raiz: <link>
```

---

#### Pós-incidente

##### Registro (pelo plantonista, até o fim do turno)

1. Registre o incidente em `<ferramenta-de-incidentes>` com: horário de início e fim, tempo até mitigar, causa provável, ações executadas (inclusive comandos ⚠️), link das evidências e se houve escalação.
2. Vincule o registro ao **ticket único de causa raiz** do alerta recorrente, `<link-do-ticket-de-causa-raiz>`. Se ele não existir, crie um em `<projeto-jira>` com o título `[RECORRENTE] Uso alto de memória no Chronos API`. **Não crie um ticket novo a cada ocorrência**: acumular as evidências no mesmo ticket é o que revela o padrão.
3. Se algum passo deste runbook estiver errado, confuso ou desatualizado, anote no ticket para a revisão do runbook.

##### Ações de causa raiz (dono: `@chronos-core`, acompanhamento: SRE)

Com cerca de 4 ocorrências por semana, o objetivo é eliminar o alerta, não ficar mais rápido em reiniciar pods.

1. **Investigar vazamento de memória.** Use os heap dumps e as métricas coletados no Passo 6 para identificar objetos que crescem com o tempo. Compare o consumo de memória com a idade dos pods ao longo de uma semana: crescimento linear, independente do tráfego, confirma vazamento.
2. **Correlacionar com deploys e dependências.** Cruze os horários das ocorrências com o histórico do Argo CD e com a latência do Ledger e o backlog do Reactor.
3. **Revisar requests e limits de memória** com base no p95/p99 real de consumo. O limite atual pode estar subdimensionado para o tráfego de hoje.
4. **Revisar a estratégia de escalonamento.** Avaliar adicionar memória como métrica do HPA, ou usar uma métrica de tráfego (requisições por segundo, requisições em andamento). Isso só faz sentido se a causa não for vazamento.
5. **Proteger contra dependências lentas.** Revisar timeouts, tamanho do pool de conexões com o Ledger e limites de requisições simultâneas, para que lentidão a jusante não vire acúmulo de memória.
6. **Melhorar o alerta.** Avaliar um alerta antecipado por **taxa de crescimento** da memória, que avise antes de chegar a 85%. Incluir no próprio alerta o link para este runbook.
7. **Medir o resultado.** Acompanhar semanalmente o número de ocorrências e o tempo médio de resolução. Meta: menos de 15 minutos por ocorrência enquanto a causa raiz não for corrigida, e zero ocorrências após a correção.
8. **Revisar este runbook** após cada ocorrência com escalação, e no mínimo a cada trimestre. Preencher todos os placeholders `<...>` na primeira revisão.


## Justificativa

O **Role** (seção `# ROLE`) define um SRE sênior especialista em EKS, Argo CD e em runbooks para plantonistas sem contexto, o que orientou o tom didático do output. O **Input** (seção `# INPUT`) traz o alerta, o problema atual (30 a 40 minutos sem procedimento) e todo o ambiente do enunciado. Os **Steps** (seção `# STEPS`) definem o raciocínio para construir o runbook: diferenciar causas, considerar que o HPA escala por CPU, coletar evidências antes de reiniciar, fazer rollback via Argo CD e criar critérios objetivos. A **Expectation** (seção `# EXPECTATION`) fixa o formato publicável, com comando, verificação esperada e próximo passo em cada etapa, e a meta de resolver em menos de 15 minutos.

## Observações sobre o output

- Os Steps aparecem claramente no resultado: a seção inicial explica que o HPA não reage a memória, o Passo 5 transforma o diagnóstico numa tabela de decisão por causa provável (vazamento, tráfego, deploy, dependência), a coleta de evidências vem antes de qualquer reinício e o rollback desativa o auto-sync do Argo CD antes de agir, o que o `argocd app rollback` exige. Os critérios de escalação e encerramento são todos objetivos (números, tempos e estados).
- Pontos fortes além do pedido: reiniciar pod a pod com `kubectl delete pod` em vez de `rollout restart` (que altera o Deployment e gera diff no Argo CD), a orientação de não reiniciar quando a causa é dependência lenta e o uso de um ticket único de causa raiz, para acumular evidências entre as ocorrências.
- Pontos fracos: o heap dump do Passo 6 é feito justamente no pod com mais memória, e a própria coleta pode levá-lo ao OOMKill, risco que o runbook não menciona; a mitigação de pico de tráfego (7B) depende de PR e merge, o que dificilmente cabe na meta de 15 minutos de madrugada; e o runbook tem mais de 20 placeholders, então ainda não é seguível de ponta a ponta até ser preenchido. Esse último ponto é consequência direta do Step 7 ("não invente valores"), uma troca consciente entre segurança e completude.
- O que faria diferente: incluir no Input os dados que faltam (label dos pods, nome da app no Argo CD, dashboards, linguagem do Chronos) e pedir na Expectation um resumo de uma página, em fluxograma, para ser consultado sob pressão antes do runbook completo.
