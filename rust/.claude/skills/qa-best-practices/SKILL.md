---
name: qa-best-practices
description: Aplica melhores práticas de QA (Quality Assurance) para validação completa de features backend em Rust. Cobre análise estática (cargo fmt --check, cargo clippy com -D warnings, cargo audit, cargo build), testes unitários e de integração com cobertura via cargo-llvm-cov (data races prevenidas em compile time pelo ownership), mutation testing com cargo-mutants, testes E2E de API com curl/HTTP, verificação de contrato OpenAPI gerado pelo utoipa (/api-docs/openapi.json e /swagger-ui), auditoria de qualidade de código (error handling com Result/thiserror/anyhow, concorrência com Tokio, traits, arquitetura), verificação de infraestrutura (PostgreSQL, NATS JetStream, health checks) e geração de relatórios de QA com evidências em `.cognup/specs/[feature]/report/`. Use sempre que validar uma feature concluída, gerar relatório de QA, executar pipeline de qualidade, revisar implementação contra PRD/TechSpec, ou aplicar tolerância zero a bugs antes de release.
---

# QA Best Practices — Validação Completa de Features Backend Rust

Skill que aplica o processo sistemático de QA backend para garantir qualidade end-to-end. Combina análise estática, testes automatizados, mutation testing, testes E2E, auditoria de código e verificação de infraestrutura em um pipeline com **tolerância zero a bugs**.

Combina com [[rust-best-practices]] (padrões de código) e [[rust-testing-best-practices]] (estratégias de teste), e — conforme o escopo da feature — com [[axum-best-practices]] (HTTP), [[postgres-best-practices]] (banco), [[nats-best-practices]] (mensageria) e [[testcontainers-rust]] (integração). A arquitetura de referência é `.claude/rules/architecture.md` (monorepo com Axum 0.8, PostgreSQL 18, NATS JetStream e composition root manual com `Arc<dyn Trait>`).

## Princípio Fundamental

> **TOLERÂNCIA ZERO A BUGS.** Uma feature só recebe status **APROVADO** quando ZERO bugs são encontrados — independente da severidade (Crítica, Alta, Média ou Baixa). Qualquer bug encontrado, por menor que seja, resulta automaticamente em **REPROVADO**.

Não existe:
- Aprovação condicional
- Aprovação com ressalvas
- Aprovação com bugs pendentes de correção
- Distinção entre bugs "bloqueantes" e "não bloqueantes"

Severidade é usada apenas para **priorização de correção**, nunca para decisão de aprovação.

## Quando Aplicar

- Ao validar uma feature backend Rust concluída antes de release
- Ao revisar implementação contra PRD, TechSpec ou Tasks
- Ao executar pipeline completo de qualidade (estática + testes + E2E)
- Ao gerar relatório de QA com evidências auditáveis
- Ao validar contrato de API contra a documentação OpenAPI gerada pelo utoipa
- Ao verificar saúde de componentes de infraestrutura (DB, broker, health endpoints)
- Antes de bump de versão (SemVer) e atualização de changelog

## Estrutura de Evidências

**REGRA CRÍTICA:** Todos os artefatos de QA DEVEM ser salvos em uma estrutura padronizada:

```
.cognup/specs/[feature]/report/
├── qa-report.md                          # Relatório final consolidado
├── bugs.md                               # Documentação detalhada de bugs
└── evidence/
    ├── static-analysis.md                # Etapa 3
    ├── test-results.md                   # Etapa 4
    ├── coverage-summary.md               # Etapa 4
    ├── mutation-testing.md               # Etapa 5
    ├── api-e2e-results.md                # Etapa 6
    ├── contract-verification.md          # Etapa 7
    ├── code-quality-audit.md             # Etapa 8
    └── infrastructure.md                 # Etapa 9
```

**OBRIGATORIEDADE DE EVIDÊNCIAS:**
- Todo teste — PASSOU ou FALHOU — DEVE ter evidência documentada
- Nenhum requisito pode ser marcado como verificado sem evidência correspondente
- Todo bug DEVE ter request/response, output de comando ou trecho de código como evidência
- Teste sem evidência = teste NÃO EXECUTADO

## Pipeline de QA — 13 Etapas

### Etapa 1: Análise da Documentação (Obrigatória)

Antes de qualquer teste:

1. **Ler o PRD** e extrair TODOS os requisitos funcionais numerados em checklist
2. **Ler a TechSpec** e anotar decisões técnicas verificáveis (endpoints, modelos, error codes, regras de negócio)
3. **Ler o arquivo de Tasks** e confirmar status de conclusão
4. **Ler arquitetura** (`.claude/rules/architecture.md`) para entender convenções
5. **Consultar skills** [[rust-best-practices]] e [[rust-testing-best-practices]] para critérios de qualidade
6. **Construir matriz de verificação** mapeando cada requisito → cenário de teste

Para cada requisito, identifique:
- Endpoint(s) envolvidos (método, path, schemas)
- Status codes HTTP esperados (sucesso e erro)
- Operações de banco que devem ser verificadas
- Operações de mensageria/eventos
- Regras de negócio e validações
- Padrões de error handling esperados

**Nunca pule esta etapa** — entender requisitos é a base da QA.

### Etapa 2: Preparação do Ambiente (Obrigatória)

```bash
# Criar diretórios de evidências
mkdir -p .cognup/specs/[feature]/report/evidence/

# Verificar toolchain
rustc --version && cargo --version
docker info

# Verificar compilação
cargo build --workspace

# Verificar dependências (lockfile íntegro)
cargo fetch && cargo build --workspace --locked

# Verificar variáveis de ambiente
cat .env  # ou env | grep APP_
```

**Protocolo de Startup da Aplicação:**

1. Verificar se já está rodando: `curl -s http://localhost:8080/health`
2. Se não, iniciar em background: `cargo run -p <app> --bin api &` (subir PostgreSQL/NATS via `docker-compose up -d` antes, se necessário)
3. Aguardar prontidão (poll no `/health` com 2-3s incrementais)
4. Verificar health endpoint retorna sucesso
5. Registrar PID para cleanup posterior

Se a aplicação falhar ao iniciar: **REPROVADO imediato**.

### Etapa 3: Análise Estática de Código (Obrigatória)

Execute todas as ferramentas e registre resultados em `evidence/static-analysis.md`:

| Ferramenta | Comando | O que Verifica |
|------------|---------|----------------|
| **Build** | `cargo build --workspace` | Erros de compilação, type safety |
| **Clippy** | `cargo clippy --workspace --all-targets -- -D warnings` | Construções suspeitas, bugs potenciais, estilo, lints |
| **Format** | `cargo fmt --all -- --check` | Conformidade de formatação (rustfmt) |
| **Audit** | `cargo audit` | Vulnerabilidades conhecidas nas dependências |
| **Lockfile** | `cargo build --workspace --locked` | `Cargo.lock` sincronizado com os `Cargo.toml` |
| **Docs** | `cargo doc --workspace --no-deps` | Doc comments e links de documentação válidos |
| **Dependências** | `cargo tree --workspace --duplicates` | Versões duplicadas/conflitantes de crates |

Para cada ferramenta:
1. Executar o comando
2. Capturar a saída **completa**
3. Marcar como **PASSOU** (zero issues) ou **FALHOU**
4. Se FALHOU: classificar cada problema e documentar

**Verificações adicionais:**
- Sem `TODO`/`FIXME` não resolvidos relacionados à feature
- Itens públicos (`pub`) possuem doc comments (`///`)
- Error handling usa thiserror nos erros de domínio e anyhow com `.context("ação")` na infraestrutura
- Sem uso de `panic!`, `unwrap()` ou `expect()` em código de produção (apenas em testes ou no bootstrap do `main`)

### Etapa 4: Testes Unitários e Integração (Obrigatória)

```bash
# Suite completa: unitários (#[cfg(test)]) + integração (tests/)
cargo test --workspace

# Cobertura por função/arquivo (cargo-llvm-cov)
cargo llvm-cov --workspace

# Cobertura total resumida
cargo llvm-cov --workspace --summary-only

# HTML para análise visual
cargo llvm-cov --workspace --html

# Testes de integração de um app específico (crate de teste separado em tests/)
cargo test -p user --test user_integration_test
```

**Análise obrigatória:**
1. Parse a saída — identificar passados (`ok`), falhos (`FAILED`), ignorados (`ignored`)
2. **ZERO falhas** é obrigatório
3. Cobertura por crate registrada
4. **Cobertura mínima: 80%** (threshold do projeto)
5. Data races: prevenidas em **compile time** pelo ownership/borrow checker (`Send`/`Sync`) — nenhum passo extra de runtime é necessário; sinalize qualquer bloco `unsafe` no código da feature

Para cada teste que falhou:
- Nome do teste, crate, mensagem de erro
- Classificar como bug e documentar

Para análise de cobertura:
- Percentuais por crate/módulo
- Módulos com cobertura baixa/zero sinalizados
- Código novo sem cobertura = bug

Referência: [[rust-testing-best-practices]] para padrões `#[cfg(test)]`, `#[tokio::test]`, mockall, testcontainers.

Salve em `evidence/test-results.md` e `evidence/coverage-summary.md`.

### Etapa 5: Mutation Testing — cargo-mutants (Obrigatória)

Mutation testing modifica código-fonte em pequenas formas (mutações). Testes que detectam = "matam" o mutante. **Mutation score alto = suite robusta**.

**Instalação:**

```bash
cargo install cargo-mutants
```

**Execução:**

```bash
# Workspace completo
cargo mutants --workspace

# Direcionado para os arquivos da feature
cargo mutants -p user -f src/application/usecase/user/create_usecase.rs

# Com exclusões (falsos positivos conhecidos)
cargo mutants --workspace --exclude-re 'impl Debug' --exclude 'src/bin/*'
```

**Análise:**
1. Cada mutante: **caught** (morto = bom) ou **missed** (sobreviveu = lacuna nos testes); mutantes `unviable` (não compilam) são descartados do cálculo
2. Calcular score: `caught / (caught + missed)` (1.0 = todos mortos)
3. **Threshold mínimo: 80%**
4. Mutantes sobreviventes documentados com arquivo, linha e mutação aplicada (ver `mutants.out/missed.txt` e `mutants.out/outcomes.json`)
5. Classificar cada sobrevivente:
   - **Lacuna real**: precisa de novo teste → bug severidade Média
   - **Falso positivo**: early exit, otimização → candidato a exclusão via `--exclude-re`

**Interpretação:**
- `mutation score >= 0.80`: aceitável
- `mutation score < 0.80`: REPROVADO — adicionar testes

Salve em `evidence/mutation-testing.md`.

### Etapa 6: Testes E2E de API (Obrigatória)

Para cada endpoint do contrato OpenAPI e/ou PRD, teste exaustivamente com `curl`:

**Padrão de comando:**

```bash
curl -s -w "\n%{http_code}" -X [METHOD] http://localhost:8080/[path] \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${TOKEN}" \
  -d '[request body]'
```

**Categorias obrigatórias:**

#### 6.1 Happy Path
- Request válido com headers e body corretos
- Status code corresponde ao esperado (200, 201, 204)
- Body bate com schema OpenAPI
- Headers corretos (`Content-Type: application/json`)

#### 6.2 Casos de Erro
- **400 Bad Request**: JSON malformado, campos obrigatórios ausentes, tipos inválidos
- **401 Unauthorized**: sem token
- **403 Forbidden**: permissões insuficientes
- **404 Not Found**: recurso inexistente
- **409 Conflict**: criação duplicada
- **422 Unprocessable Entity**: dados semanticamente inválidos
- **500**: API nunca retorna stack traces ou detalhes internos

#### 6.3 Edge Cases
- Body vazio
- Valores extremamente longos
- Caracteres especiais (unicode, SQL injection, XSS)
- Limites numéricos (0, negativos, MAX_INT)
- Requisições concorrentes ao mesmo endpoint
- Path traversal (`../../`)
- Headers inválidos ou ausentes

Para cada teste:
1. Executar request
2. Capturar resposta **completa** (status, headers, body)
3. Verificar contra comportamento esperado
4. **PASSOU** ou **FALHOU**
5. Se FALHOU: documentar request/response completos

Salve em `evidence/api-e2e-results.md`.

### Etapa 7: Verificação de Contrato API (Obrigatória)

Compare o contrato OpenAPI gerado pelo utoipa (`/api-docs/openapi.json`) contra comportamento real:

1. Baixar o contrato de `http://localhost:8080/api-docs/openapi.json` e extrair todos os endpoints
2. Para cada endpoint documentado verificar:
   - Existe e responde (não 404)
   - Método HTTP correto
   - Schema de request bate (campos obrigatórios, tipos)
   - Schema de response bate (nomes, tipos, estrutura)
   - Status codes documentados são realmente retornados
   - `Content-Type` corresponde
3. Detectar endpoints **não documentados** (rota registrada no Router mas sem `#[utoipa::path]`)
4. Detectar endpoints **fantasma** (no OpenAPI mas não implementados)

**Obter e comparar:**

```bash
# O contrato é gerado em compile time pelo utoipa — não há passo de CLI
curl -s http://localhost:8080/api-docs/openapi.json | jq '.paths | keys'

# Swagger UI servido pela API
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/swagger-ui/
```

Marcar cada endpoint: **MATCH** ou **MISMATCH**.

Salve em `evidence/contract-verification.md`.

### Etapa 8: Auditoria de Qualidade de Código Rust (Obrigatória)

Consulte [[rust-best-practices]] para critérios completos. Verificar e documentar em `evidence/code-quality-audit.md`:

#### 8.1 Error Handling

| Critério | Verificar | Evidência |
|----------|-----------|-----------|
| Erros sempre tratados | Sem `unwrap()`/`expect()` em produção; sem `let _ =` descartando `Result` sem justificativa | Listar ocorrências |
| Contexto na propagação | Uso de `.context("ação")` (anyhow) em repositórios/adapters/publishers | Funções sem contexto |
| Erros de domínio tipados | Enum `DomainError` com thiserror (`#[derive(Debug, Error)]`) por app | Apps sem erros tipados |
| Pattern matching | `match`/`matches!` sobre variantes ao invés de comparar strings de erro | Locais incorretos |
| Sem `panic!` em produção | `panic!` apenas em testes ou no bootstrap do `main` | Listar usos suspeitos |

#### 8.2 Concorrência

| Critério | Verificar | Evidência |
|----------|-----------|-----------|
| Lifecycle de tasks | Toda `tokio::spawn` tem caminho de encerramento (shutdown signal, `JoinHandle`) | Tasks sem encerramento |
| Sincronização | `Arc<Mutex<T>>`/`RwLock` para estado compartilhado; `tokio::sync::Mutex` quando o lock atravessa `.await` | Estado desprotegido / guard através de await |
| Data races | Prevenidas em compile time (ownership + `Send`/`Sync`); sem `unsafe` injustificado | Blocos `unsafe` |
| Bloqueio do runtime | Operações bloqueantes via `tokio::task::spawn_blocking` | Chamadas bloqueantes em contexto async |
| Sem leak | Tasks terminam no shutdown (`tokio::select!` com sinal, drain) | Tasks órfãs |

#### 8.3 Traits e Design

| Critério | Verificar | Evidência |
|----------|-----------|-----------|
| Traits pequenas | Focadas, poucos métodos (ISP) | Traits infladas |
| Depender de traits, construir concreto | Camadas dependem de `Arc<dyn Trait>`; construtores retornam `Self` | Violações |
| Naming | snake_case para funções/módulos, UpperCamelCase para tipos/traits, SCREAMING_SNAKE_CASE para consts; sem stuttering | Violações de C-CASE / stuttering |
| Function design | Funções pequenas, ≤ 3-4 parâmetros | Funções gigantes |
| Traits async | `#[async_trait]` com bound `Send + Sync` nas traits de domínio (necessário para `Arc<dyn Trait>`) | Violações |

#### 8.4 Qualidade de Testes

| Critério | Verificar | Evidência |
|----------|-----------|-----------|
| Unitários junto da impl | `#[cfg(test)] mod tests` no mesmo arquivo da implementação | Testes fora do padrão |
| Asserts nativos | `assert!`/`assert_eq!`/`matches!` + `#[tokio::test]` para async | Testes verbosos/incorretos |
| Mocks apenas externos | mockall (`MockUserRepository`) apenas para dependências externas (não over-mocking) | Over-mocking |
| Cenários de erro | Não apenas happy path | Funções sem testes de erro |
| Independência/paralelismo | `cargo test` roda em paralelo por padrão — sem dependência de ordem nem estado compartilhado | Testes acoplados |
| Fixtures isoladas | Containers testcontainers com cleanup automático no drop | State leak entre testes |

#### 8.5 Conformidade Arquitetural

Consultar `.claude/rules/architecture.md` do projeto.

| Critério | Verificar | Evidência |
|----------|-----------|-----------|
| Regra de dependência | `domain` não importa axum/sqlx/async-nats; `application` importa apenas `crate::domain`; `pkg` nunca importa apps | Violações de import |
| Construtores | `X::new(...) -> Self`; composition root embrulha em `Arc<dyn Trait>` | Construtores fora do padrão |
| Naming de arquivos | snake_case (`user_repository.rs`), módulos estilo `mod.rs` | Violações |
| Composition root | Construção bottom-up no `main` de cada binário (`src/bin/api.rs`, `src/bin/consumer.rs`) | Dependências construídas fora do root |
| Anotações OpenAPI | Handlers com `#[utoipa::path]` completos e DTOs com `#[derive(ToSchema)]` | Handlers sem docs |

### Etapa 9: Verificação de Infraestrutura (Obrigatória)

#### 9.1 PostgreSQL

> Referência: [[postgres-best-practices]].

- Health endpoint reporta o banco saudável (ex.: `postgres: true`)
- Se feature envolve persistência:
  - Criar recurso via POST → buscar via GET → comparar dados
  - Integridade: campos batem com o enviado
  - Migrations aplicadas (`.up.sql`/`.down.sql` em `migrations/<app>/`, criadas com `sqlx migrate add -r`); índices e constraints conforme spec (`\d "tabela"` via psql)
  - Operações multi-step atômicas (verificar rollback em falha parcial via `pool.begin()` + métodos `*_with_tx`; sem `tx.commit()`, o rollback é automático no drop)

#### 9.2 NATS JetStream

> Referência: [[nats-best-practices]].

- Health endpoint reporta a mensageria saudável (ex.: `nats: true`)
- Se feature envolve mensageria:
  - Disparar evento via API
  - Verificar publicação (`nats stream view <STREAM>` ou logs)
  - Streams/consumers duráveis criados conforme spec (subjects `events.<domínio>.<ação>`)
  - Deduplicação por `Nats-Msg-Id` e ack/nak/term corretos no consumer (`msg.ack()`, `AckKind::Nak`, `AckKind::Term`)
  - Evento publicado somente APÓS o commit no banco

#### 9.3 Health Check

- `GET /health` retorna 200
- Resposta inclui o status dos componentes: `server`, `postgres`, `nats`
- `requestId` e `timestamp` presentes e formatados corretamente

#### 9.4 Graceful Shutdown

- Aplicação lida com `SIGTERM`/`SIGINT` gracefully (`with_graceful_shutdown(shutdown_signal())`)
- Requisições em andamento completam antes do shutdown
- Conexões fechadas na ordem: HTTP → NATS (drain) → PostgreSQL (`pool.close()`)

Salve em `evidence/infrastructure.md`.

### Etapa 10: Documentação de Bugs

Para cada bug, em `report/bugs.md`:

```markdown
## BUG-[ID]: [Descrição curta]

- **Severidade**: Crítica / Alta / Média / Baixa
- **Requisito**: [ID do PRD]
- **Categoria**: Análise Estática / Falha de Teste / API E2E / Contrato / Qualidade / Infraestrutura
- **Passos para Reproduzir**:
  1. [passo]
  2. [passo]
- **Resultado Esperado**: [o que deveria acontecer]
- **Resultado Atual**: [o que aconteceu]
- **Evidência**:
  ```
  [request/response, output, mensagem de erro]
  ```
- **Ambiente**: localhost:8080
```

**Critérios de Severidade** (para priorização de correção, NÃO para aprovação):

- **Crítica**: crash, perda de dados, vulnerabilidade de segurança, `panic!` em produção, condição de corrida lógica (check-then-act sem transaction)
- **Alta**: endpoint principal quebrado, persistência incorreta, bypass de auth, suite de testes falhando
- **Média**: status code errado, error handling ausente, validação incompleta, cobertura < threshold
- **Baixa**: problemas menores de qualidade, doc comments ausentes, formatação inconsistente

**REGRA DE APROVAÇÃO:** Qualquer bug — **incluindo Baixa** — torna o QA REPROVADO.

### Etapa 11: Geração do Relatório de QA (Obrigatória)

Crie `report/qa-report.md` com este formato:

```markdown
# Relatório de QA Backend - [Feature]

## Resumo
- **Data**: [AAAA-MM-DD]
- **QA Engineer**: AI QA Validator
- **Status**: APROVADO / REPROVADO
- **Total de Requisitos**: [X]
- **Requisitos Atendidos**: [Y]
- **Requisitos Reprovados**: [Z]
- **Bugs Encontrados**: [W]
- **Total de Evidências**: [N]

## Requisitos Verificados
| ID | Requisito | Status | Evidência |

## Análise Estática
| Ferramenta | Comando | Status | Detalhes |
| Build | `cargo build --workspace` | OK / FALHOU | ... |
| Clippy | `cargo clippy --workspace --all-targets -- -D warnings` | OK / FALHOU | ... |
| Format | `cargo fmt --all -- --check` | OK / FALHOU | ... |
| Audit | `cargo audit` | OK / FALHOU | ... |

## Testes
| Crate | Total | Passed | Failed | Ignored |
- **Cobertura Total**: [X]%
- **Threshold Mínimo**: 80%
- **Data races**: prevenidas em compile time (ownership/borrow checker)

## Mutation Testing
| Métrica | Valor |
| Total de Mutantes | [X] |
| Mortos (caught) | [Y] |
| Sobreviventes (missed) | [Z] |
| Mutation Score | [Y/X] ([%]) |

## E2E
| Endpoint | Método | Cenário | Status Esperado | Recebido | Status |

## Contrato (OpenAPI/utoipa)
| Endpoint | Método | Documentado | Implementado | Match | Status |

## Auditoria de Código
### Error Handling, Concorrência, Traits, Testes, Arquitetura
(uma tabela por seção)

## Infraestrutura
| Componente | Status | Detalhes |

## Bugs
| ID | Descrição | Severidade | Categoria | Requisito | Evidência |

## Inventário de Evidências
| Arquivo | Tipo | Relacionado a |

## Conclusão
[Avaliação final com raciocínio]

## Próximos Passos
[Ações se REPROVADO, ou confirmação de prontidão se APROVADO]
```

### Etapa 12: Changelog (Pós-Aprovação)

**Só executar se QA = APROVADO.**

Seguir [Keep a Changelog](https://keepachangelog.com/) e [Conventional Commits](https://www.conventionalcommits.org/).

Categorias:
- **Added**: novas features, endpoints, entidades
- **Changed**: modificações em funcionalidades, melhorias de performance
- **Deprecated**: marcadas para remoção
- **Removed**: removidas
- **Fixed**: correções de bugs
- **Security**: patches de vulnerabilidade

Adicionar em `[Unreleased]` no `CHANGELOG.md`:

```markdown
## [Unreleased]

### Added
- [Feature] — QA validated on [YYYY-MM-DD]
  - [Endpoint/capacidade 1]
  - [Endpoint/capacidade 2]
```

Regras:
- NUNCA sobrescrever entradas existentes
- Entradas em **inglês** (padrão da indústria)
- Uma linha por feature, sub-itens opcionais
- Referenciar nome da feature do PRD

### Etapa 13: Versionamento Semântico (Pós-Aprovação)

**Só executar se Etapa 12 completou com sucesso.**

Seguir [SemVer](https://semver.org/):

| Bump | Quando Usar |
|------|-------------|
| **MAJOR** | Breaking changes (endpoints removidos, contratos alterados) |
| **MINOR** | Novas features backward-compatible |
| **PATCH** | Bug fixes, performance, refactoring interno |

Processo:

```bash
# 1. Verificar versão atual
git describe --tags --abbrev=0

# 2. Calcular nova versão
# 3. Atualizar arquivos de versão (`version` nos Cargo.toml do workspace, Makefile)
# 4. Mover [Unreleased] para [X.Y.Z] - YYYY-MM-DD no CHANGELOG.md
# 5. Commit
git add -A
git commit -m "chore(release): vX.Y.Z

[Resumo breve da feature validada]"

# 6. Tag
git tag -a vX.Y.Z -m "Release vX.Y.Z"
```

Regras:
- NUNCA fazer bump se QA = REPROVADO
- SEMPRE usar prefixo `v` na tag
- SEMPRE incluir mensagem descritiva

## Critérios de Aprovação — Checklist Final

**APROVADO** requer **TODOS**:

- [ ] 100% dos requisitos funcionais do PRD verificados e passando
- [ ] **ZERO bugs** (qualquer severidade)
- [ ] `cargo build --workspace` sem erros
- [ ] `cargo clippy --workspace --all-targets -- -D warnings` limpo
- [ ] `cargo fmt --all -- --check` sem diffs
- [ ] `cargo audit` sem vulnerabilidades (se configurado no projeto)
- [ ] Suite de testes: ZERO falhas (`cargo test --workspace`)
- [ ] Cobertura ≥ 80% (`cargo llvm-cov --workspace`)
- [ ] Mutation score ≥ 80% (`cargo mutants`)
- [ ] Sem `unsafe` injustificado nem `unwrap()`/`expect()`/`panic!` em código de produção
- [ ] Contrato OpenAPI (utoipa) ↔ implementação 100% match
- [ ] Auditoria de código sem violações
- [ ] PostgreSQL, NATS JetStream, Health endpoints saudáveis
- [ ] Graceful shutdown funcional
- [ ] Todas as evidências documentadas em `report/evidence/`
- [ ] Relatório `qa-report.md` gerado

**REPROVADO** se QUALQUER um dos itens acima falhar.

## Diretrizes Operacionais

1. **Compilação antes de testes** — se não compila, nada mais importa
2. **Capturar request/response de TODOS os bugs** — sem exceção
3. **Verificar logs** para panics, erros, warnings inesperados
4. **Testar TODOS os cenários de erro** — não apenas happy paths
5. **Tolerância zero** — bug Baixa reprova igual a Crítica
6. **Ser sistemático** — seguir checklist sequencialmente
7. **Documentar tudo** — sem evidência = não testado
8. **Warnings do clippy são falha** — sempre `-D warnings`; data races já são prevenidas em compile time pelo ownership
9. **Validar graceful shutdown** se feature toca lifecycle
10. **Cleanup pós-testes** — parar processos em background
11. **Mutation testing valida testes** — sobreviventes = lacunas
12. **Validar contra skills** [[rust-best-practices]] e [[rust-testing-best-practices]]
13. **Validar arquitetura** contra `.claude/rules/architecture.md`
14. **Teste sem evidência = NÃO EXECUTADO**

## Ferramentas Essenciais — Cheat Sheet

```bash
# Análise estática
cargo build --workspace
cargo clippy --workspace --all-targets -- -D warnings
cargo fmt --all -- --check
cargo audit
cargo build --workspace --locked
cargo doc --workspace --no-deps

# Testes
cargo test --workspace
cargo llvm-cov --workspace
cargo llvm-cov --workspace --html
cargo test -p <app> --test <nome_do_teste_de_integracao>

# Mutation testing
cargo install cargo-mutants
cargo mutants --workspace

# Teste focado
cargo test -p user create_usecase

# Benchmark
cargo bench

# Profiling (cargo-flamegraph)
cargo flamegraph --bin api

# Contrato OpenAPI (gerado em compile time pelo utoipa)
curl -s http://localhost:8080/api-docs/openapi.json | jq '.paths | keys'
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/swagger-ui/

# Health check
curl -s http://localhost:8080/health | jq .

# E2E pattern
curl -s -w "\n%{http_code}" -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{"name":"test","amount":100}'
```

## Anti-Patterns de QA (Evitar)

| Anti-Pattern | Correto |
|---|---|
| "Bug de severidade baixa, aprovo" | Tolerância zero — REPROVADO |
| Pular evidência de teste que passou | Toda evidência é obrigatória |
| Testar só happy path | Testar todos cenários de erro + edge cases |
| Cobertura como única métrica | Cobertura + mutation score + auditoria |
| Tolerar warnings do clippy | `-D warnings` — warning é falha |
| Confiar no contrato OpenAPI sem verificar | Comparar contrato utoipa vs comportamento real |
| Não documentar request/response do bug | Toda evidência reproduzível |
| Mock testando o mock | Mocks apenas para dependências externas |
| Aprovar com TODO/FIXME pendentes | Limpar TODOs da feature antes de aprovar |
| Bump de versão antes de aprovar | Etapas 12-13 SÓ após APROVADO |
| Fazer cleanup só "se der tempo" | Cleanup é parte do processo |

## Idioma do Relatório

Escrever relatório de QA e documentação de bugs em **Português (Brasil)**. Termos técnicos (tipos Rust, métodos HTTP, status codes, nomes de ferramentas) podem permanecer em inglês.

Changelog em **inglês** (padrão da indústria).
