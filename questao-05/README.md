# Questão 05 · Modernizar deployment legado

**Framework:** B-A-B (Before · After · Bridge)

**Modelo:** Claude Opus 5.5 (Anthropic, via claude.ai, esforço Médio). Escolhido pelo conhecimento de Kubernetes e segurança e pela capacidade de respeitar restrições sutis, como o selector imutável e a proibição de escrever segredos.

## Cenário

Numa revisão de produção, Doc Brown encontrou o Deployment do Chronos que o George escreveu três anos atrás, sem nenhuma prática que hoje é obrigatória na empresa. A versão moderna precisa ter alta disponibilidade, imagem versionada (nada de `latest`), secrets fora do manifest, resource requests e limits, liveness e readiness probes, `securityContext` não-root e as demais práticas de produção.

## Prompt

````text
# BEFORE
Este é o manifest atual do Deployment do Chronos, o API gateway e plataforma
core da Hill Valley Tech. Ele foi escrito há três anos e nunca foi atualizado:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      containers:
      - name: api
        image: chronos-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          value: "P@ssw0rd2023!"
        - name: JWT_SECRET
          value: "hvt-jwt-prod-secret"

Problemas do estado atual:
- Uma única réplica: qualquer falha ou deploy derruba o ponto de entrada de
  todo o tráfego da empresa.
- Imagem com tag latest: deploys não são reproduzíveis e rollback é incerto.
- Senha do banco e segredo JWT em texto puro no manifest, expostos há anos.
- Sem resource requests e limits: o scheduler não sabe quanto reservar e o
  pod pode consumir o nó inteiro.
- Sem liveness e readiness probes: o Kubernetes não detecta travamentos e
  envia tráfego para pods que ainda não estão prontos.
- Sem securityContext: o container pode rodar como root.

# AFTER
O manifest modernizado deve atender ao padrão de produção da empresa:
- Alta disponibilidade: no mínimo 3 réplicas, distribuídas entre zonas e nós,
  com rolling update sem indisponibilidade e PodDisruptionBudget.
- Imagem versionada com registry e tag semântica (nunca latest).
- DB_PASSWORD e JWT_SECRET lidos de um Secret do Kubernetes via secretKeyRef,
  sem nenhum valor sensível no manifest.
- Resource requests e limits de CPU e memória.
- Startup, liveness e readiness probes.
- securityContext não-root no pod e no container: runAsNonRoot,
  readOnlyRootFilesystem, allowPrivilegeEscalation false, capabilities
  removidas e seccompProfile RuntimeDefault.
- ServiceAccount dedicada, sem montar o token automaticamente.
- Labels recomendados do Kubernetes (app.kubernetes.io/*).
- Encerramento gracioso, para não derrubar requisições em andamento durante
  os deploys.

# BRIDGE
Para transformar o BEFORE no AFTER:
1. Mantenha name, namespace, a porta 8080 e o selector.matchLabels exatamente
   como estão: o selector de um Deployment é imutável, e mudá-lo quebraria o
   kubectl apply em produção. Labels novos entram apenas em metadata e no
   template.
2. Onde faltar informação (tag da imagem, endereço do registry, paths de
   health check, valores de CPU e memória, UID), use valores de exemplo
   realistas e marque cada um com um comentário "# TODO: confirmar".
3. Não escreva o valor de nenhum segredo, nem em base64.
4. Responda nesta estrutura:
   a) Um bloco ```yaml``` com todos os recursos necessários (Deployment,
      PodDisruptionBudget e ServiceAccount) separados por ---, comentados.
   b) Uma tabela Markdown com as colunas "Problema (Before)", "Solução
      (After)" e "Onde no manifest", uma linha por problema.
   c) Os comandos para criar o Secret sem expor valores no terminal ou no
      Git, e para validar o manifest antes de aplicar.
   d) Os passos para migrar com segurança em produção, incluindo o que fazer
      com as credenciais que ficaram expostas no manifest antigo.
````

## Output

### a) Manifests

```yaml
# ---------- ServiceAccount dedicada ----------
apiVersion: v1
kind: ServiceAccount
metadata:
  name: chronos-api
  namespace: production
  labels:
    app.kubernetes.io/name: chronos-api
    app.kubernetes.io/component: api-gateway
    app.kubernetes.io/part-of: chronos
    app.kubernetes.io/managed-by: kubectl
# O Chronos não chama a API do Kubernetes: o token não é montado
automountServiceAccountToken: false
---
# ---------- PodDisruptionBudget ----------
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: chronos-api
  namespace: production
  labels:
    app.kubernetes.io/name: chronos-api
    app.kubernetes.io/component: api-gateway
    app.kubernetes.io/part-of: chronos
spec:
  # Manutenção de nós (drain) nunca deixa menos de 2 pods no ar
  minAvailable: 2
  selector:
    matchLabels:
      app: chronos-api          # mesmo label do selector do Deployment
---
# ---------- Deployment ----------
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api             # mantido
  namespace: production         # mantido
  labels:
    app.kubernetes.io/name: chronos-api
    app.kubernetes.io/instance: chronos-api-production
    app.kubernetes.io/version: "2.4.1"            # TODO: confirmar (igual à tag da imagem)
    app.kubernetes.io/component: api-gateway
    app.kubernetes.io/part-of: chronos
    app.kubernetes.io/managed-by: kubectl
spec:
  replicas: 3                   # mínimo para alta disponibilidade
  revisionHistoryLimit: 10      # mantém revisões para rollback
  # Selector IMUTÁVEL: mantido exatamente como estava
  selector:
    matchLabels:
      app: chronos-api
  # Rolling update sem indisponibilidade: sobe 1 pod novo antes de tirar 1 antigo
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  minReadySeconds: 10           # pod precisa ficar pronto 10s antes de contar como disponível
  template:
    metadata:
      labels:
        app: chronos-api        # label antigo obrigatório (casa com o selector)
        app.kubernetes.io/name: chronos-api
        app.kubernetes.io/instance: chronos-api-production
        app.kubernetes.io/version: "2.4.1"        # TODO: confirmar
        app.kubernetes.io/component: api-gateway
        app.kubernetes.io/part-of: chronos
    spec:
      serviceAccountName: chronos-api
      automountServiceAccountToken: false

      # Encerramento gracioso: tempo total para terminar requisições em andamento
      terminationGracePeriodSeconds: 45           # TODO: confirmar (maior que o timeout máximo de requisição + preStop)

      # securityContext do pod: não-root e seccomp padrão
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001                          # TODO: confirmar UID da imagem
        runAsGroup: 10001                         # TODO: confirmar
        fsGroup: 10001                            # TODO: confirmar
        seccompProfile:
          type: RuntimeDefault

      # Distribuição entre zonas e entre nós
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule          # exige espalhar entre zonas
        labelSelector:
          matchLabels:
            app: chronos-api
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway         # preferência por nós diferentes
        labelSelector:
          matchLabels:
            app: chronos-api

      containers:
      - name: api                                 # mantido
        # Registry + tag semântica; nunca latest
        image: registry.hillvalleytech.com/platform/chronos-api:2.4.1   # TODO: confirmar registry e tag
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8080                     # mantido
          protocol: TCP

        # Segredos lidos do Secret; nenhum valor no manifest
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: chronos-api-secrets
              key: DB_PASSWORD
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: chronos-api-secrets
              key: JWT_SECRET

        # Reserva para o scheduler e teto de consumo no nó
        resources:
          requests:
            cpu: "500m"                           # TODO: confirmar com métricas reais
            memory: "512Mi"                       # TODO: confirmar
          limits:
            memory: "1Gi"                         # TODO: confirmar
            cpu: "1"                              # TODO: confirmar (avaliar remover para evitar throttling)

        # Startup: dá até 5 min para o boot sem a liveness matar o pod
        startupProbe:
          httpGet:
            path: /healthz                        # TODO: confirmar path
            port: http
          periodSeconds: 5
          failureThreshold: 60
        # Liveness: reinicia o container se travar
        livenessProbe:
          httpGet:
            path: /healthz                        # TODO: confirmar (não deve depender do banco)
            port: http
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        # Readiness: só recebe tráfego quando estiver pronto
        readinessProbe:
          httpGet:
            path: /ready                          # TODO: confirmar path
            port: http
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3

        # Encerramento gracioso: espera o endpoint sair dos load balancers antes do SIGTERM
        lifecycle:
          preStop:
            sleep:
              seconds: 10                         # requer Kubernetes 1.30+; TODO: confirmar versão do cluster

        # securityContext do container
        securityContext:
          runAsNonRoot: true
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
          seccompProfile:
            type: RuntimeDefault

        # Filesystem é somente leitura: /tmp gravável em memória/disco efêmero
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir:
          sizeLimit: 128Mi                        # TODO: confirmar se a app grava em outro diretório
```

### b) Problemas e soluções

| Problema (Before) | Solução (After) | Onde no manifest |
|---|---|---|
| Uma única réplica; falha ou deploy derruba todo o tráfego | 3 réplicas espalhadas entre zonas e nós, rolling update com `maxUnavailable: 0` e PDB com `minAvailable: 2` | `spec.replicas`, `spec.strategy`, `topologySpreadConstraints`, recurso `PodDisruptionBudget` |
| Imagem `latest`, deploys não reproduzíveis | Registry explícito e tag semântica, refletida em `app.kubernetes.io/version` | `containers[].image`, labels de versão |
| Senha do banco e segredo JWT em texto puro | `valueFrom.secretKeyRef` apontando para o Secret `chronos-api-secrets` | `containers[].env` |
| Sem requests e limits | Requests de CPU e memória e limits de CPU e memória | `containers[].resources` |
| Sem probes | `startupProbe`, `livenessProbe` e `readinessProbe` HTTP | `containers[]` |
| Sem securityContext, pode rodar como root | Pod e container com `runAsNonRoot`, UID fixo, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, `drop: ALL`, `seccompProfile: RuntimeDefault`; `/tmp` em `emptyDir` | `spec.template.spec.securityContext`, `containers[].securityContext`, `volumes` |
| Usa a ServiceAccount `default` com token montado | ServiceAccount dedicada com `automountServiceAccountToken: false` | Recurso `ServiceAccount`, `serviceAccountName` |
| Sem labels padronizados | Labels `app.kubernetes.io/*` em metadata e template; `app: chronos-api` mantido no selector | `metadata.labels`, `template.metadata.labels` |
| Deploys derrubam requisições em andamento | `preStop` com espera, `terminationGracePeriodSeconds` e `minReadySeconds` | `lifecycle.preStop`, `terminationGracePeriodSeconds`, `minReadySeconds` |

### c) Comandos: Secret e validação

```bash
# 1. Ler os valores sem eco na tela e sem gravar no histórico do shell
read -rs -p "DB_PASSWORD: " DB_PASSWORD; echo
read -rs -p "JWT_SECRET: "  JWT_SECRET;  echo

# 2. Criar o Secret via process substitution: os valores não aparecem
#    na linha de comando (ps), no histórico, nem em arquivo no disco ou no Git
kubectl create secret generic chronos-api-secrets \
  --namespace production \
  --from-file=DB_PASSWORD=<(printf '%s' "$DB_PASSWORD") \
  --from-file=JWT_SECRET=<(printf '%s' "$JWT_SECRET")

# 3. Limpar as variáveis da sessão
unset DB_PASSWORD JWT_SECRET

# 4. Conferir que as chaves existem, sem exibir os valores
kubectl describe secret chronos-api-secrets -n production

# 5. Validação estática do schema (kubeconform)
kubeconform -strict -summary chronos-api.yaml

# 6. Validação no servidor: schema, admission controllers e políticas, sem aplicar
kubectl apply --dry-run=server -f chronos-api.yaml

# 7. Ver exatamente o que vai mudar em produção
kubectl diff -f chronos-api.yaml
```

Para manter o Secret versionado no Git com segurança, a alternativa é Sealed Secrets, External Secrets Operator ou SOPS. Os valores nunca vão em texto nem em base64 no repositório.

### d) Migração segura em produção

**1. Trate as credenciais antigas como comprometidas.** Elas estão no histórico do Git há anos, então qualquer pessoa com acesso ao repositório, a um fork ou a um backup pode tê-las. Não basta movê-las para um Secret: elas precisam ser trocadas.

**2. Gere credenciais novas.** No PostgreSQL, crie uma senha nova, de preferência para um usuário novo, e mantenha a antiga válida por enquanto para não derrubar os pods atuais. Gere também um `JWT_SECRET` novo e forte, por exemplo com `openssl rand -base64 48`, digitado diretamente no `read -s` e nunca colado em chat ou ticket.

**3. Crie o Secret com as credenciais novas**, usando os comandos da seção c. O Secret tem de existir antes do apply, ou os pods novos ficam em `CreateContainerConfigError`.

**4. Confirme os TODOs.** Antes do apply, verifique que a imagem roda com UID 10001 e só grava em `/tmp`, que os paths `/healthz` e `/ready` existem, que o cluster é 1.30+ (por causa do `preStop.sleep`) e que há nós em pelo menos 3 zonas com capacidade para os requests. Faça isso primeiro em staging, com o mesmo manifest.

**5. Aplique e acompanhe.** Rode `kubectl apply -f chronos-api.yaml` e acompanhe com `kubectl rollout status deployment/chronos-api -n production`, junto com a taxa de erro e a latência do gateway. Como o selector não mudou, o apply funciona normalmente. Em caso de problema, `kubectl rollout undo` volta à revisão anterior.

**6. Planeje o impacto da troca do JWT.** Trocar o `JWT_SECRET` invalida todos os tokens emitidos, e os usuários precisarão fazer login de novo. Se a aplicação aceitar dois segredos durante uma janela de transição, use esse recurso. Caso contrário, faça a troca em horário de baixo tráfego e avise o suporte antes.

**7. Revogue as credenciais antigas.** Com todos os pods novos rodando, remova a senha antiga do banco e revise os logs de acesso ao PostgreSQL desde 2023, procurando conexões de origens inesperadas.

**8. Limpe o repositório.** Remova os valores do manifest e avalie reescrever o histórico do Git (`git filter-repo`). Lembre que isso não substitui a rotação, porque clones antigos continuam com as credenciais. Adicione um scanner de segredos no CI, como gitleaks ou trufflehog, para impedir que isso se repita.

**9. Registre o incidente**, com o tempo de exposição, as ações tomadas e os responsáveis, conforme a política de segurança da empresa.

## Justificativa

O **Before** (seção `# BEFORE`) apresenta o manifest atual junto com um diagnóstico explícito de cada problema e do seu impacto (ponto único de falha, deploys não reproduzíveis, segredos expostos). O **After** (seção `# AFTER`) descreve o estado desejado como uma lista verificável do padrão de produção, cobrindo os itens do enunciado e as "demais práticas" que ele deixa em aberto. A **Bridge** (seção `# BRIDGE`) define como fazer a transição com segurança: preservar o selector imutável, marcar suposições com TODO, nunca escrever segredos e entregar o caminho de migração.

## Observações sobre o output

- O modelo cumpriu todo o After e respeitou as restrições da Bridge: manteve o `selector` e o label `app: chronos-api`, marcou com TODO cada valor que inventou (registry, tag, UID, paths, recursos) e não escreveu nenhum segredo. Também tratou as credenciais antigas como comprometidas, com rotação, revogação, revisão de logs e prevenção (gitleaks no CI).
- Foi além do pedido em pontos relevantes: `emptyDir` em `/tmp` para o `readOnlyRootFilesystem` não quebrar a aplicação, o alerta de que trocar o `JWT_SECRET` desloga todos os usuários e a criação do Secret sem os valores passarem pela linha de comando.
- Pontos fracos: o `topologySpreadConstraints` por zona com `DoNotSchedule` pode deixar pods em `Pending` se o cluster tiver menos de 3 zonas ou faltar capacidade em uma delas; o `preStop.sleep` só funciona em Kubernetes 1.30+; e o output é longo, com 9 passos de migração, mais do que um revisor precisaria para aprovar o manifest.
- O que faria diferente: informar no Before a versão do cluster e o número de zonas disponíveis, eliminando dois TODOs, e limitar no Bridge o tamanho da seção de migração.
