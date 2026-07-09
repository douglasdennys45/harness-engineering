---
description: Executa a próxima tarefa disponível da feature - implementação frontend Next.js completa com testes, review, commit e QA
argument-hint: [num-nome-funcionalidade] [numero-da-task (opcional)]
---

Você é um assistente IA responsável por implementar tarefas de forma correta e completa em uma aplicação frontend Next.js. Identifique a próxima tarefa disponível, prepare o contexto e **IMPLEMENTE** até a conclusão.

## Regras Inegociáveis

<critical>
1. A tarefa só está completa quando **todos os testes passam com 100% de sucesso** e a validação técnica (lint, typecheck, build) está limpa.
2. A tarefa só está completa após o @task-reviewer retornar **APPROVED**. Se retornar APPROVED WITH OBSERVATIONS ou CHANGES REQUESTED: corrija, rode o @task-reviewer novamente e repita até APPROVED (o arquivo de review deve ser atualizado a cada re-review).
3. Toda tarefa finalizada exige **commit** no padrão Conventional Commits via skill `conventional-commit`.
4. Após completar a tarefa, **marque como completa em tasks.md**.
5. TODA integração com o backend via SSR (Server Components, Server Actions, Route Handlers). O client NUNCA chama o backend diretamente. Sem exceções.
6. Não se apresse: leia os arquivos necessários, verifique os testes e faça reasoning explícito antes de implementar (you are not lazy).
7. NÃO PULE NENHUM PASSO. Implemente soluções adequadas, **sem gambiarras**.
</critical>

## Localização dos Arquivos

| Artefato | Caminho |
|---|---|
| PRD | `.cognup/specs/[num-nome-funcionalidade]/prd.md` |
| Tech Spec | `.cognup/specs/[num-nome-funcionalidade]/techspec.md` |
| Lista de tarefas | `.cognup/specs/[num-nome-funcionalidade]/tasks.md` |
| Tarefa individual | `.cognup/specs/[num-nome-funcionalidade]/tasks/[num]_task.md` |
| Review da tarefa | `.cognup/specs/[num-nome-funcionalidade]/tasks/reviews/[num]_task_review.md` |
| Relatório de QA | `.cognup/specs/[num-nome-funcionalidade]/report/qa-report.md` |
| Bugs do QA | `.cognup/specs/[num-nome-funcionalidade]/report/bugs.md` |
| Regras do projeto | @.claude/rules (se existirem) + skill `nextjs-clean-architecture` |

Argumentos recebidos: `$ARGUMENTS` (feature e, opcionalmente, o número da task). Se nenhum argumento for informado, identifique a feature ativa e a próxima task não concluída em `tasks.md`.

## Fluxo de Execução

### 1. Branch (Gitflow) — apenas na tarefa 1

<critical>Antes de iniciar a **tarefa número 1** de uma feature, crie a branch seguindo Gitflow. A branch é criada uma única vez; as tarefas seguintes continuam nela.</critical>

1. `git status` — garanta que não há mudanças não commitadas pendentes de outra feature.
2. `git branch --show-current` — verifique a branch atual.
3. Se necessário: `git checkout develop && git pull origin develop`.
4. Crie a branch:
   - **Feature**: `git checkout -b feature/[num-nome-funcionalidade]`
   - **Bugfix**: `git checkout -b bugfix/[num-nome-funcionalidade]`
   - **Hotfix**: `git checkout -b hotfix/[num-nome-funcionalidade]`

Se a branch da feature já existir, apenas faça checkout nela.

### 2. Preparação e Análise

Leia **nesta ordem** e extraia apenas o que é relevante para a task atual:

1. `tasks.md` — identifique a próxima task não concluída e suas dependências.
2. `tasks/[num]_task.md` — definição detalhada da tarefa.
3. `techspec.md` — requisitos técnicos que afetam esta task (renderização, contratos de API, componentes).
4. `prd.md` — contexto de produto relevante.
5. Skill `nextjs-clean-architecture` — camadas, regras de dependência e nomenclatura do projeto.
6. Código existente das tasks anteriores da mesma feature (para reaproveitar padrões já estabelecidos) e componentes shadcn instalados (`npx shadcn@latest info`).

Produza o resumo antes de codar:

```
ID da Tarefa: [ID ou número]
Nome da Tarefa: [nome ou descrição breve]
Contexto PRD: [pontos principais]
Requisitos Tech Spec: [requisitos técnicos principais]
Dependências: [tasks anteriores das quais depende + estado atual delas]
Camadas Afetadas: [domain / application / infra (SSR) / presentation / rotas]
Objetivos Principais: [objetivos primários]
Riscos/Desafios: [riscos identificados]

Plano:
1. [primeiro passo]
2. [segundo passo]
3. [...]
```

### 3. Implementação com Skills

<critical>Consulte as skills relevantes ANTES e DURANTE a implementação. Não implemente sem verificar as boas práticas aplicáveis ao contexto.</critical>

| Skill | Quando utilizar |
|-------|----------------|
| `next-best-practices` | Qualquer código Next.js: file conventions, RSC boundaries, data patterns, async APIs, metadata, error handling, route handlers, image/font optimization |
| `nextjs-clean-architecture` | Estrutura de camadas, regras de dependência, nomenclatura, use cases, DTOs/mappers, stores |
| `shadcn` | Adicionar, compor ou estilizar componentes UI com shadcn/ui |
| `frontend-design` | Criar interfaces com alta qualidade visual — componentes, páginas e layouts |
| `responsive-design` | Layouts responsivos — container queries, fluid typography, CSS Grid, mobile-first |
| `web-design-guidelines` | Revisar a UI implementada quanto a acessibilidade, UX e boas práticas |
| `playwright-generate-test` | Gerar testes E2E Playwright baseados nos cenários da task |
| `webapp-testing` | Verificar a funcionalidade no browser — screenshots, logs, debug de UI |
| `conventional-commit` | Mensagem de commit ao finalizar cada tarefa |

Regras durante a implementação:

- **Server Components por padrão**; Client Components apenas quando há interatividade. Leitura via fetch em Server Components, mutações via Server Actions — nenhum `fetch`/`useEffect` de Client Component para o backend.
- Reaproveite componentes shadcn instalados e registries antes de criar componentes custom.
- Implemente estados de loading, erro e vazio (`loading.tsx`, `error.tsx`, empty states) — não apenas o happy path.
- Use o Context7 MCP **apenas quando** as skills e regras locais não cobrirem a dúvida (API de biblioteca externa, comportamento de versão específica).
- Escreva os testes exigidos pela task (unitário e/ou integração) junto com a implementação, não depois.

### 4. Validação Técnica (gate obrigatório)

Execute na raiz do projeto (adapte ao package manager do projeto) e **não prossiga com falhas**:

```bash
npm run lint
npx tsc --noEmit
npm run test
npm run build
```

Se a task tiver testes E2E, execute também os specs Playwright da task. Para tasks com entrega visual, **verifique o resultado no browser** com a skill `webapp-testing` (fluxo funcionando, responsividade, console sem erros) antes de considerar a validação completa.

Falhou? Corrija a causa raiz e rode novamente. Testes com 100% de sucesso são pré-condição para o commit.

### 5. Commit da Tarefa (obrigatório a cada tarefa)

<critical>NÃO finalize a tarefa sem realizar o commit. O commit faz parte do ciclo de conclusão.</critical>

1. `git status` e `git diff` — revise as mudanças.
2. `git add <files>` — stage apenas arquivos da task.
3. Gere a mensagem com a skill `conventional-commit`:
   - Formato: `type(scope): description`
   - **type**: `feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert`
   - **scope**: contexto da mudança (ex: `auth`, `checkout`, `ui`)
   - Exemplos: `feat(auth): add login form with server action`, `test(checkout): add e2e tests for payment flow`
4. `git commit -m "type(scope): description"`

### 6. Review (loop até APPROVED)

1. Execute o agente @task-reviewer.
2. Verifique o **Status** no arquivo `tasks/reviews/[num]_task_review.md` gerado:
   - **APPROVED** → marque a task como completa em `tasks.md` e siga adiante.
   - **APPROVED WITH OBSERVATIONS** ou **CHANGES REQUESTED** → aplique as correções indicadas.
3. Correções aplicadas: revalide (passo 4), commite as correções (`fix`/`refactor` conforme o caso) e execute o @task-reviewer novamente. O agente atualiza o mesmo arquivo de review.
4. Repita até o status ser **APPROVED**. Não finalize a tarefa com status diferente de APPROVED.

### 7. QA (apenas quando TODAS as tasks da feature estiverem concluídas)

1. Com todas as tasks completas em `tasks.md` e todos os reviews APPROVED, execute o agente @qa-validator.
2. Verifique o status em `report/qa-report.md`:
   - **APROVADO** → siga para o passo 8.
   - **REPROVADO** com bugs em `report/bugs.md` → execute o agente @executor-bug para corrigir todos os bugs documentados.
3. Após o @executor-bug finalizar, execute novamente o @qa-validator.
4. Repita o ciclo (qa-validator → executor-bug → qa-validator) até o status ser **APROVADO**.

### 8. Merge, Tag e Push (apenas com feature 100% concluída)

<critical>Só execute com: todas as tasks completas em tasks.md + todos os reviews APPROVED + QA APROVADO. Sempre use `--no-ff` no merge.</critical>

1. Confirme as três condições acima nos arquivos correspondentes.
2. `git checkout develop && git pull origin develop`
3. `git merge feature/[num-nome-funcionalidade] --no-ff -m "merge: feature/[num-nome-funcionalidade] into develop"`
4. Gere a tag seguindo **Semantic Versioning**:
   - Última tag: `git tag --sort=-v:refname | head -1` (sem tag existente, inicie em `v0.1.0`)
   - **feat** → MINOR (`v1.0.0` → `v1.1.0`) | **fix** → PATCH (`v1.0.0` → `v1.0.1`) | **breaking change** → MAJOR (`v1.0.0` → `v2.0.0`)
   - `git tag -a v[X.Y.Z] -m "release: v[X.Y.Z] - [breve descrição da feature]"`
5. `git push origin develop && git push origin v[X.Y.Z]`

## Definition of Done (checklist por tarefa)

- [ ] Task, PRD, techspec e regras de arquitetura lidos e resumo produzido
- [ ] Implementação segue as camadas da skill `nextjs-clean-architecture` e a regra SSR (nenhuma chamada direta do client ao backend)
- [ ] Skills consultadas conforme o contexto da mudança
- [ ] Lint, typecheck, testes e build passando 100% (+ verificação no browser para entregas visuais)
- [ ] Estados de loading, erro e vazio implementados
- [ ] Commit realizado no padrão Conventional Commits
- [ ] @task-reviewer executado com status final **APPROVED**
- [ ] Task marcada como completa em `tasks.md`
- [ ] (Última task) Ciclo @qa-validator → @executor-bug repetido até QA **APROVADO**
- [ ] (Última task) Merge `--no-ff` em develop, tag SemVer e push realizados

## Implementação

Após o resumo e o plano, **comece imediatamente a implementar a tarefa**: execute os comandos necessários, faça as alterações de código, siga os padrões do projeto e garanta que todos os requisitos sejam atendidos.
