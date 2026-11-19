# Questão 06 · Módulo Terraform no padrão interno

**Framework:** C-A-R-E (Context · Action · Result · Example)

**Modelo:** Claude Opus 5.5 (Anthropic, via claude.ai, esforço Médio). Escolhido por gerar código de infraestrutura em vários arquivos mantendo a consistência entre eles e por conhecer as mudanças recentes do provider AWS.

## Cenário

Strickland (segurança e compliance) publicou o padrão interno de IaC: tags obrigatórias `Owner`, `CostCenter` e `Environment` em todo recurso; prefixo `hvt-` nos nomes; todo bucket S3 com encryption (SSE-S3 no mínimo), versioning, block public access total e logging; variáveis em `variables.tf` com `description` e `type`. Doc Brown pediu um módulo Terraform reutilizável para buckets S3 aderentes ao padrão, com exemplo de uso, no mesmo estilo do módulo de VPC da empresa.

## Prompt

```text
# CONTEXT
A Hill Valley Tech tem um padrão interno de IaC, publicado pelo Strickland
(head de segurança e compliance), que todo módulo Terraform novo precisa
seguir:
- Tags obrigatórias em todo recurso: Owner, CostCenter, Environment.
- Prefixo hvt- nos nomes de recursos.
- Todo bucket S3 com: encryption habilitada (SSE-S3 no mínimo), versioning
  ativo, block public access total e logging configurado.
- Variáveis de entrada em variables.tf, com description e type obrigatórios.

O Doc Brown (CTO) pediu um módulo Terraform reutilizável para criar buckets
S3 aderentes a esse padrão. O módulo será consumido por todos os times da
empresa, que vão usá-lo sem conhecer os detalhes do padrão.

# ACTION
Escreva o módulo Terraform completo para criar um bucket S3 aderente ao
padrão, seguindo estas regras:
1. Use AWS provider >= 5.0 e os recursos separados de configuração do bucket
   (aws_s3_bucket_versioning, aws_s3_bucket_server_side_encryption_configuration,
   aws_s3_bucket_public_access_block, aws_s3_bucket_logging), não os blocos
   inline depreciados dentro de aws_s3_bucket.
2. Monte o nome do bucket sempre com o prefixo hvt- a partir das variáveis,
   sem permitir que o consumidor remova o prefixo.
3. Use SSE-S3 (AES256) por padrão, com opção de usar SSE-KMS informando uma
   chave KMS.
4. Receba o bucket de destino dos logs de acesso como variável obrigatória.
5. Valide as entradas com blocos validation: environment restrito a dev,
   staging ou production, e o nome final do bucket respeitando as regras de
   nomes do S3 (minúsculas, até 63 caracteres).
6. Desabilite ACLs (object ownership BucketOwnerEnforced) e não permita
   force_destroy por padrão.
7. Aplique as tags obrigatórias em todos os recursos que suportam tags e
   permita que o consumidor adicione tags extras sem sobrescrever as
   obrigatórias. Se algum recurso não suportar tags, explique.
8. Exponha outputs úteis (id, arn, nome e domínio do bucket).

# RESULT
Entregue os arquivos do módulo, cada um em seu próprio bloco de código, com o
nome do arquivo como título:
- versions.tf
- variables.tf (toda variável com description e type)
- main.tf
- outputs.tf
- examples/basic/main.tf, com um exemplo de uso real consumindo o módulo
- README.md curto, com descrição, exemplo de uso e tabela de variáveis

Depois dos arquivos, inclua:
- Uma tabela com as colunas "Regra do padrão" e "Onde é atendida",
  uma linha por regra do Strickland.
- Os comandos para validar o módulo antes de publicar.
- Os pré-requisitos que ficam fora do módulo (por exemplo, permissões no
  bucket de logs).

# EXAMPLE
Siga o mesmo estilo do módulo de VPC que já existe na empresa: variáveis com
description e type, tags centralizadas em locals.common_tags e aplicadas com
merge(), e nomes com prefixo hvt- seguido do ambiente.

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}

locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }
}

resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags = merge(local.common_tags, {
    Name = "hvt-vpc-${var.environment}"
  })
}
```

## Output

### versions.tf

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}
```

### variables.tf

```hcl
variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "environment deve ser dev, staging ou production."
  }
}

variable "name" {
  description = "Sufixo do nome do bucket. O nome final será hvt-<environment>-<name>. Não inclua o prefixo hvt- nem o ambiente."
  type        = string

  validation {
    condition     = can(regex("^[a-z0-9]([a-z0-9-]*[a-z0-9])?$", var.name))
    error_message = "name deve conter apenas letras minúsculas, números e hífens, começando e terminando com letra ou número."
  }

  validation {
    condition     = !startswith(var.name, "hvt-")
    error_message = "Não inclua o prefixo hvt- em name: o módulo adiciona automaticamente."
  }

  validation {
    # 63 (limite S3) - 15 ("hvt-" + "production" + "-", pior caso)
    condition     = length(var.name) <= 48
    error_message = "name deve ter no máximo 48 caracteres para que o nome final caiba no limite de 63 do S3."
  }
}

variable "owner" {
  description = "Time ou pessoa responsável pelo bucket (tag obrigatória Owner)"
  type        = string

  validation {
    condition     = length(trimspace(var.owner)) > 0
    error_message = "owner não pode ser vazio."
  }
}

variable "cost_center" {
  description = "Centro de custo do bucket (tag obrigatória CostCenter)"
  type        = string

  validation {
    condition     = length(trimspace(var.cost_center)) > 0
    error_message = "cost_center não pode ser vazio."
  }
}

variable "log_bucket_name" {
  description = "Nome de um bucket existente, na mesma região, que receberá os logs de acesso deste bucket"
  type        = string

  validation {
    condition     = length(trimspace(var.log_bucket_name)) > 0
    error_message = "log_bucket_name é obrigatório: todo bucket precisa de logging configurado."
  }
}

variable "log_prefix" {
  description = "Prefixo dos logs de acesso no bucket de destino. Se null, usa '<nome-do-bucket>/'"
  type        = string
  default     = null
}

variable "kms_key_arn" {
  description = "ARN de uma chave KMS para usar SSE-KMS. Se null, usa SSE-S3 (AES256)"
  type        = string
  default     = null

  validation {
    condition     = var.kms_key_arn == null || can(regex("^arn:aws[a-zA-Z-]*:kms:", var.kms_key_arn))
    error_message = "kms_key_arn deve ser um ARN de chave KMS válido ou null."
  }
}

variable "force_destroy" {
  description = "Permite destruir o bucket mesmo com objetos. Mantenha false em production"
  type        = bool
  default     = false
}

variable "tags" {
  description = "Tags extras. Não sobrescrevem as obrigatórias (Owner, CostCenter, Environment)"
  type        = map(string)
  default     = {}
}
```

### main.tf

```hcl
locals {
  # Nome sempre montado pelo módulo: o consumidor não controla o prefixo
  bucket_name = "hvt-${var.environment}-${var.name}"

  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }

  # Tags extras primeiro: as obrigatórias vêm depois no merge e prevalecem
  tags = merge(var.tags, local.common_tags, {
    Name = local.bucket_name
  })

  use_kms = var.kms_key_arn != null
}

resource "aws_s3_bucket" "this" {
  bucket        = local.bucket_name
  force_destroy = var.force_destroy
  tags          = local.tags

  lifecycle {
    # Segunda barreira: valida o nome final completo
    precondition {
      condition     = length(local.bucket_name) <= 63 && can(regex("^[a-z0-9][a-z0-9-]*[a-z0-9]$", local.bucket_name))
      error_message = "O nome final do bucket (${local.bucket_name}) viola as regras de nome do S3."
    }

    precondition {
      condition     = var.log_bucket_name != local.bucket_name
      error_message = "O bucket de logs não pode ser o próprio bucket."
    }
  }
}

# ACLs desabilitadas: o dono do bucket é dono de todos os objetos
resource "aws_s3_bucket_ownership_controls" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    object_ownership = "BucketOwnerEnforced"
  }
}

# Block public access total
resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Versioning ativo
resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Criptografia: SSE-S3 por padrão, SSE-KMS se uma chave for informada
resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = local.use_kms ? "aws:kms" : "AES256"
      kms_master_key_id = local.use_kms ? var.kms_key_arn : null
    }
    # Bucket key reduz chamadas (e custo) ao KMS
    bucket_key_enabled = local.use_kms
  }
}

# Logs de acesso enviados ao bucket central
resource "aws_s3_bucket_logging" "this" {
  bucket = aws_s3_bucket.this.id

  target_bucket = var.log_bucket_name
  target_prefix = coalesce(var.log_prefix, "${local.bucket_name}/")
}
```

### outputs.tf

```hcl
output "bucket_id" {
  description = "ID do bucket (igual ao nome)"
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "ARN do bucket"
  value       = aws_s3_bucket.this.arn
}

output "bucket_name" {
  description = "Nome final do bucket, com prefixo hvt- e ambiente"
  value       = aws_s3_bucket.this.bucket
}

output "bucket_domain_name" {
  description = "Domínio global do bucket (<nome>.s3.amazonaws.com)"
  value       = aws_s3_bucket.this.bucket_domain_name
}

output "bucket_regional_domain_name" {
  description = "Domínio regional do bucket, recomendado para CloudFront e acesso na mesma região"
  value       = aws_s3_bucket.this.bucket_regional_domain_name
}
```

### examples/basic/main.tf

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# Bucket de relatórios do time de dados, com SSE-S3 (padrão)
module "reports_bucket" {
  source = "../../"

  environment     = "staging"
  name            = "data-reports"
  owner           = "time-dados"
  cost_center     = "CC-1042"
  log_bucket_name = "hvt-staging-s3-access-logs"

  tags = {
    Project = "lift"
  }
}

# Variante com SSE-KMS
module "invoices_bucket" {
  source = "../../"

  environment     = "production"
  name            = "billing-invoices"
  owner           = "time-financeiro"
  cost_center     = "CC-2001"
  log_bucket_name = "hvt-production-s3-access-logs"
  kms_key_arn     = "arn:aws:kms:us-east-1:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab"
}

output "reports_bucket_name" {
  value = module.reports_bucket.bucket_name # hvt-staging-data-reports
}
```

### README.md

````markdown
# Módulo Terraform: s3-bucket (padrão HVT)

Cria um bucket S3 aderente ao padrão de IaC da Hill Valley Tech: nome com
prefixo `hvt-<ambiente>-`, criptografia (SSE-S3 ou SSE-KMS), versioning,
block public access total, ACLs desabilitadas, logs de acesso e tags
obrigatórias. O consumidor não precisa configurar nada disso: o módulo
garante o padrão.

## Uso

```hcl
module "reports_bucket" {
  source = "git::https://github.com/hillvalleytech/terraform-modules.git//s3-bucket?ref=v1.0.0"

  environment     = "staging"
  name            = "data-reports"          # nome final: hvt-staging-data-reports
  owner           = "time-dados"
  cost_center     = "CC-1042"
  log_bucket_name = "hvt-staging-s3-access-logs"
}
```

## Variáveis

| Nome | Tipo | Obrigatória | Padrão | Descrição |
|---|---|---|---|---|
| environment | string | sim | — | dev, staging ou production |
| name | string | sim | — | Sufixo do nome (sem `hvt-`), até 48 caracteres |
| owner | string | sim | — | Tag Owner |
| cost_center | string | sim | — | Tag CostCenter |
| log_bucket_name | string | sim | — | Bucket existente que recebe os logs de acesso |
| log_prefix | string | não | `<nome-do-bucket>/` | Prefixo dos logs |
| kms_key_arn | string | não | null (SSE-S3) | Chave KMS para SSE-KMS |
| force_destroy | bool | não | false | Permite apagar bucket com objetos |
| tags | map(string) | não | {} | Tags extras (não sobrescrevem as obrigatórias) |

## Outputs

`bucket_id`, `bucket_arn`, `bucket_name`, `bucket_domain_name`, `bucket_regional_domain_name`.
````

### Sobre tags

Só o `aws_s3_bucket` recebe tags. Os recursos de versioning, encryption, public access block, logging e ownership controls não têm argumento `tags` no provider, porque não são recursos AWS independentes com ARN próprio. Eles são configurações do bucket, gravadas nele via API (`PutBucketVersioning`, `PutBucketEncryption` etc.). Como herdam o contexto do bucket, a tag do bucket cobre o rastreamento de dono e custo. Se o time quiser tags em todos os recursos AWS taggeáveis de um projeto, a alternativa complementar é `default_tags` no provider do consumidor.

### Conformidade com o padrão

| Regra do padrão | Onde é atendida |
|---|---|
| Tags obrigatórias Owner, CostCenter, Environment | `local.common_tags` aplicado com `merge()` em `aws_s3_bucket.this`; as obrigatórias vêm por último no merge e não podem ser sobrescritas por `var.tags`; `owner` e `cost_center` validados como não vazios |
| Prefixo `hvt-` nos nomes | `local.bucket_name = "hvt-${var.environment}-${var.name}"`; o consumidor só informa o sufixo, e a validação rejeita `name` começando com `hvt-` |
| Encryption (SSE-S3 no mínimo) | `aws_s3_bucket_server_side_encryption_configuration.this`: AES256 por padrão, `aws:kms` se `kms_key_arn` for informado; não há opção para desligar |
| Versioning ativo | `aws_s3_bucket_versioning.this` com `status = "Enabled"` fixo |
| Block public access total | `aws_s3_bucket_public_access_block.this` com as 4 flags em `true`, reforçado por `BucketOwnerEnforced` em `aws_s3_bucket_ownership_controls.this` |
| Logging configurado | `aws_s3_bucket_logging.this`; `log_bucket_name` é obrigatória e sem default |
| Variáveis com description e type | Todas as variáveis em `variables.tf`, com validações de `environment`, nome, KMS e campos obrigatórios |

### Validação antes de publicar

```bash
# Formatação
terraform fmt -check -recursive

# Sintaxe e tipos do módulo
terraform init -backend=false
terraform validate

# Lint com regras do provider AWS
tflint --init && tflint --recursive

# Análise de segurança e compliance
checkov -d . --framework terraform
trivy config .

# Plano real a partir do exemplo (requer credenciais e o bucket de logs existente)
cd examples/basic
terraform init
terraform plan

# Testar as validações: estes devem FALHAR no plan
terraform plan -var='...'   # ex.: environment = "prod", name = "hvt-x", name com maiúsculas
```

Para automatizar o teste das validações, vale adicionar um `tests/*.tftest.hcl` com `expect_failures`, rodado com `terraform test`.

### Pré-requisitos fora do módulo

**Bucket de logs.** Ele precisa existir antes, na mesma região do bucket criado, e não pode ser o próprio bucket. Como o padrão usa `BucketOwnerEnforced`, a permissão de escrita deve vir de uma bucket policy que libere `s3:PutObject` para o principal `logging.s3.amazonaws.com`, com condições `aws:SourceAccount` e `aws:SourceArn`. Grants de ACL não funcionam nesse modo. O bucket de logs deve usar SSE-S3: a AWS não suporta SSE-KMS como criptografia padrão do bucket de destino de server access logs.

**Chave KMS.** Quando `kms_key_arn` for usado, a key policy deve permitir uso pelos papéis que leem e gravam no bucket, e a chave deve estar na mesma região.

**Permissões IAM de quem aplica.** O papel do pipeline precisa de `s3:CreateBucket` e das ações `s3:PutBucket*` (versioning, encryption, public access block, logging, ownership controls, tagging), mais `kms:DescribeKey` se usar KMS.

**Block public access na conta.** É uma camada adicional recomendada, configurada uma vez por conta com `aws_s3_account_public_access_block`, fora deste módulo.

**Versionamento do módulo.** Os times devem consumir o módulo por tag (`?ref=v1.0.0`), nunca pela branch principal.

**Lifecycle de versões antigas.** Com versioning ativo, versões antigas acumulam custo. Regras de expiração (`aws_s3_bucket_lifecycle_configuration`) ficam a cargo de cada time ou podem virar uma evolução opcional do módulo.

## Justificativa

O **Context** (seção `# CONTEXT`) apresenta o padrão do Strickland como regras explícitas e explica que o módulo será usado por times que não conhecem o padrão, então ele precisa garanti-lo sozinho. A **Action** (seção `# ACTION`) detalha como implementar cada regra, incluindo decisões que evitam erros comuns, como usar os recursos separados do provider 5.x em vez de blocos depreciados e impedir a remoção do prefixo `hvt-`. O **Result** (seção `# RESULT`) define os arquivos entregues, o exemplo de uso e a tabela de conformidade, e o **Example** (seção `# EXAMPLE`) traz o módulo de VPC como referência de estilo, refletida no output em `locals.common_tags`, `merge()` e variáveis com `description` e `type`.

## Observações sobre o output

- O módulo cumpre todas as regras do padrão e usa os recursos separados do provider 5.x. O consumidor não consegue fugir do padrão: o prefixo é montado pelo módulo, a criptografia não pode ser desligada, o logging não tem default e as tags obrigatórias vêm por último no `merge()`, então não são sobrescritas.
- O modelo explicou corretamente por que só o bucket recebe tags (os demais recursos são configurações do bucket, sem ARN próprio) e listou pré-requisitos que costumam quebrar o primeiro `apply`, como a bucket policy do bucket de logs e a restrição de SSE-KMS no destino dos logs.
- Pontos fracos: o comando para testar as validações ficou incompleto (`terraform plan -var='...'`), e nem funcionaria no exemplo, que tem os valores fixos no código; e o módulo não inclui uma bucket policy exigindo TLS (`aws:SecureTransport`), prática comum que o padrão não exige mas que um módulo "de segurança" costuma trazer.
- O que faria diferente: pedir no Result um arquivo `tests/*.tftest.hcl` com casos que devem falhar, em vez de comandos manuais, e perguntar no Action se há outras práticas de segurança recomendadas além do padrão.
