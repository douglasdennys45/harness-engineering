---
name: task-reviewer-best-practices
description: Aplica melhores práticas de Task Review para revisão sistemática de tarefas concluídas em aplicações frontend Angular com SSR. Cobre identificação da tarefa em `.cognup/specs/[feature]/tasks/[num]_task.md`, análise de diff (`git diff`/`git log`), validação contra boas práticas de Angular (signals, zoneless, control flow, @defer, SSR/hydration, render modes), arquitetura do projeto (skill angular-clean-architecture — camadas, Native Federation, boundaries Nx), regra BFF (browser nunca chama o backend interno), TypeScript strict, Angular Material, responsividade, acessibilidade (WCAG), performance (Core Web Vitals), testes (Vitest, Angular Testing Library, Playwright), execução de verificações automatizadas (typecheck, lint, test, build via Nx), classificação de problemas (Crítico/Major/Minor/Positivo) e geração ou re-escrita do artefato em `.cognup/specs/[feature]/tasks/reviews/[num]_task_review.md` em Português (BR) com status em inglês (APPROVED/APPROVED WITH OBSERVATIONS/CHANGES REQUESTED). Use sempre que uma tarefa frontend foi concluída via workflow `run-task.md` e precisa ser revisada antes do merge, ao gerar artefato de revisão, ao reexecutar review após correções, ou ao avaliar aderência de uma implementação Angular aos padrões do projeto.
---

# Task Reviewer Best Practices — Revisão Sistemática de Tarefas Frontend Angular

Skill que aplica o processo metódico de **Task Review** sobre tarefas concluídas usando o workflow `run-task.md` em aplicações frontend Angular com SSR. Combina identificação da tarefa, análise de diff, validação contra os padrões do projeto, verificações automatizadas, classificação de problemas e geração do artefato `[num]_task_review.md` rastreável.

Combina com [[angular-best-practices]] (padrões Angular) e [[angular-clean-architecture]] (arquitetura, camadas e federação do projeto) — sempre —, e conforme o escopo do diff com [[angular-material]] (componentes UI), [[frontend-design]] (qualidade visual), [[responsive-design]] (responsividade), [[web-design-guidelines]] (guidelines de interface), [[webapp-testing]] (validação visual no browser) e [[playwright-generate-test]] (testes E2E).

## Princípio Fundamental

> **Revisão completa, justa e rastreável.** O revisor identifica a tarefa, lê integralmente os arquivos modificados (não apenas os diffs), valida contra os padrões do projeto, executa verificações automatizadas e gera o artefato `[num]_task_review.md` com problemas classificados por severidade. Cada problema referencia arquivo, linha e correção sugerida; cada boa prática observada é reconhecida.

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
| Padrões do projeto | skill [[angular-clean-architecture]] + `@.claude/rules` (se existirem) |

**REGRA CRÍTICA:** o artefato de revisão é gravado SEMPRE em `.cognup/specs/[nome-da-funcionalidade]/tasks/reviews/[num]_task_review.md` — nunca ao lado do arquivo da task, nunca na raiz da feature. Se a pasta não existir, crie antes: `mkdir -p .cognup/specs/[nome-da-funcionalidade]/tasks/reviews`.

## Quando Aplicar

- Quando uma tarefa frontend foi finalizada via workflow `run-task.md` e o usuário pediu revisão
- Quando o usuário cita um número de tarefa específico (`task 5`, `task 12`) para revisar
- Proativamente após uma implementação significativa em Angular ser concluída
- Em re-revisão: quando `tasks/reviews/[num]_task_review.md` já existe com status diferente de APPROVED
- Antes de abrir Pull Request de uma tarefa concluída
- Ao validar aderência de código novo às convenções do projeto (camadas, federação, BFF, Material, responsividade, a11y)

## Missão do Revisor

Você é um revisor de frontend de elite com domínio profundo em **Angular moderno (signals, zoneless, standalone), SSR/hydration com @angular/ssr, Native Federation (microfrontends), TypeScript e padrões modernos de design frontend**. Seu olhar é meticuloso para performance, acessibilidade, SEO e qualidade de UI/UX.

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
- Para arquivos grandes, focar nos componentes/classes alterados mas verificar imports, tipos e providers relacionados
- Identificar quais **projetos Nx** foram tocados (`npx nx show projects --affected`) — o escopo do review inclui os efeitos em apps que consomem as libs alteradas

### Etapa 3: Preparação Técnica

Antes de iniciar a revisão detalhada, carregue o contexto técnico do projeto:

1. **Consultar [[angular-clean-architecture]]** — camadas por domínio, regras de dependência, federação, regra BFF, nomenclatura — e `@.claude/rules` (se existirem)
2. **Consultar `CLAUDE.md`** (se existir) — convenções específicas e comandos de build/test
3. **Carregar [[angular-best-practices]]** como referência Angular (sempre)
4. **Carregar [[angular-material]]** se o diff toca componentes UI
5. **Carregar as skills de escopo aplicáveis ao diff**: [[frontend-design]] e [[web-design-guidelines]] se tocou UI visual; [[responsive-design]] se tocou layouts; [[webapp-testing]] para validação visual; [[playwright-generate-test]] se tocou testes E2E
6. **Não carregar skills fora do escopo** — ex.: não carregue responsive-design para uma task que só cria entidades. Registre no artefato quais foram usadas e quais foram N/A

A revisão precisa ser fiel **ao projeto que você está revisando** — não imponha padrões que o projeto não adota. Mas valide rigorosamente os padrões que o projeto **declarou** adotar.

### Etapa 4: Conduzir a Revisão

Revise o código contra os critérios abaixo, agrupados por área. Para cada violação, registre **arquivo + linha + descrição + correção sugerida**.

#### 4.1 Padrões Gerais de Código

- **Idioma do código**: tudo em inglês (variáveis, funções, componentes, comentários)
- **Nomenclatura**: camelCase para funções/variáveis, PascalCase para classes/interfaces, kebab-case para arquivos/diretórios, sufixos da skill de arquitetura (Dto, Mapper, Repository, Store)
- **Nomes claros**: sem abreviações; funções iniciam com verbo e executam uma única ação
- **Constantes**: sem números mágicos — usar constantes nomeadas
- **Parâmetros**: máximo de 3 (usar objeto se mais)
- **Condicionais**: máximo de 2 níveis de aninhamento, preferir early returns
- **Tamanho**: métodos até ~50 linhas; componentes até ~300 linhas — extrair quando ultrapassar
- **TypeScript strict**: `const` sobre `let`, nunca `var`, **nunca `any`**; tipos explícitos em APIs públicas
- **Imports**: path aliases do projeto; sem deep imports em libs; sem dependências circulares

#### 4.2 Angular Moderno e SSR

> Referência: [[angular-best-practices]].

| Critério | Verificar |
|---|---|
| **Standalone + OnPush** | componentes standalone com `ChangeDetectionStrategy.OnPush`; sem NgModules novos |
| **Signals** | `input()`/`output()`/`model()`; estado com `signal`/`computed`; sem `effect()` para derivar estado; sem subscribe manual setando variável de classe |
| **Zoneless** | sem dependência de Zone.js (`NgZone.run`, `onStable`); callbacks externos escrevem em signals |
| **Control flow** | `@if`/`@for`/`@switch` com `track` estável; `@empty` para vazios |
| **Regra BFF** | leitura inicial via HttpClient no SSR (transfer cache); mutações via BFF do domínio; **nenhum código do browser chama o backend interno** (sem URL de backend interno no bundle client, sem SDKs de backend, sem secrets) |
| **Render modes** | rotas novas declaradas em `app.routes.server.ts` (Server/Prerender/Client) com justificativa alinhada à techspec |
| **SSR-safety** | render determinístico (sem `Date.now()`/`Math.random()`/browser APIs no render); browser-only atrás de `afterNextRender()`; auth via cookies, não localStorage |
| **Hydration** | `provideClientHydration(withEventReplay(), withIncrementalHydration())`; sem mismatches; `ngSkipHydration` apenas justificado |
| **@defer** | conteúdo pesado/abaixo da dobra com `@defer` + placeholder dimensionado; componentes federados sempre com `@defer` + fallback |
| **Streaming/estados** | loading (skeleton), erro (retry), vazio e populado implementados |
| **Error handling** | rota 404 wildcard; falha de lazy load/remote com fallback; sem `catch` vazio |
| **HTTP** | `HttpClient`/`httpResource` (nunca fetch manual em repository); interceptors para auth/erros; sem waterfalls evitáveis |
| **Formulários** | Reactive Forms tipados; validação client E servidor; erros associados aos campos; estado pending no submit |

#### 4.3 Arquitetura do Projeto

> Referência: [[angular-clean-architecture]].

- Regras de dependência entre camadas respeitadas (domain → application → infra → presentation)
- Artefatos no lugar correto: entidades/contratos em domain, use cases em application, repositories/interceptors/stores em infra, componentes/páginas em presentation
- Nomenclatura e sufixos conforme a skill (Dto, Mapper, Repository, Store)
- Composition root apenas em `app.config.ts` (providers ligando token → implementação)
- Sem imports entre domínios fora dos canais permitidos (inputs via shell, Event Bus tipado)
- Exposição federada apenas via `presentation/exposed/`; componentes expostos auto-contidos
- Tags e boundaries Nx respeitados (o lint acusa; confirme que não foram suprimidos com `eslint-disable`)

#### 4.4 Padrões de Componente

- Composição sobre herança (`ng-content`, componentes pequenos)
- Estado colocalizado próximo de onde é usado; DI para dependências, `inject()` sobre construtor
- Inputs explícitos e tipados; sem lógica pesada em funções chamadas no template
- Keys estáveis (`track item.id`) em listas — nunca `$index` em listas dinâmicas
- Pipes puros para transformação; `computed()` para derivação

#### 4.5 Performance e Core Web Vitals

- **LCP**: imagens acima da dobra com `NgOptimizedImage` + `priority`; fontes otimizadas
- **INP**: handlers leves; trabalho pesado diferido (`@defer`, `afterNextRender`)
- **CLS**: imagens com dimensões; placeholders de `@defer` com tamanho estável; sem layout shifts
- **Bundle**: lazy loading por rota; `@defer` para libs pesadas; budgets respeitados
- **Navegação**: `routerLink` para navegação interna
- Transfer cache aproveitado (sem dupla busca servidor+client do mesmo dado)

#### 4.6 UI/UX e Design

> Referência: [[frontend-design]] e [[web-design-guidelines]] (busque as guidelines atualizadas via WebFetch de `https://raw.githubusercontent.com/vercel-labs/web-interface-guidelines/main/command.md`; reporte findings no formato `arquivo:linha`).

- Estilos via design tokens do tema (`var(--mat-sys-*)`); sem hex hardcoded, sem CSS inline
- Design system coeso (cores, espaçamento, tipografia); dark mode funcionando via tokens
- Estados de **loading, erro e vazio** implementados com feedback visual — não apenas happy path
- Micro-interações: hover, focus e active definidos; transições suaves; `prefers-reduced-motion` respeitado
- O design evita estéticas genéricas de IA (gradientes roxos em fundo branco, layouts previsíveis)

#### 4.7 Angular Material (quando aplicável)

> Referência: [[angular-material]].

- Hierarquia de reuso respeitada: `libs/ui` → Material/CDK → custom
- Sem `::ng-deep` nem seletores `.mdc-*`/`.mat-mdc-*`; customização via `overrides`/tokens
- Formulários: `mat-label` + `mat-error` + `mat-hint`; appearance única do projeto
- Dialogs com título e ações; dados via `MAT_DIALOG_DATA` tipado; abertos apenas em handlers (nunca no render SSR)
- Tabelas com `trackBy`, estado vazio e estratégia mobile
- Ícones com `aria-hidden`/`aria-label` corretos

#### 4.8 Responsividade (quando aplicável)

> Referência: [[responsive-design]].

- Mobile-first: estilos base para mobile, breakpoints ascendentes
- Container queries (`@container`) para componentes que vivem em contextos variados
- Tipografia fluida com `clamp()`; grids com `auto-fit`/`auto-fill` + `minmax()`
- `dvh` em vez de `vh` para alturas full-screen mobile
- Touch targets mínimos de 44x44px; navegação mobile com `aria-expanded`/`aria-controls`
- Tabelas com scroll horizontal ou layout de cards em telas pequenas; **nenhum overflow horizontal**

#### 4.9 Acessibilidade (a11y)

- HTML semântico (`<header>`, `<main>`, `<nav>`, `<section>`); headings em hierarquia lógica
- ARIA apenas onde HTML semântico/Material não basta; `aria-label` em ícones interativos
- Navegação por teclado completa; focus trapping em overlays (CDK); `:focus-visible` estilizado
- Contraste WCAG AA (4.5:1 texto, 3:1 elementos gráficos)
- `alt` descritivo em imagens (decorativas com `alt=""`)
- Formulários: labels associados aos inputs; erros anunciados programaticamente
- Título por rota; foco gerenciado na navegação

#### 4.10 Testes

> Referência: [[webapp-testing]] e [[playwright-generate-test]].

| Critério | Verificar |
|---|---|
| Cobertura | código novo tem testes correspondentes (exigidos pela task) |
| Framework | Vitest para unitários; Angular Testing Library para componentes; Playwright para E2E |
| Independência | cada teste funciona isoladamente; padrão AAA |
| Foco no usuário | testes validam o que o usuário vê/faz (queries por role/label), não a implementação |
| Nomes | descritivos, iniciando com "should" |
| Mocks | repositories mockados pelo **token**; MSW para infra; fixtures baseadas nos contratos de API da techspec |
| E2E | fluxos críticos cobertos contra o app em SSR (HTML do servidor validado) |

### Etapa 5: Verificação Automatizada

Execute todas as ferramentas (via Nx, nos projetos afetados) e registre saídas para anexar ao artefato:

```bash
# 1. Type checking — bloqueante
npx nx affected -t typecheck   # fallback: npx tsc --noEmit -p tsconfig.base.json

# 2. Linter (inclui boundaries Nx)
npx nx affected -t lint

# 3. Testes
npx nx affected -t test

# 4. Build de produção (browser + server bundles) — bloqueante
npx nx affected -t build
```

**Validação visual** (quando a task tem entrega de UI): use [[webapp-testing]] para capturar screenshots da interface implementada, verificar interações e validar estados de loading/erro/vazio no browser.

**Regras:**

- Erro de typecheck → **Crítico** automático
- Falha no build → **Crítico** automático
- Qualquer teste falhando → **Crítico**
- Erros de lint (incluindo violação de boundary) → **Major** (warnings: **Minor**); violação de boundary de camada/domínio → **Crítico**
- Erros no console do browser durante a validação visual (incluindo NG05xx de hydration) → **Major** (ou Crítico se quebra o fluxo)

### Etapa 6: Classificar Problemas

Cada problema encontrado recebe **uma** classificação:

#### **CRÍTICO** (bloqueante para merge)

- Bugs funcionais; funcionalidade quebrada (não compila, build falha, testes falham)
- **Browser chamando o backend interno diretamente** (violação da regra BFF) ou secret exposto no bundle client
- **Mismatches de hydration** (NG0500/NG0501) ou render não-determinístico no caminho SSR
- Problemas de segurança (XSS via `innerHTML`/`bypassSecurityTrust*` injustificado, dados sensíveis no client)
- Mutação sem validação no servidor/BFF
- Tipos `any` em código novo
- Rota nova sem render mode declarado ou tratamento de erro ausente em fluxo crítico
- Bloqueadores de acessibilidade (fluxo principal inoperável por teclado/screen reader)
- Violações das regras de dependência entre camadas ou entre domínios ([[angular-clean-architecture]]), incluindo `eslint-disable` de boundary

#### **MAJOR** (deve ser corrigido)

- Padrões legados em código novo (`*ngIf`/`*ngFor`, `@Input()`/`@Output()` decorators, NgModule, subscribe manual com variável de classe)
- Componente sem OnPush; `effect()` derivando estado; estado derivável armazenado
- Data fetching com waterfalls evitáveis; transfer cache desperdiçado (dupla busca)
- Testes ausentes para código novo
- Estados de loading/erro/vazio não implementados
- Regressões de performance (imagem sem `NgOptimizedImage`, lib pesada sem `@defer`, budget estourado)
- Lacunas de SEO (título/meta ausentes em página pública; conteúdo indexável em `RenderMode.Client` sem justificativa)
- Problemas de a11y não bloqueantes (contraste, labels, foco)
- Uso incorreto de Angular Material (`::ng-deep`, hex hardcoded, dialog sem título)
- Violações das Web Interface Guidelines
- Overflow horizontal ou touch targets < 44px em mobile
- Violações de padrões do projeto (nomenclatura, camada errada, Dto/Mapper ausente)

#### **MINOR** (sugestão)

- Sugestões de estilo; otimizações opcionais
- Inconsistências visuais menores; comentários redundantes
- Warnings de lint

#### **POSITIVO** (reconhecer)

- SSR-safety e render modes bem escolhidos; incremental hydration bem aplicada
- Signals e derivação de estado exemplares
- Estados de loading/erro/vazio completos
- Acessibilidade bem implementada; responsividade sólida
- Boa cobertura de testes incluindo edge cases
- Componentes Material bem compostos; design de qualidade

### Etapa 7: Gerar/Atualizar o Artefato de Revisão

1. `mkdir -p .cognup/specs/[nome-da-funcionalidade]/tasks/reviews`
2. Crie ou **sobrescreva** `.cognup/specs/[nome-da-funcionalidade]/tasks/reviews/[num]_task_review.md`

#### Formato Obrigatório

```markdown
# Revisão: Task [num] - [Título da Task]

**Revisor**: AI Frontend Reviewer
**Data**: [AAAA-MM-DD]
**Arquivo da task**: .cognup/specs/[nome-da-funcionalidade]/tasks/[num]_task.md
**Status**: [APPROVED | APPROVED WITH OBSERVATIONS | CHANGES REQUESTED]
**Re-revisão**: [Não | Sim (após correções)]

## Resumo

[Resumo breve do que foi implementado e avaliação geral de qualidade]

## Skills Técnicas Utilizadas

| Skill | Utilizada | Observações |
|-------|-----------|-------------|
| angular-best-practices | Sim | [Breve nota] |
| angular-clean-architecture | Sim | [Breve nota] |
| angular-material | [Sim/N/A] | [Breve nota] |
| frontend-design | [Sim/N/A] | [Breve nota] |
| responsive-design | [Sim/N/A] | [Breve nota] |
| web-design-guidelines | [Sim/N/A] | [Breve nota] |
| webapp-testing | [Sim/N/A] | [Breve nota] |
| playwright-generate-test | [Sim/N/A] | [Breve nota] |

## Arquivos Revisados

| Arquivo | Status | Problemas |
|---------|--------|-----------|
| [caminho do arquivo] | [OK / Problemas / Crítico] | [contagem] |

## Verificações Automatizadas

| Verificação | Comando | Resultado | Detalhes |
|---|---|---|---|
| Type check | `npx nx affected -t typecheck` | OK / FALHOU | ... |
| Lint (+ boundaries) | `npx nx affected -t lint` | OK / FALHOU | ... |
| Testes | `npx nx affected -t test` | OK / FALHOU | ... |
| Build (browser + server) | `npx nx affected -t build` | OK / FALHOU | ... |
| Validação visual | webapp-testing | OK / FALHOU / N/A | ... |

## Problemas Encontrados

### Problemas Críticos

[Liste cada problema crítico com: arquivo, linha, descrição e correção sugerida com exemplo de código]
[Se nenhum: "Nenhum problema crítico encontrado."]

### Problemas Major

[Mesmo formato]

### Problemas Minor

[Mesmo formato]

## Destaques Positivos

[Liste o que foi bem feito — SSR-safety, signals, a11y, responsividade, testes, etc.]

## Conformidade com Padrões

| Padrão | Status |
|--------|--------|
| Angular moderno e SSR (angular-best-practices) | [OK / Atenção / Crítico] |
| Regra BFF (browser não chama backend interno) | [OK / Crítico] |
| Render modes e hydration | [OK / Atenção / Crítico] |
| Arquitetura (angular-clean-architecture) | [OK / Atenção / Crítico] |
| Boundaries Nx (camadas e domínios) | [OK / Crítico] |
| TypeScript / Padrões de Código | [OK / Atenção / Crítico] |
| Performance e Core Web Vitals | [OK / Atenção / Crítico] |
| UI/UX e Design | [OK / Atenção / Crítico / N/A] |
| Angular Material | [OK / Atenção / Crítico / N/A] |
| Responsividade | [OK / Atenção / Crítico / N/A] |
| Acessibilidade (a11y) | [OK / Atenção / Crítico] |
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

## Diretrizes Operacionais

1. **Caminho de saída fixo** — o review vai SEMPRE em `.cognup/specs/[feature]/tasks/reviews/[num]_task_review.md`
2. **Ler antes de julgar** — leia integralmente os arquivos modificados, não apenas o diff
3. **Seja específico** — sempre referencie arquivo e número da linha
4. **Forneça solução** — não apenas aponte; sugira correção com exemplo de código
5. **Verifique testes** — código novo sem teste correspondente é **Major** no mínimo
6. **Execute as verificações** — typecheck, lint, test, build via Nx (+ validação visual quando há UI)
7. **Cheque contra requisitos** — o implementado corresponde ao solicitado em `[num]_task.md` e no `techspec.md`?
8. **Valide a regra BFF** — nenhuma chamada direta do browser ao backend interno passa despercebida
9. **Valide SSR-safety** — render determinístico, render modes declarados, hydration sem mismatch
10. **Aplique as skills conforme o escopo** — registre no artefato quais foram usadas e quais foram N/A
11. **Reconheça o bom** — destaques positivos são parte do artefato
12. **Re-revisão sobrescreve** — não acumule reviews; o arquivo sempre reflete o estado mais recente
13. **Idioma** — artefato em Português (BR); exemplos de código e valores de Status em inglês
14. **Atualize memória** — registre padrões recorrentes do projeto para revisões futuras

## Anti-Patterns de Task Review (Evitar)

| Anti-Pattern | Correto |
|---|---|
| "LGTM 👍" sem revisar | Toda revisão produz artefato `[num]_task_review.md` |
| Gravar o review fora de `tasks/reviews/` | Caminho fixo: `.cognup/specs/[feature]/tasks/reviews/` |
| Status em português (APROVADO) | Status em inglês (APPROVED) — é o que o workflow verifica |
| Revisar apenas o diff (sem ler arquivos completos) | Ler contexto completo dos arquivos modificados |
| "Tudo certo" sem rodar verificações | Sempre executar typecheck, lint, test e build |
| Aprovar com build ou teste falhando | Falha de build/teste = Crítico, CHANGES REQUESTED |
| Ignorar chamada ao backend no client "porque funciona" | Browser chamando backend interno = Crítico (regra BFF) |
| Ignorar warning NG05xx de hydration | Mismatch de hydration = Crítico |
| Aceitar `eslint-disable` de boundary | Violação de boundary = Crítico |
| Apontar problema sem solução | Sempre sugerir correção com exemplo de código |
| Ignorar testes ausentes | Código novo sem teste = Major |
| Não classificar problemas | Toda observação tem severidade explícita |
| Manter review antigo após correções | Re-revisão **sobrescreve** o arquivo |
| Importar padrões de outro projeto | Validar contra os padrões **deste** projeto (skills + rules) |
| Carregar todas as skills sempre | Carregar apenas as aplicáveis ao escopo do diff |
| Só apontar erros, ignorar acertos | Sempre incluir seção "Destaques Positivos" |
| Revisar UI sem olhar o browser | Usar webapp-testing para validação visual quando há entrega de UI |

## Cheat Sheet — Comandos do Reviewer

```bash
# Identificar tarefa
ls .cognup/specs/*/tasks/*_task.md
ls -la .cognup/specs/[feature]/tasks/reviews/*_task_review.md 2>/dev/null   # detectar re-revisão

# Identificar arquivos alterados (Gitflow: base develop)
git diff develop...HEAD --stat
git log develop...HEAD --oneline
git status --short
npx nx show projects --affected

# Ler diff completo de um arquivo
git diff develop...HEAD -- apps/billing/src/app/presentation/pages/invoices.component.ts

# Verificações automatizadas
npx nx affected -t typecheck
npx nx affected -t lint
npx nx affected -t test
npx nx affected -t build

# Candidatos a violação da regra BFF / SSR-safety no diff
git diff develop...HEAD -G"fetch\(|localStorage|window\." --name-only
git diff develop...HEAD -G"new Date\(\)|Math.random" --name-only

# Padrões legados em código novo
git diff develop...HEAD -G"\*ngIf|\*ngFor|@Input\(|@Output\(|::ng-deep" --name-only

# Identificar testes do diff
git diff develop...HEAD --name-only -- '*.spec.ts'

# Criar a pasta do review e gravar o artefato
mkdir -p .cognup/specs/[feature]/tasks/reviews
```

## Idioma

Artefato de revisão em **Português (Brasil)**. Exemplos de código e valores de **Status** permanecem em **inglês**.

## Manutenção de Memória do Revisor

Ao longo das revisões, registre padrões e violações recorrentes para construir conhecimento institucional sobre o codebase:

- Problemas recorrentes de SSR/hydration entre tarefas
- Padrões de SSR-safety e render modes efetivamente em uso
- Violações recorrentes da regra BFF (browser chamando backend interno)
- Padrões e regressões de performance
- Aderência ao design system (tokens Material) e consistência de UI
- Lacunas e padrões de acessibilidade
- Padrões de uso do Angular Material e violações recorrentes
- Organização de arquivos e convenções Nx/Angular adotadas
- Dependências e bibliotecas que o projeto utiliza

Essas notas tornam revisões futuras mais rápidas e consistentes.
