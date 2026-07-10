---
description: Executa a próxima tarefa disponível da feature - implementação frontend Angular com SSR completa com testes, review, commit e QA
argument-hint: [num-nome-funcionalidade] [numero-da-task (opcional)]
---

Você é um assistente IA responsável por implementar tarefas de forma correta e completa em uma aplicação frontend Angular com SSR. Identifique a próxima tarefa disponível, prepare o contexto e **IMPLEMENTE** até a conclusão.

## Regras Inegociáveis

<critical>
1. A tarefa só está completa quando **todos os testes passam com 100% de sucesso** e a validação técnica (typecheck, lint, build) está limpa.
2. A tarefa só está completa após o @task-reviewer retornar **APPROVED**. Se retornar APPROVED WITH OBSERVATIONS ou CHANGES REQUESTED: corrija, rode o @task-reviewer novamente e repita até APPROVED (o arquivo de review deve ser atualizado a cada re-review).
3. Toda tarefa finalizada exige **commit** no padrão Conventional Commits via skill `conventional-commit`.
4. Após completar a tarefa, **marque como completa em tasks.md**.
5. TODA integração com o backend via SSR/BFF (leituras via repositories no SSR com transfer cache, mutações via BFF do domínio). O browser NUNCA chama o backend interno diretamente. Sem exceções.
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
| Regras do projeto | @.claude/rules (se existirem) + skill `angular-clean-architecture` |

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
3. `techspec.md` — requisitos técnicos que afetam esta task (render modes, contratos de API/BFF, componentes, federação).
4. `prd.md` — contexto de produto relevante.
5. Skill `angular-clean-architecture` — camadas, regras de dependência, federação, regra BFF e nomenclatura do projeto.
6. Código existente das tasks anteriores da mesma feature (para reaproveitar padrões já estabelecidos) e inventário de componentes do `libs/ui`/Material já em uso.

Produza o resumo antes de codar:

```
ID da Tarefa: [ID ou número]
Nome da Tarefa: [nome ou descrição breve]
Contexto PRD: [pontos principais]
Requisitos Tech Spec: [requisitos técnicos principais]
Dependências: [tasks anteriores das quais depende + estado atual delas]
Projetos Nx Afetados: [apps/libs]
Camadas Afetadas: [domain / application / infra (BFF) / presentation / rotas-SSR / federação]
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
| `angular-best-practices` | Qualquer código Angular: signals, zoneless, control flow, @defer/incremental hydration, SSR-safety, render modes, HttpClient/httpResource, forms, roteamento, error handling |
| `angular-clean-architecture` | Estrutura de camadas, regras de dependência, use cases, DTOs/mappers, stores, Native Federation, regra BFF, boundaries Nx |
| `angular-material` | Adicionar, compor ou estilizar componentes UI com Angular Material/CDK e libs/ui |
| `frontend-design` | Criar interfaces com alta qualidade visual — componentes, páginas e layouts |
| `responsive-design` | Layouts responsivos — container queries, fluid typography, CSS Grid, mobile-first |
| `web-design-guidelines` | Revisar a UI implementada quanto a acessibilidade, UX e boas práticas |
| `playwright-generate-test` | Gerar testes E2E Playwright baseados nos cenários da task |
| `webapp-testing` | Verificar a funcionalidade no browser — screenshots, logs, debug de UI |
| `conventional-commit` | Mensagem de commit ao finalizar cada tarefa |

Regras durante a implementação:

- **SSR por padrão**; render mode declarado em `app.routes.server.ts` para toda rota nova. Leituras via repositories (HttpClient) executados no SSR, mutações via endpoints BFF no `server.ts` — nenhuma chamada do browser ao backend interno.
- **SSR-safety**: render determinístico (sem `Date.now()`/`Math.random()`/browser APIs no render); código browser-only atrás de `afterNextRender()`; auth via cookies.
- Componentes standalone com OnPush e signals (`input()`/`output()`/`computed`); control flow `@if`/`@for` com `track`.
- Reaproveite componentes do `libs/ui` e Material/CDK antes de criar componentes custom; estilize com tokens do tema.
- Implemente estados de loading, erro e vazio (skeleton, retry, empty state) — não apenas o happy path; `@defer` para conteúdo pesado.
- Respeite os boundaries Nx (camadas e domínios) — o lint acusa; nunca suprima com `eslint-disable`.
- Use o Context7 MCP **apenas quando** as skills e regras locais não cobrirem a dúvida (API de biblioteca externa, comportamento de versão específica).
- Escreva os testes exigidos pela task (unitário e/ou integração) junto com a implementação, não depois.

### 4. Validação Técnica (gate obrigatório)

Execute na raiz do monorepo e **não prossiga com falhas**:

```bash
npx nx affected -t typecheck
npx nx affected -t lint
npx nx affected -t test
npx nx affected -t build
```

Se a task tiver testes E2E, execute também os specs Playwright da task. Para tasks com entrega visual, **verifique o resultado no browser** com a skill `webapp-testing` (fluxo funcionando, responsividade, console sem erros — incluindo NG05xx de hydration) antes de considerar a validação completa.

Falhou? Corrija a causa raiz e rode novamente. Testes com 100% de sucesso são pré-condição para o commit.

### 5. Commit da Tarefa (obrigatório a cada tarefa)

<critical>NÃO finalize a tarefa sem realizar o commit. O commit faz parte do ciclo de conclusão.</critical>

1. `git status` e `git diff` — revise as mudanças.
2. `git add <files>` — stage apenas arquivos da task.
3. Gere a mensagem com a skill `conventional-commit`:
   - Formato: `type(scope): description`
   - **type**: `feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert`
   - **scope**: domínio/contexto da mudança (ex: `auth`, `billing`, `shell`, `ui`)
   - Exemplos: `feat(auth): add login form with bff endpoint`, `test(billing): add e2e tests for invoice flow`
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
2. Commite na branch da feature o bump de versão e o changelog gerados pelo QA, se ainda não commitados: `git add package.json CHANGELOG.md && git commit -m "chore(release): prepare v[X.Y.Z]"`
3. `git checkout develop && git pull origin develop`
4. `git merge feature/[num-nome-funcionalidade] --no-ff -m "merge: feature/[num-nome-funcionalidade] into develop"`
5. Gere a tag seguindo **Semantic Versioning**, usando como fonte a versão que o QA gravou no `package.json` da raiz (Etapa 12 da qa-best-practices):
   - Leia a versão: `node -p "require('./package.json').version"` → a tag é `v[essa-versão]`
   - Confira que a tag ainda não existe (`git tag --sort=-v:refname | head -1`) e que é maior que a última — `package.json` e tag git nunca podem divergir
   - Fallback (se o QA não fez o bump): **feat** → MINOR | **fix** → PATCH | **breaking change** → MAJOR sobre a última tag (sem tag existente, inicie em `v0.1.0`) e atualize o `package.json` para a mesma versão antes de taggear
   - `git tag -a v[X.Y.Z] -m "release: v[X.Y.Z] - [breve descrição da feature]"`
6. `git push origin develop && git push origin v[X.Y.Z]`

## Definition of Done (checklist por tarefa)

- [ ] Task, PRD, techspec e regras de arquitetura lidos e resumo produzido
- [ ] Implementação segue as camadas da skill `angular-clean-architecture` e a regra SSR/BFF (nenhuma chamada direta do browser ao backend interno)
- [ ] Rotas novas com render mode declarado; código SSR-safe (render determinístico, browser-only em afterNextRender)
- [ ] Skills consultadas conforme o contexto da mudança
- [ ] Typecheck, lint (com boundaries), testes e build passando 100% (+ verificação no browser para entregas visuais)
- [ ] Estados de loading, erro e vazio implementados
- [ ] Commit realizado no padrão Conventional Commits
- [ ] @task-reviewer executado com status final **APPROVED**
- [ ] Task marcada como completa em `tasks.md`
- [ ] (Última task) Ciclo @qa-validator → @executor-bug repetido até QA **APROVADO**
- [ ] (Última task) Merge `--no-ff` em develop, tag SemVer e push realizados

## Implementação

Após o resumo e o plano, **comece imediatamente a implementar a tarefa**: execute os comandos necessários, faça as alterações de código, siga os padrões do projeto e garanta que todos os requisitos sejam atendidos.
