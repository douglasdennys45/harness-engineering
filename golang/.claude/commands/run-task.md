---
description: Executa a proxima tarefa disponivel da feature - implementacao Go completa com testes, review, commit e QA
argument-hint: [num-nome-funcionalidade] [numero-da-task (opcional)]
---

Voce e um assistente IA responsavel por implementar tarefas de forma correta e completa em um servico Go. Identifique a proxima tarefa disponivel, prepare o contexto e **IMPLEMENTE** ate a conclusao.

## Regras Inegociaveis

<critical>
1. A tarefa so esta completa quando **todos os testes passam com 100% de sucesso** (`go test ./... -race -count=1`).
2. A tarefa so esta completa apos o @task-reviewer retornar **APPROVED**. Se retornar APPROVED WITH OBSERVATIONS ou CHANGES REQUESTED: corrija, rode o @task-reviewer novamente e repita ate APPROVED (o arquivo de review deve ser atualizado a cada re-review).
3. Toda tarefa finalizada exige **commit** no padrao Conventional Commits via skill `conventional-commit`.
4. Apos completar a tarefa, **marque como completa em tasks.md**.
5. Nao se apresse: leia os arquivos necessarios, verifique os testes e faca reasoning explicito antes de implementar (you are not lazy).
6. NAO PULE NENHUM PASSO. Implemente solucoes adequadas, **sem gambiarras**.
</critical>

## Localizacao dos Arquivos

| Artefato | Caminho |
|---|---|
| PRD | `.cognup/specs/[num-nome-funcionalidade]/prd.md` |
| Tech Spec | `.cognup/specs/[num-nome-funcionalidade]/techspec.md` |
| Lista de tarefas | `.cognup/specs/[num-nome-funcionalidade]/tasks.md` |
| Tarefa individual | `.cognup/specs/[num-nome-funcionalidade]/tasks/[num]_task.md` |
| Review da tarefa | `.cognup/specs/[num-nome-funcionalidade]/tasks/reviews/[num]_task_review.md` |
| Relatorio de QA | `.cognup/specs/[num-nome-funcionalidade]/report/qa-report.md` |
| Bugs do QA | `.cognup/specs/[num-nome-funcionalidade]/report/bugs.md` |
| Regras do projeto | @.claude/rules |

Argumentos recebidos: `$ARGUMENTS` (feature e, opcionalmente, o numero da task). Se nenhum argumento for informado, identifique a feature ativa e a proxima task nao concluida em `tasks.md`.

## Fluxo de Execucao

### 1. Branch (Gitflow) — apenas na tarefa 1

<critical>Antes de iniciar a **tarefa numero 1** de uma feature, crie a branch seguindo Gitflow. A branch e criada uma unica vez; as tarefas seguintes continuam nela.</critical>

1. `git status` — garanta que nao ha mudancas nao commitadas pendentes de outra feature.
2. `git branch --show-current` — verifique a branch atual.
3. Se necessario: `git checkout develop && git pull origin develop`.
4. Crie a branch:
   - **Feature**: `git checkout -b feature/[num-nome-funcionalidade]`
   - **Bugfix**: `git checkout -b bugfix/[num-nome-funcionalidade]`
   - **Hotfix**: `git checkout -b hotfix/[num-nome-funcionalidade]`

Se a branch da feature ja existir, apenas faca checkout nela.

### 2. Preparacao e Analise

Leia **nesta ordem** e extraia apenas o que e relevante para a task atual:

1. `tasks.md` — identifique a proxima task nao concluida e suas dependencias.
2. `tasks/[num]_task.md` — definicao detalhada da tarefa.
3. `techspec.md` — requisitos tecnicos que afetam esta task.
4. `prd.md` — contexto de produto relevante.
5. `@.claude/rules/architecture.md` — camadas, nomenclatura e o guia "Adicionando uma Nova Feature".
6. Codigo existente das tasks anteriores da mesma feature (para reaproveitar padroes ja estabelecidos).

Produza o resumo antes de codar:

```
ID da Tarefa: [ID ou numero]
Nome da Tarefa: [nome ou descricao breve]
Contexto PRD: [pontos principais]
Requisitos Tech Spec: [requisitos tecnicos principais]
Dependencias: [tasks anteriores das quais depende + estado atual delas]
Camadas Afetadas: [domain / application / infrastructure / pkg / migrations]
Objetivos Principais: [objetivos primarios]
Riscos/Desafios: [riscos identificados]

Plano:
1. [primeiro passo]
2. [segundo passo]
3. [...]
```

### 3. Implementacao com Skills

<critical>Consulte as skills relevantes ANTES e DURANTE a implementacao. Nao implemente sem verificar as boas praticas aplicaveis ao contexto.</critical>

| Skill | Quando utilizar |
|-------|----------------|
| `go-best-practices` | Qualquer codigo Go: error handling, concurrency, context, interfaces, nomenclatura |
| `go-testing-best-practices` | Testes: table-driven, testify, gomock, benchmarks, race detection, cobertura |
| `fiber-v3-best-practices` | Controllers, rotas, middlewares, bind/validacao HTTP (camada infrastructure/controller) |
| `postgres-best-practices` | Repositorios, queries SQL, migrations, transactions (camada infrastructure/repository) |
| `nats-best-practices` | Publishers, subscribers, streams, eventos JetStream |
| `testcontainers-go` | Testes de integracao com PostgreSQL/NATS reais |
| `conventional-commit` | Mensagem de commit ao finalizar cada tarefa |

Regras durante a implementacao:

- Siga rigorosamente `@.claude/rules/architecture.md`: Clean Architecture, regra de dependencia entre camadas, nomenclatura kebab-case, construtores retornando interface de dominio, composicao FX no `main.go`.
- Reaproveite o que ja existe em `pkg/` antes de criar codigo novo.
- Use o Context7 MCP **apenas quando** as skills e regras locais nao cobrirem a duvida (API de biblioteca externa, comportamento de versao especifica).
- Escreva os testes exigidos pela task (unitario e/ou integracao) junto com a implementacao, nao depois.

### 4. Validacao Tecnica (gate obrigatorio)

Execute na raiz do workspace e **nao prossiga com falhas**:

```bash
go build ./...
go vet ./...
go test ./... -race -count=1
```

Se existir `golangci-lint` configurado no projeto, execute tambem `golangci-lint run`.

Falhou? Corrija a causa raiz e rode novamente. Testes com 100% de sucesso sao pre-condicao para o commit.

### 5. Commit da Tarefa (obrigatorio a cada tarefa)

<critical>NAO finalize a tarefa sem realizar o commit. O commit faz parte do ciclo de conclusao.</critical>

1. `git status` e `git diff` — revise as mudancas.
2. `git add <files>` — stage apenas arquivos da task.
3. Gere a mensagem com a skill `conventional-commit`:
   - Formato: `type(scope): description`
   - **type**: `feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert`
   - **scope**: contexto de dominio da mudanca (ex: `order`, `user`, `health`, `config`)
   - Exemplos: `feat(order): add create order use case and repository`, `test(order): add table-driven tests for order validation`
4. `git commit -m "type(scope): description"`

### 6. Review (loop ate APPROVED)

1. Execute o agente @task-reviewer.
2. Verifique o **Status** no arquivo `tasks/reviews/[num]_task_review.md` gerado:
   - **APPROVED** → marque a task como completa em `tasks.md` e siga adiante.
   - **APPROVED WITH OBSERVATIONS** ou **CHANGES REQUESTED** → aplique as correcoes indicadas.
3. Correcoes aplicadas: revalide (passo 4), commite as correcoes (`fix`/`refactor` conforme o caso) e execute o @task-reviewer novamente. O agente atualiza o mesmo arquivo de review.
4. Repita ate o status ser **APPROVED**. Nao finalize a tarefa com status diferente de APPROVED.

### 7. QA (apenas quando TODAS as tasks da feature estiverem concluidas)

1. Com todas as tasks completas em `tasks.md` e todos os reviews APPROVED, execute o agente @qa-validator.
2. Verifique o status em `report/qa-report.md`:
   - **APROVADO** → siga para o passo 8.
   - **REPROVADO** com bugs em `report/bugs.md` → execute o agente @executor-bug para corrigir todos os bugs documentados.
3. Apos o @executor-bug finalizar, execute novamente o @qa-validator.
4. Repita o ciclo (qa-validator → executor-bug → qa-validator) ate o status ser **APROVADO**.

### 8. Merge, Tag e Push (apenas com feature 100% concluida)

<critical>So execute com: todas as tasks completas em tasks.md + todos os reviews APPROVED + QA APROVADO. Sempre use `--no-ff` no merge.</critical>

1. Confirme as tres condicoes acima nos arquivos correspondentes.
2. `git checkout develop && git pull origin develop`
3. `git merge feature/[num-nome-funcionalidade] --no-ff -m "merge: feature/[num-nome-funcionalidade] into develop"`
4. Gere a tag seguindo **Semantic Versioning**:
   - Ultima tag: `git tag --sort=-v:refname | head -1` (sem tag existente, inicie em `v0.1.0`)
   - **feat** → MINOR (`v1.0.0` → `v1.1.0`) | **fix** → PATCH (`v1.0.0` → `v1.0.1`) | **breaking change** → MAJOR (`v1.0.0` → `v2.0.0`)
   - `git tag -a v[X.Y.Z] -m "release: v[X.Y.Z] - [breve descricao da feature]"`
5. `git push origin develop && git push origin v[X.Y.Z]`

## Definition of Done (checklist por tarefa)

- [ ] Task, PRD, techspec e regras de arquitetura lidos e resumo produzido
- [ ] Implementacao segue Clean Architecture e nomenclatura de `architecture.md`
- [ ] Skills consultadas conforme o contexto da mudanca
- [ ] `go build ./...`, `go vet ./...` e `go test ./... -race -count=1` passando 100%
- [ ] Commit realizado no padrao Conventional Commits
- [ ] @task-reviewer executado com status final **APPROVED**
- [ ] Task marcada como completa em `tasks.md`
- [ ] (Ultima task) Ciclo @qa-validator → @executor-bug repetido ate QA **APROVADO**
- [ ] (Ultima task) Merge `--no-ff` em develop, tag SemVer e push realizados

## Implementacao

Apos o resumo e o plano, **comece imediatamente a implementar a tarefa**: execute os comandos necessarios, faca as alteracoes de codigo, siga os padroes do projeto e garanta que todos os requisitos sejam atendidos.
