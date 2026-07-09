---
name: task-reviewer-best-practices
description: Aplica melhores práticas de Task Review para revisão sistemática de tarefas concluídas em serviços Go (Golang). Cobre identificação da tarefa em `.cognup/specs/[feature]/tasks/[num]_task.md`, análise de diff (`git diff`/`git log`), validação contra padrões idiomáticos do Go (Effective Go, Code Review Comments), Clean Architecture (`.claude/rules/architecture.md`), tratamento de erros (`%w`, `errors.Is/As`, sentinel errors), concorrência segura (`context.Context`, goroutines, mutex, channels), Fiber v3 (BaseController, BindAndHandle, ErrorMapper, StructValidator), PostgreSQL 18 (queries manuais, migrations, Transactor/WithTx), NATS JetStream (ack/nak/term, dedup, idempotência), uber-go/fx, slog, testes (table-driven, testify, gomock, testcontainers-go), execução de verificações automatizadas (`go build`, `go vet`, `go test -race`, `gofmt`, `golangci-lint`), classificação de problemas (Crítico/Major/Minor/Positivo) e geração ou re-escrita do artefato em `.cognup/specs/[feature]/tasks/reviews/[num]_task_review.md` em Português (BR) com status em inglês (APPROVED/APPROVED WITH OBSERVATIONS/CHANGES REQUESTED). Use sempre que uma tarefa Go foi concluída via workflow `run-task.md` e precisa ser revisada antes do merge, ao gerar artefato de revisão, ao reexecutar review após correções, ou ao avaliar aderência de uma implementação Go aos padrões do projeto.
---

# Task Reviewer Best Practices — Revisão Sistemática de Tarefas Go

Skill que aplica o processo metódico de **Task Review** sobre tarefas concluídas usando o workflow `run-task.md` em serviços Go (Golang). Combina identificação da tarefa, análise de diff, validação contra padrões idiomáticos, verificações automatizadas, classificação de problemas e geração do artefato `[num]_task_review.md` rastreável.

Combina com [[go-best-practices]] (padrões de código idiomático Go) e [[go-testing-best-practices]] (estratégias de teste), e — conforme o escopo do diff — com as skills de domínio [[fiber-v3-best-practices]] (HTTP), [[postgres-best-practices]] (banco), [[nats-best-practices]] (mensageria) e [[testcontainers-go]] (testes de integração). A arquitetura de referência é a definida em `.claude/rules/architecture.md` (Clean Architecture em monorepo com Fiber v3, PostgreSQL 18, NATS JetStream e uber-go/fx).

## Princípio Fundamental

> **Revisão completa, justa e rastreável.** O revisor identifica a tarefa, lê integralmente os arquivos modificados (não apenas os diffs), valida contra padrões idiomáticos do Go, executa verificações automatizadas e gera o artefato `[num]_task_review.md` com problemas classificados por severidade. Cada problema referencia arquivo, linha e correção sugerida; cada boa prática observada é reconhecida.

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

- Quando uma tarefa Go foi finalizada via workflow `run-task.md` e o usuário pediu revisão
- Quando o usuário cita um número de tarefa específico (`task 5`, `task 12`) para revisar
- Proativamente após uma implementação significativa em Go ser concluída
- Em re-revisão: quando `tasks/reviews/[num]_task_review.md` já existe com status diferente de APPROVED
- Antes de abrir Pull Request de uma tarefa concluída
- Ao validar aderência de código novo às convenções do projeto (Clean Architecture, fx, Fiber v3, PostgreSQL, NATS)

## Missão do Revisor

Você é um revisor de código de elite com domínio profundo em **Go (Golang), sistemas distribuídos, REST APIs, Clean Architecture e engenharia de software**. Seu olhar é meticuloso para código idiomático Go, qualidade, manutenibilidade e aderência aos padrões do projeto.

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

1. **Consultar `.claude/rules/architecture.md`** — camadas (domain/application/infrastructure), regra de dependência, nomenclatura kebab-case, convenções de construtores, BaseController, Transactor, composição FX, guia "Adicionando uma Nova Feature"
2. **Consultar `CLAUDE.md`** (se existir) — convenções específicas e comandos de build/test
3. **Confirmar a stack** (padrão do projeto): **Go 1.26, Fiber v3, PostgreSQL 18, NATS JetStream, uber-go/fx, log/slog, go-playground/validator/v10, swaggo, testify/gomock/testcontainers-go** — valide desvios pelo `go.mod`
4. **Carregar [[go-best-practices]]** como referência idiomática (sempre)
5. **Carregar as skills de domínio aplicáveis ao diff**: [[fiber-v3-best-practices]] se tocou controllers/rotas/middlewares; [[postgres-best-practices]] se tocou repositórios/queries/migrations; [[nats-best-practices]] se tocou publishers/subscribers/streams; [[go-testing-best-practices]] e [[testcontainers-go]] se tocou testes
6. **Não carregar skills fora do escopo** — ex.: não carregue NATS para uma task que só cria entidade de domínio. Registre no artefato quais foram usadas e quais foram N/A

A revisão precisa ser fiel **ao projeto que você está revisando** — não imponha padrões que o projeto não adota. Mas valide rigorosamente os padrões que o projeto **declarou** adotar.

### Etapa 4: Conduzir a Revisão

Revise o código contra os critérios abaixo, agrupados por área. Para cada violação, registre **arquivo + linha + descrição + correção sugerida**.

#### 4.1 Padrões Gerais de Código

- **Idioma do código**: tudo em inglês (variáveis, funções, structs, comentários)
- **Nomenclatura clara**: identificadores exportados descritivos; nomes curtos apenas em escopos pequenos (Go permite `i`, `r`, `err` localmente)
- **Constantes**: sem números mágicos — usar `const` ou `iota`
- **Funções**: uma única ação clara, verbos para mutações (`CreateOrder`, `UpdateUser`)
- **Parâmetros**: > 3-4 parâmetros indica necessidade de option struct ou functional options
- **Condicionais**: sem aninhamento profundo, **early returns** e **guard clauses**
- **Tamanho de método**: lógica complexa < 50 linhas; se ultrapassar, extrair helper
- **Tamanho de arquivo**: responsabilidade única; dividir quando exceder ~500 linhas
- **Comentários**: identificadores exportados **DEVEM** ter doc comments (`// FunctionName does...`); evitar comentários que repetem o código

#### 4.2 Go Idioms (Effective Go + Code Review Comments)

> Referências: [Effective Go](https://go.dev/doc/effective_go), [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments). Use [[go-best-practices]] para detalhes.

| Critério | Verificar |
|---|---|
| **Nomenclatura** | `MixedCaps`/`mixedCaps`; **nunca** `snake_case` para identificadores Go |
| **Acrônimos** | `HTTPClient`, `ID`, `URL` (totalmente em maiúsculas) — **não** `HttpClient`, `Id` |
| **Pacotes** | curto, lowercase, palavra única; sem `_` ou `mixedCaps`; evitar `util`, `common`, `helper` |
| **Receivers** | 1-2 letras, consistentes entre métodos do mesmo tipo; **nunca** `this`/`self` |
| **Interfaces de método único** | terminam em `-er` (`Reader`, `Writer`, `Stringer`) |
| **Sentinel errors** | `ErrSomething` (ex.: `ErrNotFound`) |
| **Error types** | terminam em `Error` (ex.: `ValidationError`) |
| **Não-exportado por padrão** | exportar apenas API pública |
| **Zero values** | aproveitar zero values significativos; evitar `new()` quando zero value é útil |
| **Composição sobre herança** | usar embedding, não hierarquias profundas |
| **Aceitar interfaces, retornar structs** | exceto construtores que retornam interface por contrato do projeto (use cases/repos/publishers/adapters) |
| **Interfaces pequenas** | 1-2 métodos; compor maiores via embedding |
| **Getter sem prefixo `Get`** | campo `owner` → getter `Owner()`, não `GetOwner()` |

#### 4.3 Tratamento de Erros

| Regra | Como Verificar |
|---|---|
| Sempre verificar erros | Sem `data, _ := ...` (exceto com comentário justificando) |
| Wrap com contexto | `fmt.Errorf("operation: %w", err)` |
| Sentinel errors | `var ErrNotFound = errors.New("not found")` + `errors.Is(err, ErrNotFound)` |
| Custom error types | `errors.As(err, &target)` quando precisa de contexto extra |
| Sem `panic` em biblioteca | `panic` apenas em `main` para estados irrecuperáveis (ou em `init`) |
| Mensagens de erro | lowercase, sem pontuação no final, descrevem o que falhou |
| Não logar e retornar | escolha **logar na boundary** OU **retornar**, nunca ambos |

#### 4.4 Concorrência

| Regra | Como Verificar |
|---|---|
| Gerenciamento de ciclo de vida | toda goroutine tem `context.Context`, `sync.WaitGroup` ou `errgroup.Group` |
| Channels sobre memória compartilhada | preferir channels quando apropriado |
| Estado compartilhado protegido | `sync.Mutex`/`sync.RWMutex` com seções críticas pequenas |
| Propagação de contexto | `ctx context.Context` é o **primeiro** parâmetro; nunca armazenar em structs |
| Sem goroutine leaks | toda goroutine tem caminho de saída claro |
| `select` com `ctx.Done()` | tratar cancelamento em goroutines de longa duração |
| Race-free | `go test -race ./...` passa sem detecções |

#### 4.5 Estrutura do Projeto (Clean Architecture — `.claude/rules/architecture.md`)

| Camada | Pode importar |
|---|---|
| `internal/domain/` | apenas stdlib |
| `internal/application/` | apenas `internal/domain/` |
| `internal/infrastructure/` | `internal/domain/`, `pkg/`, libs externas |
| `pkg/` | **nunca** `internal/` de nenhum app |
| `apps/<app>/cmd/<binary>/main.go` | tudo (composição do grafo FX daquele binário) |

Validar também:

- **Use cases**: 1 ação = 1 interface = 1 arquivo, método único `Perform(ctx, input)`; implementação em `application/usecase/<contexto>/<ação>-usecase.go`
- **Construtores**: use cases/repos/publishers/adapters retornam **interface de domínio**; controllers/subscribers retornam **ponteiro concreto**
- **Libs externas via inversão de dependência**: interface em `domain/<capacidade>/` (ex.: `crypt/`, `token/`), implementação em `infrastructure/adapter/<lib>-adapter.go`
- **Eventos publicados DEPOIS do commit** no banco, nunca dentro da transaction
- Convenções de nomenclatura de arquivos: kebab-case (`order-repository.go`)
- Sem imports circulares; direção limpa de dependências; sem estado global

#### 4.6 REST/HTTP (Fiber v3, quando aplicável)

> Referência: [[fiber-v3-best-practices]] e o padrão BaseController de `architecture.md`.

| Critério | Verificar |
|---|---|
| Versão | `github.com/gofiber/fiber/v3` |
| Handler signature | `func(c fiber.Ctx) error` (não `*fiber.Ctx`) |
| BaseController | controllers embutem `pkgctrl.BaseController`; rotas via `ctrl.Handle(...)` ou `pkgctrl.BindAndHandle(...)` |
| RequestContext | handlers usam o `RequestContext` já populado (requestId, parentId, IP, logger) — nunca extrair manualmente |
| Recursos | inglês, plural, kebab-case (`/orders`, `/payment-methods`); máximo 3 níveis |
| Status codes | constantes `fiber.Status*` (não números mágicos) |
| Binding | `c.Bind().Body(&input)` ao invés de `json.Unmarshal(c.Body(), &v)` |
| Validação | tags `json` + `validate` (go-playground/validator) via `StructValidator` — nunca validação manual no handler |
| Erros de domínio | `return err` — mapeamento HTTP via `ErrorMapper`; novos erros registrados no `ProvideErrorMapper` do app |
| `c.Context()` | sempre extrair antes de passar para camadas internas ou goroutines |
| Zero-allocation | copiar `c.Body()`, `c.Params()`, `c.Query()` antes de persistir além do handler |
| Middlewares | `recover` registrado primeiro; CORS, helmet, requestid configurados |
| Graceful shutdown | `app.Shutdown()` no hook `OnStop` do fx |
| Anotações Swagger | swaggo em todos os handlers públicos |
| Controllers | implementam `RegisterRoutes(app *fiber.App)`; lógica delegada ao use case |

#### 4.7 Logging (slog)

- Logging estruturado em JSON via `slog`
- Levels: `DEBUG` (dev), `INFO` (eventos notáveis), `WARN` (recuperável), `ERROR` (falha)
- Nunca arquivos — usar stdout/stderr
- Nunca logar dados sensíveis (senhas, tokens, PII, CPF, cartão)
- Mensagens claras + campos de contexto (`slog.String("orderId", id)`)
- Nunca silenciar erros (`_ = err`)
- Incluir `requestId`/`parentId` para rastreabilidade em handlers e subscribers

#### 4.8 Banco de Dados (PostgreSQL 18, quando aplicável)

> Referência: [[postgres-best-practices]] e os padrões de repositório de `architecture.md`.

- SQL manual com placeholders posicionais (`$1`, `$2`) — **nunca** concatenação de strings; sem ORM
- Sem `SELECT *` — listar colunas explicitamente; queries como `const` dentro do método
- `sql.ErrNoRows` → retornar `nil, nil` (ausência não é erro)
- `defer rows.Close()` sempre; verificar `rows.Err()` após iteração
- `INSERT ... RETURNING id` para obter ID gerado
- Paginação por **cursor**, nunca `OFFSET` em tabelas grandes
- Migrations `up.sql`/`down.sql` em `migrations/<app>/` (tipos, índices, constraints nomeadas conforme a skill)
- Operações multi-step atômicas via `Transactor.RunInTx` + métodos `WithTx` nos repositórios
- Repositórios retornam **interfaces de domínio**, não tipos concretos

#### 4.9 Mensageria (NATS JetStream, quando aplicável)

> Referência: [[nats-best-practices]].

- Publishers implementam interfaces de evento do domínio (`event.OrderEvent`)
- `Nats-Msg-Id` = ID da entidade para deduplicação server-side
- Subscribers delegam para use cases — **sem lógica de negócio no consumer**
- Ack/Nak/Term adequados: `msg.Ack()` em sucesso, `msg.Nak()` para erro recuperável (redelivery com backoff), `msg.Term()` para mensagem corrompida/inválida
- Consumers duráveis com `MaxDeliver`/`AckWait`; idempotência no consumo (`INSERT ... ON CONFLICT DO NOTHING`)
- Subjects hierárquicos `events.<domínio>.<ação>`; um stream por domínio
- Lifecycle gerenciado via fx (`OnStart`/`OnStop`); reconnect e drain no shutdown
- Publicar **depois** do commit no banco, nunca antes ou dentro da transaction

#### 4.10 Injeção de Dependência (uber-go/fx)

- Providers registrados no `main.go` de cada binário afetado (API e/ou Consumer)
- `config.InfraModule` **sempre primeiro**
- Cada binário fornece **apenas o que precisa** — uma API não fornece subscriber, e vice-versa
- Lifecycle hooks (`OnStart`/`OnStop`) para startup e shutdown graceful
- Construtores seguem convenção:
  - Repos/publishers/use cases/adapters → **interface de domínio**
  - Controllers/subscribers/entidades → **ponteiro concreto**

#### 4.11 Testes

> Referência completa: [[go-testing-best-practices]] e [[testcontainers-go]].

| Critério | Verificar |
|---|---|
| Framework | pacote padrão `testing` |
| Independência | testes podem rodar em qualquer ordem |
| Paralelismo | `t.Parallel()` quando seguro |
| Table-driven | `tests := []struct{...}` + `t.Run(tc.name, ...)` |
| Asserções | `testify` (`assert`/`require`) |
| Mocks | gerados via `gomock` sobre interfaces de domínio — sem monkey-patching |
| Integração | testcontainers-go com serviços reais: PostgreSQL (módulo `postgres`) para repositories, NATS (módulo `nats`) para publishers/subscribers; cleanup correto dos containers |
| HTTP | `app.Test()` do Fiber v3 / `httptest` para handlers |
| Helpers | `t.Helper()` em funções helper |
| Edge cases | input vazio, nil pointers, error paths, valores no limite |
| Race detector | `go test -race ./...` limpo |
| Cobertura | verificar com `go test -cover`; código novo coberto |
| Benchmarks | `Benchmark*` em caminhos críticos quando relevante |

### Etapa 5: Verificação Automatizada

Execute todas as ferramentas e registre saídas para anexar ao artefato:

```bash
# 1. Compilação — bloqueante
go build ./...

# 2. Vet — construções suspeitas
go vet ./...

# 3. Testes com race detector e cobertura
go test -race -count=1 -coverprofile=coverage.out ./...
go tool cover -func=coverage.out | tail -1

# 4. Linter (se disponível no projeto)
golangci-lint run ./...

# 5. Formatação
gofmt -l .

# 6. Imports organizados (se goimports disponível)
goimports -l .

# 7. go.mod limpo
go mod tidy -diff
go mod verify
```

**Regras:**

- Qualquer falha em `go build` → **Crítico** automático
- Qualquer detecção em `go vet` → **Major** (ou Crítico se for race/concorrência)
- Qualquer teste falhando → **Crítico**
- Race detector detectou → **Crítico**
- Arquivos listados em `gofmt -l .` → **Major**
- `golangci-lint` issues → classificar por severidade do linter
- Cobertura abaixo do threshold do projeto (típico 80%) → **Major**

### Etapa 6: Classificar Problemas

Cada problema encontrado recebe **uma** classificação:

#### **CRÍTICO** (bloqueante para merge)

- Bugs funcionais
- Problemas de segurança (SQL injection, XSS, exposição de credenciais, JWT mal validado)
- Funcionalidade quebrada (não compila, testes falham)
- **Goroutine leaks**
- **Race conditions**
- Erros retornados não verificados
- `panic` em código de biblioteca
- Violações da regra de dependência da Clean Architecture (`pkg/` importando `internal/`, `application/` importando `infrastructure/`)
- Evento publicado dentro de transaction
- Logging de dados sensíveis (PII, senhas, tokens)
- `c.Body()`/`c.Params()` armazenados sem copy em handler Fiber

#### **MAJOR** (deve ser corrigido)

- Violações de idiomas Go (`snake_case`, `GetX()`, acrônimos errados)
- Padrões do projeto desrespeitados (construtor com retorno errado, arquivo em pasta errada, use case sem interface `Perform`)
- Validação manual no handler em vez de tags `validate`
- Erro de domínio não registrado no `ErrorMapper`
- Testes ausentes para código novo
- Nomenclatura inadequada (nomes genéricos, abreviações ruins)
- Contexto de erro ausente (sem `%w`)
- Identificadores exportados sem doc comment
- Uso incorreto de uber-go/fx (provider faltando, ordem errada)
- Cobertura significativamente abaixo do threshold
- `gofmt`/`goimports` não aplicados
- Mensagens de erro com Maiúscula inicial ou pontuação final
- `rows.Close()` sem `defer` ou `rows.Err()` não verificado
- `SELECT *` ou paginação por OFFSET em tabela grande

#### **MINOR** (sugestão)

- Sugestões de estilo
- Otimizações opcionais
- Padrões não-idiomáticos que ainda funcionam
- Comentários redundantes
- Variáveis com nomes pouco descritivos em escopo amplo
- Funções um pouco longas mas ainda coesas

#### **POSITIVO** (reconhecer)

- Go idiomático bem aplicado
- Boa cobertura de testes incluindo edge cases
- Tratamento de erros consistente com `%w` e `errors.Is/As`
- Uso correto de `context.Context` em toda a pilha
- Interfaces pequenas e focadas
- Estrutura aderente à Clean Architecture
- Doc comments completos
- Uso correto de `t.Parallel()`, `t.Helper()`, table-driven tests
- Graceful shutdown bem implementado

### Etapa 7: Gerar/Atualizar o Artefato de Revisão

1. `mkdir -p .cognup/specs/[nome-da-funcionalidade]/tasks/reviews`
2. Crie ou **sobrescreva** `.cognup/specs/[nome-da-funcionalidade]/tasks/reviews/[num]_task_review.md`

#### Formato Obrigatório

```markdown
# Revisão: Task [num] - [Título da Task]

**Revisor**: AI Go Code Reviewer
**Data**: [AAAA-MM-DD]
**Arquivo da task**: .cognup/specs/[nome-da-funcionalidade]/tasks/[num]_task.md
**Status**: [APPROVED | APPROVED WITH OBSERVATIONS | CHANGES REQUESTED]
**Re-revisão**: [Não | Sim (após correções)]

## Resumo

[Resumo breve do que foi implementado e avaliação geral de qualidade]

## Skills Técnicas Utilizadas

| Skill | Utilizada | Observações |
|-------|-----------|-------------|
| go-best-practices | Sim | [Breve nota] |
| go-testing-best-practices | [Sim/N/A] | [Breve nota] |
| fiber-v3-best-practices | [Sim/N/A] | [Breve nota] |
| postgres-best-practices | [Sim/N/A] | [Breve nota] |
| nats-best-practices | [Sim/N/A] | [Breve nota] |
| testcontainers-go | [Sim/N/A] | [Breve nota] |

## Arquivos Revisados

| Arquivo | Status | Problemas |
|---------|--------|-----------|
| [caminho do arquivo] | [OK / Problemas / Crítico] | [contagem] |

## Verificações Automatizadas

| Verificação | Comando | Resultado | Detalhes |
|---|---|---|---|
| Build | `go build ./...` | OK / FALHOU | ... |
| Vet | `go vet ./...` | OK / FALHOU | ... |
| Testes | `go test -race -count=1 ./...` | OK / FALHOU | ... |
| Cobertura | `go tool cover -func` | [X]% | threshold: [Y]% |
| Lint | `golangci-lint run` | OK / FALHOU / N/A | ... |
| Formatação | `gofmt -l .` | OK / FALHOU | ... |

## Problemas Encontrados

### Problemas Críticos

[Liste cada problema crítico com: arquivo, linha, descrição e correção sugerida com exemplo de código]
[Se nenhum: "Nenhum problema crítico encontrado."]

### Problemas Major

[Mesmo formato]

### Problemas Minor

[Mesmo formato]

## Destaques Positivos

[Liste o que foi bem feito — Go idiomático, cobertura, design limpo, etc.]

## Conformidade com Padrões

| Padrão | Status |
|--------|--------|
| Go Idioms & Effective Go (go-best-practices) | [OK / Atenção / Crítico] |
| Tratamento de Erros | [OK / Atenção / Crítico] |
| Concorrência | [OK / Atenção / Crítico / N/A] |
| Clean Architecture (architecture.md) | [OK / Atenção / Crítico] |
| HTTP/Fiber v3 | [OK / Atenção / Crítico / N/A] |
| PostgreSQL (queries, migrations, tx) | [OK / Atenção / Crítico / N/A] |
| NATS JetStream | [OK / Atenção / Crítico / N/A] |
| Logging (slog) | [OK / Atenção / Crítico / N/A] |
| Injeção de Dependência (uber-go/fx) | [OK / Atenção / Crítico / N/A] |
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
6. **Execute as verificações** — `go build`, `go vet`, `go test -race`, `gofmt`, `golangci-lint`
7. **Cheque contra requisitos** — o implementado corresponde ao solicitado em `[num]_task.md` e no `techspec.md`?
8. **Aplique [[go-best-practices]]** para validar idiomas Go; **[[go-testing-best-practices]]** para testes; e as skills de domínio aplicáveis ao diff
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
| "Tudo certo" sem rodar testes | Sempre executar `go build`, `go vet`, `go test -race` |
| Apontar problema sem solução | Sempre sugerir correção com exemplo de código |
| Ignorar testes ausentes | Código novo sem teste = Major |
| Não classificar problemas | Toda observação tem severidade explícita (Crítico/Major/Minor/Positivo) |
| Aprovar com testes falhando | Falha de teste = Crítico, CHANGES REQUESTED |
| Aprovar com `go build` quebrado | Build falha = Crítico imediato |
| Manter review antigo após correções | Re-revisão **sobrescreve** o arquivo |
| Reportar status sem rastreabilidade | Cada decisão tem referência a arquivo:linha + comando executado |
| Importar padrões de outro projeto | Validar contra os padrões **deste** projeto (`.claude/rules/`, skills) |
| Carregar todas as skills sempre | Carregar apenas as aplicáveis ao escopo do diff |
| Só apontar erros, ignorar acertos | Sempre incluir seção "Destaques Positivos" |
| Esquecer race detector | Sempre `go test -race ./...` |
| Aceitar `panic` em código de biblioteca | `panic` apenas em `main` ou `init`, casos terminais |
| Aceitar erro logado E retornado | Logar **na boundary** OU retornar — nunca ambos |

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
git diff develop...HEAD -- apps/user/internal/application/usecase/user/create-usecase.go

# Verificações automatizadas
go build ./...
go vet ./...
go test -race -count=1 -coverprofile=coverage.out ./...
go tool cover -func=coverage.out | tail -1            # cobertura total
golangci-lint run ./...
gofmt -l .
goimports -l .
go mod tidy -diff
go mod verify

# Testes específicos do diff
go test -race -run TestCreateUseCase ./apps/user/internal/application/usecase/user/

# Identificar handlers HTTP modificados
git diff develop...HEAD --name-only -- 'apps/*/internal/infrastructure/controller/*'

# Identificar testes
git diff develop...HEAD --name-only -- '*_test.go'

# Criar a pasta do review e gravar o artefato
mkdir -p .cognup/specs/[feature]/tasks/reviews

# Re-gerar swagger e comparar (se API + swaggo)
swag init -g apps/<app>/cmd/api/main.go --parseDependency --parseInternal
git diff docs/
```

## Idioma

Artefato de revisão em **Português (Brasil)**. Exemplos de código e valores de **Status** permanecem em **inglês**.

## Manutenção de Memória do Revisor

Ao longo das revisões, registre padrões e violações recorrentes para construir conhecimento institucional sobre o codebase:

- Violações recorrentes entre tarefas (ex.: time se esquece de copiar `c.Body()`)
- Padrões arquiteturais consolidados no projeto (camadas, organização por contexto de domínio)
- Abordagens comuns de teste e lacunas frequentes (mocks, testcontainers)
- Convenções de nomenclatura efetivamente em uso (vs. as documentadas)
- Dependências e libs adotadas
- Padrões de tratamento de erros consolidados
- Padrões de concorrência (errgroup, context propagation)
- Padrões de uso do uber-go/fx e violações recorrentes
- Endpoints/recursos mais propensos a regressões

Essas notas tornam revisões futuras mais rápidas e consistentes.
