# Hill Valley Tech · Prompt Engineering

Desafio do **Módulo 02 · Cap 01 – Fundamentos de Prompt Engineering** da Pós **AIOps e IA na Engenharia de Cloud** (DevOpsPro).

## Contexto

A Hill Valley Tech é uma empresa fictícia com cinco sistemas em produção:

| Sistema | Papel |
|---|---|
| **Chronos** | API gateway e plataforma core, ponto de entrada de todo o tráfego |
| **Ledger** | Data warehouse em PostgreSQL com histórico de transações e eventos |
| **Reactor** | Processamento assíncrono por filas de mensagens |
| **Beacon** | Observabilidade: métricas, logs e alertas usados pelo plantão |
| **Lift** | Produto em beta, amadurecido à parte do core |

E um time enxuto:

| Pessoa | Papel |
|---|---|
| **Doc Brown** | CTO, direção técnica |
| **Jennifer Parker** | PM, prioriza o que o produto entrega |
| **Lorraine Baines** | Líder de SRE e responsável pelo plantão; cobra runbooks e procedimentos documentados |
| **George McFly** | Engenheiro sênior veterano, autor de boa parte do sistema legado |
| **Goldie Wilson** | CEO, olhar de custo e crescimento |
| **Strickland** | Head de segurança e compliance, aprova os padrões internos de código |

Em cada questão, a entrega é um **prompt de IA** aplicando o framework indicado, executado em um modelo, com o **output registrado** e a **justificativa** mostrando como os componentes do framework aparecem no prompt.

## Questões

| # | Cenário | Framework | Modelo |
|---|---|---|---|
| [01](questao-01/) | Dockerfile para o Lift | R-T-F | ChatGPT (OpenAI) |
| [02](questao-02/) | Script de backup do Ledger | R-T-F | ChatGPT (OpenAI) |
| [03](questao-03/) | Relatório de redução de custos cloud | T-A-G | Claude Opus 5.5 (Anthropic) |
| [04](questao-04/) | Relatório mensal de transações do Ledger | T-A-G | Claude Opus 5.5 (Anthropic) |
| [05](questao-05/) | Modernizar deployment legado | B-A-B | Claude Opus 5.5 (Anthropic) |
| [06](questao-06/) | Módulo Terraform no padrão interno | C-A-R-E | Claude Opus 5.5 (Anthropic) |
| [07](questao-07/) | Runbook para alerta recorrente | R-I-S-E | Claude Opus 5.5 (Anthropic) |
| [08](questao-08/) | Postmortem técnico de incidente em produção | R-I-S-E (escolhido) | Claude Opus 5.5 (Anthropic) |

## Organização do repositório

Cada questão tem a sua própria pasta (`questao-01/`, `questao-02/`…) com um único `README.md` sempre com a mesma estrutura: framework e **Modelo** no topo (com o motivo da escolha), depois o cenário, o **Prompt**, o **Output** e a **Justificativa**, seguidos de observações sobre a qualidade do output.

Motivos da escolha:

- **Leitura direta no GitHub:** o `README.md` de cada pasta é renderizado automaticamente, então a questão inteira pode ser lida sem abrir arquivos separados.
- **Uma questão, um contexto:** prompt, output e justificativa ficam juntos, o que facilita comparar o que foi pedido com o que o modelo entregou.
- **Prompts delimitados:** cada prompt está num bloco de código isolado, com o texto exato usado, pronto para copiar e reexecutar.
- **Histórico pelo Git:** quando um prompt é refeito (por exemplo, trocando o modelo), a versão anterior continua no histórico de commits.

Limitação conhecida: como o prompt fica embutido no README, não é possível versioná-lo ou reutilizá-lo isoladamente (por exemplo, como um arquivo `prompt.md` independente). Essa decisão será revisitada no Capítulo 4 (Criação e Versionamento de Prompts).
