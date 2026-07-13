---
name: qa-best-practices
description: Aplica melhores práticas de QA (Quality Assurance) para validação completa de features backend em Node.js (TypeScript). Cobre análise estática (prettier --check, eslint, tsc --noEmit, pnpm build), testes unitários e de integração com vitest e cobertura v8, mutation testing com Stryker, testes E2E de API com curl/HTTP, verificação de contrato OpenAPI gerado via swagger-jsdoc, auditoria de qualidade de código (error handling com cause, promises não aguardadas, tipagem any, arquitetura), verificação de infraestrutura (PostgreSQL, NATS JetStream, health checks) e geração de relatórios de QA com evidências em `.cognup/specs/[feature]/report/`. Use sempre que validar uma feature concluída, gerar relatório de QA, executar pipeline de qualidade, revisar implementação contra PRD/TechSpec, ou aplicar tolerância zero a bugs antes de release.
---

# QA Best Practices — Validação Completa de Features Backend Node.js

Skill que aplica o processo sistemático de QA backend para garantir qualidade end-to-end. Combina análise estática, testes automatizados, mutation testing, testes E2E, auditoria de código e verificação de infraestrutura em um pipeline com **tolerância zero a bugs**.

Combina com [[nodejs-best-practices]] (padrões de código) e [[node-testing-best-practices]] (estratégias de teste), e — conforme o escopo da feature — com [[hyper-express-best-practices]] (HTTP), [[postgres-best-practices]] (banco), [[nats-best-practices]] (mensageria) e [[testcontainers-node]] (integração). A arquitetura de referência é `.claude/rules/architecture.md` (monorepo pnpm workspaces com Hyper-Express, PostgreSQL 18, NATS JetStream e awilix).

## Princípio Fundamental

> **TOLERÂNCIA ZERO A BUGS.** Uma feature só recebe status **APROVADO** quando ZERO bugs são encontrados — independente da severidade (Crítica, Alta, Média ou Baixa). Qualquer bug encontrado, por menor que seja, resulta automaticamente em **REPROVADO**.

Não existe:
- Aprovação condicional
- Aprovação com ressalvas
- Aprovação com bugs pendentes de correção
- Distinção entre bugs "bloqueantes" e "não bloqueantes"

Severidade é usada apenas para **priorização de correção**, nunca para decisão de aprovação.

## Quando Aplicar

- Ao validar uma feature backend Node.js/TypeScript concluída antes de release
- Ao revisar implementação contra PRD, TechSpec ou Tasks
- Ao executar pipeline completo de qualidade (estática + testes + E2E)
- Ao gerar relatório de QA com evidências auditáveis
- Ao validar contrato de API contra documentação OpenAPI (swagger-jsdoc)
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
5. **Consultar skills** [[nodejs-best-practices]] e [[node-testing-best-practices]] para critérios de qualidade
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
node --version   # Node.js 22 LTS
pnpm --version
docker info

# Verificar dependências e integridade do lockfile
pnpm install --frozen-lockfile

# Verificar compilação
pnpm build

# Verificar variáveis de ambiente
cat .env  # ou env | grep APP_
```

**Protocolo de Startup da Aplicação:**

1. Verificar se já está rodando: `curl -s http://localhost:8080/health`
2. Se não, iniciar em background: `pnpm --filter <app> start &` (subir PostgreSQL/NATS via `docker-compose up -d` e aplicar migrations via `dbmate up` antes, se necessário)
3. Aguardar prontidão (poll no `/health` com 2-3s incrementais)
4. Verificar health endpoint retorna sucesso
5. Registrar PID para cleanup posterior

Se a aplicação falhar ao iniciar: **REPROVADO imediato**.

### Etapa 3: Análise Estática de Código (Obrigatória)

Execute todas as ferramentas e registre resultados em `evidence/static-analysis.md`:

| Ferramenta | Comando | O que Verifica |
|------------|---------|----------------|
| **Typecheck** | `pnpm typecheck` (`tsc --noEmit`) | Erros de tipo, strict mode, type safety |
| **Build** | `pnpm build` (`tsc`) | Compilação completa do workspace |
| **Lint** | `pnpm lint` (eslint + typescript-eslint) | Qualidade, estilo, bugs potenciais |
| **Format** | `pnpm format:check` (`prettier --check`) | Conformidade de formatação |
| **Lockfile** | `pnpm install --frozen-lockfile` | Integridade de dependências |
| **Audit** | `pnpm audit --prod` | Vulnerabilidades conhecidas em dependências |

Para cada ferramenta:
1. Executar o comando
2. Capturar a saída **completa**
3. Marcar como **PASSOU** (zero issues) ou **FALHOU**
4. Se FALHOU: classificar cada problema e documentar

**Verificações adicionais:**
- Sem `TODO`/`FIXME` não resolvidos relacionados à feature
- APIs exportadas possuem TSDoc
- Error handling usa `Error` com `cause` (`new Error("context", { cause: err })`) ou classes de erro de domínio
- Sem `any` (explícito ou implícito), sem `@ts-ignore`/`@ts-expect-error` não justificados
- Sem `console.log` em código de produção (apenas pino)
- Sem `process.exit()` fora do bootstrap/shutdown

### Etapa 4: Testes Unitários e Integração (Obrigatória)

```bash
# Suite completa com cobertura (provider v8)
pnpm test -- --coverage   # vitest run --coverage

# Cobertura por arquivo/diretório
cat coverage/coverage-summary.json | jq .

# HTML para análise visual
open coverage/index.html

# Testes de integração (projeto/config separado com testcontainers)
pnpm test:integration     # vitest run --config vitest.integration.config.ts
```

**Análise obrigatória:**
1. Parse a saída — identificar passados, falhos, pulados
2. **ZERO falhas** é obrigatório
3. Cobertura por pacote/diretório registrada
4. **Cobertura mínima: 80%** (threshold do projeto)
5. **Zero flakiness**: suite roda sem `retry` configurado e passa em execuções repetidas — Node não possui race detector como Go; o rigor equivalente é exigir testes determinísticos (sem retry, sem dependência de ordem) e **testes de concorrência** onde couber (requisições paralelas, `Promise.all` sobre o mesmo recurso, idempotência de consumers)

Para cada teste que falhou:
- Nome do teste, arquivo, mensagem de erro
- Classificar como bug e documentar

Para análise de cobertura:
- Percentuais por diretório
- Diretórios com cobertura baixa/zero sinalizados
- Código novo sem cobertura = bug

Referência: [[node-testing-best-practices]] para padrões `test.each`, `vi.fn`/`vi.mock`, fixtures e [[testcontainers-node]] para integração.

Salve em `evidence/test-results.md` e `evidence/coverage-summary.md`.

### Etapa 5: Mutation Testing — Stryker (Obrigatória)

Mutation testing modifica código-fonte em pequenas formas (mutações). Testes que detectam = "matam" o mutante. **Mutation score alto = suite robusta**.

**Instalação:**

```bash
pnpm add -D @stryker-mutator/core @stryker-mutator/vitest-runner
```

**Execução:**

```bash
# Projeto completo (usa stryker.config.json na raiz do pacote)
pnpm dlx stryker run

# Direcionado para os arquivos da feature
pnpm dlx stryker run --mutate "apps/<app>/src/application/**/*.ts,apps/<app>/src/domain/**/*.ts"

# Com exclusões (falsos positivos conhecidos)
pnpm dlx stryker run --mutate "src/**/*.ts,!src/**/*.spec.ts,!src/infrastructure/config/**"
```

**Análise:**
1. Cada mutação: **Killed** (morto = bom) ou **Survived** (sobreviveu = lacuna nos testes)
2. Score reportado pelo Stryker: `killed / total`
3. **Threshold mínimo: 80%** (configurar `thresholds.break: 80` no `stryker.config.json`)
4. Mutantes sobreviventes documentados com mutator, arquivo, linha e diff
5. Classificar cada sobrevivente:
   - **Lacuna real**: precisa de novo teste → bug severidade Média
   - **Falso positivo**: early exit, otimização, código de config → candidato a exclusão no `mutate`

**Interpretação:**
- `mutation score >= 80%`: aceitável
- `mutation score < 80%`: REPROVADO — adicionar testes

Salve em `evidence/mutation-testing.md` (incluindo o resumo do report HTML do Stryker).

### Etapa 6: Testes E2E de API (Obrigatória)

Para cada endpoint do OpenAPI e/ou PRD, teste exaustivamente com `curl`:

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
- **400 Bad Request**: JSON malformado, campos obrigatórios ausentes, tipos inválidos (rejeitados pelo schema zod)
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
- Limites numéricos (0, negativos, `Number.MAX_SAFE_INTEGER`)
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

Compare a spec OpenAPI gerada via **swagger-jsdoc** contra comportamento real:

1. Gerar/ler a spec (`docs/openapi.json` ou endpoint `/docs/openapi.json`) e extrair todos os endpoints
2. Para cada endpoint documentado verificar:
   - Existe e responde (não 404)
   - Método HTTP correto
   - Schema de request bate (campos obrigatórios, tipos — coerente com o schema zod do handler)
   - Schema de response bate (nomes, tipos, estrutura)
   - Status codes documentados são realmente retornados
   - `Content-Type` corresponde
3. Detectar endpoints **não documentados** (existem mas não têm anotações `@openapi` no JSDoc)
4. Detectar endpoints **fantasma** (na spec mas não implementados)

**Regenerar e comparar:**

```bash
pnpm openapi:generate   # script que roda swagger-jsdoc e grava docs/openapi.json
git diff docs/openapi.json
```

Marcar cada endpoint: **MATCH** ou **MISMATCH**.

Salve em `evidence/contract-verification.md`.

### Etapa 8: Auditoria de Qualidade de Código TypeScript (Obrigatória)

Consulte [[nodejs-best-practices]] para critérios completos. Verificar e documentar em `evidence/code-quality-audit.md`:

#### 8.1 Error Handling

| Critério | Verificar | Evidência |
|----------|-----------|-----------|
| Erros nunca engolidos | Sem `catch {}` vazio ou `catch (e) { /* nada */ }` | Listar ocorrências |
| Encadeamento com `cause` | `new Error("context", { cause: err })` ao re-lançar | Funções sem contexto |
| Classes de erro de domínio | Erros tipados por entidade (`UserNotFoundError extends DomainError`) | Entidades sem erros tipados |
| `instanceof` correto | Verificação por classe, nunca por `error.message` | Locais incorretos |
| Nunca lançar não-Error | Sem `throw "string"` ou `throw { ... }` | Listar usos suspeitos |
| Rejections tratadas | Sem `unhandledRejection` nos logs; handlers async com try/catch na boundary | Logs da aplicação |

#### 8.2 Assincronismo e Concorrência

| Critério | Verificar | Evidência |
|----------|-----------|-----------|
| Promises aguardadas | Sem floating promises (`no-floating-promises` do typescript-eslint) | Ocorrências |
| Paralelismo intencional | `Promise.all`/`allSettled` para operações independentes | Awaits sequenciais desnecessários |
| Cancelamento | `AbortSignal` propagado em operações canceláveis/timeouts | Operações sem cancelamento |
| Sem async órfão | Toda operação em background tem lifecycle e shutdown | Timers/loops sem cleanup |
| Concorrência testada | Testes com requisições/operações paralelas onde couber | Testes de concorrência |

#### 8.3 Tipagem e Design

| Critério | Verificar | Evidência |
|----------|-----------|-----------|
| Strict mode | `tsconfig` com `strict: true` respeitado; sem `any` | Ocorrências de `any`/asserções |
| Fronteiras validadas | Input externo validado com zod antes de virar tipo de domínio | Entradas sem validação |
| Interfaces pequenas | Focadas, poucos métodos (ISP) | Interfaces inchadas |
| Naming | camelCase para valores, PascalCase para tipos/classes, sem stuttering | Violações (`order.OrderRepository`) |
| Function design | Funções pequenas, ≤ 3-4 parâmetros (usar objeto de opções) | Funções gigantes |
| ESM consistente | `import`/`export` (sem `require`), imports organizados | Violações |

#### 8.4 Qualidade de Testes

| Critério | Verificar | Evidência |
|----------|-----------|-----------|
| Table-driven | `test.each`/`it.each` para variações do mesmo cenário | Testes sem padrão |
| Asserções vitest | `expect` com matchers específicos (`toEqual`, `toMatchObject`, `rejects.toThrow`) | Testes verbosos |
| Mocks apenas externos | `vi.fn`/mocks apenas para dependências externas (não over-mocking) | Over-mocking |
| Cenários de erro | Não apenas happy path | Funções sem testes de erro |
| Sem retry/flakiness | Sem `retry` no config; sem `sleep` arbitrário — usar espera por condição | Configs e ocorrências |
| Fixtures isoladas | `beforeEach`/`afterEach` com estado limpo; containers com cleanup | State leak entre testes |

#### 8.5 Conformidade Arquitetural

Consultar `.claude/rules/architecture.md` do projeto.

| Critério | Verificar | Evidência |
|----------|-----------|-----------|
| Regra de dependência | Domain não importa Application/Infrastructure | Violações de import |
| Use cases | 1 ação = 1 classe = 1 arquivo, método único `perform` | Use cases fora do padrão |
| Naming de arquivos | kebab-case (`create-usecase.ts`, `user-repository.ts`) | Violações |
| Composição awilix | Registrations corretas no container (composition root) | Container do app |
| Anotações OpenAPI | Handlers possuem blocos `@openapi` completos (swagger-jsdoc) | Handlers sem docs |

### Etapa 9: Verificação de Infraestrutura (Obrigatória)

#### 9.1 PostgreSQL

> Referência: [[postgres-best-practices]].

- Health endpoint reporta o banco saudável (ex.: `postgres: true`)
- Se feature envolve persistência:
  - Criar recurso via POST → buscar via GET → comparar dados
  - Integridade: campos batem com o enviado
  - Migrations aplicadas (`dbmate status`; arquivos com `-- migrate:up`/`-- migrate:down` em `db/migrations/<app>/`); índices e constraints conforme spec (`\d "tabela"` via psql)
  - Operações multi-step atômicas (verificar rollback em falha parcial via `Transactor.runInTx`)

#### 9.2 NATS JetStream

> Referência: [[nats-best-practices]].

- Health endpoint reporta a mensageria saudável (ex.: `nats: true`)
- Se feature envolve mensageria:
  - Disparar evento via API
  - Verificar publicação (`nats stream view <STREAM>` ou logs)
  - Streams/consumers duráveis criados conforme spec (subjects `events.<domínio>.<ação>`)
  - Deduplicação por `Nats-Msg-Id` e ack/nak/term corretos no consumer
  - Evento publicado somente APÓS o commit no banco

#### 9.3 Health Check

- `GET /health` retorna 200
- Resposta inclui o status dos componentes: `server`, `postgres`, `nats`
- `requestId` e `timestamp` presentes e formatados corretamente

#### 9.4 Graceful Shutdown

- Aplicação lida com `SIGTERM`/`SIGINT` gracefully (handlers de shutdown + `container.dispose()` do awilix)
- Requisições em andamento completam antes do shutdown
- Conexões fechadas na ordem: HTTP (`server.close()`) → NATS (drain) → PostgreSQL (`pool.end()`)

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

- **Crítica**: crash, perda de dados, vulnerabilidade de segurança, unhandled rejection derrubando o processo, corrupção por concorrência
- **Alta**: endpoint principal quebrado, persistência incorreta, bypass de auth, suite de testes falhando
- **Média**: status code errado, error handling ausente, validação incompleta, cobertura < threshold
- **Baixa**: problemas menores de qualidade, TSDoc ausente, formatação inconsistente

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
| Typecheck | `pnpm typecheck` | OK / FALHOU | ... |
| Build | `pnpm build` | OK / FALHOU | ... |
| Lint | `pnpm lint` | OK / FALHOU | ... |
| Format | `pnpm format:check` | OK / FALHOU | ... |

## Testes
| Pacote/Diretório | Total | Passed | Failed | Skipped |
- **Cobertura Total**: [X]%
- **Threshold Mínimo**: 80%
- **Flakiness/Retry**: [Sim/Não]

## Mutation Testing (Stryker)
| Métrica | Valor |
| Total de Mutantes | [X] |
| Mortos (Killed) | [Y] |
| Sobreviventes (Survived) | [Z] |
| Mutation Score | [Y/X] ([%]) |

## E2E
| Endpoint | Método | Cenário | Status Esperado | Recebido | Status |

## Contrato (OpenAPI)
| Endpoint | Método | Documentado | Implementado | Match | Status |

## Auditoria de Código
### Error Handling, Assincronismo, Tipagem, Testes, Arquitetura
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
# 3. Atualizar version nos package.json afetados (pnpm version --no-git-tag-version)
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
- [ ] `pnpm typecheck` (tsc --noEmit) sem erros
- [ ] `pnpm build` sem erros
- [ ] `pnpm lint` (eslint) sem issues
- [ ] `pnpm format:check` (prettier) limpo
- [ ] Suite de testes (`pnpm test`): ZERO falhas
- [ ] Cobertura ≥ 80%
- [ ] Mutation score (Stryker) ≥ 80%
- [ ] Zero flakiness (sem retry, sem promessas não aguardadas nos logs)
- [ ] Contrato OpenAPI ↔ implementação 100% match
- [ ] Auditoria de código sem violações
- [ ] PostgreSQL, NATS JetStream, Health endpoints saudáveis
- [ ] Graceful shutdown funcional
- [ ] Todas as evidências documentadas em `report/evidence/`
- [ ] Relatório `qa-report.md` gerado

**REPROVADO** se QUALQUER um dos itens acima falhar.

## Diretrizes Operacionais

1. **Typecheck/build antes de testes** — se não compila, nada mais importa
2. **Capturar request/response de TODOS os bugs** — sem exceção
3. **Verificar logs** para unhandled rejections, erros, warnings inesperados
4. **Testar TODOS os cenários de erro** — não apenas happy paths
5. **Tolerância zero** — bug Baixa reprova igual a Crítica
6. **Ser sistemático** — seguir checklist sequencialmente
7. **Documentar tudo** — sem evidência = não testado
8. **Sempre `vitest run`** (nunca modo watch) e sem retry ao validar
9. **Validar graceful shutdown** se feature toca lifecycle
10. **Cleanup pós-testes** — parar processos em background
11. **Mutation testing valida testes** — sobreviventes = lacunas
12. **Validar contra skills** [[nodejs-best-practices]] e [[node-testing-best-practices]]
13. **Validar arquitetura** contra `.claude/rules/architecture.md`
14. **Teste sem evidência = NÃO EXECUTADO**

## Ferramentas Essenciais — Cheat Sheet

```bash
# Análise estática
pnpm typecheck              # tsc --noEmit
pnpm build                  # tsc
pnpm lint                   # eslint
pnpm format:check           # prettier --check
pnpm install --frozen-lockfile
pnpm audit --prod

# Testes
pnpm test -- --coverage     # vitest run --coverage (provider v8)
pnpm test:integration       # vitest run --config vitest.integration.config.ts
pnpm test -- --run src/application/usecase/order   # diretório específico

# Mutation testing
pnpm dlx stryker run
pnpm dlx stryker run --mutate "apps/<app>/src/application/**/*.ts"

# Teste focado (repetido para detectar flakiness)
pnpm vitest run -t "creates order" --no-file-parallelism

# Benchmark (quando relevante)
pnpm vitest bench

# Migrations
dbmate status
dbmate up

# OpenAPI (swagger-jsdoc)
pnpm openapi:generate
git diff docs/openapi.json

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
| Aprovar com `retry` mascarando flakiness | Suite determinística, sem retry |
| Confiar na spec OpenAPI sem verificar | Comparar OpenAPI vs comportamento real |
| Não documentar request/response do bug | Toda evidência reproduzível |
| Mock testando o mock | Mocks apenas para dependências externas |
| Aprovar com TODO/FIXME pendentes | Limpar TODOs da feature antes de aprovar |
| Bump de versão antes de aprovar | Etapas 12-13 SÓ após APROVADO |
| Fazer cleanup só "se der tempo" | Cleanup é parte do processo |

## Idioma do Relatório

Escrever relatório de QA e documentação de bugs em **Português (Brasil)**. Termos técnicos (tipos TypeScript, métodos HTTP, status codes, nomes de ferramentas) podem permanecer em inglês.

Changelog em **inglês** (padrão da indústria).
