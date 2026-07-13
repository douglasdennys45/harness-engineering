---
name: qa-best-practices
description: Aplica melhores praticas de QA (Quality Assurance) para validacao completa de features backend em Python. Cobre analise estatica (ruff format --check, ruff check, mypy --strict), testes unitarios e de integracao com pytest e cobertura (pytest-cov), mutation testing com mutmut, testes E2E de API com httpx/curl, verificacao de contrato OpenAPI nativo do Robyn, auditoria de qualidade de codigo (error handling com raise from, corrotinas nao aguardadas, tipagem Any, arquitetura), verificacao de infraestrutura (PostgreSQL, NATS JetStream, health checks) e geracao de relatorios de QA com evidencias em `.cognup/specs/[feature]/report/`. Use sempre que validar uma feature concluida, gerar relatorio de QA, executar pipeline de qualidade, revisar implementacao contra PRD/TechSpec, ou aplicar tolerancia zero a bugs antes de release.
---

# QA Best Practices — Validacao Completa de Features Backend Python

Skill que aplica o processo sistematico de QA backend para garantir qualidade end-to-end. Combina analise estatica, testes automatizados, mutation testing, testes E2E, auditoria de codigo e verificacao de infraestrutura em um pipeline com **tolerancia zero a bugs**.

Combina com [[python-best-practices]] (padroes de codigo) e [[python-testing-best-practices]] (estrategias de teste), e — conforme o escopo da feature — com [[robyn-best-practices]] (HTTP), [[postgres-best-practices]] (banco), [[nats-best-practices]] (mensageria) e [[testcontainers-python]] (integracao). A arquitetura de referencia e `.claude/rules/architecture.md` (monorepo uv com Robyn, PostgreSQL 18, NATS JetStream e dependency-injector).

## Principio Fundamental

> **TOLERANCIA ZERO A BUGS.** Uma feature so recebe status **APROVADO** quando ZERO bugs sao encontrados — independente da severidade. Qualquer bug, por menor que seja, resulta em **REPROVADO**.

Nao existe: aprovacao condicional, com ressalvas, ou com bugs pendentes. Severidade e usada apenas para **priorizacao de correcao**.

## Quando Aplicar

- Ao validar uma feature backend Python concluida antes de release
- Ao revisar implementacao contra PRD, TechSpec ou Tasks
- Ao executar pipeline completo de qualidade (estatica + testes + E2E)
- Ao gerar relatorio de QA com evidencias auditaveis
- Ao validar contrato de API contra o OpenAPI nativo do Robyn
- Antes de bump de versao (SemVer) e atualizacao de changelog

## Estrutura de Evidencias

**REGRA CRITICA:** todos os artefatos de QA DEVEM ser salvos em:

```
.cognup/specs/[feature]/report/
├── qa-report.md                          # Relatorio final consolidado
├── bugs.md                               # Documentacao detalhada de bugs
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

**OBRIGATORIEDADE:** todo teste (PASSOU ou FALHOU) tem evidencia; todo bug tem request/response, output ou trecho de codigo. Teste sem evidencia = teste NAO EXECUTADO.

## Pipeline de QA — 13 Etapas

### Etapa 1: Analise da Documentacao (Obrigatoria)

1. **Ler o PRD** e extrair TODOS os requisitos funcionais numerados em checklist
2. **Ler a TechSpec** e anotar decisoes verificaveis (endpoints, modelos, error codes, regras)
3. **Ler o arquivo de Tasks** e confirmar status
4. **Ler arquitetura** (`.claude/rules/architecture.md`)
5. **Consultar skills** [[python-best-practices]] e [[python-testing-best-practices]]
6. **Construir matriz de verificacao** mapeando cada requisito -> cenario de teste

### Etapa 2: Preparacao do Ambiente (Obrigatoria)

```bash
mkdir -p .cognup/specs/[feature]/report/evidence/

python --version   # 3.13+
uv --version
docker info

uv sync --locked   # integridade do lockfile
make typecheck     # compilacao logica (mypy)
cat .env
```

**Startup da Aplicacao:**
1. Verificar se ja roda: `curl -s http://localhost:8080/health`
2. Se nao, subir infra (`docker-compose up -d`, `dbmate up`) e a API: `uv run python -m user.bin.api &` (ou `uv run robyn ... `)
3. Poll no `/health` ate prontidao
4. Registrar PID para cleanup

Se a aplicacao falhar ao iniciar: **REPROVADO imediato**.

### Etapa 3: Analise Estatica (Obrigatoria)

Registre em `evidence/static-analysis.md`:

| Ferramenta | Comando | O que Verifica |
|---|---|---|
| **Typecheck** | `make typecheck` (`mypy --strict`) | Erros de tipo, strict, sem `Any` |
| **Lint** | `make lint` (`ruff check`) | Qualidade, estilo, bugs potenciais, async |
| **Format** | `make format-check` (`ruff format --check`) | Conformidade de formatacao |
| **Lockfile** | `uv sync --locked` | Integridade de dependencias |

**Verificacoes adicionais:**
- Sem `TODO`/`FIXME` nao resolvidos da feature
- APIs publicas com docstring
- Error handling usa `raise ... from err` ou classes de erro de dominio
- Sem `Any` (explicito/implicito), sem `# type: ignore` nao justificado
- Sem `print` em codigo de producao (apenas structlog)
- Sem `time.sleep`/I/O sincrono bloqueante no caminho async

### Etapa 4: Testes Unitarios e Integracao (Obrigatoria)

```bash
uv run pytest --cov --cov-report=term-missing   # unitarios + cobertura
uv run pytest -m integration                      # integracao (testcontainers)
```

**Analise obrigatoria:**
1. **ZERO falhas** e obrigatorio
2. Cobertura por modulo registrada; **minimo 80%**
3. **Zero flakiness**: suite roda sem retry e passa em execucoes repetidas; testes deterministicos (sem sleep, sem dependencia de ordem); testes de concorrencia onde couber (requisicoes paralelas, idempotencia de consumers)

Codigo novo sem cobertura = bug. Salve em `evidence/test-results.md` e `coverage-summary.md`.

### Etapa 5: Mutation Testing — mutmut (Obrigatoria)

```bash
uv run mutmut run --paths-to-mutate apps/<app>/src/<app>/domain,apps/<app>/src/<app>/application
uv run mutmut results
```

**Analise:**
1. Cada mutacao: **killed** (bom) ou **survived** (lacuna nos testes)
2. **Threshold minimo: 80%** de mutantes mortos
3. Mutantes sobreviventes documentados; classificar: lacuna real (novo teste, bug Media) ou falso positivo (candidato a exclusao)

Salve em `evidence/mutation-testing.md`.

### Etapa 6: Testes E2E de API (Obrigatoria)

Para cada endpoint, teste com `curl`/`httpx`:

```bash
curl -s -w "\n%{http_code}" -X POST http://localhost:8080/v1/users \
  -H "Content-Type: application/json" \
  -d '{"name":"test","email":"t@e.com"}'
```

**Categorias obrigatorias:**
- **Happy path**: request valido -> 200/201/204, body bate com schema
- **Casos de erro**: 400 (JSON malformado, campos ausentes, tipos invalidos rejeitados pelo Pydantic), 401, 403, 404, 409, 422; **500 nunca vaza stack trace**
- **Edge cases**: body vazio, valores longos, unicode, SQL injection, XSS, limites numericos, requisicoes concorrentes, path traversal

Documente request/response completos. Salve em `evidence/api-e2e-results.md`.

### Etapa 7: Verificacao de Contrato API (Obrigatoria)

Compare o OpenAPI nativo do Robyn contra o comportamento real:

```bash
curl -s http://localhost:8080/openapi.json > /tmp/openapi.json
```

Para cada endpoint documentado: existe e responde (nao 404), metodo correto, schema de request/response bate (coerente com o modelo Pydantic), status codes documentados sao retornados, `Content-Type` correto. Detectar endpoints nao documentados (sem `openapi_tags`/docstring) e fantasma (no spec, nao implementados). Salve em `evidence/contract-verification.md`.

### Etapa 8: Auditoria de Qualidade de Codigo Python (Obrigatoria)

Consulte [[python-best-practices]]. Documente em `evidence/code-quality-audit.md`:

#### 8.1 Error Handling
| Criterio | Verificar |
|---|---|
| Erros nunca engolidos | Sem `except: pass`; boundary loga + age |
| Encadeamento com `from` | `raise X("context") from err` |
| Classes de erro de dominio | `UserNotFoundError(DomainError)` |
| `isinstance` correto | nunca por `str(err)` |
| `CancelledError` re-lancado | nunca engolido |

#### 8.2 Assincronismo e Concorrencia
| Criterio | Verificar |
|---|---|
| Corrotinas aguardadas | sem `RuntimeWarning: never awaited` |
| Tasks rastreadas | `create_task` com referencia + tratamento |
| Paralelismo | `asyncio.gather`/`TaskGroup` para independentes |
| Timeouts | `asyncio.timeout` em I/O externa |
| Sem bloqueio do loop | sem `time.sleep`/`requests`; CPU-bound em `to_thread` |
| Concorrencia testada | requisicoes/operacoes paralelas onde couber |

#### 8.3 Tipagem e Design
| Criterio | Verificar |
|---|---|
| mypy strict | sem `Any`; sintaxe moderna de tipos |
| Fronteiras validadas | input externo validado com Pydantic antes do dominio |
| Protocols pequenos | focados (ISP) |
| Naming | snake_case/PascalCase corretos; sem camelCase |
| Imports | absolutos, organizados |

#### 8.4 Qualidade de Testes
| Criterio | Verificar |
|---|---|
| Table-driven | `parametrize` para variacoes |
| `pytest.raises` | erro esperado por tipo |
| Fakes/mocks | so para dependencias externas (Protocols) |
| Cenarios de erro | nao apenas happy path |
| Sem flakiness | sem sleep, sem dependencia de ordem |
| Fixtures isoladas | estado limpo por teste; containers com cleanup |

#### 8.5 Conformidade Arquitetural
| Criterio | Verificar |
|---|---|
| Regra de dependencia | domain nao importa application/infrastructure |
| Use cases | 1 acao = 1 Protocol = 1 arquivo, `perform` |
| Naming de arquivos | snake_case |
| Composicao dependency-injector | providers corretos no composition root |
| OpenAPI | handlers com docstring + `openapi_tags` |

### Etapa 9: Verificacao de Infraestrutura (Obrigatoria)

#### 9.1 PostgreSQL
> Referencia: [[postgres-best-practices]].
- Health reporta o banco saudavel; criar via POST -> buscar via GET -> comparar; migrations aplicadas (`dbmate status`); indices/constraints conforme spec; operacoes multi-step atomicas (rollback em falha via `run_in_tx`).

#### 9.2 NATS JetStream
> Referencia: [[nats-best-practices]].
- Health reporta a mensageria saudavel; disparar evento via API; verificar publicacao; streams/consumers duraveis conforme spec; dedup por `Nats-Msg-Id` e ack/nak/term corretos; evento publicado **apos** o commit.

#### 9.3 Health Check
- `GET /health` retorna 200 com status de `server`, `postgres`, `nats`; `request_id`/`timestamp` presentes.

#### 9.4 Graceful Shutdown
- SIGTERM/SIGINT tratados (`@app.shutdown_handler`/signal handlers); requisicoes/mensagens em andamento completam; ordem: HTTP -> NATS drain -> PG disconnect.

Salve em `evidence/infrastructure.md`.

### Etapa 10: Documentacao de Bugs

Para cada bug, em `report/bugs.md`:

```markdown
## BUG-[ID]: [Descricao curta]
- **Severidade**: Critica / Alta / Media / Baixa
- **Requisito**: [ID do PRD]
- **Categoria**: Analise Estatica / Falha de Teste / API E2E / Contrato / Qualidade / Infraestrutura
- **Passos para Reproduzir**: 1. ... 2. ...
- **Resultado Esperado**: ...
- **Resultado Atual**: ...
- **Evidencia**:
  ```
  [request/response, output, mensagem de erro]
  ```
- **Ambiente**: localhost:8080
```

**Severidade** (para priorizacao, NAO para aprovacao):
- **Critica**: crash, perda de dados, vulnerabilidade, corrotina nao aguardada perdendo escrita, corrupcao por concorrencia
- **Alta**: endpoint quebrado, persistencia incorreta, bypass de auth, suite falhando
- **Media**: status code errado, error handling ausente, validacao incompleta, cobertura < threshold
- **Baixa**: qualidade menor, docstring ausente, formatacao

**REGRA DE APROVACAO:** qualquer bug — **incluindo Baixa** — torna o QA REPROVADO.

### Etapa 11: Geracao do Relatorio de QA (Obrigatoria)

Crie `report/qa-report.md`:

```markdown
# Relatorio de QA Backend - [Feature]
## Resumo
- Data / QA Engineer / Status: APROVADO / REPROVADO
- Total de Requisitos / Atendidos / Reprovados / Bugs / Evidencias
## Requisitos Verificados
| ID | Requisito | Status | Evidencia |
## Analise Estatica
| Ferramenta | Comando | Status | Detalhes |
| Typecheck | `make typecheck` | OK/FALHOU | |
| Lint | `make lint` | OK/FALHOU | |
| Format | `make format-check` | OK/FALHOU | |
## Testes
| Modulo | Total | Passed | Failed | Skipped |
- Cobertura Total / Threshold 80% / Flakiness
## Mutation Testing (mutmut)
| Total | Killed | Survived | Score |
## E2E | Contrato (OpenAPI) | Auditoria | Infraestrutura
## Bugs
| ID | Descricao | Severidade | Categoria | Requisito | Evidencia |
## Inventario de Evidencias
## Conclusao / Proximos Passos
```

### Etapa 12: Changelog (Pos-Aprovacao)

**So se QA = APROVADO.** Keep a Changelog + Conventional Commits. Adicionar em `[Unreleased]` do `CHANGELOG.md` (entradas em ingles). Ver [[changelog-automation]].

### Etapa 13: Versionamento Semantico (Pos-Aprovacao)

**So se Etapa 12 completou.** SemVer: MAJOR (breaking), MINOR (feature), PATCH (fix). `pyproject.toml` version + `CHANGELOG.md` + tag `vX.Y.Z`. NUNCA bump se REPROVADO.

## Criterios de Aprovacao — Checklist Final

**APROVADO** requer **TODOS**:

- [ ] 100% dos requisitos do PRD verificados e passando
- [ ] **ZERO bugs** (qualquer severidade)
- [ ] `make typecheck` (mypy --strict) sem erros
- [ ] `make lint` (ruff) sem issues
- [ ] `make format-check` (ruff format) limpo
- [ ] Suite (`make test`): ZERO falhas
- [ ] Cobertura >= 80%
- [ ] Mutation score (mutmut) >= 80%
- [ ] Zero flakiness (sem retry, sem corrotina nao aguardada)
- [ ] Contrato OpenAPI <-> implementacao 100% match
- [ ] Auditoria de codigo sem violacoes
- [ ] PostgreSQL, NATS, Health saudaveis
- [ ] Graceful shutdown funcional
- [ ] Evidencias em `report/evidence/`
- [ ] `qa-report.md` gerado

**REPROVADO** se QUALQUER item falhar.

## Ferramentas Essenciais — Cheat Sheet

```bash
# Analise estatica
make typecheck              # mypy --strict
make lint                   # ruff check
make format-check           # ruff format --check
uv sync --locked

# Testes
uv run pytest --cov --cov-report=term-missing
uv run pytest -m integration
uv run pytest apps/user/tests/unit/application   # dir especifico

# Mutation
uv run mutmut run --paths-to-mutate apps/user/src/user/domain,apps/user/src/user/application
uv run mutmut results

# Teste focado (repetido para detectar flakiness)
uv run pytest -k "create_user" -p no:randomly

# Migrations
dbmate status && dbmate up

# OpenAPI
curl -s http://localhost:8080/openapi.json | python -m json.tool

# Health / E2E
curl -s http://localhost:8080/health | python -m json.tool
curl -s -w "\n%{http_code}" -X POST http://localhost:8080/v1/users -H "Content-Type: application/json" -d '{"name":"test","email":"t@e.com"}'
```

## Anti-Patterns de QA (Evitar)

| Anti-Pattern | Correto |
|---|---|
| "Bug de severidade baixa, aprovo" | Tolerancia zero — REPROVADO |
| Pular evidencia de teste que passou | Toda evidencia obrigatoria |
| Testar so happy path | Todos cenarios de erro + edge cases |
| Cobertura como unica metrica | Cobertura + mutation + auditoria |
| Aprovar com retry mascarando flakiness | Suite deterministica |
| Confiar no OpenAPI sem verificar | Comparar spec vs comportamento real |
| Nao documentar request/response do bug | Toda evidencia reproduzivel |
| Aprovar com TODO/FIXME pendentes | Limpar TODOs antes |
| Bump antes de aprovar | Etapas 12-13 so apos APROVADO |

## Idioma do Relatorio

Relatorio de QA e bugs em **Portugues (Brasil)**. Termos tecnicos e changelog em ingles.
