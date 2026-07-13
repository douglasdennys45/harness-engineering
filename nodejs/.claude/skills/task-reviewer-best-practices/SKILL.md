---
name: task-reviewer-best-practices
description: Aplica melhores práticas de Task Review para revisão sistemática de tarefas concluídas em serviços Node.js (TypeScript). Cobre identificação da tarefa em `.cognup/specs/[feature]/tasks/[num]_task.md`, análise de diff (`git diff`/`git log`), validação contra boas práticas de TypeScript (strict types, sem any, ESM, imports organizados), Clean Architecture (`.claude/rules/architecture.md`), tratamento de erros (Error com `cause`, classes de erro de domínio, `instanceof`), assincronismo seguro (async/await, promises não aguardadas, AbortSignal), Hyper-Express (BaseController, bindAndHandle com zod, ErrorMapper, set_error_handler), PostgreSQL 18 (queries manuais com `pg`, migrations dbmate, Transactor/withTx), NATS JetStream (ack/nak/term, dedup, idempotência), awilix, pino, testes (vitest, vi.fn, testcontainers), execução de verificações automatizadas (`pnpm typecheck`, `pnpm lint`, `pnpm format:check`, `pnpm test`, `pnpm build`), classificação de problemas (Crítico/Major/Minor/Positivo) e geração ou re-escrita do artefato em `.cognup/specs/[feature]/tasks/reviews/[num]_task_review.md` em Português (BR) com status em inglês (APPROVED/APPROVED WITH OBSERVATIONS/CHANGES REQUESTED). Use sempre que uma tarefa Node.js foi concluída via workflow `run-task.md` e precisa ser revisada antes do merge, ao gerar artefato de revisão, ao reexecutar review após correções, ou ao avaliar aderência de uma implementação TypeScript aos padrões do projeto.
---

# Task Reviewer Best Practices — Revisão Sistemática de Tarefas Node.js

Skill que aplica o processo metódico de **Task Review** sobre tarefas concluídas usando o workflow `run-task.md` em serviços Node.js (TypeScript). Combina identificação da tarefa, análise de diff, validação contra boas práticas de TypeScript, verificações automatizadas, classificação de problemas e geração do artefato `[num]_task_review.md` rastreável.

Combina com [[nodejs-best-practices]] (padrões de código TypeScript moderno) e [[node-testing-best-practices]] (estratégias de teste), e — conforme o escopo do diff — com as skills de domínio [[hyper-express-best-practices]] (HTTP), [[postgres-best-practices]] (banco), [[nats-best-practices]] (mensageria) e [[testcontainers-node]] (testes de integração). A arquitetura de referência é a definida em `.claude/rules/architecture.md` (Clean Architecture em monorepo pnpm workspaces com Hyper-Express, PostgreSQL 18, NATS JetStream e awilix).

## Princípio Fundamental

> **Revisão completa, justa e rastreável.** O revisor identifica a tarefa, lê integralmente os arquivos modificados (não apenas os diffs), valida contra boas práticas de TypeScript/Node.js, executa verificações automatizadas e gera o artefato `[num]_task_review.md` com problemas classificados por severidade. Cada problema referencia arquivo, linha e correção sugerida; cada boa prática observada é reconhecida.

A revisão produz exatamente **um** dos três status (em inglês — é o que o workflow `run-task.md` e os agentes de correção verificam):

- **APPROVED** — pronto para merge
- **APPROVED WITH OBSERVATIONS** — segue mas requer nova revisão após melhorias
- **CHANGES REQUESTED** — requer correções obrigatórias e nova execução do reviewer

Apenas **APPROVED** finaliza a revisão. Os outros dois disparam o ciclo de correção → re-revisão até obter APPROVED.

## Localização dos Artefatos

| Artefato | Caminho |
|---|---|
| PRD | `.cognup/specs/[nome-da-funcionalidade]/prd.md` |
| Tech Spec | `.cognup/specs/[nome-da-funcionalidade]/techspec.md` |
| Lista de tarefas | `.cognup/specs/[nome-da-funcionalidade]/tasks.md` |
| Tarefa individual | `.cognup/specs/[nome-da-funcionalidade]/tasks/[num]_task.md` |
| **Review (SAÍDA)** | `.cognup/specs/[nome-da-funcionalidade]/tasks/reviews/[num]_task_review.md` |
| Regras do projeto | `.claude/rules/architecture.md` |

**REGRA CRÍTICA:** o artefato de revisão é gravado SEMPRE em `.cognup/specs/[nome-da-funcionalidade]/tasks/reviews/[num]_task_review.md` — nunca ao lado do arquivo da task, nunca na raiz da feature. Se a pasta não existir, crie antes: `mkdir -p .cognup/specs/[nome-da-funcionalidade]/tasks/reviews`.

## Quando Aplicar

- Quando uma tarefa Node.js/TypeScript foi finalizada via workflow `run-task.md` e o usuário pediu revisão
- Quando o usuário cita um número de tarefa específico (`task 5`, `task 12`) para revisar
- Proativamente após uma implementação significativa em TypeScript ser concluída
- Em re-revisão: quando `tasks/reviews/[num]_task_review.md` já existe com status diferente de APPROVED
- Antes de abrir Pull Request de uma tarefa concluída
- Ao validar aderência de código novo às convenções do projeto (Clean Architecture, awilix, Hyper-Express, PostgreSQL, NATS)

## Missão do Revisor

Você é um revisor de código de elite com domínio profundo em **Node.js, TypeScript, sistemas distribuídos, REST APIs, Clean Architecture e engenharia de software**. Seu olhar é meticuloso para TypeScript idiomático e estrito, qualidade, manutenibilidade e aderência aos padrões do projeto.

Em toda revisão, seu trabalho é:

1. Identificar qual tarefa foi concluída encontrando o `[num]_task.md` correspondente
2. Compreender o que foi solicitado na tarefa
3. Revisar **TODAS** as alterações de código relacionadas
4. Executar verificações automatizadas
5. Classificar problemas (Crítico/Major/Minor/Positivo)
6. Gerar/sobrescrever `tasks/reviews/[num]_task_review.md` com avaliação final em Português (BR)

## Pipeline de Revisão — 7 Etapas

### Etapa 1: Identificar a Tarefa e o Contexto

Localize o arquivo da tarefa concluída:

- Identifique a feature em `.cognup/specs/[nome-da-funcionalidade]/` (se não informada, use a feature com atividade mais recente)
- Localize a task em `tasks/[num]_task.md` (fallback: procure `[num]_task.md` dentro da pasta da feature)
- Se nenhum número for fornecido, encontre em `tasks.md` a última task marcada como concluída
- **Ler integralmente** a task, o `techspec.md` e o trecho relevante do `prd.md` antes de qualquer julgamento

**Detecção de Re-revisão:** Se `tasks/reviews/[num]_task_review.md` já existir com **Status** diferente de **APPROVED** (ex.: APPROVED WITH OBSERVATIONS, CHANGES REQUESTED):

- Trata-se de **re-revisão** após correções
- Execute a revisão completa no código **atual**
- **Sobrescreva** o arquivo com a nova avaliação
- Marque `**Re-revisão**: Sim (após correções)` no cabeçalho

```bash
# Localizar a task
ls .cognup/specs/*/tasks/*_task.md

# Verificar se já existe review (para detecção de re-revisão)
ls -la .cognup/specs/[feature]/tasks/reviews/*_task_review.md 2>/dev/null
```

### Etapa 2: Identificar Arquivos Alterados

Use git para identificar o escopo real da tarefa:

```bash
# Diff vs branch base (o projeto usa Gitflow: base é develop)
git diff develop...HEAD --stat

# Commits da branch atual
git log develop...HEAD --oneline

# Arquivos modificados não commitados (caso a tarefa ainda não tenha sido commitada)
git status --short
git diff HEAD --stat
```

**Regras críticas:**

- Listar **todos** os arquivos alterados (não apenas os "mais relevantes")
- Ler o **contexto completo** de cada arquivo modificado, não apenas o trecho do diff — bugs frequentemente vivem nas linhas adjacentes não modificadas
- Para arquivos novos, ler 100% do conteúdo
- Para arquivos grandes, focar nas funções/métodos alterados mas verificar imports, constantes, tipos relacionados

### Etapa 3: Preparação Técnica

Antes de iniciar a revisão detalhada, carregue o contexto técnico do projeto:

1. **Consultar `.claude/rules/architecture.md`** — camadas (domain/application/infrastructure), regra de dependência, nomenclatura kebab-case, convenções de construtores, BaseController, Transactor, composição do container awilix, guia "Adicionando uma Nova Feature"
2. **Consultar `CLAUDE.md`** (se existir) — convenções específicas e comandos de build/test
3. **Confirmar a stack** (padrão do projeto): **Node.js 22 LTS, TypeScript 5 strict ESM, Hyper-Express, PostgreSQL 18 (`pg`), NATS JetStream (`nats`), awilix, pino, zod, swagger-jsdoc, vitest/testcontainers, dbmate** — valide desvios pelos `package.json` do workspace
4. **Carregar [[nodejs-best-practices]]** como referência idiomática (sempre)
5. **Carregar as skills de domínio aplicáveis ao diff**: [[hyper-express-best-practices]] se tocou controllers/rotas/middlewares; [[postgres-best-practices]] se tocou repositórios/queries/migrations; [[nats-best-practices]] se tocou publishers/subscribers/streams; [[node-testing-best-practices]] e [[testcontainers-node]] se tocou testes
6. **Não carregar skills fora do escopo** — ex.: não carregue NATS para uma task que só cria entidade de domínio. Registre no artefato quais foram usadas e quais foram N/A

A revisão precisa ser fiel **ao projeto que você está revisando** — não imponha padrões que o projeto não adota. Mas valide rigorosamente os padrões que o projeto **declarou** adotar.

### Etapa 4: Conduzir a Revisão

Revise o código contra os critérios abaixo, agrupados por área. Para cada violação, registre **arquivo + linha + descrição + correção sugerida**.

#### 4.1 Padrões Gerais de Código

- **Idioma do código**: tudo em inglês (variáveis, funções, classes, comentários)
- **Nomenclatura clara**: identificadores exportados descritivos; nomes curtos apenas em escopos pequenos (`i`, `err` localmente)
- **Constantes**: sem números mágicos — usar `const` nomeadas ou objetos `as const`
- **Funções**: uma única ação clara, verbos para mutações (`createOrder`, `updateUser`)
- **Parâmetros**: > 3-4 parâmetros indica necessidade de objeto de opções tipado
- **Condicionais**: sem aninhamento profundo, **early returns** e **guard clauses**
- **Tamanho de método**: lógica complexa < 50 linhas; se ultrapassar, extrair helper
- **Tamanho de arquivo**: responsabilidade única; dividir quando exceder ~500 linhas
- **Comentários**: APIs exportadas **DEVEM** ter TSDoc (`/** ... */`); evitar comentários que repetem o código

#### 4.2 TypeScript Idiomático (strict + ESM)

> Referência: [[nodejs-best-practices]] para detalhes.

| Critério | Verificar |
|---|---|
| **Nomenclatura** | `camelCase` para valores/funções, `PascalCase` para tipos/classes/interfaces; **nunca** `snake_case` |
| **Acrônimos** | consistentes com o projeto (`httpClient`, `userId`, `parseUrl`) — sem mistura |
| **Arquivos** | kebab-case (`create-usecase.ts`); sem `util.ts`/`helpers.ts` genéricos |
| **Strict types** | sem `any` (explícito ou implícito); `unknown` + narrowing nas fronteiras |
| **Asserções** | sem `as` desnecessário, sem non-null assertion (`!`) sem justificativa |
| **`@ts-ignore`** | proibido; `@ts-expect-error` apenas com comentário explicando |
| **ESM** | `import`/`export` (nunca `require`); imports com extensão conforme padrão do projeto |
| **Imports organizados** | blocos: node builtins (`node:`) → third-party → projeto interno |
| **`const` sobre `let`** | sem `var`; imutabilidade por padrão (`readonly`, `ReadonlyArray`) |
| **Enums vs unions** | preferir union types/`as const` a `enum` (conforme convenção do projeto) |
| **Não-exportado por padrão** | exportar apenas API pública do módulo; sem default exports (named exports) |
| **Discriminated unions** | para estados/resultados com variantes, com narrowing exaustivo |
| **Composição sobre herança** | herança apenas para hierarquia de erros; preferir composição/interfaces |
| **Getters sem prefixo `get`** | campo `owner` → getter `owner`, não `getOwner()` (métodos de busca em repositórios podem usar `findBy*`) |

#### 4.3 Tratamento de Erros

| Regra | Como Verificar |
|---|---|
| Erros nunca engolidos | Sem `catch {}` vazio ou catch que só loga e segue como sucesso |
| Encadeamento com `cause` | `throw new Error("operation failed", { cause: err })` ao re-lançar com contexto |
| Classes de erro de domínio | `class UserNotFoundError extends DomainError` — hierarquia definida no domínio |
| `instanceof` para decisão | nunca decidir por `error.message` ou strings mágicas |
| Nunca lançar não-Error | sem `throw "string"`/`throw {}`; catch tipa `unknown` e faz narrowing |
| Mensagens de erro | lowercase, sem pontuação no final, descrevem o que falhou |
| Não logar e relançar | escolha **logar na boundary** OU **relançar**, nunca ambos |

#### 4.4 Assincronismo e Concorrência

| Regra | Como Verificar |
|---|---|
| Promises aguardadas | sem floating promises — regra `no-floating-promises` habilitada e respeitada |
| Fire-and-forget explícito | `void promise` apenas com tratamento de erro interno e justificativa |
| Paralelismo intencional | `Promise.all`/`allSettled` para operações independentes; await sequencial só quando há dependência |
| Propagação de cancelamento | `AbortSignal` propagado em operações canceláveis/timeouts (`AbortSignal.timeout`) |
| Sem async em constructor | inicialização assíncrona via factory/método `init`, nunca no constructor |
| Sem operação órfã | timers, loops de consumo e background jobs têm caminho de encerramento claro no shutdown |
| Sem estado compartilhado mutável | cuidado com módulos singleton mutados por requests concorrentes |

#### 4.5 Estrutura do Projeto (Clean Architecture — `.claude/rules/architecture.md`)

| Camada | Pode importar |
|---|---|
| `src/domain/` | apenas stdlib do Node (idealmente nada de infra) |
| `src/application/` | apenas `src/domain/` |
| `src/infrastructure/` | `src/domain/`, `packages/` compartilhados, libs externas |
| `packages/` (workspace) | **nunca** `src/` interno de nenhum app |
| `apps/<app>/src/main.ts` | tudo (composition root — container awilix daquele binário) |

Validar também:

- **Use cases**: 1 ação = 1 interface = 1 arquivo, método único `perform(input)`; implementação em `application/usecase/<contexto>/<ação>-usecase.ts`
- **Contratos**: use cases/repos/publishers/adapters tipados por **interfaces de domínio**; controllers/subscribers são classes concretas
- **Libs externas via inversão de dependência**: interface em `domain/<capacidade>/` (ex.: `crypt/`, `token/`), implementação em `infrastructure/adapter/<lib>-adapter.ts`
- **Eventos publicados DEPOIS do commit** no banco, nunca dentro da transaction
- Convenções de nomenclatura de arquivos: kebab-case (`order-repository.ts`)
- Sem imports circulares; direção limpa de dependências; sem estado global

#### 4.6 REST/HTTP (Hyper-Express, quando aplicável)

> Referência: [[hyper-express-best-practices]] e o padrão BaseController de `architecture.md`.

| Critério | Verificar |
|---|---|
| Framework | `hyper-express` (Server/Router nativos) |
| Handler signature | `(request: Request, response: Response)` async — sem callbacks aninhados |
| BaseController | controllers estendem/compõem o `BaseController`; rotas via `ctrl.handle(...)` ou `bindAndHandle(...)` |
| RequestContext | handlers usam o `RequestContext` já populado (requestId, parentId, IP, logger) — nunca extrair manualmente |
| Recursos | inglês, plural, kebab-case (`/orders`, `/payment-methods`); máximo 3 níveis |
| Status codes | constantes nomeadas (não números mágicos espalhados) |
| Binding | `await request.json()` centralizado no `bindAndHandle` — nunca parse manual no handler |
| Validação | schemas **zod** por endpoint (`schema.parse`/`safeParse` no `bindAndHandle`) — nunca validação manual no handler |
| Erros de domínio | `throw err` — mapeamento HTTP via `ErrorMapper`; fallback global via `server.set_error_handler`; novos erros registrados no `provideErrorMapper` do app |
| Timeouts/limites | body limit e timeouts configurados no servidor |
| Middlewares | requestId, CORS e logging configurados; error handler global registrado |
| Graceful shutdown | `server.close()` no handler de shutdown (SIGTERM/SIGINT) |
| Anotações OpenAPI | blocos `@openapi` (swagger-jsdoc) em todos os handlers públicos |
| Controllers | implementam `registerRoutes(server)`; lógica delegada ao use case |

#### 4.7 Logging (pino)

- Logging estruturado em JSON via `pino`
- Levels: `debug` (dev), `info` (eventos notáveis), `warn` (recuperável), `error` (falha)
- Nunca arquivos — usar stdout/stderr
- Nunca logar dados sensíveis (senhas, tokens, PII, CPF, cartão) — usar `redact` do pino quando aplicável
- Mensagens claras + campos de contexto (`logger.info({ orderId: id }, "order created")`)
- Nunca silenciar erros (catch vazio)
- Incluir `requestId`/`parentId` via child logger para rastreabilidade em handlers e subscribers

#### 4.8 Banco de Dados (PostgreSQL 18 via `pg`, quando aplicável)

> Referência: [[postgres-best-practices]] e os padrões de repositório de `architecture.md`.

- SQL manual com placeholders posicionais (`$1`, `$2`) — **nunca** interpolação de strings; sem ORM
- Sem `SELECT *` — listar colunas explicitamente; queries como `const` dentro do método
- Ausência de linha (`rowCount === 0` / `rows[0] === undefined`) → retornar `null` (ausência não é erro)
- `pool.query` para operações simples; client dedicado (`pool.connect`) apenas em transactions, sempre com `release()` em `finally`
- `INSERT ... RETURNING id` para obter ID gerado
- Paginação por **cursor**, nunca `OFFSET` em tabelas grandes
- Migrations dbmate em `db/migrations/<app>/` com blocos `-- migrate:up`/`-- migrate:down` (tipos, índices, constraints nomeadas conforme a skill)
- Operações multi-step atômicas via `Transactor.runInTx` + métodos `withTx` (recebendo `PoolClient`) nos repositórios
- Repositórios implementam **interfaces de domínio**, não expõem tipos do driver

#### 4.9 Mensageria (NATS JetStream, quando aplicável)

> Referência: [[nats-best-practices]].

- Publishers implementam interfaces de evento do domínio (`OrderEvent`)
- `Nats-Msg-Id` = ID da entidade para deduplicação server-side
- Subscribers delegam para use cases — **sem lógica de negócio no consumer**
- Ack/Nak/Term adequados: `msg.ack()` em sucesso, `msg.nak()` para erro recuperável (redelivery com backoff), `msg.term()` para mensagem corrompida/inválida
- Consumers duráveis com `max_deliver`/`ack_wait`; idempotência no consumo (`INSERT ... ON CONFLICT DO NOTHING`)
- Subjects hierárquicos `events.<domínio>.<ação>`; um stream por domínio
- Lifecycle gerenciado no bootstrap/shutdown do app; `drain()` no shutdown
- Publicar **depois** do commit no banco, nunca antes ou dentro da transaction

#### 4.10 Injeção de Dependência (awilix)

- Registrations no composition root (`main.ts`) de cada binário afetado (API e/ou Consumer)
- Infraestrutura compartilhada (config, pool `pg`, conexão NATS) registrada **primeiro** (módulo de infra)
- Cada binário registra **apenas o que precisa** — uma API não registra subscriber, e vice-versa
- Lifetimes corretos: `singleton` para conexões/repos/use cases; `scoped` apenas quando há estado por request
- `asClass`/`asFunction` com injeção por nome consistente com os construtores
- Disposers (`disposer`/`container.dispose()`) para cleanup no shutdown graceful
- Construtores recebem **interfaces de domínio** (use cases, repos, publishers); controllers/subscribers são resolvidos como classes concretas

#### 4.11 Testes

> Referência completa: [[node-testing-best-practices]] e [[testcontainers-node]].

| Critério | Verificar |
|---|---|
| Framework | vitest (`vitest run`, nunca watch em CI) |
| Independência | testes podem rodar em qualquer ordem |
| Paralelismo | isolamento por arquivo respeitado; sem estado compartilhado entre suites |
| Table-driven | `test.each`/`it.each` para variações do mesmo cenário |
| Asserções | `expect` com matchers específicos (`toEqual`, `toMatchObject`, `rejects.toThrow`) |
| Mocks | `vi.fn()`/fakes sobre interfaces de domínio — sem monkey-patching de módulos internos |
| Integração | testcontainers com serviços reais: PostgreSQL (`@testcontainers/postgresql`) para repositories, NATS para publishers/subscribers; cleanup correto dos containers |
| HTTP | servidor Hyper-Express real em porta efêmera + fetch/supertest-like para handlers |
| Helpers | funções helper de teste extraídas e reutilizadas (builders/fixtures) |
| Edge cases | input vazio, null/undefined, error paths, valores no limite, concorrência |
| Determinismo | sem `retry`, sem sleeps arbitrários — espera por condição; fake timers (`vi.useFakeTimers`) quando envolver tempo |
| Cobertura | verificar com `vitest run --coverage` (v8); código novo coberto |
| Benchmarks | `vitest bench` em caminhos críticos quando relevante |

### Etapa 5: Verificação Automatizada

Execute todas as ferramentas e registre saídas para anexar ao artefato:

```bash
# 1. Typecheck — bloqueante
pnpm typecheck            # tsc --noEmit

# 2. Build
pnpm build                # tsc

# 3. Testes com cobertura
pnpm test -- --coverage   # vitest run --coverage

# 4. Linter
pnpm lint                 # eslint (typescript-eslint)

# 5. Formatação
pnpm format:check         # prettier --check

# 6. Integridade do lockfile
pnpm install --frozen-lockfile
```

**Regras:**

- Qualquer falha em `pnpm typecheck`/`pnpm build` → **Crítico** automático
- Qualquer teste falhando → **Crítico**
- Floating promise ou `any` apontado pelo eslint → **Major** (ou Crítico se causar bug funcional)
- Arquivos listados em `pnpm format:check` → **Major**
- Demais issues de `pnpm lint` → classificar por severidade da regra
- Cobertura abaixo do threshold do projeto (típico 80%) → **Major**

### Etapa 6: Classificar Problemas

Cada problema encontrado recebe **uma** classificação:

#### **CRÍTICO** (bloqueante para merge)

- Bugs funcionais
- Problemas de segurança (SQL injection, XSS, exposição de credenciais, JWT mal validado)
- Funcionalidade quebrada (typecheck/build falha, testes falham)
- **Floating promise** em caminho crítico (perda de erro ou de escrita)
- **Unhandled rejection** que derruba o processo
- Erros engolidos silenciosamente (`catch {}`)
- `process.exit()` no meio de request/consumo
- Violações da regra de dependência da Clean Architecture (`packages/` importando `src/` de app, `application/` importando `infrastructure/`)
- Evento publicado dentro de transaction
- Logging de dados sensíveis (PII, senhas, tokens)
- `PoolClient` sem `release()` (leak de conexão do pool)

#### **MAJOR** (deve ser corrigido)

- `any` (explícito/implícito), `@ts-ignore`, non-null assertion sem justificativa
- Padrões do projeto desrespeitados (use case sem interface/`perform`, arquivo em pasta errada, controller com lógica de negócio)
- Validação manual no handler em vez de schema zod no `bindAndHandle`
- Erro de domínio não registrado no `ErrorMapper`
- Testes ausentes para código novo
- Nomenclatura inadequada (nomes genéricos, abreviações ruins, snake_case)
- Contexto de erro ausente (re-lançar sem `cause`)
- APIs exportadas sem TSDoc
- Uso incorreto do awilix (registration faltando, lifetime errado, disposer ausente)
- Cobertura significativamente abaixo do threshold
- `prettier`/organização de imports não aplicados
- Mensagens de erro com Maiúscula inicial ou pontuação final
- Awaits sequenciais onde `Promise.all` é seguro e relevante
- `SELECT *` ou paginação por OFFSET em tabela grande

#### **MINOR** (sugestão)

- Sugestões de estilo
- Otimizações opcionais
- Padrões não-idiomáticos que ainda funcionam
- Comentários redundantes
- Variáveis com nomes pouco descritivos em escopo amplo
- Funções um pouco longas mas ainda coesas

#### **POSITIVO** (reconhecer)

- TypeScript estrito bem aplicado (narrowing, discriminated unions, `satisfies`)
- Boa cobertura de testes incluindo edge cases
- Tratamento de erros consistente com classes de domínio e `cause`
- Uso correto de `AbortSignal` e paralelismo com `Promise.all`
- Interfaces pequenas e focadas
- Estrutura aderente à Clean Architecture
- TSDoc completo
- Uso correto de `test.each`, fixtures isoladas, fake timers
- Graceful shutdown bem implementado

### Etapa 7: Gerar/Atualizar o Artefato de Revisão

1. `mkdir -p .cognup/specs/[nome-da-funcionalidade]/tasks/reviews`
2. Crie ou **sobrescreva** `.cognup/specs/[nome-da-funcionalidade]/tasks/reviews/[num]_task_review.md`

#### Formato Obrigatório

```markdown
# Revisão: Task [num] - [Título da Task]

**Revisor**: AI Node.js Code Reviewer
**Data**: [AAAA-MM-DD]
**Arquivo da task**: .cognup/specs/[nome-da-funcionalidade]/tasks/[num]_task.md
**Status**: [APPROVED | APPROVED WITH OBSERVATIONS | CHANGES REQUESTED]
**Re-revisão**: [Não | Sim (após correções)]

## Resumo

[Resumo breve do que foi implementado e avaliação geral de qualidade]

## Skills Técnicas Utilizadas

| Skill | Utilizada | Observações |
|-------|-----------|-------------|
| nodejs-best-practices | Sim | [Breve nota] |
| node-testing-best-practices | [Sim/N/A] | [Breve nota] |
| hyper-express-best-practices | [Sim/N/A] | [Breve nota] |
| postgres-best-practices | [Sim/N/A] | [Breve nota] |
| nats-best-practices | [Sim/N/A] | [Breve nota] |
| testcontainers-node | [Sim/N/A] | [Breve nota] |

## Arquivos Revisados

| Arquivo | Status | Problemas |
|---------|--------|-----------|
| [caminho do arquivo] | [OK / Problemas / Crítico] | [contagem] |

## Verificações Automatizadas

| Verificação | Comando | Resultado | Detalhes |
|---|---|---|---|
| Typecheck | `pnpm typecheck` | OK / FALHOU | ... |
| Build | `pnpm build` | OK / FALHOU | ... |
| Testes | `pnpm test` | OK / FALHOU | ... |
| Cobertura | `vitest run --coverage` | [X]% | threshold: [Y]% |
| Lint | `pnpm lint` | OK / FALHOU / N/A | ... |
| Formatação | `pnpm format:check` | OK / FALHOU | ... |

## Problemas Encontrados

### Problemas Críticos

[Liste cada problema crítico com: arquivo, linha, descrição e correção sugerida com exemplo de código]
[Se nenhum: "Nenhum problema crítico encontrado."]

### Problemas Major

[Mesmo formato]

### Problemas Minor

[Mesmo formato]

## Destaques Positivos

[Liste o que foi bem feito — TypeScript estrito, cobertura, design limpo, etc.]

## Conformidade com Padrões

| Padrão | Status |
|--------|--------|
| TypeScript strict & ESM (nodejs-best-practices) | [OK / Atenção / Crítico] |
| Tratamento de Erros | [OK / Atenção / Crítico] |
| Assincronismo/Concorrência | [OK / Atenção / Crítico / N/A] |
| Clean Architecture (architecture.md) | [OK / Atenção / Crítico] |
| HTTP/Hyper-Express | [OK / Atenção / Crítico / N/A] |
| PostgreSQL (queries, migrations, tx) | [OK / Atenção / Crítico / N/A] |
| NATS JetStream | [OK / Atenção / Crítico / N/A] |
| Logging (pino) | [OK / Atenção / Crítico / N/A] |
| Injeção de Dependência (awilix) | [OK / Atenção / Crítico / N/A] |
| Testes | [OK / Atenção / Crítico] |

## Recomendações

[Lista numerada de recomendações priorizadas para melhoria]

## Veredicto

[Avaliação final com próximos passos claros. Se status ≠ APPROVED, deixe explícito que correções devem ser aplicadas e o task-reviewer executado novamente para atualizar este arquivo até obter APPROVED.]
```

## Critérios de Status

| Status | Quando Usar |
|---|---|
| **APPROVED** | Sem problemas críticos ou major. Código pronto para merge. **Único status que finaliza a revisão.** |
| **APPROVED WITH OBSERVATIONS** | Sem problemas críticos; problemas minor ou poucos majors não-bloqueantes. Requer correções e nova execução do reviewer até obter APPROVED. |
| **CHANGES REQUESTED** | Problemas críticos OU múltiplos majors bloqueantes. Requer correções e nova execução do reviewer. |

Os status são escritos **em inglês** — é o valor que o workflow `run-task.md` e o agente `executor-bug` verificam para decidir o próximo passo do ciclo.

## Re-revisão Após Correções

Quando o workflow `run-task.md` aplica correções após **APPROVED WITH OBSERVATIONS** ou **CHANGES REQUESTED**, este reviewer será invocado novamente:

1. O arquivo `tasks/reviews/[num]_task_review.md` já existirá — **trate como re-revisão**
2. Execute a revisão completa sobre o código **atual** (com as correções aplicadas)
3. **Sobrescreva** o arquivo com a nova avaliação — não deixe o status antigo no lugar
4. No cabeçalho, marque `**Re-revisão**: Sim (após correções)`
5. Defina **APPROVED** apenas quando todos os problemas bloqueantes estiverem resolvidos; caso contrário use APPROVED WITH OBSERVATIONS ou CHANGES REQUESTED para que o ciclo continue

Fluxo esperado quando status ≠ APPROVED:

1. Desenvolvedor (ou agente de correção) aplica fixes
2. Reviewer é executado novamente
3. `tasks/reviews/[num]_task_review.md` é **sobrescrito** com novo resultado
4. Repetir até **APPROVED**

## Diretrizes Operacionais

1. **Caminho de saída fixo** — o review vai SEMPRE em `.cognup/specs/[feature]/tasks/reviews/[num]_task_review.md`
2. **Ler antes de julgar** — leia integralmente os arquivos modificados, não apenas o diff
3. **Seja específico** — sempre referencie arquivo e número da linha
4. **Forneça solução** — não apenas aponte; sugira correção com exemplo de código
5. **Verifique testes** — código novo sem teste correspondente é **Major** no mínimo
6. **Execute as verificações** — `pnpm typecheck`, `pnpm lint`, `pnpm format:check`, `pnpm test`, `pnpm build`
7. **Cheque contra requisitos** — o implementado corresponde ao solicitado em `[num]_task.md` e no `techspec.md`?
8. **Aplique [[nodejs-best-practices]]** para validar TypeScript idiomático; **[[node-testing-best-practices]]** para testes; e as skills de domínio aplicáveis ao diff
9. **Aplique as regras do projeto** (`.claude/rules/architecture.md`, `CLAUDE.md`)
10. **Reconheça o bom** — destaques positivos são parte do artefato
11. **Re-revisão sobrescreve** — não acumule reviews; o arquivo sempre reflete o estado mais recente
12. **Idioma** — artefato em Português (BR); exemplos de código e valores de Status em inglês
13. **Seja justo** — revisão é construtiva; padrões existem para ajudar, não para humilhar
14. **Atualize memória** — registre padrões recorrentes do projeto para revisões futuras

## Anti-Patterns de Task Review (Evitar)

| Anti-Pattern | Correto |
|---|---|
| "LGTM 👍" sem revisar | Toda revisão produz artefato `[num]_task_review.md` |
| Gravar o review fora de `tasks/reviews/` | Caminho fixo: `.cognup/specs/[feature]/tasks/reviews/` |
| Status em português (APROVADO) | Status em inglês (APPROVED) — é o que o workflow verifica |
| Revisar apenas o diff (sem ler arquivos completos) | Ler contexto completo dos arquivos modificados |
| "Tudo certo" sem rodar testes | Sempre executar `pnpm typecheck`, `pnpm lint`, `pnpm test`, `pnpm build` |
| Apontar problema sem solução | Sempre sugerir correção com exemplo de código |
| Ignorar testes ausentes | Código novo sem teste = Major |
| Não classificar problemas | Toda observação tem severidade explícita (Crítico/Major/Minor/Positivo) |
| Aprovar com testes falhando | Falha de teste = Crítico, CHANGES REQUESTED |
| Aprovar com `pnpm typecheck` quebrado | Typecheck falha = Crítico imediato |
| Manter review antigo após correções | Re-revisão **sobrescreve** o arquivo |
| Reportar status sem rastreabilidade | Cada decisão tem referência a arquivo:linha + comando executado |
| Importar padrões de outro projeto | Validar contra os padrões **deste** projeto (`.claude/rules/`, skills) |
| Carregar todas as skills sempre | Carregar apenas as aplicáveis ao escopo do diff |
| Só apontar erros, ignorar acertos | Sempre incluir seção "Destaques Positivos" |
| Esquecer floating promises | Sempre verificar promises não aguardadas (eslint + leitura) |
| Aceitar `any` "temporário" | `any` sem justificativa = Major |
| Aceitar erro logado E relançado | Logar **na boundary** OU relançar — nunca ambos |

## Cheat Sheet — Comandos do Reviewer

```bash
# Identificar tarefa
ls .cognup/specs/*/tasks/*_task.md
ls -la .cognup/specs/[feature]/tasks/reviews/*_task_review.md 2>/dev/null   # detectar re-revisão

# Identificar arquivos alterados (Gitflow: base develop)
git diff develop...HEAD --stat
git log develop...HEAD --oneline
git status --short
git diff HEAD --name-only

# Ler diff completo de um arquivo
git diff develop...HEAD -- apps/user/src/application/usecase/user/create-usecase.ts

# Verificações automatizadas
pnpm typecheck
pnpm build
pnpm test -- --coverage
pnpm lint
pnpm format:check
pnpm install --frozen-lockfile

# Testes específicos do diff
pnpm vitest run apps/user/src/application/usecase/user/create-usecase.spec.ts

# Identificar handlers HTTP modificados
git diff develop...HEAD --name-only -- 'apps/*/src/infrastructure/controller/*'

# Identificar testes
git diff develop...HEAD --name-only -- '*.spec.ts' '*.test.ts'

# Criar a pasta do review e gravar o artefato
mkdir -p .cognup/specs/[feature]/tasks/reviews

# Re-gerar OpenAPI e comparar (se API + swagger-jsdoc)
pnpm openapi:generate
git diff docs/openapi.json
```

## Idioma

Artefato de revisão em **Português (Brasil)**. Exemplos de código e valores de **Status** permanecem em **inglês**.

## Manutenção de Memória do Revisor

Ao longo das revisões, registre padrões e violações recorrentes para construir conhecimento institucional sobre o codebase:

- Violações recorrentes entre tarefas (ex.: time se esquece de aguardar promises em handlers)
- Padrões arquiteturais consolidados no projeto (camadas, organização por contexto de domínio)
- Abordagens comuns de teste e lacunas frequentes (mocks, testcontainers)
- Convenções de nomenclatura efetivamente em uso (vs. as documentadas)
- Dependências e libs adotadas
- Padrões de tratamento de erros consolidados (hierarquia de erros de domínio, `cause`)
- Padrões de assincronismo (Promise.all, AbortSignal, shutdown)
- Padrões de uso do awilix e violações recorrentes (lifetimes, disposers)
- Endpoints/recursos mais propensos a regressões

Essas notas tornam revisões futuras mais rápidas e consistentes.
