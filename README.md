# Harness Engineering

Coleção de **harnesses de desenvolvimento orientado a specs** para [Claude Code](https://claude.com/claude-code). Cada harness define um pipeline completo de engenharia — da ideia ao merge — em que o trabalho avança por fases com artefatos versionados, aprovação explícita do usuário no planejamento e gates automatizados (review + QA) na execução.

A ideia central: em vez de pedir código diretamente ao agente, o processo força **discovery antes de decidir, spec antes de implementar e validação antes de mergear** — com skills de boas práticas guiando cada etapa e agents especializados fechando o ciclo de qualidade.

## Harnesses Disponíveis

| Harness | Foco | Stack |
|---|---|---|
| [`golang/`](golang/README.md) | Backend — microserviços | Go 1.26, Fiber v3, PostgreSQL 18, NATS JetStream, uber-go/fx, Clean Architecture em monorepo |
| [`nextjs/`](nextjs/README.md) | Frontend — aplicações web | Next.js (App Router), React Server Components, TypeScript, Tailwind, shadcn/ui, integração com backend 100% via SSR |

Ambos compartilham o mesmo desenho de processo; mudam apenas as skills técnicas, os critérios de review/QA e as convenções de arquitetura de cada stack.

## O Pipeline (comum a todos os harnesses)

```
 /create-prd  ─────►  /create-techspec  ─────►  /create-tasks  ─────►  /run-task (N vezes)
     │                       │                        │                       │
     ▼                       ▼                        ▼                       ▼
   prd.md               techspec.md            tasks.md + tasks/        código + testes
 (o quê / por quê)      (como)                 [num]_task.md            + review + QA
                                                                        + commit + merge
```

1. **`/create-prd`** — discovery de produto obrigatório (brainstorm interativo + pesquisa de mercado) antes de escrever os requisitos. Foco no *o quê* e *por quê*.
2. **`/create-techspec`** — brainstorm técnico sobre o PRD: 2-5 abordagens com trade-offs validadas contra as skills da stack, reusar vs construir, estratégia de testes desde o início. Foco no *como*.
3. **`/create-tasks`** — decomposição em até 10 tarefas incrementais, cada uma um entregável funcional com testes próprios. A lista high-level é aprovada pelo usuário antes de gerar qualquer arquivo.
4. **`/run-task`** — executa tarefa a tarefa: branch Gitflow, implementação guiada pelas skills, gate de validação técnica, commit em Conventional Commits, review em loop até **APPROVED**, QA com **tolerância zero a bugs** até **APROVADO**, e merge `--no-ff` + tag SemVer + push.

Todos os artefatos de uma feature vivem em `.cognup/specs/[num-nome-funcionalidade]/` — PRD, tech spec, tasks, reviews e relatórios de QA com evidências.

## Anatomia de um Harness

Cada diretório é um `.claude/` completo, pronto para ser copiado para um projeto real:

```
<stack>/.claude/
├── commands/     # Os 4 slash commands que dirigem o pipeline
├── agents/       # Subagents finos: task-reviewer, qa-validator, executor-bug
├── skills/       # Skills de processo (review, QA, bugfix, brainstorming, PRD, commits)
│                 # + skills técnicas da stack (consultadas conforme o escopo)
├── templates/    # Estrutura obrigatória de cada artefato (PRD, techspec, tasks)
└── rules/        # Arquitetura canônica (golang) — no nextjs esse papel é da
                  # skill nextjs-clean-architecture
```

**Padrão de design dos agents:** cada agent é um arquivo fino (~40-50 linhas) contendo apenas o contrato com o workflow (caminhos de saída fixos, valores exatos de status, regras inegociáveis) e delegando todo o processo para uma skill obrigatória (`task-reviewer-best-practices`, `qa-best-practices`, `bugfix-best-practices`) — que é a única fonte de verdade do pipeline daquele agent.

**Ciclo de qualidade:** `run-task` → commit → `task-reviewer` (loop até APPROVED) → ... → última task → `qa-validator` (tolerância zero) → se REPROVADO, `executor-bug` corrige na causa raiz com teste de regressão por bug → re-QA → merge.

## Como Usar em um Projeto

1. Copie o `.claude/` do harness da sua stack para a raiz do projeto:
   ```bash
   cp -r harness-engineering/nextjs/.claude  meu-projeto/
   # ou
   cp -r harness-engineering/golang/.claude  meu-projeto/
   ```
2. Abra o Claude Code no projeto e rode o pipeline:
   ```
   /create-prd        # descreva a ideia e conduza o discovery
   /create-techspec   # defina a abordagem técnica
   /create-tasks      # decomponha em tarefas
   /run-task          # execute até o merge
   ```

## Princípios do Processo

1. **Explore antes de decidir; decida antes de especificar** — nenhum artefato é gerado sem discovery e aprovação do usuário.
2. **Nenhum arquivo foge dos templates** — os artefatos são previsíveis e comparáveis entre features.
3. **Testes acompanham cada tarefa** — nunca são uma fase separada no final.
4. **Tarefa só termina verde** — validação técnica 100% + review APPROVED são pré-condição de conclusão.
5. **Tolerância zero no QA** — qualquer bug, de qualquer severidade, reprova a feature.
6. **Bugs morrem na causa raiz** — cada correção carrega um teste de regressão.
7. **Rastreabilidade total** — todo requisito vira cenário de teste, toda decisão fica registrada em artefato versionado.

## Adicionando um Novo Harness

Para criar um harness de outra stack (ex.: Python, Kotlin), siga o padrão existente:

1. Crie `<stack>/.claude/` com os 4 commands espelhando os existentes (a estrutura do processo não muda — adapte camadas, gates de validação e ferramentas de teste da stack)
2. Adicione as skills técnicas da stack e a referência de arquitetura (via `rules/` ou uma skill de arquitetura)
3. Crie as 3 skills de processo (`task-reviewer-best-practices`, `qa-best-practices`, `bugfix-best-practices`) com os critérios da stack
4. Crie os 3 agents finos apontando para elas, preservando o contrato (status `APPROVED`/`APROVADO`, caminhos `tasks/reviews/` e `report/`)
5. Documente no `README.md` do harness e adicione a linha na tabela acima
