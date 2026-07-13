---
name: bugfix-best-practices
description: Aplica melhores praticas de Bugfix para correcao sistematica de bugs documentados pelo QA em servicos Python. Cobre leitura do arquivo `.cognup/specs/[feature]/report/bugs.md`, analise de causa raiz (nunca sintomas), planejamento por bug, implementacao da correcao conforme Clean Architecture (`.claude/rules/architecture.md`) e stack do projeto (Robyn, PostgreSQL 18, NATS JetStream, dependency-injector, structlog, Pydantic), criacao de testes de regressao (unitario com pytest/AsyncMock, integracao com testcontainers, E2E de API), validacao completa (`make typecheck`, `make lint`, `make format-check`, `make test`), atualizacao do bugs.md com status das correcoes, relatorio final, commits no padrao Conventional Commits e ciclo de revisao (task-reviewer ate APPROVED, depois qa-validator ate APROVADO). Use sempre que houver bugs documentados em bugs.md para corrigir, quando o QA reprovar uma feature, ao corrigir regressoes, ou ao criar testes de regressao para bugs encontrados.
---

# Bugfix Best Practices — Correcao Sistematica de Bugs Python

Skill que aplica o processo metodico de **Bugfix** sobre bugs documentados pelo QA (`qa-best-practices`) em servicos Python. Combina analise de causa raiz, correcao conforme os padroes do projeto, testes de regressao, validacao completa, rastreabilidade no `bugs.md` e o ciclo de revisao ate aprovacao.

Combina com [[python-best-practices]] (padroes de codigo Python moderno) e [[python-testing-best-practices]] (testes de regressao), e — conforme o componente afetado — com [[robyn-best-practices]] (HTTP), [[postgres-best-practices]] (banco), [[nats-best-practices]] (mensageria) e [[testcontainers-python]] (integracao). Commits seguem [[conventional-commit]]. A arquitetura de referencia e `.claude/rules/architecture.md` (monorepo uv com Robyn, PostgreSQL 18, NATS JetStream e dependency-injector).

## Principio Fundamental

> **Causa raiz, nunca sintoma.** Todo bug e corrigido na origem — sem gambiarras, sem supressao de sintomas, sem `try/except` defensivo escondendo o defeito. Cada correcao vem com testes de regressao que falhariam se a correcao fosse revertida. O bugfix so termina quando TODOS os bugs do `bugs.md` estao corrigidos, TODOS os testes passam e o ciclo de revisao retorna APPROVED.

Regras absolutas:
- **TODOS os bugs** listados no `bugs.md` devem ser corrigidos
- **Cada bug corrigido** ganha teste(s) de regressao que simulam o cenario original
- **Nenhuma correcao superficial** — se o sintoma some mas a causa permanece, o bug nao foi corrigido
- **Nada finaliza com teste falhando** — `make test` (pytest) a 100% e pre-condicao

## Localizacao dos Artefatos

| Artefato | Caminho |
|---|---|
| **Bugs (ENTRADA)** | `.cognup/specs/[nome-da-funcionalidade]/report/bugs.md` |
| Relatorio de QA | `.cognup/specs/[nome-da-funcionalidade]/report/qa-report.md` |
| PRD / Tech Spec / Tasks | `.cognup/specs/[nome-da-funcionalidade]/` |
| Regras do projeto | `.claude/rules/architecture.md` |

## Quando Aplicar

- Quando o QA (`qa-validator`) reprovou a feature e documentou bugs em `report/bugs.md`
- Quando o usuario pede para corrigir bugs documentados
- Ao corrigir regressoes detectadas apos merge
- Ao criar testes de regressao para bugs encontrados manualmente (documente-os primeiro no `bugs.md`)

## Pipeline de Bugfix — 8 Etapas

### Etapa 1: Analise de Contexto

- Ler `report/bugs.md` e extrair **TODOS** os bugs (ID, severidade, categoria, passos, evidencia)
- Ler `report/qa-report.md` para entender a reprovacao
- Ler PRD e TechSpec para os requisitos afetados
- Ler `.claude/rules/architecture.md`
- Carregar [[python-best-practices]] (sempre) e as skills de dominio conforme os componentes afetados

**NUNCA pule esta etapa** — entender o contexto e fundamental para corrigir a causa raiz.

### Etapa 2: Planejamento das Correcoes

Para cada bug, gerar um resumo **antes de codar**:

```
BUG ID: [ID]
Severidade: [Critica/Alta/Media/Baixa]
Componente Afetado: [camada + arquivo(s)]
Causa Raiz: [o que de fato esta errado e por que]
Arquivos a Modificar: [lista]
Estrategia de Correcao: [abordagem, conforme architecture.md]
Testes de Regressao Planejados:
  - [unitario]: [descricao]
  - [integracao]: [descricao]
  - [E2E]: [descricao]
```

**Ordem: Critica -> Alta -> Media -> Baixa.**

Distinguir causa raiz de sintoma:

| Sintoma (nao corrigir so isso) | Causa raiz (corrigir isto) |
|---|---|
| Endpoint retorna 500 | Erro nao mapeado no `ErrorMapper` ou nao tratado no use case |
| Dado nao persiste | Transaction sem commit, `RETURNING` ausente, rollback silencioso, corrotina de escrita nao aguardada |
| Evento nao chega ao consumer | Publish antes do commit, subject errado, falta de `Nats-Msg-Id` |
| Teste flakey | Corrotina nao aguardada, dependencia de ordem, estado compartilhado, sleep real onde cabia controle de tempo |
| Crash intermitente | Excecao nao tratada em caminho de erro, `except Exception` engolindo `CancelledError`, event loop bloqueado por I/O sincrono |

### Etapa 3: Implementacao das Correcoes

Para cada bug:
1. **Localizar o codigo afetado** — ler os arquivos por completo
2. **Reproduzir** — de preferencia com um teste que falha (red); no minimo, reasoning explicito
3. **Implementar na causa raiz** — conforme os padroes (camadas, Protocols de dominio, error handling com classes e `from`, eventos pos-commit)
4. **Verificar tipos** — `make typecheck` apos cada correcao
5. **Rodar os testes existentes** — nenhum quebrou

Regras:
- Respeitar a regra de dependencia (`domain` -> `application` -> `infrastructure`)
- Nao introduzir codigo fora dos padroes para "resolver rapido"
- Se a correcao exigir mudanca arquitetural, documentar a justificativa
- Novos bugs descobertos entram no `bugs.md` (formato do QA) e sao corrigidos no mesmo ciclo

### Etapa 4: Testes de Regressao

Para cada bug corrigido, criar testes que:
- **Simulem o cenario original** — o teste DEVE falhar se a correcao for revertida
- **Validem o comportamento correto** — passa com a correcao
- **Cubram edge cases relacionados** — variacoes (None, vazio, limites, concorrencia)

> Referencia: [[python-testing-best-practices]] (parametrize, AsyncMock, fixtures) e [[testcontainers-python]].

| Tipo | Quando Usar | Ferramentas |
|---|---|---|
| Unitario | Bug em logica isolada de funcao/use case | pytest + `parametrize` + fakes/`AsyncMock` sobre Protocols |
| Integracao | Bug na comunicacao com infra (repository, publisher/subscriber) | testcontainers (PostgreSQL/NATS reais) |
| E2E | Bug visivel no fluxo completo da API | API Robyn em porta livre + httpx/curl |

Nomeie os testes de forma rastreavel ao bug (ex.: `test_bug_03_duplicate_email_returns_409`).

### Etapa 5: Validacao Completa

```bash
make typecheck        # mypy --strict
make lint             # ruff check
make format-check     # ruff format --check
make test             # pytest — 100% de sucesso
```

**A tarefa NAO esta completa se qualquer verificacao falhar.**

### Etapa 6: Atualizacao do bugs.md

Apos corrigir cada bug, anexar ao final da entrada:

```markdown
- **Status:** Corrigido
- **Correcao aplicada:** [descricao breve da correcao na causa raiz]
- **Testes de regressao:** [lista dos testes criados, com caminho do arquivo]
```

Regras: **nunca apagar** a documentacao original; todo bug termina com **Status: Corrigido**; novos bugs entram no mesmo arquivo.

### Etapa 7: Relatorio Final

```markdown
# Relatorio de Bugfix - [Nome da Funcionalidade]
## Resumo
- Total de Bugs / Corrigidos / Novos Descobertos e Corrigidos / Testes de Regressao Criados
## Detalhes por Bug
| ID | Severidade | Status | Causa Raiz | Correcao | Testes Criados |
## Validacao
- `make typecheck`: SEM ERROS
- `make lint`: LIMPO
- `make format-check`: LIMPO
- `make test`: TODOS PASSANDO
```

### Etapa 8: Commit, Revisao e Re-QA

1. **Commit por bugfix** no padrao [[conventional-commit]]: `fix(scope): description` (ex: `fix(order): map duplicate email error to 409 conflict`); stage apenas os arquivos do bugfix; testes de regressao no mesmo commit.
2. **Revisao**: execute o agente **@task-reviewer** ([[task-reviewer-best-practices]]) — status != APPROVED -> ajuste, commite, re-execute ate **APPROVED**.
3. **Re-QA**: com todos os bugs corrigidos e review APPROVED, execute o agente **@qa-validator** ([[qa-best-practices]]) — **APROVADO** encerra; **REPROVADO** com novos bugs -> reinicie o pipeline (Etapa 1).

Ciclo: `qa-validator (REPROVADO) -> bugfix -> task-reviewer (APPROVED) -> qa-validator` — ate **APROVADO**.

## Diretrizes Operacionais

1. **Todos os bugs, sempre** — nao encerrar com bug pendente
2. **Ler antes de modificar** — codigo-fonte completo dos arquivos afetados
3. **Causa raiz documentada** — no planejamento e no relatorio
4. **Regressao obrigatoria** — bug sem teste de regressao = nao corrigido
5. **Ordem por severidade** — Critica -> Alta -> Media -> Baixa
6. **Sem gambiarras** — mesmos padroes do codigo novo (`architecture.md` + skills)
7. **Validacao completa antes do commit** — typecheck, lint, format-check, test
8. **Commits no padrao** — `fix(scope): description`, um por bugfix
9. **Rastreabilidade no bugs.md** — status anexado sem apagar o original
10. **Ciclo ate aprovacao** — task-reviewer ate APPROVED, depois qa-validator ate APROVADO
11. **Use o Context7 MCP apenas quando** as skills e regras locais nao cobrirem a duvida

## Anti-Patterns de Bugfix (Evitar)

| Anti-Pattern | Correto |
|---|---|
| Silenciar o sintoma (`except: return None`) | Corrigir a causa raiz |
| `try/except` amplo para esconder crash | Eliminar o acesso invalido que causa o crash |
| Corrigir sem teste de regressao | Todo bug ganha teste que falha se revertido |
| Teste que passa mesmo sem a correcao | Validar red -> green |
| Corrigir so os bugs "importantes" | TODOS, em ordem de severidade |
| Marcar "Corrigido" sem rodar a suite | `make test` a 100% antes |
| Apagar a descricao original do bug | Anexar status mantendo rastreabilidade |
| Um commit gigante com tudo | Um `fix(scope)` por bugfix |
| Ignorar bug novo descoberto | Documentar no bugs.md e corrigir no mesmo ciclo |
| Finalizar sem review | task-reviewer ate APPROVED, depois qa-validator ate APROVADO |
| Correcao que viola a regra de dependencia | Conformidade com `architecture.md` |
| Retry/sleep para "resolver" flakey | Corrigir a corrotina nao aguardada ou o estado compartilhado |

## Cheat Sheet — Comandos do Bugfix

```bash
cat .cognup/specs/[feature]/report/bugs.md
cat .cognup/specs/[feature]/report/qa-report.md

# Reproduzir com teste focado (red)
uv run pytest apps/<app>/tests/unit/application/usecase/order -k "bug_03"

# Validacao completa (gate antes do commit)
make typecheck && make lint && make format-check && make test

# Cobertura dos modulos corrigidos
uv run pytest --cov=apps/<app>/src --cov-report=term-missing

# Commit por bugfix
git add <files>
git commit -m "fix(order): map duplicate email error to 409 conflict"

# E2E rapido no endpoint corrigido
curl -s -w "\n%{http_code}" -X POST http://localhost:8080/v1/orders \
  -H "Content-Type: application/json" -d '{"userId":"...","totalCents":100}'
```

## Idioma

Planejamento, relatorio e `bugs.md` em **Portugues (Brasil)**. Codigo, nomes de testes e commits em **ingles**.

## Manutencao de Memoria

Registre causas raiz recorrentes (erro nao registrado no ErrorMapper, publish dentro de transaction, corrotina nao aguardada, event loop bloqueado), componentes propensos a bugs, lacunas de teste, correcoes que exigiram mudanca arquitetural, testes flakey e suas causas.
