# Questão 01 · Dockerfile para o Lift

**Framework:** R-T-F (Role · Task · Format)
**Modelo utilizado:** ChatGPT (OpenAI)

## Cenário

O Lift vai sair das VMs e entrar no cluster Kubernetes da empresa. É uma API Python/Flask na porta 8080, com dependências em `requirements.txt` e duas variáveis de ambiente obrigatórias no runtime: `DATABASE_URL` e `API_KEY`. Em produção sobe com `gunicorn --bind 0.0.0.0:8080 --workers 4 app:app`. Falta o Dockerfile, seguindo as boas práticas de criação.

## Prompt

````text
# ROLE
Você é um engenheiro DevOps sênior, especialista em containerização de aplicações
Python e em segurança de imagens Docker para clusters Kubernetes em produção.

# TASK
Escreva o Dockerfile do "Lift", uma API Python/Flask que está saindo de VMs e
vai rodar no cluster Kubernetes da empresa.

Contexto do projeto:
- Estrutura:
  lift/
  ├── app.py
  ├── requirements.txt
  ├── lib/
  │   ├── auth.py
  │   └── storage.py
  └── tests/
      └── test_app.py
- requirements.txt:
  Flask==3.0.0
  gunicorn==21.2.0
  requests==2.31.0
  python-dotenv==1.0.0
  psycopg2-binary==2.9.9
- A aplicação escuta na porta 8080.
- Em produção sobe com: gunicorn --bind 0.0.0.0:8080 --workers 4 app:app
- Precisa das variáveis de ambiente DATABASE_URL e API_KEY no runtime.

Requisitos (boas práticas obrigatórias):
1. Imagem base oficial, slim e com versão fixa (Python 3.12, compatível com
   psycopg2-binary 2.9.9).
2. Multi-stage build: dependências instaladas em um estágio de build e só o
   necessário copiado para a imagem final.
3. Aproveitar o cache de camadas: copiar requirements.txt e instalar as
   dependências antes de copiar o código.
4. pip install com --no-cache-dir.
5. Executar como usuário não-root.
6. Definir PYTHONDONTWRITEBYTECODE=1 e PYTHONUNBUFFERED=1.
7. Não incluir a pasta tests/ na imagem final.
8. NÃO definir DATABASE_URL nem API_KEY no Dockerfile: elas serão injetadas pelo
   Kubernetes (Secret). Não use ARG/ENV com valores de segredo.
9. EXPOSE 8080 e CMD em exec form (formato JSON) com o comando gunicorn acima.
10. Não adicione HEALTHCHECK: no Kubernetes a saúde é verificada por probes.

# FORMAT
Responda exatamente nesta ordem, sem texto fora dessas seções:
1. Um bloco de código ```dockerfile``` com o Dockerfile completo, com comentários
   curtos em português explicando cada etapa.
2. Um bloco de código com o arquivo .dockerignore correspondente.
3. Uma tabela Markdown com as colunas "Boa prática" e "Onde foi aplicada",
   uma linha por requisito.
4. Um bloco ```bash``` com os comandos para build e para rodar localmente,
   passando DATABASE_URL e API_KEY com -e.
````

## Output

### 1. Dockerfile

```dockerfile
# ==========================================
# Estágio 1: Build das dependências
# ==========================================
FROM python:3.12.8-slim AS builder

# Define diretório de trabalho
WORKDIR /app

# Evita arquivos .pyc e mantém logs em tempo real
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Copia primeiro os requisitos para aproveitar o cache
COPY requirements.txt .

# Instala as dependências em um diretório isolado
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt


# ==========================================
# Estágio 2: Imagem final de produção
# ==========================================
FROM python:3.12.8-slim AS production

# Define diretório de trabalho
WORKDIR /app

# Configura o comportamento do Python
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Cria usuário sem privilégios administrativos
RUN groupadd --system appgroup && \
    useradd --system --gid appgroup --no-create-home appuser

# Copia somente as dependências instaladas no build
COPY --from=builder /install /usr/local

# Copia os arquivos da aplicação
COPY --chown=appuser:appgroup app.py .
COPY --chown=appuser:appgroup lib/ ./lib/

# Executa a aplicação sem privilégios de root
USER appuser

# Documenta a porta utilizada pela API
EXPOSE 8080

# Inicia a aplicação com Gunicorn
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "4", "app:app"]
```

### 2. .dockerignore

```gitignore
# Testes
tests/

# Ambiente virtual
venv/
.venv/
env/

# Cache e arquivos temporários
__pycache__/
*.py[cod]
*.pyo
*.pyd
.pytest_cache/
.mypy_cache/

# Arquivos de configuração local
.env
.env.*
!.env.example

# Controle de versão
.git/
.gitignore

# IDEs
.vscode/
.idea/

# Documentação
README.md
*.md

# Logs
*.log

# Arquivos do sistema
.DS_Store
Thumbs.db
```

### 3. Tabela de boas práticas

| Boa prática | Onde foi aplicada |
|---|---|
| 1. Imagem oficial Python 3.12 slim com versão fixa | `FROM python:3.12.8-slim` nos dois estágios. |
| 2. Multi-stage build | Estágios `builder` e `production`. |
| 3. Aproveitamento do cache | `COPY requirements.txt` antes da instalação e da cópia do código. |
| 4. Instalação sem cache | `pip install --no-cache-dir`. |
| 5. Usuário não-root | Criação do usuário `appuser` e instrução `USER appuser`. |
| 6. Variáveis do Python | `PYTHONDONTWRITEBYTECODE=1` e `PYTHONUNBUFFERED=1`. |
| 7. Exclusão dos testes | `tests/` no `.dockerignore` e ausência de cópia dessa pasta no estágio final. |
| 8. Segredos externos | `DATABASE_URL` e `API_KEY` não são declaradas no Dockerfile. Devem ser injetadas pelo Kubernetes via Secret. |
| 9. Porta e inicialização | `EXPOSE 8080` e `CMD` em formato JSON com Gunicorn e quatro workers. |
| 10. Sem HEALTHCHECK | Não foi declarado. A verificação será realizada pelas probes do Kubernetes. |

### 4. Comandos para build e execução local

```bash
# Build da imagem
docker build -t lift:latest .

# Executar o container localmente
docker run -d \
  --name lift-api \
  -p 8080:8080 \
  -e DATABASE_URL="postgresql://usuario:senha@localhost:5432/lift" \
  -e API_KEY="sua-chave-api" \
  lift:latest

# Verificar os logs
docker logs -f lift-api

# Parar o container
docker stop lift-api

# Remover o container
docker rm lift-api
```

> **Observação importante:** para execução local, se o PostgreSQL estiver instalado na máquina host, `localhost` dentro do container não aponta para o host. No Linux, você pode utilizar `--add-host=host.docker.internal:host-gateway` e trocar o endereço da conexão para `host.docker.internal`.

## Justificativa

O prompt foi dividido em três seções explícitas (`# ROLE`, `# TASK`, `# FORMAT`), uma para cada componente do framework.

| Componente | Onde aparece no prompt | Por que importa |
|---|---|---|
| **Role** | Seção `# ROLE`: engenheiro DevOps sênior, especialista em containerização Python e segurança de imagens para Kubernetes em produção. | Direciona o modelo para as decisões de quem já colocou imagens em produção, puxando segurança e otimização por padrão, e não apenas "um Dockerfile que funciona". |
| **Task** | Seção `# TASK`: objetivo, todo o contexto do enunciado (estrutura, `requirements.txt`, porta, comando do Gunicorn, variáveis de ambiente) e 10 requisitos numerados. | Transforma "seguir as boas práticas" (vago) em critérios verificáveis. Também previne erros comuns, como embutir segredos na imagem via `ENV`/`ARG` ou declarar um `HEALTHCHECK` que o Kubernetes ignora. |
| **Format** | Seção `# FORMAT`: ordem fixa de quatro entregáveis (Dockerfile comentado, `.dockerignore`, tabela de conformidade, comandos de teste). | Deixa a saída pronta para uso e auditável: a tabela permite conferir requisito por requisito se o modelo cumpriu tudo. |

### Análise do resultado

O output atendeu aos 10 requisitos, como mostra a tabela de conformidade gerada pelo próprio modelo. Destaques:

- A imagem foi fixada em uma versão de patch (`3.12.8-slim`), mais restrita que o pedido, o que deixa o build reproduzível.
- O estágio de build instala as dependências com `--prefix=/install`, e o estágio final copia só esse diretório, sem resíduos do build.
- O código entra com `--chown=appuser:appgroup`, e só `app.py` e `lib/` são copiados, então `tests/` fica fora da imagem por dois caminhos (cópia seletiva e `.dockerignore`).
- Nenhum segredo aparece no Dockerfile. Os valores só são passados em tempo de execução (`-e` no teste local, Secret no Kubernetes).

Ponto de atenção: o modelo adicionou uma observação final fora das quatro seções, desviando levemente da instrução "sem texto fora dessas seções". O conteúdo é útil (explica por que `localhost` não alcança o banco do host), mas mostra que restrições negativas de formato nem sempre são seguidas à risca, e poderiam ser reforçadas no prompt.
