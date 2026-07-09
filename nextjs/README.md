# Harness de Engenharia — Next.js

Este diretório contém um **harness de desenvolvimento orientado a specs** para aplicações frontend Next.js usando Claude Code. Ele define um pipeline completo — da ideia ao merge — em que cada fase produz um artefato versionado, e a implementação só avança com aprovação explícita (do usuário nas fases de planejamento, dos agents de review/QA na fase de execução).

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
    └── screenshots/                # Evidências visuais dos testes de QA
```

## Estrutura do `.claude/`

| Pasta | Conteúdo |
|---|---|
| `commands/` | Os 4 slash commands que dirigem o pipeline |
| `agents/` | Subagents especializados: `task-reviewer`, `qa-validator`, `executor-bug` |
| `skills/` | Boas práticas consultadas pelos commands e agents (Next.js, arquitetura, shadcn, design, responsividade, testes, commits, QA, review, bugfix, brainstorming, PRD) |
| `templates/` | Estrutura obrigatória dos artefatos (PRD, Tech Spec, tasks, task individual) |

A referência canônica de arquitetura é a skill `nextjs-clean-architecture` (camadas por domínio, regras de dependência, nomenclatura).

## Regra de Integração Frontend-Backend (SSR Obrigatório)

Regra transversal a todas as fases: **toda comunicação com o backend passa pela camada SSR do Next.js**.

- **Server Components (RSC)**: `fetch` direto às APIs do backend na renderização — leitura de dados
- **Server Actions (`'use server'`)**: mutações (formulários, ações do usuário) — escrita/atualização
- **Route Handlers (`app/api/`)**: endpoints auxiliares (webhooks, integrações externas, streaming)

Client Components **nunca** chamam o backend diretamente (sem `fetch`/`useEffect` para APIs, sem SDKs de backend no client). Violações são classificadas como **Crítico** no review e como **bug** no QA.

## As 4 Fases

### 1. `/create-prd` — Requisitos de Produto

Conduz um **discovery obrigatório** antes de escrever qualquer coisa: brainstorm interativo (uma pergunta por vez, 2+ rodadas), pesquisa de mercado via web search (mínimo 3 buscas), 2-3 abordagens com trade-offs. Só gera o PRD após o usuário confirmar a direção do produto.

- **Foco:** o QUÊ e o POR QUÊ — nunca soluções de design.
- **Saída:** `prd.md` seguindo o template (Visão Geral, Objetivos, Histórias de Usuário, Funcionalidades, UX, Restrições, Fora de Escopo), com requisitos numerados e critérios mensuráveis. Máximo ~2.000 palavras.

### 2. `/create-techspec` — Especificação Técnica

Brainstorm técnico sobre o PRD: lê o codebase (incluindo os componentes shadcn instalados via `npx shadcn@latest info`), propõe 2-5 abordagens de frontend validadas contra as skills do projeto — estratégia de renderização (RSC vs Client, SSR/SSG/ISR), componentes a reusar vs criar, integração via SSR, responsividade —, avalia bibliotecas existentes vs desenvolvimento customizado e define a estratégia de testes desde o início. Só gera a spec após o usuário confirmar a direção técnica.

- **Foco:** o COMO — não repete requisitos do PRD.
- **Saída:** `techspec.md` seguindo o template (Resumo Executivo, Arquitetura Frontend, Design de Implementação, Pontos de Integração, Testes, Sequenciamento, Performance e UX, Considerações Técnicas). Máximo ~2.000 palavras.

### 3. `/create-tasks` — Decomposição em Tarefas

Lê PRD + Tech Spec + codebase, mapeia os artefatos por camada (contratos/tipos → camada SSR → componentes UI → composição de páginas → E2E) e propõe 2-3 abordagens de decomposição. **Mostra a lista high-level e aguarda aprovação explícita** antes de gerar qualquer arquivo.

- Cada tarefa é um **entregável funcional e incremental** com testes próprios — preferindo fatias verticais navegáveis (página + camada SSR + componentes) a camadas horizontais soltas.
- Testes unitários e de integração acompanham cada tarefa; apenas o E2E do fluxo completo pode ser a tarefa final.
- **Saída:** `tasks.md` (lista + árvore de dependências) e `tasks/[num]_task.md` para cada tarefa (formato X.0 / subtarefas X.Y, máximo 10 tarefas).

### 4. `/run-task` — Execução

Implementa a próxima tarefa disponível (ou a informada via argumento) até a conclusão:

1. **Branch (só na task 1):** cria `feature/[num-nome]` a partir de `develop` (Gitflow).
2. **Preparação:** lê tasks.md → task → techspec → prd → skill `nextjs-clean-architecture`, e produz um resumo + plano antes de codar.
3. **Implementação:** Server Components por padrão, componentes shadcn antes de custom, estados de loading/erro/vazio obrigatórios, e consulta às skills conforme o contexto (shadcn para UI, responsive-design para layouts, next-best-practices para arquitetura). Testes escritos junto com o código.
4. **Validação (gate):** `npm run lint`, `npx tsc --noEmit`, `npm run test`, `npm run build` — 100% de sucesso é pré-condição; entregas visuais são verificadas no browser via skill `webapp-testing`.
5. **Commit:** Conventional Commits (`type(scope): description`) via skill `conventional-commit`.
6. **Review (loop):** executa o agent `@task-reviewer` e repete correção → revalidação → re-review até status **APPROVED**.
7. **QA (só quando TODAS as tasks terminarem):** executa `@qa-validator`; se **REPROVADO**, o `@executor-bug` corrige os bugs de `report/bugs.md` e o ciclo QA → bugfix → QA repete até **APROVADO**.
8. **Merge e release:** merge `--no-ff` em `develop`, tag SemVer (feat → MINOR, fix → PATCH, breaking → MAJOR) e push.

## Os 3 Agents

| Agent | Quando roda | O que faz | Status de saída |
|---|---|---|---|
| `task-reviewer` | Após cada tarefa commitada | Revisa o diff contra Next.js/SSR, arquitetura, React, performance, UI/UX, shadcn, responsividade, acessibilidade e testes; roda as verificações automatizadas; grava `tasks/reviews/[num]_task_review.md` | `APPROVED` \| `APPROVED WITH OBSERVATIONS` \| `CHANGES REQUESTED` |
| `qa-validator` | Quando todas as tasks estão APPROVED | Pipeline de QA completo via browser MCP: análise estática, testes, E2E, acessibilidade (WCAG 2.2), verificação visual de estados, auditoria de design e responsividade em 5 breakpoints — tudo com screenshot de evidência. **Tolerância zero:** qualquer bug = REPROVADO | `APROVADO` \| `REPROVADO` |
| `executor-bug` | Quando o QA reprova | Corrige todos os bugs de `report/bugs.md` na **causa raiz**, cria teste de regressão por bug, valida visualmente no browser, atualiza o bugs.md e commita | Ciclo repete até QA aprovar |

Cada agent carrega sua skill de processo obrigatória (`task-reviewer-best-practices`, `qa-best-practices`, `bugfix-best-practices`), que é a fonte de verdade do seu pipeline.

## Skills

**De processo:** `brainstorming` (discovery das fases 1-3), `prd`, `task-reviewer-best-practices`, `qa-best-practices`, `bugfix-best-practices`, `conventional-commit`, `changelog-automation`.

**Técnicas (consultadas conforme o escopo da mudança):**

| Skill | Quando |
|---|---|
| `next-best-practices` | Qualquer código Next.js (file conventions, RSC boundaries, data patterns, metadata, error handling) |
| `nextjs-clean-architecture` | Estrutura de camadas, regras de dependência, nomenclatura, DTOs/mappers, stores |
| `shadcn` | Adicionar, compor ou estilizar componentes UI |
| `frontend-design` | Interfaces com alta qualidade visual |
| `responsive-design` | Layouts responsivos (container queries, fluid typography, mobile-first) |
| `web-design-guidelines` | Revisão de UI contra as Web Interface Guidelines (a11y, UX) |
| `webapp-testing` | Verificação no browser (screenshots, logs, debug de UI) |
| `playwright-generate-test` | Geração de testes E2E Playwright |

## Stack e Arquitetura

- **Next.js (App Router)** com **React Server Components** e **TypeScript** (strict, sem `any`)
- **Tailwind CSS** + **shadcn/ui** como design system (componentes existentes antes de custom)
- **Integração com backend 100% via SSR** — Server Components (leitura), Server Actions (mutações), Route Handlers (webhooks/streaming)
- **Clean Architecture por domínio** (skill `nextjs-clean-architecture`): domain → application → infra (camada SSR, DTOs, mappers) → presentation (componentes, páginas, stores)
- **Mobile-first e responsivo por padrão**; acessibilidade WCAG 2.2 validada no QA
- **Testes**: Testing Library (unitários/integração com mocks dos contratos de API) + Playwright (E2E contra o app em SSR)

## Regras Inegociáveis do Processo

1. Nenhum artefato é gerado sem discovery + aprovação do usuário (fases 1-3).
2. Nenhum arquivo foge dos templates de `.claude/templates/`.
3. Integração frontend-backend sempre via SSR — o client nunca chama o backend diretamente.
4. Tarefa só está completa com lint, typecheck, testes e build 100% verdes **e** review APPROVED.
5. Todo commit segue Conventional Commits; toda tarefa finalizada é commitada.
6. Merge em `develop` só com: todas as tasks completas + todos os reviews APPROVED + QA APROVADO.
7. Bugs são corrigidos na causa raiz, cada um com teste de regressão.

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
