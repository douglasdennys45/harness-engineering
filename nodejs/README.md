# Harness de Engenharia — Node.js

Este diretório contém um **harness de desenvolvimento orientado a specs** para serviços Node.js usando Claude Code. Ele define um pipeline completo — da ideia ao merge — em que cada fase produz um artefato versionado, e a implementação só avança com aprovação explícita (do usuário nas fases de planejamento, dos agents de review/QA na fase de execução).

## Visão Geral do Fluxo

```
 /create-prd  ─────►  /create-techspec  ─────►  /create-tasks  ─────►  /run-task (N vezes)
     │                       │                        │                       │
     ▼                       ▼                        ▼                       ▼
   prd.md               techspec.md            tasks.md + tasks/        código + testes
 (o quê / por quê)      (como)                 [num]_task.md            + review + QA
                                                                        + commit + merge
```

Todos os artefatos de uma feature vivem em:

```
.cognup/specs/[num-nome-funcionalidade]/
├── prd.md                          # Requisitos de produto
├── techspec.md                     # Especificação técnica
├── tasks.md                        # Lista de tarefas + árvore de dependências
├── tasks/
│   ├── [num]_task.md               # Definição detalhada de cada tarefa
│   └── reviews/
│       └── [num]_task_review.md    # Artefato de review de cada tarefa
└── report/
    ├── qa-report.md                # Relatório de QA da feature
    ├── bugs.md                     # Bugs encontrados pelo QA
    └── evidence/                   # Evidências dos testes de QA
```

## Estrutura do `.claude/`

| Pasta | Conteúdo |
|---|---|
| `commands/` | Os 4 slash commands que dirigem o pipeline |
| `agents/` | Subagents especializados: `task-reviewer`, `qa-validator`, `executor-bug` |
| `skills/` | Boas práticas consultadas pelos commands e agents (Node.js, Hyper-Express, PostgreSQL, NATS, testes, commits, QA, review, bugfix, brainstorming, PRD) |
| `templates/` | Estrutura obrigatória dos artefatos (PRD, Tech Spec, tasks, task individual) |
| `rules/` | `architecture.md` — Clean Architecture, monorepo, convenções de nomenclatura e o guia passo a passo de nova feature |

## As 4 Fases

### 1. `/create-prd` — Requisitos de Produto

Conduz um **discovery obrigatório** antes de escrever qualquer coisa: brainstorm interativo (uma pergunta por vez, 2+ rodadas), pesquisa de mercado via web search (mínimo 3 buscas), 2-3 abordagens com trade-offs. Só gera o PRD após o usuário confirmar a direção do produto.

- **Foco:** o QUÊ e o POR QUÊ — nunca soluções de design.
- **Saída:** `prd.md` seguindo o template (Visão Geral, Objetivos, Histórias de Usuário, Funcionalidades, UX, Restrições, Fora de Escopo), com requisitos numerados e critérios mensuráveis. Máximo ~2.000 palavras.

### 2. `/create-techspec` — Especificação Técnica

Brainstorm técnico sobre o PRD: lê o codebase, propõe 2-5 abordagens arquiteturais validadas contra as skills de domínio (Node.js, Hyper-Express, Postgres, NATS) e as rules do projeto, avalia reusar vs construir, e define a estratégia de testes desde o início. Só gera a spec após o usuário confirmar a direção técnica.

- **Foco:** o COMO — não repete requisitos do PRD.
- **Saída:** `techspec.md` seguindo o template (Resumo Executivo, Arquitetura, Design de Implementação, Integrações, Testes, Sequenciamento, Monitoramento, Considerações Técnicas). Máximo ~2.000 palavras.

### 3. `/create-tasks` — Decomposição em Tarefas

Lê PRD + Tech Spec + codebase, mapeia os artefatos por camada (domain → application → infrastructure → migrations → composição awilix) e propõe 2-3 abordagens de decomposição. **Mostra a lista high-level e aguarda aprovação explícita** antes de gerar qualquer arquivo.

- Cada tarefa é um **entregável funcional e incremental** com testes próprios (nunca uma tarefa separada só de testes).
- Sequenciamento respeita a Clean Architecture; tarefas paralelas e dependências são marcadas.
- **Saída:** `tasks.md` (lista + árvore de dependências) e `tasks/[num]_task.md` para cada tarefa (formato X.0 / subtarefas X.Y, máximo 10 tarefas).

### 4. `/run-task` — Execução

Implementa a próxima tarefa disponível (ou a informada via argumento) até a conclusão:

1. **Branch (só na task 1):** cria `feature/[num-nome]` a partir de `develop` (Gitflow).
2. **Preparação:** lê tasks.md → task → techspec → prd → rules, e produz um resumo + plano antes de codar.
3. **Implementação:** segue `rules/architecture.md` e consulta as skills conforme o contexto (Hyper-Express para controllers, Postgres para repositórios, NATS para mensageria, etc.). Testes escritos junto com o código.
4. **Validação (gate):** `pnpm typecheck`, `pnpm lint`, `pnpm test` (vitest run) e `pnpm build` — 100% de sucesso é pré-condição.
5. **Commit:** Conventional Commits (`type(scope): description`) via skill `conventional-commit`.
6. **Review (loop):** executa o agent `@task-reviewer` e repete correção → revalidação → re-review até status **APPROVED**.
7. **QA (só quando TODAS as tasks terminarem):** executa `@qa-validator`; se **REPROVADO**, o `@executor-bug` corrige os bugs de `report/bugs.md` e o ciclo QA → bugfix → QA repete até **APROVADO**.
8. **Merge e release:** merge `--no-ff` em `develop`, tag SemVer (feat → MINOR, fix → PATCH, breaking → MAJOR) e push.

## Os 3 Agents

| Agent | Quando roda | O que faz | Status de saída |
|---|---|---|---|
| `task-reviewer` | Após cada tarefa commitada | Revisa o diff contra os idiomas TypeScript/Node.js, Clean Architecture e a stack; roda as verificações automatizadas; grava `tasks/reviews/[num]_task_review.md` | `APPROVED` \| `APPROVED WITH OBSERVATIONS` \| `CHANGES REQUESTED` |
| `qa-validator` | Quando todas as tasks estão APPROVED | Pipeline de QA completo: análise estática, testes unitários/integração, mutation testing (Stryker), E2E de API, contrato OpenAPI, auditoria de código e infra. **Tolerância zero:** qualquer bug = REPROVADO | `APROVADO` \| `REPROVADO` |
| `executor-bug` | Quando o QA reprova | Corrige todos os bugs de `report/bugs.md` na **causa raiz**, cria teste de regressão por bug, atualiza o bugs.md e commita | Ciclo repete até QA aprovar |

Cada agent carrega sua skill de processo obrigatória (`task-reviewer-best-practices`, `qa-best-practices`, `bugfix-best-practices`), que é a fonte de verdade do seu pipeline.

## Skills

**De processo:** `brainstorming` (discovery das fases 1-3), `prd`, `task-reviewer-best-practices`, `qa-best-practices`, `bugfix-best-practices`, `conventional-commit`, `changelog-automation`.

**Técnicas (consultadas conforme o escopo da mudança):**

| Skill | Quando |
|---|---|
| `nodejs-best-practices` | Qualquer código TypeScript/Node.js |
| `node-testing-best-practices` | Testes (vitest, vi.fn/vi.mock, test.each, Stryker) |
| `testcontainers-node` | Testes de integração com Postgres/NATS reais |
| `hyper-express-best-practices` | Controllers, rotas, middlewares HTTP |
| `postgres-best-practices` | Schema, queries, migrations, transactions |
| `nats-best-practices` | Publishers, subscribers, JetStream |

## Stack e Arquitetura

Definidas em `.claude/rules/architecture.md` (referência canônica):

- **Node.js 22 LTS + TypeScript 5** (strict, ESM) em monorepo **pnpm workspaces** com N apps independentes (`apps/<app>/`) + biblioteca compartilhada (`pkg/`)
- **Clean Architecture** por app: `src/domain/` (entidades + interfaces: `entity/`, `usecase/<contexto>/`, `repository/`, `event/`, `<lib>/`) → `src/application/usecase/<contexto>/` (use cases) → `src/infrastructure/` (config, controllers, repos, publishers, subscribers, adapters) → `src/bin/` (composition root awilix: `api.ts` e `consumer.ts`)
- **Hyper-Express** (HTTP), **PostgreSQL 18** via `pg` (SQL manual, sem ORM; migrations com **dbmate**), **NATS JetStream** via `nats` (mensageria entre apps), **awilix** (DI), **pino** (logging estruturado), **zod** (validação), **swagger-jsdoc** (OpenAPI)
- **Testes:** vitest + testcontainers (Postgres/NATS reais) + Stryker (mutation testing); **qualidade:** ESLint + Prettier
- 1 use case = 1 interface = 1 arquivo, com método único `perform(input)`; arquivos em kebab-case
- Database-per-service; comunicação entre apps somente via eventos NATS

## Regras Inegociáveis do Processo

1. Nenhum artefato é gerado sem discovery + aprovação do usuário (fases 1-3).
2. Nenhum arquivo foge dos templates de `.claude/templates/`.
3. Tarefa só está completa com testes 100% verdes (`pnpm test`) **e** review APPROVED.
4. Todo commit segue Conventional Commits; toda tarefa finalizada é commitada.
5. Merge em `develop` só com: todas as tasks completas + todos os reviews APPROVED + QA APROVADO.
6. Bugs são corrigidos na causa raiz, cada um com teste de regressão.

## Como Começar

```
# 1. Descreva a ideia e conduza o discovery de produto
/create-prd

# 2. Defina a abordagem técnica sobre o PRD gerado
/create-techspec

# 3. Decomponha em tarefas incrementais
/create-tasks

# 4. Execute tarefa a tarefa até o merge
/run-task [num-nome-funcionalidade]
```
