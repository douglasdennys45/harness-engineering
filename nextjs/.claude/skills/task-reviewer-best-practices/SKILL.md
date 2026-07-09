---
name: task-reviewer-best-practices
description: Aplica melhores práticas de Task Review para revisão sistemática de tarefas concluídas em aplicações frontend Next.js. Cobre identificação da tarefa em `.cognup/specs/[feature]/tasks/[num]_task.md`, análise de diff (`git diff`/`git log`), validação contra boas práticas de Next.js (RSC boundaries, SSR/SSG/ISR, data patterns, hidratação), arquitetura do projeto (skill nextjs-clean-architecture), regra de integração via SSR (client nunca chama o backend), React patterns, TypeScript strict, shadcn/ui, responsividade, acessibilidade (WCAG), performance (Core Web Vitals), testes (Testing Library, Playwright), execução de verificações automatizadas (`tsc --noEmit`, lint, test, build), classificação de problemas (Crítico/Major/Minor/Positivo) e geração ou re-escrita do artefato em `.cognup/specs/[feature]/tasks/reviews/[num]_task_review.md` em Português (BR) com status em inglês (APPROVED/APPROVED WITH OBSERVATIONS/CHANGES REQUESTED). Use sempre que uma tarefa frontend foi concluída via workflow `run-task.md` e precisa ser revisada antes do merge, ao gerar artefato de revisão, ao reexecutar review após correções, ou ao avaliar aderência de uma implementação Next.js aos padrões do projeto.
---

# Task Reviewer Best Practices — Revisão Sistemática de Tarefas Frontend Next.js

Skill que aplica o processo metódico de **Task Review** sobre tarefas concluídas usando o workflow `run-task.md` em aplicações frontend Next.js. Combina identificação da tarefa, análise de diff, validação contra os padrões do projeto, verificações automatizadas, classificação de problemas e geração do artefato `[num]_task_review.md` rastreável.

Combina com [[next-best-practices]] (padrões Next.js) e [[nextjs-clean-architecture]] (arquitetura e camadas do projeto) — sempre —, e conforme o escopo do diff com [[shadcn]] (componentes UI), [[frontend-design]] (qualidade visual), [[responsive-design]] (responsividade), [[web-design-guidelines]] (guidelines de interface), [[webapp-testing]] (validação visual no browser) e [[playwright-generate-test]] (testes E2E).

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
| Padrões do projeto | skill [[nextjs-clean-architecture]] + `@.claude/rules` (se existirem) |

**REGRA CRÍTICA:** o artefato de revisão é gravado SEMPRE em `.cognup/specs/[nome-da-funcionalidade]/tasks/reviews/[num]_task_review.md` — nunca ao lado do arquivo da task, nunca na raiz da feature. Se a pasta não existir, crie antes: `mkdir -p .cognup/specs/[nome-da-funcionalidade]/tasks/reviews`.

## Quando Aplicar

- Quando uma tarefa frontend foi finalizada via workflow `run-task.md` e o usuário pediu revisão
- Quando o usuário cita um número de tarefa específico (`task 5`, `task 12`) para revisar
- Proativamente após uma implementação significativa em Next.js ser concluída
- Em re-revisão: quando `tasks/reviews/[num]_task_review.md` já existe com status diferente de APPROVED
- Antes de abrir Pull Request de uma tarefa concluída
- Ao validar aderência de código novo às convenções do projeto (camadas, SSR, shadcn, responsividade, a11y)

## Missão do Revisor

Você é um revisor de frontend de elite com domínio profundo em **Next.js (App Router), React Server Components, SSR/SSG/ISR, TypeScript e padrões modernos de design frontend**. Seu olhar é meticuloso para performance, acessibilidade, SEO e qualidade de UI/UX.

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
- Para arquivos grandes, focar nos componentes/funções alterados mas verificar imports, tipos e providers relacionados

### Etapa 3: Preparação Técnica

Antes de iniciar a revisão detalhada, carregue o contexto técnico do projeto:

1. **Consultar [[nextjs-clean-architecture]]** — camadas por domínio, regras de dependência, nomenclatura, DTOs/mappers, stores — e `@.claude/rules` (se existirem)
2. **Consultar `CLAUDE.md`** (se existir) — convenções específicas e comandos de build/test
3. **Carregar [[next-best-practices]]** como referência Next.js (sempre)
4. **Verificar contexto shadcn** (se `components.json` existir): execute `npx shadcn@latest info --json` e carregue [[shadcn]]
5. **Carregar as skills de escopo aplicáveis ao diff**: [[frontend-design]] e [[web-design-guidelines]] se tocou UI visual; [[responsive-design]] se tocou layouts; [[webapp-testing]] para validação visual; [[playwright-generate-test]] se tocou testes E2E
6. **Não carregar skills fora do escopo** — ex.: não carregue responsive-design para uma task que só cria schemas Zod. Registre no artefato quais foram usadas e quais foram N/A

A revisão precisa ser fiel **ao projeto que você está revisando** — não imponha padrões que o projeto não adota. Mas valide rigorosamente os padrões que o projeto **declarou** adotar.

### Etapa 4: Conduzir a Revisão

Revise o código contra os critérios abaixo, agrupados por área. Para cada violação, registre **arquivo + linha + descrição + correção sugerida**.

#### 4.1 Padrões Gerais de Código

- **Idioma do código**: tudo em inglês (variáveis, funções, componentes, comentários)
- **Nomenclatura**: camelCase para funções/variáveis, PascalCase para componentes/interfaces, kebab-case para arquivos/diretórios
- **Nomes claros**: sem abreviações; funções iniciam com verbo e executam uma única ação
- **Constantes**: sem números mágicos — usar constantes nomeadas
- **Parâmetros**: máximo de 3 (usar objeto se mais)
- **Condicionais**: máximo de 2 níveis de aninhamento, preferir early returns
- **Tamanho**: métodos até ~50 linhas; componentes até ~300 linhas — extrair quando ultrapassar
- **TypeScript strict**: `const` sobre `let`, nunca `var`, **nunca `any`**; tipos explícitos para props
- **Imports**: `import`/`export`, nunca `require`; sem dependências circulares

#### 4.2 Next.js e Renderização Server-Side

> Referência: [[next-best-practices]].

| Critério | Verificar |
|---|---|
| **Server vs Client** | Server Components por padrão; `'use client'` apenas quando necessário (event handlers, hooks, APIs do browser) |
| **Integração via SSR** | leitura via `fetch` em Server Components; mutações via Server Actions; **nenhum Client Component chama o backend** (sem `fetch`/`useEffect` para APIs, sem SDKs de backend no client) |
| **Data fetching** | caching adequado (`revalidate`, `cache: 'no-store'`/`'force-cache'`); sem waterfalls (paralelizar com `Promise.all`) |
| **Server Actions** | `'use server'` correto; input **validado no servidor** (Zod) |
| **Streaming** | `loading.tsx` e `<Suspense>` nos pontos de espera |
| **Error handling** | `error.tsx` e `not-found.tsx` nas rotas críticas |
| **Async APIs (Next 15+)** | `params`, `searchParams`, `cookies()`, `headers()` tratados como assíncronos |
| **Metadata/SEO** | `generateMetadata` ou export `metadata` por página; título, descrição e OG preenchidos |
| **Route Handlers** | `route.ts` com métodos HTTP corretos e `NextResponse` com status apropriados |
| **Static vs Dynamic** | páginas estáticas usam `generateStaticParams`; rotas dinâmicas justificadas |
| **Hidratação** | sem mismatches servidor/cliente (datas, random, browser APIs no render) |
| **File conventions** | `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx` conforme App Router |

#### 4.3 Arquitetura do Projeto

> Referência: [[nextjs-clean-architecture]].

- Regras de dependência entre camadas respeitadas (domain → application → infra → presentation)
- Artefatos no lugar correto: entidades/tipos em domain, use cases em application, integração SSR/DTOs/mappers em infra, componentes/páginas/stores em presentation
- Nomenclatura e sufixos conforme a skill (DTO, Mapper, Repository)
- Sem imports entre domínios fora dos canais permitidos (props, eventos tipados)

#### 4.4 React Patterns

- Apenas componentes funcionais; composição sobre herança (`children`, slots)
- Estado colocalizado próximo de onde é usado; Context API com providers próximos aos consumers
- Props explícitas e tipadas; sem spread de props
- Custom hooks com prefixo `use` para lógica reutilizável
- Memoização (`useMemo`/`useCallback`) apenas com ganho mensurável
- Keys estáveis em listas — nunca index em listas dinâmicas

#### 4.5 Performance e Core Web Vitals

- **LCP**: imagens acima da dobra com `priority` no `next/image`; fontes via `next/font`
- **INP**: handlers pesados diferidos; sem JS bloqueante no client
- **CLS**: imagens com `width`/`height` ou `fill` + `sizes`; sem layout shifts
- **Bundle**: boundaries `'use client'` granulares; `next/dynamic` para componentes pesados; lazy loading abaixo da dobra
- **Navegação**: `<Link>` do Next.js para prefetch automático
- `next/image` em vez de `<img>`; `next/font` sem FOIT/FOUT

#### 4.6 UI/UX e Design

> Referência: [[frontend-design]] e [[web-design-guidelines]] (busque as guidelines atualizadas via WebFetch de `https://raw.githubusercontent.com/vercel-labs/web-interface-guidelines/main/command.md`; reporte findings no formato `arquivo:linha`).

- Tailwind consistente: `cn()`/`clsx()` para classes condicionais; sem CSS inline
- Design system coeso (cores, espaçamento, tipografia); dark mode com variantes `dark:` quando o projeto suporta
- Estados de **loading, erro e vazio** implementados com feedback visual — não apenas happy path
- Micro-interações: hover, focus e active definidos; transições suaves; `prefers-reduced-motion` respeitado
- O design evita estéticas genéricas de IA (gradientes roxos em fundo branco, layouts previsíveis)

#### 4.7 shadcn/ui (quando aplicável)

> Referência: [[shadcn]] (Critical Rules).

- Componentes shadcn existentes usados antes de markup customizado (`Alert`, `Empty`, `Badge`, `Skeleton`)
- Estilização: `className` para layout, não estilização; `gap-*` em vez de `space-x/y-*`; `size-*` para dimensões iguais; cores semânticas (`bg-primary`, `text-muted-foreground`)
- Formulários: `FieldGroup` + `Field`; validação com `data-invalid` + `aria-invalid`
- Composição: Items dentro de seus Groups; `asChild`/`render` para triggers; títulos obrigatórios em Dialog/Sheet/Drawer
- Ícones: `data-icon` em botões; sem classes de dimensionamento em ícones dentro de componentes

#### 4.8 Responsividade (quando aplicável)

> Referência: [[responsive-design]].

- Mobile-first: estilos base para mobile, breakpoints ascendentes (`sm:`, `md:`, `lg:`, `xl:`)
- Container queries (`@container`) para componentes que vivem em contextos variados
- Tipografia fluida com `clamp()`; grids com `auto-fit`/`auto-fill` + `minmax()`
- `next/image` com `sizes` correto; `dvh` em vez de `vh` para alturas full-screen mobile
- Touch targets mínimos de 44x44px; navegação mobile com `aria-expanded`/`aria-controls`
- Tabelas com scroll horizontal ou layout de cards em telas pequenas; **nenhum overflow horizontal**

#### 4.9 Acessibilidade (a11y)

- HTML semântico (`<header>`, `<main>`, `<nav>`, `<section>`); headings em hierarquia lógica
- ARIA apenas onde HTML semântico não basta; `aria-label` em ícones interativos
- Navegação por teclado completa; focus trapping em modais; `:focus-visible` estilizado
- Contraste WCAG AA (4.5:1 texto, 3:1 elementos gráficos)
- `alt` descritivo em imagens (decorativas com `alt=""`)
- Formulários: labels associados aos inputs; erros anunciados programaticamente

#### 4.10 Testes

> Referência: [[webapp-testing]] e [[playwright-generate-test]].

| Critério | Verificar |
|---|---|
| Cobertura | código novo tem testes correspondentes (exigidos pela task) |
| Framework | `@testing-library/react` para componentes; Playwright para E2E |
| Independência | cada teste funciona isoladamente; padrão AAA |
| Foco no usuário | testes validam o que o usuário vê/faz, não a implementação |
| Nomes | descritivos, iniciando com "should" |
| Mocks | Server Actions mockadas; fixtures baseadas nos contratos de API da techspec |
| E2E | fluxos críticos cobertos contra o app em SSR |

### Etapa 5: Verificação Automatizada

Execute todas as ferramentas (adapte ao package manager do projeto) e registre saídas para anexar ao artefato:

```bash
# 1. Type checking — bloqueante
npx tsc --noEmit

# 2. Linter
npm run lint

# 3. Testes
npm run test

# 4. Build de produção — bloqueante
npm run build
```

**Validação visual** (quando a task tem entrega de UI): use [[webapp-testing]] para capturar screenshots da interface implementada, verificar interações e validar estados de loading/erro/vazio no browser.

**Regras:**

- Erro em `tsc --noEmit` → **Crítico** automático
- Falha em `npm run build` → **Crítico** automático
- Qualquer teste falhando → **Crítico**
- Erros de lint → **Major** (warnings: **Minor**)
- Erros no console do browser durante a validação visual → **Major** (ou Crítico se quebra o fluxo)

### Etapa 6: Classificar Problemas

Cada problema encontrado recebe **uma** classificação:

#### **CRÍTICO** (bloqueante para merge)

- Bugs funcionais; funcionalidade quebrada (não compila, build falha, testes falham)
- **Client Component chamando o backend diretamente** (violação da regra SSR)
- **Mismatches de hidratação** servidor/cliente
- Problemas de segurança (XSS, dados sensíveis expostos no client, secrets em código client-side)
- Server Action sem validação de input no servidor
- Tipos `any` em código novo
- `error.tsx`/tratamento de erro ausente em fluxo crítico
- Bloqueadores de acessibilidade (fluxo principal inoperável por teclado/screen reader)
- Violações da regra de dependência entre camadas ([[nextjs-clean-architecture]])

#### **MAJOR** (deve ser corrigido)

- Componente client desnecessário (`'use client'` sem interatividade)
- Data fetching com waterfalls evitáveis; caching incorreto
- Testes ausentes para código novo
- Estados de loading/erro/vazio não implementados
- Regressões de performance (imagem sem `next/image`, fonte sem `next/font`, bundle inflado)
- Lacunas de SEO (metadata ausente em página pública)
- Problemas de a11y não bloqueantes (contraste, labels, foco)
- Uso incorreto de componentes shadcn (violação das Critical Rules)
- Violações das Web Interface Guidelines
- Overflow horizontal ou touch targets < 44px em mobile
- Violações de padrões do projeto (nomenclatura, camada errada, DTO/mapper ausente)

#### **MINOR** (sugestão)

- Sugestões de estilo; otimizações opcionais
- Memoização desnecessária ou ausente sem impacto medido
- Inconsistências visuais menores; comentários redundantes
- Warnings de lint

#### **POSITIVO** (reconhecer)

- RSC boundaries bem definidos; integração SSR exemplar
- Estados de loading/erro/vazio completos
- Acessibilidade bem implementada; responsividade sólida
- Boa cobertura de testes incluindo edge cases
- Componentes shadcn bem compostos; design de qualidade

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
| next-best-practices | Sim | [Breve nota] |
| nextjs-clean-architecture | Sim | [Breve nota] |
| shadcn | [Sim/N/A] | [Breve nota] |
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
| Type check | `npx tsc --noEmit` | OK / FALHOU | ... |
| Lint | `npm run lint` | OK / FALHOU | ... |
| Testes | `npm run test` | OK / FALHOU | ... |
| Build | `npm run build` | OK / FALHOU | ... |
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

[Liste o que foi bem feito — RSC boundaries, a11y, responsividade, testes, etc.]

## Conformidade com Padrões

| Padrão | Status |
|--------|--------|
| Next.js e SSR (next-best-practices) | [OK / Atenção / Crítico] |
| Integração via SSR (client não chama backend) | [OK / Crítico] |
| Arquitetura (nextjs-clean-architecture) | [OK / Atenção / Crítico] |
| React Patterns | [OK / Atenção / Crítico] |
| TypeScript / Padrões de Código | [OK / Atenção / Crítico] |
| Performance e Core Web Vitals | [OK / Atenção / Crítico] |
| UI/UX e Design | [OK / Atenção / Crítico / N/A] |
| shadcn/ui | [OK / Atenção / Crítico / N/A] |
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
6. **Execute as verificações** — `tsc --noEmit`, lint, test, build (+ validação visual quando há UI)
7. **Cheque contra requisitos** — o implementado corresponde ao solicitado em `[num]_task.md` e no `techspec.md`?
8. **Valide a regra SSR** — nenhuma chamada direta do client ao backend passa despercebida
9. **Aplique as skills conforme o escopo** — registre no artefato quais foram usadas e quais foram N/A
10. **Reconheça o bom** — destaques positivos são parte do artefato
11. **Re-revisão sobrescreve** — não acumule reviews; o arquivo sempre reflete o estado mais recente
12. **Idioma** — artefato em Português (BR); exemplos de código e valores de Status em inglês
13. **Atualize memória** — registre padrões recorrentes do projeto para revisões futuras

## Anti-Patterns de Task Review (Evitar)

| Anti-Pattern | Correto |
|---|---|
| "LGTM 👍" sem revisar | Toda revisão produz artefato `[num]_task_review.md` |
| Gravar o review fora de `tasks/reviews/` | Caminho fixo: `.cognup/specs/[feature]/tasks/reviews/` |
| Status em português (APROVADO) | Status em inglês (APPROVED) — é o que o workflow verifica |
| Revisar apenas o diff (sem ler arquivos completos) | Ler contexto completo dos arquivos modificados |
| "Tudo certo" sem rodar verificações | Sempre executar tsc, lint, test e build |
| Aprovar com build ou teste falhando | Falha de build/teste = Crítico, CHANGES REQUESTED |
| Ignorar `fetch` no client "porque funciona" | Client chamando backend = Crítico (regra SSR) |
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

# Ler diff completo de um arquivo
git diff develop...HEAD -- app/checkout/page.tsx

# Verificações automatizadas
npx tsc --noEmit
npm run lint
npm run test
npm run build

# Contexto shadcn (se components.json existir)
npx shadcn@latest info --json

# Identificar componentes client no diff (candidatos a violação SSR)
git diff develop...HEAD -G"use client" --name-only
git diff develop...HEAD -G"useEffect.*fetch|fetch\(" --name-only

# Identificar testes do diff
git diff develop...HEAD --name-only -- '*.test.tsx' '*.test.ts' '*.spec.ts'

# Criar a pasta do review e gravar o artefato
mkdir -p .cognup/specs/[feature]/tasks/reviews
```

## Idioma

Artefato de revisão em **Português (Brasil)**. Exemplos de código e valores de **Status** permanecem em **inglês**.

## Manutenção de Memória do Revisor

Ao longo das revisões, registre padrões e violações recorrentes para construir conhecimento institucional sobre o codebase:

- Problemas recorrentes de SSR/hidratação entre tarefas
- Padrões de boundaries entre Server e Client Components efetivamente em uso
- Violações recorrentes da regra SSR (client chamando backend)
- Padrões e regressões de performance
- Aderência ao design system e consistência de UI
- Lacunas e padrões de acessibilidade
- Padrões de uso do shadcn/ui e violações recorrentes
- Organização de arquivos e convenções Next.js adotadas
- Dependências e bibliotecas que o projeto utiliza

Essas notas tornam revisões futuras mais rápidas e consistentes.
