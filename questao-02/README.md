# Questão 02 · Script de backup do Ledger

**Framework:** R-T-F (Role · Task · Format)

**Modelo:** ChatGPT, plano gratuito (OpenAI, modelo padrão, sem o modo Pensar). Mantido o mesmo modelo da questão 01 para observar como ele se comporta com o mesmo framework numa tarefa diferente: um script bash longo e cheio de requisitos de robustez.

## Cenário

Lorraine (SRE) identificou que o Ledger, o PostgreSQL que o George levantou numa EC2 anos atrás, nunca teve backup automatizado. Ela quer fechar essa dependência com uma cron diária. O script precisa fazer o dump com `pg_dump`, compactar com `gzip`, subir para o bucket S3 `hvt-ledger-backups` via `aws s3 cp`, manter 30 dias de retenção no S3, registrar cada execução em `/var/log/ledger-backup.log` com timestamp e sair com exit code adequado em caso de falha.

## Prompt

````text
# ROLE
Você é um engenheiro SRE sênior, especialista em PostgreSQL, automação com bash
e AWS, com experiência em rotinas de backup de bancos críticos em produção.

# TASK
Escreva um script bash de backup diário do "Ledger", o data warehouse PostgreSQL
da empresa, que hoje não tem nenhuma rotina de backup automatizada. O script
será executado via cron diária em uma instância EC2.

Ambiente:
- Host: ledger-db.internal.hvt.io
- Porta: 5432
- Banco: ledger_prod
- Usuário de backup: backup_user
- Senha: variável de ambiente PGPASSWORD, populada pelo AWS Secrets Manager via
  IAM role da instância
- Região AWS: us-east-1
- SO: Ubuntu 22.04 LTS
- Diretório de trabalho: /var/backups/ledger (80 GB livres)
- Tamanho médio do dump compactado: ~12 GB
- Bucket S3 de destino: hvt-ledger-backups
- Log: /var/log/ledger-backup.log

O script deve:
1. Fazer o dump com pg_dump e compactar com gzip (pipe direto, sem gerar arquivo
   intermediário sem compressão).
2. Nomear o arquivo com data e hora em UTC (ex.: ledger_prod_2026-11-19T030000Z.sql.gz).
3. Enviar o arquivo para o bucket com aws s3 cp, informando a região.
4. Manter 30 dias de retenção no S3, removendo backups mais antigos que isso.
5. Registrar cada etapa em /var/log/ledger-backup.log com timestamp, incluindo
   início, fim, duração e tamanho do arquivo.
6. Sair com exit code diferente de zero em qualquer falha, com códigos distintos
   por tipo de erro (documentados no cabeçalho do script).

Requisitos de robustez (obrigatórios):
- Usar set -euo pipefail, para que uma falha do pg_dump no pipe não seja
  mascarada pelo gzip.
- Validar antes de começar: PGPASSWORD definida, comandos pg_dump, gzip e aws
  instalados, e espaço livre suficiente no diretório de trabalho.
- Impedir execuções simultâneas com flock.
- Usar trap para remover arquivos parciais em caso de falha.
- Validar a integridade do arquivo com gzip -t antes do upload.
- Confirmar que o objeto chegou ao S3 antes de apagar a cópia local.
- Nunca escrever a senha no log nem na saída do script.
- Usar variáveis no topo do script para todas as configurações.

# FORMAT
Responda exatamente nesta ordem, sem texto fora dessas seções:
1. Um bloco ```bash``` com o script completo, com cabeçalho (descrição, uso e
   tabela de exit codes) e comentários curtos em português em cada etapa.
2. Um bloco com a linha de crontab para rodar todos os dias às 03:00 UTC.
3. Uma tabela Markdown com as colunas "Requisito" e "Como foi atendido",
   uma linha por item pedido (os 6 itens e os requisitos de robustez).
4. Uma lista curta com as permissões IAM mínimas que a role da instância
   precisa no bucket.
````

## Output

### 1. Script completo · `ledger-backup.sh`

```bash
#!/bin/bash
#
# ============================================================
# LEDGER - BACKUP AUTOMATIZADO POSTGRESQL
# ============================================================
#
# Descrição:
#   Realiza backup diário do PostgreSQL, comprime com gzip,
#   valida a integridade e envia o arquivo para o Amazon S3.
#   Remove backups locais após confirmação do upload e mantém
#   30 dias de retenção no bucket.
#
# Uso:
#   sudo /usr/local/sbin/ledger-backup.sh
#
# Pré-requisitos:
#   - PostgreSQL client (pg_dump)
#   - gzip
#   - AWS CLI configurada com IAM Role
#   - PGPASSWORD fornecida pelo AWS Secrets Manager
#   - flock (utilitário de lock do Linux)
#
# EXIT CODES:
#   0  = Backup concluído com sucesso
#   1  = Erro inesperado
#   2  = Configuração inválida ou PGPASSWORD ausente
#   3  = Dependência não instalada
#   4  = Espaço em disco insuficiente
#   5  = Outra execução já está em andamento
#   6  = Falha no pg_dump
#   7  = Falha no gzip ou integridade inválida
#   8  = Falha no upload ou verificação no S3
#   9  = Falha na política de retenção
#
# ============================================================

set -euo pipefail

# ------------------------------------------------------------
# 1. CONFIGURAÇÕES
# ------------------------------------------------------------

DB_HOST="ledger-db.internal.hvt.io"
DB_PORT="5432"
DB_NAME="ledger_prod"
DB_USER="backup_user"

AWS_REGION="us-east-1"
S3_BUCKET="hvt-ledger-backups"
S3_PREFIX=""

WORK_DIR="/var/backups/ledger"
LOG_FILE="/var/log/ledger-backup.log"
LOCK_FILE="/var/lock/ledger-backup.lock"

RETENTION_DAYS=30

# Espaço mínimo exigido: 15 GiB
MIN_FREE_GB=15

# ------------------------------------------------------------
# 2. INICIALIZAÇÃO E LOG
# ------------------------------------------------------------

mkdir -p "$WORK_DIR"
mkdir -p "$(dirname "$LOCK_FILE")"

touch "$LOG_FILE"

START_EPOCH=$(date -u +%s)
BACKUP_FILE=""
PARTIAL_FILE=""
BACKUP_SUCCESS=false

# Registra mensagens com timestamp UTC
log() {
    echo "[$(date -u '+%Y-%m-%dT%H:%M:%SZ')] $*"
}

# Direciona a saída para o arquivo de log
exec >> "$LOG_FILE" 2>&1

# ------------------------------------------------------------
# 3. TRATAMENTO DE ERROS E LIMPEZA
# ------------------------------------------------------------

cleanup() {
    local EXIT_STATUS=$?

    if [[ "$BACKUP_SUCCESS" != "true" ]]; then
        if [[ -n "$PARTIAL_FILE" && -f "$PARTIAL_FILE" ]]; then
            log "Removendo arquivo parcial."
            rm -f -- "$PARTIAL_FILE"
        fi
    fi

    local END_EPOCH
    END_EPOCH=$(date -u +%s)

    local DURATION=$((END_EPOCH - START_EPOCH))

    if [[ "$EXIT_STATUS" -eq 0 ]]; then
        log "BACKUP FINALIZADO COM SUCESSO."
    else
        log "BACKUP FINALIZADO COM ERRO. Exit code: $EXIT_STATUS"
    fi

    log "Duração total: ${DURATION} segundos."
    log "============================================================"

    exit "$EXIT_STATUS"
}

trap cleanup EXIT

# ------------------------------------------------------------
# 4. VALIDAÇÃO DE PRÉ-REQUISITOS
# ------------------------------------------------------------

log "============================================================"
log "Iniciando backup do banco $DB_NAME."

# Verifica a variável de senha sem exibir seu conteúdo
if [[ -z "${PGPASSWORD:-}" ]]; then
    log "ERRO: PGPASSWORD não definida."
    exit 2
fi

# Verifica as dependências
for CMD in pg_dump gzip aws flock; do
    if ! command -v "$CMD" >/dev/null 2>&1; then
        log "ERRO: Dependência não encontrada: $CMD"
        exit 3
    fi
done

# Verifica espaço disponível no diretório de trabalho
AVAILABLE_KB=$(df -Pk "$WORK_DIR" | awk 'NR==2 {print $4}')
MIN_FREE_KB=$((MIN_FREE_GB * 1024 * 1024))

if [[ "$AVAILABLE_KB" -lt "$MIN_FREE_KB" ]]; then
    log "ERRO: Espaço insuficiente em disco."
    log "Disponível: $((AVAILABLE_KB / 1024 / 1024)) GiB."
    log "Mínimo exigido: ${MIN_FREE_GB} GiB."
    exit 4
fi

log "Validação de dependências e espaço concluída."

# ------------------------------------------------------------
# 5. BLOQUEIO DE EXECUÇÕES SIMULTÂNEAS
# ------------------------------------------------------------

exec 9>"$LOCK_FILE"

if ! flock -n 9; then
    log "ERRO: Outra execução do backup já está em andamento."
    exit 5
fi

log "Lock adquirido com sucesso."

# ------------------------------------------------------------
# 6. DEFINIÇÃO DO NOME DO ARQUIVO
# ------------------------------------------------------------

TIMESTAMP=$(date -u '+%Y-%m-%dT%H%M%SZ')

FILE_NAME="${DB_NAME}_${TIMESTAMP}.sql.gz"

BACKUP_FILE="${WORK_DIR}/${FILE_NAME}"
PARTIAL_FILE="${BACKUP_FILE}.part"

S3_KEY="${S3_PREFIX:+${S3_PREFIX}/}${FILE_NAME}"

log "Arquivo de backup: $FILE_NAME"

# ------------------------------------------------------------
# 7. EXECUÇÃO DO PG_DUMP COM COMPRESSÃO
# ------------------------------------------------------------

log "Iniciando pg_dump e compressão gzip."

if pg_dump \
    --host="$DB_HOST" \
    --port="$DB_PORT" \
    --username="$DB_USER" \
    --dbname="$DB_NAME" \
    --format=plain \
    --no-password \
    --verbose \
    | gzip -c > "$PARTIAL_FILE"; then

    log "Dump e compressão concluídos."

else
    PIPE_STATUS=("${PIPESTATUS[@]}")

    if [[ "${PIPE_STATUS[0]}" -ne 0 ]]; then
        log "ERRO: pg_dump falhou. Código: ${PIPE_STATUS[0]}"
        exit 6
    fi

    log "ERRO: gzip falhou. Código: ${PIPE_STATUS[1]}"
    exit 7
fi

# ------------------------------------------------------------
# 8. VALIDAÇÃO DA INTEGRIDADE
# ------------------------------------------------------------

log "Validando integridade do arquivo gzip."

if ! gzip -t "$PARTIAL_FILE"; then
    log "ERRO: Arquivo gzip corrompido."
    exit 7
fi

log "Integridade do arquivo confirmada."

# Renomeia somente depois de validar o arquivo
mv -- "$PARTIAL_FILE" "$BACKUP_FILE"
PARTIAL_FILE=""

# Registra tamanho do backup
BACKUP_SIZE=$(stat -c '%s' "$BACKUP_FILE")
BACKUP_SIZE_GB=$(awk -v size="$BACKUP_SIZE" \
    'BEGIN {printf "%.2f", size / 1024 / 1024 / 1024}')

log "Tamanho do backup: ${BACKUP_SIZE_GB} GiB."

# ------------------------------------------------------------
# 9. UPLOAD PARA O AMAZON S3
# ------------------------------------------------------------

log "Iniciando upload para o S3."

if ! aws s3 cp \
    "$BACKUP_FILE" \
    "s3://${S3_BUCKET}/${S3_KEY}" \
    --region "$AWS_REGION" \
    --only-show-errors; then

    log "ERRO: Falha no upload para o S3."
    exit 8
fi

log "Upload concluído. Verificando objeto remoto."

# ------------------------------------------------------------
# 10. CONFIRMAÇÃO DO UPLOAD
# ------------------------------------------------------------

if ! REMOTE_SIZE=$(aws s3api head-object \
    --bucket "$S3_BUCKET" \
    --key "$S3_KEY" \
    --region "$AWS_REGION" \
    --query 'ContentLength' \
    --output text); then

    log "ERRO: Não foi possível confirmar o objeto no S3."
    exit 8
fi

if [[ "$REMOTE_SIZE" != "$BACKUP_SIZE" ]]; then
    log "ERRO: Tamanho do arquivo remoto diferente do local."
    log "Local: $BACKUP_SIZE | Remoto: $REMOTE_SIZE"
    exit 8
fi

log "Upload confirmado. Tamanho remoto: $REMOTE_SIZE bytes."

# ------------------------------------------------------------
# 11. REMOÇÃO DA CÓPIA LOCAL
# ------------------------------------------------------------

if ! rm -f -- "$BACKUP_FILE"; then
    log "ERRO: Não foi possível remover o backup local."
    exit 1
fi

log "Cópia local removida após confirmação do S3."

# ------------------------------------------------------------
# 12. POLÍTICA DE RETENÇÃO NO S3
# ------------------------------------------------------------

log "Iniciando limpeza de backups antigos."

CUTOFF_EPOCH=$(date -u -d "${RETENTION_DAYS} days ago" +%s)

if ! S3_LIST=$(aws s3 ls \
    "s3://${S3_BUCKET}/${S3_PREFIX}" \
    --recursive \
    --region "$AWS_REGION"); then

    log "ERRO: Falha ao listar objetos do S3."
    exit 9
fi

while IFS= read -r LINE; do

    [[ -z "$LINE" ]] && continue

    # Extrai data, horário e chave do objeto
    OBJECT_DATE=$(awk '{print $1}' <<< "$LINE")
    OBJECT_TIME=$(awk '{print $2}' <<< "$LINE")
    OBJECT_KEY=$(awk '{print $4}' <<< "$LINE")

    [[ -z "$OBJECT_KEY" ]] && continue

    # Ignora objetos que não sejam backups do Ledger
    [[ "$OBJECT_KEY" != *.sql.gz ]] && continue
    [[ "$OBJECT_KEY" != "${DB_NAME}_"* ]] && continue

    OBJECT_EPOCH=$(date -u -d \
        "${OBJECT_DATE} ${OBJECT_TIME} UTC" +%s) || {
        log "AVISO: Não foi possível interpretar data de $OBJECT_KEY."
        continue
    }

    # Exclui objetos com mais de 30 dias
    if [[ "$OBJECT_EPOCH" -lt "$CUTOFF_EPOCH" ]]; then

        log "Removendo backup expirado: $OBJECT_KEY"

        if ! aws s3 rm \
            "s3://${S3_BUCKET}/${OBJECT_KEY}" \
            --region "$AWS_REGION" \
            --only-show-errors; then

            log "ERRO: Falha ao remover $OBJECT_KEY."
            exit 9
        fi

        log "Backup expirado removido."
    fi

done <<< "$S3_LIST"

log "Política de retenção concluída."

# ------------------------------------------------------------
# 13. FINALIZAÇÃO
# ------------------------------------------------------------

BACKUP_SUCCESS=true

log "Backup do Ledger concluído."
log "Destino: s3://${S3_BUCKET}/${S3_KEY}"
log "Tamanho: ${BACKUP_SIZE_GB} GiB."

exit 0
```

### 2. Crontab · execução diária às 03:00 UTC

```cron
# Define o fuso horário do cron como UTC
CRON_TZ=UTC

# Backup diário do Ledger às 03:00
0 3 * * * /usr/local/sbin/ledger-backup.sh
```

### 3. Matriz de requisitos

| Requisito | Como foi atendido |
|---|---|
| 1. Dump com gzip direto | `pg_dump` conectado ao `gzip` por pipe, sem arquivo intermediário descompactado. |
| 2. Nome com data/hora UTC | `date -u '+%Y-%m-%dT%H%M%SZ'`. |
| 3. Upload para S3 | `aws s3 cp` com região `us-east-1`. |
| 4. Retenção de 30 dias | Lista objetos e remove backups `.sql.gz` mais antigos que 30 dias. |
| 5. Logging completo | Função `log()` registra timestamps UTC, início, etapas, tamanho e duração. |
| 6. Exit codes distintos | Códigos de 0 a 9 documentados no cabeçalho. |
| `set -euo pipefail` | Interrompe em erros, variáveis não definidas e falhas em pipelines. |
| Validar PGPASSWORD | Verifica se a variável existe e não está vazia, sem imprimir seu conteúdo. |
| Validar dependências | Verifica `pg_dump`, `gzip`, `aws` e `flock`. |
| Validar espaço disponível | Exige pelo menos 15 GiB livres. |
| Impedir concorrência | Utiliza `flock -n` com arquivo de lock. |
| Remover arquivos parciais | `trap cleanup EXIT` remove arquivos `.part` em caso de falha. |
| Validar integridade | Executa `gzip -t` antes do upload. |
| Confirmar upload | `aws s3api head-object` verifica existência e tamanho do objeto. |
| Proteger a senha | Não imprime nem registra `PGPASSWORD`. |
| Configurações centralizadas | Variáveis de configuração concentradas no início do script. |

### 4. Permissões IAM mínimas

A IAM Role da instância EC2 precisa das seguintes permissões no bucket `hvt-ledger-backups`:

| Permissão IAM | Finalidade |
|---|---|
| `s3:ListBucket` | Listar backups existentes para verificar a retenção. |
| `s3:PutObject` | Enviar novos backups para o bucket. |
| `s3:GetObject` | Consultar os metadados dos objetos com `head-object`. |
| `s3:DeleteObject` | Remover backups expirados após 30 dias. |
| `s3:AbortMultipartUpload` | Cancelar uploads multipart incompletos em caso de falha. |

Exemplo de política IAM:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "LedgerBackupBucketAccess",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::hvt-ledger-backups"
    },
    {
      "Sid": "LedgerBackupObjectAccess",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:AbortMultipartUpload"
      ],
      "Resource": "arn:aws:s3:::hvt-ledger-backups/*"
    }
  ]
}
```

> **Atenção para produção:** a role também precisa ter permissão de leitura do segredo no AWS Secrets Manager (`secretsmanager:GetSecretValue`) caso a própria instância EC2 seja responsável por recuperar e exportar `PGPASSWORD`. Se o bucket utilizar criptografia SSE-KMS com uma chave gerenciada pelo cliente, também serão necessárias permissões KMS apropriadas.
>
> **Recomendação SRE:** antes de colocar em produção, faça um teste de restauração do dump em outro PostgreSQL. Um backup só deve ser considerado confiável depois que sua restauração for validada.

## Justificativa

O **Role** (seção `# ROLE`) define um SRE sênior especialista em PostgreSQL, bash e AWS, puxando uma postura defensiva coerente com a demanda da Lorraine: tratar falhas e não perder dados. A **Task** (seção `# TASK`) traz todo o ambiente do enunciado, os 6 comportamentos esperados e requisitos de robustez explícitos (`pipefail`, `flock`, `trap`, `gzip -t`, confirmação no S3), que fecham brechas como subir um dump vazio como se fosse backup válido. O **Format** (seção `# FORMAT`) pede script com tabela de exit codes, linha de crontab, matriz de requisitos e permissões IAM, entregando tudo o que é preciso para colocar em produção.

## Observações sobre o output

- Pontos fortes: o dump é gravado como `.part` e só é renomeado depois do `gzip -t`; o `PIPESTATUS` separa falha do `pg_dump` (exit 6) da falha do `gzip` (exit 7); o tamanho no S3 é comparado com o local antes de apagar a cópia; e a retenção só roda depois de um upload confirmado.
- Pontos fracos: o mínimo de 15 GiB livres é pouco acima do dump atual (~12 GB) e vai falhar com o crescimento do banco; a retenção feita pelo script seria mais confiável como regra de *S3 Lifecycle*; o `aws s3 ls` mostra datas no fuso local, mas o script as interpreta como UTC; e o modelo adicionou notas fora das seções pedidas e entregou as permissões IAM como tabela e JSON em vez de "lista curta".
- O que faria diferente: informar a versão do servidor PostgreSQL (o cliente padrão do Ubuntu 22.04 é o 14), pedir margem de disco proporcional ao último backup e reforçar a restrição de formato com "não adicione notas ou recomendações finais".
