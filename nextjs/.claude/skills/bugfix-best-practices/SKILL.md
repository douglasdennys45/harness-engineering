---
name: bugfix-best-practices
description: Aplica melhores práticas de Bugfix para correção sistemática de bugs documentados pelo QA em aplicações frontend Next.js. Cobre leitura do arquivo `.cognup/specs/[feature]/report/bugs.md`, análise de causa raiz (nunca sintomas), planejamento por bug, implementação da correção conforme os padrões do projeto (nextjs-clean-architecture, next-best-practices, regra SSR), criação de testes de regressão (unitário com Testing Library, integração com mocks dos contratos, E2E com Playwright), validação visual no browser (webapp-testing), validação completa (tsc --noEmit, lint, test, build), atualização do bugs.md com status das correções, relatório final, commits no padrão Conventional Commits e ciclo de revisão (task-reviewer até APPROVED, depois qa-validator até APROVADO). Use sempre que houver bugs documentados em bugs.md para corrigir, quando o QA reprovar uma feature, ao corrigir regressões, ou ao criar testes de regressão para bugs encontrados.
---

# Bugfix Best Practices — Correção Sistemática de Bugs Frontend Next.js

Skill que aplica o processo metódico de **Bugfix** sobre bugs documentados pelo QA (`qa-best-practices`) em aplicações frontend Next.js. Combina análise de causa raiz, correção conforme os padrões do projeto, testes de regressão, validação completa, rastreabilidade no `bugs.md` e o ciclo de revisão até aprovação.

Combina com [[next-best-practices]] (padrões Next.js) e [[nextjs-clean-architecture]] (arquitetura e camadas) — sempre —, e conforme o componente afetado com [[shadcn]] (componentes UI), [[frontend-design]] (bugs visuais), [[responsive-design]] (bugs de layout/responsividade), [[web-design-guidelines]] (conformidade de interface), [[webapp-testing]] (validação no browser) e [[playwright-generate-test]] (testes de regressão E2E). Commits seguem [[conventional-commit]].

## Princípio Fundamental

> **Causa raiz, nunca sintoma.** Todo bug é corrigido na origem do problema — sem gambiarras, sem supressão de sintomas, sem CSS defensivo escondendo o defeito. Cada correção vem acompanhada de testes de regressão que falhariam se a correção fosse revertida. O bugfix só termina quando TODOS os bugs do `bugs.md` estão corrigidos, TODOS os testes passam e o ciclo de revisão retorna APPROVED.

Regras absolutas:

- **TODOS os bugs** listados no `bugs.md` devem ser corrigidos — não existe correção parcial
- **Cada bug corrigido** ganha teste(s) de regressão que simulam o cenário original
- **Nenhuma correção superficial** — se o sintoma some mas a causa permanece, o bug não foi corrigido
- **Nada finaliza com teste falhando** — lint, typecheck, testes e build 100% limpos são pré-condição

## Localização dos Artefatos

| Artefato | Caminho |
|---|---|
| **Bugs (ENTRADA)** | `.cognup/specs/[nome-da-funcionalidade]/report/bugs.md` |
| Relatório de QA | `.cognup/specs/[nome-da-funcionalidade]/report/qa-report.md` |
| Screenshots do QA | `.cognup/specs/[nome-da-funcionalidade]/report/screenshots/` |
| PRD | `.cognup/specs/[nome-da-funcionalidade]/prd.md` |
| Tech Spec | `.cognup/specs/[nome-da-funcionalidade]/techspec.md` |
| Tasks | `.cognup/specs/[nome-da-funcionalidade]/tasks.md` |
| Reviews das tasks | `.cognup/specs/[nome-da-funcionalidade]/tasks/reviews/` |
| Padrões do projeto | skill [[nextjs-clean-architecture]] + `@.claude/rules` (se existirem) |

## Quando Aplicar

- Quando o QA (`qa-best-practices` / agente qa-validator) reprovou a feature e documentou bugs em `report/bugs.md`
- Quando o usuário pede para corrigir bugs documentados de uma feature
- Ao corrigir regressões detectadas após merge
- Ao criar testes de regressão para bugs encontrados manualmente (documente-os primeiro no `bugs.md`)

## Pipeline de Bugfix — 8 Etapas

### Etapa 1: Análise de Contexto

- Ler `report/bugs.md` e extrair **TODOS** os bugs documentados (ID, severidade, categoria, passos de reprodução, screenshot)
- Ler o `report/qa-report.md` para entender o contexto da reprovação
- Ler o PRD para entender os requisitos afetados por cada bug
- Ler a TechSpec para entender as decisões técnicas relevantes (renderização, contratos, componentes)
- Consultar [[nextjs-clean-architecture]] para garantir conformidade nas correções
- Carregar [[next-best-practices]] (sempre) e as skills de escopo conforme os componentes afetados: [[shadcn]] para bugs em componentes UI, [[responsive-design]] para bugs de layout/breakpoint, [[frontend-design]]/[[web-design-guidelines]] para bugs visuais e de UX

**NUNCA pule esta etapa** — entender o contexto completo é fundamental para corrigir a causa raiz e não o sintoma.

### Etapa 2: Planejamento das Correções

Para cada bug, gerar um resumo de planejamento **antes de codar**:

```
BUG ID: [ID do bug]
Severidade: [Crítica/Alta/Média/Baixa]
Componente Afetado: [camada + arquivo(s)]
Causa Raiz: [análise da causa raiz — o que de fato está errado e por quê]
Arquivos a Modificar: [lista de arquivos]
Estratégia de Correção: [descrição da abordagem, conforme os padrões do projeto]
Testes de Regressão Planejados:
  - [Teste unitário]: [descrição]
  - [Teste de integração]: [descrição]
  - [Teste E2E]: [descrição]
```

**Ordem de correção por severidade: Crítica → Alta → Média → Baixa.**

Distinguir causa raiz de sintoma:

| Sintoma (não corrigir só isso) | Causa raiz (corrigir isto) |
|---|---|
| Erro de hidratação no console | Render não-determinístico (data/random/browser API) no Server Component ou HTML inválido |
| Dado não aparece na tela | Fetch com cache errado, Server Action sem revalidate, contrato de API divergente |
| Formulário "não envia" | Server Action sem validação/tratamento de erro, estado de erro não exibido |
| Layout quebrado no mobile | Falta de estratégia mobile-first, largura fixa, ausência de wrap/minmax |
| Flash de conteúdo errado (FOUC) | Client Component fazendo trabalho que pertence ao servidor, `use client` desnecessário |
| Erro só em produção | Diferença de comportamento dev/prod (cache, prerender) não coberta pelo build local |
| Teste flakey | Espera fixa em vez de espera por condição, estado compartilhado entre testes |

### Etapa 3: Implementação das Correções

Para cada bug, seguir esta sequência:

1. **Localizar o código afetado** — ler e entender os arquivos envolvidos por completo, não apenas o trecho suspeito
2. **Reproduzir o problema** — de preferência com um teste que falha (red) ou reproduzindo o fluxo no browser via [[webapp-testing]]; no mínimo, reasoning explícito sobre o fluxo que causa o bug
3. **Implementar a correção na causa raiz** — conforme os padrões do projeto (camadas, RSC boundaries, integração via SSR, componentes shadcn)
4. **Verificar tipagem** — `npx tsc --noEmit` após cada correção
5. **Executar os testes existentes** — garantir que nenhum teste quebrou com a mudança

Regras durante a implementação:

- Respeitar a regra SSR: **nenhuma correção pode introduzir chamada direta do client ao backend**
- Respeitar as camadas e regras de dependência de [[nextjs-clean-architecture]]
- Não introduzir código fora dos padrões do projeto para "resolver rápido"
- Se a correção exigir mudança arquitetural significativa, documentar a justificativa no relatório
- Se descobrir **novos bugs** durante a correção, documentá-los no `bugs.md` (mesmo formato do QA) e corrigi-los no mesmo ciclo

### Etapa 4: Testes de Regressão

Para cada bug corrigido, criar testes que:

- **Simulem o cenário original do bug** — o teste DEVE falhar se a correção for revertida
- **Validem o comportamento correto** — o teste passa com a correção aplicada
- **Cubram edge cases relacionados** — variações do mesmo problema (vazio, limites, viewports, erros do backend)

| Tipo | Quando Usar | Ferramentas |
|------|-------------|-------------|
| Teste unitário | Bug em lógica isolada (componente, hook, util, schema) | Testing Library + fixtures/mocks de Server Actions |
| Teste de integração | Bug na comunicação entre componentes/camadas (formulário + Server Action, composições) | Testing Library + interceptação dos contratos de API |
| Teste E2E | Bug visível na interface ou no fluxo completo do usuário | Playwright via [[playwright-generate-test]] |

Nomeie os testes de forma rastreável ao bug (ex.: `should keep submit disabled while pending (BUG-03)`).

### Etapa 5: Validação Completa

**5.1 Validação técnica** — execute na raiz do projeto (adapte ao package manager) e **não prossiga com falhas**:

```bash
npx tsc --noEmit
npm run lint
npm run test
npm run build
```

**5.2 Validação visual no browser** (obrigatória para bugs visuais/de fluxo) — use [[webapp-testing]]:

1. Navegue até a área afetada e reproduza o fluxo que causava o bug
2. Verifique que o comportamento está correto (incluindo o breakpoint afetado, para bugs responsivos)
3. Capture screenshot de evidência da correção
4. Verifique `browser_console_messages` — sem erros novos

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
- `npx tsc --noEmit`: SEM ERROS
- `npm run lint`: LIMPO
- `npm run test`: TODOS PASSANDO
- `npm run build`: OK
- Validação visual no browser: OK (screenshots capturados)
```

### Etapa 8: Commit, Revisão e Re-QA

1. **Commit por bugfix** no padrão [[conventional-commit]]:
   - `fix(scope): description` — o scope é o contexto da mudança (ex.: `checkout`, `auth`, `ui`)
   - Exemplo: `fix(checkout): show server validation errors on payment form`
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
6. **Sem gambiarras** — correções seguem os mesmos padrões de qualidade do código novo (skills do projeto)
7. **Regra SSR intocável** — nenhuma correção introduz chamada direta do client ao backend
8. **Validação completa antes do commit** — typecheck, lint, test, build + verificação visual para bugs de UI
9. **Commits no padrão** — `fix(scope): description` via [[conventional-commit]], um commit por bugfix
10. **Rastreabilidade no bugs.md** — status anexado sem apagar a documentação original
11. **Ciclo até aprovação** — task-reviewer até APPROVED, depois qa-validator até APROVADO
12. **Novos bugs entram no ciclo** — descobriu, documentou no `bugs.md`, corrigiu
13. **Use o Context7 MCP apenas quando** as skills e regras locais não cobrirem a dúvida (API de lib externa, comportamento de versão específica)

## Anti-Patterns de Bugfix (Evitar)

| Anti-Pattern | Correto |
|---|---|
| Esconder o erro (`catch` vazio, optional chaining defensivo) | Corrigir a causa raiz do erro |
| `suppressHydrationWarning` para "resolver" hidratação | Eliminar o render não-determinístico que causa o mismatch |
| `!important`/largura fixa para "resolver" layout | Corrigir a estratégia de layout (mobile-first, grid/flex corretos) |
| Mover fetch para o client para "simplificar" | Manter a integração via SSR — corrigir a camada servidor |
| Corrigir sem teste de regressão | Todo bug ganha teste que falha se a correção for revertida |
| Teste de regressão que passa mesmo sem a correção | Validar o teste contra o código bugado (red → green) |
| Corrigir só os bugs "importantes" | TODOS os bugs do bugs.md, em ordem de severidade |
| Marcar "Corrigido" sem rodar a suite completa | Typecheck, lint, test e build 100% antes de atualizar o bugs.md |
| Corrigir bug visual sem olhar o browser | Validar a correção visualmente com webapp-testing |
| Apagar a descrição original do bug | Anexar status mantendo a rastreabilidade |
| Um commit gigante com todas as correções | Um commit `fix(scope)` por bugfix |
| Ignorar bug novo descoberto no caminho | Documentar no bugs.md e corrigir no mesmo ciclo |
| Finalizar sem review | task-reviewer até APPROVED, depois qa-validator até APROVADO |
| Espera fixa para "resolver" teste flakey | Esperar por condição/estado, corrigir a causa da instabilidade |

## Cheat Sheet — Comandos do Bugfix

```bash
# Ler os bugs e o contexto
cat .cognup/specs/[feature]/report/bugs.md
cat .cognup/specs/[feature]/report/qa-report.md

# Reproduzir com teste focado (red)
npm run test -- --run [nome-do-teste-ou-arquivo]

# Validação completa (gate antes do commit)
npx tsc --noEmit
npm run lint
npm run test
npm run build

# Executar specs Playwright da área afetada
npx playwright test [spec-do-fluxo]

# Commit por bugfix (Conventional Commits)
git add <files>
git commit -m "fix(checkout): show server validation errors on payment form"
```

## Idioma

Planejamento, relatório e atualizações do `bugs.md` em **Português (Brasil)**. Código, nomes de testes e mensagens de commit em **inglês**.

## Manutenção de Memória

Ao longo dos bugfixes, registre padrões para acelerar correções futuras:

- Causas raiz recorrentes (ex.: hidratação por render não-determinístico, cache de fetch incorreto, contrato de API divergente)
- Componentes mais propensos a bugs e seus modos de falha
- Lacunas sistemáticas de teste que deixam bugs passarem (edge cases, breakpoints não cobertos)
- Correções que exigiram mudança arquitetural e suas justificativas
- Testes flakey conhecidos e suas causas
