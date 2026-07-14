---
name: bugfix-best-practices
description: Aplica melhores práticas de Bugfix para correção sistemática de bugs documentados pelo QA em serviços Rust. Cobre leitura do arquivo `.cognup/specs/[feature]/report/bugs.md`, análise de causa raiz (nunca sintomas), planejamento por bug, implementação da correção conforme Clean Architecture (`.claude/rules/architecture.md`) e stack do projeto (Axum 0.8, PostgreSQL 18/sqlx, NATS JetStream/async-nats, composition root manual + `Arc<dyn Trait>`, tracing, thiserror/anyhow), criação de testes de regressão (unitário com `#[cfg(test)] mod tests` + mockall, integração com testcontainers, E2E de API com reqwest), validação completa (`cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo build`, `cargo test`), atualização do bugs.md com status das correções, relatório final, commits no padrão Conventional Commits e ciclo de revisão (task-reviewer até APPROVED, depois qa-validator até APROVADO). Use sempre que houver bugs documentados em bugs.md para corrigir, quando o QA reprovar uma feature, ao corrigir regressões, ou ao criar testes de regressão para bugs encontrados.
---

# Bugfix Best Practices — Correção Sistemática de Bugs Rust

Skill que aplica o processo metódico de **Bugfix** sobre bugs documentados pelo QA (`qa-best-practices`) em serviços Rust. Combina análise de causa raiz, correção conforme os padrões do projeto, testes de regressão, validação completa, rastreabilidade no `bugs.md` e o ciclo de revisão até aprovação.

Combina com [[rust-best-practices]] (padrões de código idiomático Rust) e [[rust-testing-best-practices]] (testes de regressão), e — conforme o componente afetado — com [[axum-best-practices]] (HTTP), [[postgres-best-practices]] (banco), [[nats-best-practices]] (mensageria) e [[testcontainers-rust]] (integração). Commits seguem [[conventional-commit]]. A arquitetura de referência é `.claude/rules/architecture.md` (monorepo com Axum 0.8, PostgreSQL 18, NATS JetStream e composition root manual com `Arc<dyn Trait>`).

## Princípio Fundamental

> **Causa raiz, nunca sintoma.** Todo bug é corrigido na origem do problema — sem gambiarras, sem supressão de sintomas, sem `if` defensivo ou `unwrap_or_default()` escondendo o defeito. Cada correção vem acompanhada de testes de regressão que falhariam se a correção fosse revertida. O bugfix só termina quando TODOS os bugs do `bugs.md` estão corrigidos, TODOS os testes passam e o ciclo de revisão retorna APPROVED.

Regras absolutas:

- **TODOS os bugs** listados no `bugs.md` devem ser corrigidos — não existe correção parcial
- **Cada bug corrigido** ganha teste(s) de regressão que simulam o cenário original
- **Nenhuma correção superficial** — se o sintoma some mas a causa permanece, o bug não foi corrigido
- **Nada finaliza com teste falhando** — `cargo test --workspace` com 100% de sucesso é pré-condição

## Localização dos Artefatos

| Artefato | Caminho |
|---|---|
| **Bugs (ENTRADA)** | `.cognup/specs/[nome-da-funcionalidade]/report/bugs.md` |
| Relatório de QA | `.cognup/specs/[nome-da-funcionalidade]/report/qa-report.md` |
| PRD | `.cognup/specs/[nome-da-funcionalidade]/prd.md` |
| Tech Spec | `.cognup/specs/[nome-da-funcionalidade]/techspec.md` |
| Tasks | `.cognup/specs/[nome-da-funcionalidade]/tasks.md` |
| Reviews das tasks | `.cognup/specs/[nome-da-funcionalidade]/tasks/reviews/` |
| Regras do projeto | `.claude/rules/architecture.md` |

## Quando Aplicar

- Quando o QA (`qa-best-practices` / agente qa-validator) reprovou a feature e documentou bugs em `report/bugs.md`
- Quando o usuário pede para corrigir bugs documentados de uma feature
- Ao corrigir regressões detectadas após merge
- Ao criar testes de regressão para bugs encontrados manualmente (documente-os primeiro no `bugs.md`)

## Pipeline de Bugfix — 8 Etapas

### Etapa 1: Análise de Contexto

- Ler `report/bugs.md` e extrair **TODOS** os bugs documentados (ID, severidade, categoria, passos de reprodução, evidência)
- Ler o `report/qa-report.md` para entender o contexto da reprovação
- Ler o PRD para entender os requisitos afetados por cada bug
- Ler a TechSpec para entender as decisões técnicas relevantes
- Ler `.claude/rules/architecture.md` para garantir conformidade nas correções
- Carregar [[rust-best-practices]] (sempre) e as skills de domínio conforme os componentes afetados: [[axum-best-practices]] para bugs em controllers/handlers, [[postgres-best-practices]] para bugs em repositórios/queries/migrations, [[nats-best-practices]] para bugs em publishers/subscribers

**NUNCA pule esta etapa** — entender o contexto completo é fundamental para corrigir a causa raiz e não o sintoma.

### Etapa 2: Planejamento das Correções

Para cada bug, gerar um resumo de planejamento **antes de codar**:

```
BUG ID: [ID do bug]
Severidade: [Crítica/Alta/Média/Baixa]
Componente Afetado: [camada + arquivo(s)]
Causa Raiz: [análise da causa raiz — o que de fato está errado e por quê]
Arquivos a Modificar: [lista de arquivos]
Estratégia de Correção: [descrição da abordagem, conforme architecture.md]
Testes de Regressão Planejados:
  - [Teste unitário]: [descrição]
  - [Teste de integração]: [descrição]
  - [Teste E2E]: [descrição]
```

**Ordem de correção por severidade: Crítica → Alta → Média → Baixa.**

Distinguir causa raiz de sintoma:

| Sintoma (não corrigir só isso) | Causa raiz (corrigir isto) |
|---|---|
| Endpoint retorna 500 | Variante de erro não mapeada no `HttpDomainError` ou erro não tratado no use case |
| Dado não persiste | Transaction sem `tx.commit()`, `RETURNING id` ausente, rollback silencioso no drop |
| Evento não chega ao consumer | Publish antes do commit, subject errado, falta de `Nats-Msg-Id` |
| Teste flakey | Race lógica sem transaction, dependência de ordem, estado compartilhado entre testes |
| Panic intermitente | `unwrap()`/`expect()` em caminho de erro não coberto, `.await` bloqueado por I/O síncrono |

### Etapa 3: Implementação das Correções

Para cada bug, seguir esta sequência:

1. **Localizar o código afetado** — ler e entender os arquivos envolvidos por completo, não apenas o trecho suspeito
2. **Reproduzir o problema** — de preferência com um teste que falha (red); no mínimo, reasoning explícito sobre o fluxo que causa o bug
3. **Implementar a correção na causa raiz** — conforme os padrões do projeto (camadas, construtores `new()`, error handling com `.context("ação")?` e variantes de `DomainError`, eventos pós-commit)
4. **Verificar compilação** — `cargo build --workspace` após cada correção
5. **Executar os testes existentes** — garantir que nenhum teste quebrou com a mudança

Regras durante a implementação:

- Respeitar a regra de dependência das camadas (`domain` → `application` → `infrastructure`); `domain` nunca importa `axum`/`sqlx`/`async-nats`
- Não introduzir código novo fora dos padrões de `architecture.md` para "resolver rápido"
- Se a correção exigir mudança arquitetural significativa, documentar a justificativa no relatório
- Se descobrir **novos bugs** durante a correção, documentá-los no `bugs.md` (mesmo formato do QA) e corrigi-los no mesmo ciclo

### Etapa 4: Testes de Regressão

Para cada bug corrigido, criar testes que:

- **Simulem o cenário original do bug** — o teste DEVE falhar se a correção for revertida
- **Validem o comportamento correto** — o teste passa com a correção aplicada
- **Cubram edge cases relacionados** — variações do mesmo problema (`None`, vazio, limites, concorrência)

> Referência: [[rust-testing-best-practices]] (`#[cfg(test)] mod tests`, `#[tokio::test]`, mockall) e [[testcontainers-rust]] (serviços reais).

| Tipo | Quando Usar | Ferramentas |
|------|-------------|-------------|
| Teste unitário | Bug em lógica isolada de função/método/use case | `#[cfg(test)] mod tests` no próprio arquivo + `#[tokio::test]` + mockall |
| Teste de integração | Bug na comunicação entre módulos ou com infra (repository, publisher/subscriber) | testcontainers (PostgreSQL/NATS reais) em `tests/` |
| Teste E2E | Bug visível no fluxo completo da API | API Axum em porta livre + reqwest/curl contra a API |

Nomeie os testes de forma rastreável ao bug (ex.: `create_order_duplicate_email_returns_conflict_bug03` ou `#[tokio::test] async fn bug_03_duplicate_email_returns_409()`).

### Etapa 5: Validação Completa

Execute na raiz do workspace e **não prossiga com falhas**:

```bash
cargo fmt --all -- --check                                  # nenhum diff
cargo clippy --workspace --all-targets -- -D warnings       # warning é falha
cargo build --workspace                                     # sem erros
cargo test --workspace                                      # 100% de sucesso
cargo audit                                                 # se configurado no projeto
```

**A tarefa NÃO está completa se qualquer verificação falhar.** Falhou? Corrija a causa raiz e rode novamente.

### Etapa 6: Atualização do bugs.md

Após corrigir cada bug, atualizar `report/bugs.md` adicionando ao final da entrada do bug:

```markdown
- **Status:** Corrigido
- **Correção aplicada:** [descrição breve da correção na causa raiz]
- **Testes de regressão:** [lista dos testes criados, com caminho do arquivo]
```

Regras:

- **Nunca apagar** a documentação original do bug — apenas anexar o status da correção (rastreabilidade)
- Todo bug do arquivo deve terminar com **Status: Corrigido** antes de encerrar o ciclo
- Novos bugs descobertos durante a correção entram no mesmo arquivo, no formato padrão do QA

### Etapa 7: Relatório Final

Gerar um resumo final ao término das correções:

```markdown
# Relatório de Bugfix - [Nome da Funcionalidade]

## Resumo
- Total de Bugs: [X]
- Bugs Corrigidos: [Y]
- Novos Bugs Descobertos e Corrigidos: [W]
- Testes de Regressão Criados: [Z]

## Detalhes por Bug
| ID | Severidade | Status | Causa Raiz | Correção | Testes Criados |
|----|------------|--------|------------|----------|----------------|
| BUG-01 | Alta | Corrigido | [causa] | [descrição] | [lista] |

## Validação
- `cargo fmt --all -- --check`: SEM DIFFS
- `cargo clippy --workspace --all-targets -- -D warnings`: LIMPO
- `cargo build --workspace`: SEM ERROS
- `cargo test --workspace`: TODOS PASSANDO
- `cargo audit`: [OK / N/A]
```

### Etapa 8: Commit, Revisão e Re-QA

1. **Commit por bugfix** no padrão [[conventional-commit]]:
   - `fix(scope): description` — o scope é o contexto de domínio (ex.: `order`, `user`)
   - Exemplo: `fix(order): map duplicate email error to 409 conflict`
   - Stage apenas os arquivos do bugfix; testes de regressão entram no mesmo commit da correção
2. **Revisão**: execute o agente **@task-reviewer** (processo em `task-reviewer-best-practices`):
   - Status diferente de **APPROVED** (APPROVED WITH OBSERVATIONS ou CHANGES REQUESTED) → ajuste os problemas, commite (`fix`/`refactor`) e execute o @task-reviewer novamente
   - Repita até **APPROVED** — o arquivo `tasks/reviews/[num]_task_review.md` é atualizado a cada re-revisão
3. **Re-QA**: com todos os bugs corrigidos e review APPROVED, execute o agente **@qa-validator** (processo em `qa-best-practices`):
   - **APROVADO** → ciclo encerrado
   - **REPROVADO** com novos bugs em `report/bugs.md` → reinicie este pipeline (Etapa 1)

O ciclo completo é: `qa-validator (REPROVADO) → bugfix → task-reviewer (APPROVED) → qa-validator` — repetir até **APROVADO**.

## Diretrizes Operacionais

1. **Todos os bugs, sempre** — não existe encerrar o ciclo com bug pendente no `bugs.md`
2. **Ler antes de modificar** — sempre leia o código-fonte completo dos arquivos afetados
3. **Causa raiz documentada** — cada correção registra a causa raiz no planejamento e no relatório
4. **Regressão obrigatória** — bug corrigido sem teste de regressão = bug não corrigido
5. **Ordem por severidade** — Crítica → Alta → Média → Baixa
6. **Sem gambiarras** — correções seguem os mesmos padrões de qualidade do código novo (`architecture.md` + skills)
7. **Validação completa antes do commit** — `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo build`, `cargo test`
8. **Commits no padrão** — `fix(scope): description` via [[conventional-commit]], um commit por bugfix
9. **Rastreabilidade no bugs.md** — status anexado sem apagar a documentação original
10. **Ciclo até aprovação** — task-reviewer até APPROVED, depois qa-validator até APROVADO
11. **Novos bugs entram no ciclo** — descobriu, documentou no `bugs.md`, corrigiu
12. **Use o Context7 MCP apenas quando** as skills e regras locais não cobrirem a dúvida (API de crate externa, comportamento de versão específica)

## Anti-Patterns de Bugfix (Evitar)

| Anti-Pattern | Correto |
|---|---|
| Silenciar o sintoma (`let _ = repo.create(...).await;`) | Corrigir a causa raiz do erro e propagar com `?` |
| `unwrap_or_default()` para esconder erro | Tratar o `Err`/`None` na origem e retornar `DomainError` adequado |
| Corrigir sem teste de regressão | Todo bug ganha teste que falha se a correção for revertida |
| Teste de regressão que passa mesmo sem a correção | Validar o teste contra o código bugado (red → green) |
| Corrigir só os bugs "importantes" | TODOS os bugs do bugs.md, em ordem de severidade |
| Marcar "Corrigido" sem rodar a suite completa | `cargo test --workspace` a 100% antes de atualizar o bugs.md |
| Apagar a descrição original do bug | Anexar status mantendo a rastreabilidade |
| Um commit gigante com todas as correções | Um commit `fix(scope)` por bugfix |
| Ignorar bug novo descoberto no caminho | Documentar no bugs.md e corrigir no mesmo ciclo |
| Finalizar sem review | task-reviewer até APPROVED, depois qa-validator até APROVADO |
| Correção que viola a regra de dependência | Conformidade com `architecture.md` mesmo sob pressão |
| `unwrap()`/`expect()`/`panic!` para "resolver" caminho de erro | Tratar o `Result`/`Option` e mapear para `DomainError` |
| Publicar evento dentro da transaction | `tx.commit()` primeiro, `event.publish(...)` depois |

## Cheat Sheet — Comandos do Bugfix

```bash
# Ler os bugs e o contexto
cat .cognup/specs/[feature]/report/bugs.md
cat .cognup/specs/[feature]/report/qa-report.md

# Reproduzir com teste focado (red)
cargo test -p billing --lib create_usecase bug_03

# Validação completa (gate antes do commit)
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo build --workspace
cargo test --workspace

# Cobertura dos crates corrigidos (cargo-llvm-cov)
cargo llvm-cov -p billing --summary-only

# Commit por bugfix (Conventional Commits)
git add <files>
git commit -m "fix(order): map duplicate email error to 409 conflict"

# Detectar regressão em endpoint corrigido (E2E rápido)
curl -s -w "\n%{http_code}" -X POST http://localhost:8080/v1/orders \
  -H "Content-Type: application/json" \
  -d '{"userId":"...","totalCents":100}'
```

## Idioma

Planejamento, relatório e atualizações do `bugs.md` em **Português (Brasil)**. Código, nomes de testes e mensagens de commit em **inglês**.

## Manutenção de Memória

Ao longo dos bugfixes, registre padrões para acelerar correções futuras:

- Causas raiz recorrentes (ex.: variante de erro não mapeada no `HttpDomainError`, publish dentro de transaction, `unwrap()` em caminho de erro)
- Componentes mais propensos a bugs e seus modos de falha
- Lacunas sistemáticas de teste que deixam bugs passarem (edge cases não cobertos, mutantes sobreviventes no cargo-mutants)
- Correções que exigiram mudança arquitetural e suas justificativas
- Testes flakey conhecidos e suas causas
