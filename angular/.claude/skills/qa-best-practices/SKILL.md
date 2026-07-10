---
name: qa-best-practices
description: Aplica melhores práticas de QA (Quality Assurance) para validação completa de features frontend Angular com SSR. Cobre análise da documentação (PRD/TechSpec/Tasks), análise estática (typecheck, lint com boundaries Nx, build browser+server), testes unitários e de integração (Vitest), testes E2E via browser MCP (Playwright) contra o app em SSR, validação de SSR/hydration (HTML do servidor, console sem NG05xx, regra BFF no network), verificação de acessibilidade (WCAG 2.2), verificação visual de estados (vazio/populado/erro/loading), auditoria de design (frontend-design, web-design-guidelines, angular-material) e responsividade por breakpoint (375/768/1024/1280/1536px), documentação de bugs com screenshots obrigatórios e geração de relatório de QA com evidências em `.cognup/specs/[feature]/report/`, além de changelog e versionamento SemVer pós-aprovação. Use sempre que validar uma feature frontend concluída, gerar relatório de QA, executar pipeline de qualidade, revisar implementação contra PRD/TechSpec, ou aplicar tolerância zero a bugs antes de release.
---

# QA Best Practices — Validação Completa de Features Frontend Angular com SSR

Skill que aplica o processo sistemático de QA frontend para garantir qualidade end-to-end. Combina análise estática, testes automatizados, testes E2E no browser contra o app em SSR, verificação de acessibilidade, auditoria visual/design/responsividade e documentação com evidências em um pipeline com **tolerância zero a bugs**.

Combina com [[angular-best-practices]] (padrões Angular/SSR), [[webapp-testing]] (testes no browser com Playwright) e [[playwright-generate-test]] (geração de testes E2E) — sempre —, e conforme o escopo com [[angular-material]] (componentes UI), [[frontend-design]] (qualidade visual), [[responsive-design]] (responsividade) e [[web-design-guidelines]] (guidelines de interface). A arquitetura de referência é a skill [[angular-clean-architecture]]. Changelog e commits seguem [[changelog-automation]] e [[conventional-commit]].

## Princípio Fundamental

> **TOLERÂNCIA ZERO A BUGS.** Uma feature só recebe status **APROVADO** quando ZERO bugs são encontrados — independente da severidade (Crítica, Alta, Média ou Baixa). Qualquer bug encontrado, por menor que seja, resulta automaticamente em **REPROVADO**.

Não existe:
- Aprovação condicional
- Aprovação com ressalvas
- Aprovação com bugs pendentes de correção
- Distinção entre bugs "bloqueantes" e "não bloqueantes"

Severidade é usada apenas para **priorização de correção**, nunca para decisão de aprovação.

## Quando Aplicar

- Ao validar uma feature frontend Angular concluída antes de release (todas as tasks completas e reviews APPROVED)
- Ao revisar implementação contra PRD, TechSpec ou Tasks
- Ao executar pipeline completo de qualidade (estática + testes + E2E no browser)
- Ao gerar relatório de QA com evidências auditáveis
- Ao verificar acessibilidade, responsividade e qualidade de design de uma entrega
- Antes de bump de versão (SemVer) e atualização de changelog

## Estrutura de Evidências

**REGRA CRÍTICA:** Todos os artefatos de QA DEVEM ser salvos em uma estrutura padronizada:

```
.cognup/specs/[feature]/report/
├── qa-report.md          # Relatório final consolidado
├── bugs.md               # Documentação detalhada de bugs
└── screenshots/          # Evidências visuais
    ├── RF-01-antes.png
    ├── RF-01-resultado.png
    ├── acessibilidade-indicador-foco.png
    ├── visual-dashboard-vazio.png
    ├── responsivo-mobile-dashboard.png
    ├── BUG-01-screenshot.png
    └── ...
```

Crie os diretórios antes de iniciar: `mkdir -p .cognup/specs/[feature]/report/screenshots/`

**OBRIGATORIEDADE DE EVIDÊNCIAS:**
- Todo teste — PASSOU ou FALHOU — DEVE ter pelo menos um screenshot como evidência
- Nenhum requisito pode ser marcado como verificado sem screenshot correspondente em `report/screenshots/`
- Todo bug DEVE ter screenshot; sem screenshot, o bug não está documentado
- Teste sem evidência = teste NÃO EXECUTADO
- Nomes descritivos referenciando o ID do requisito ou bug (ex.: `RF-01-resultado.png`, `BUG-02-screenshot.png`)

## Pipeline de QA — 12 Etapas

### Etapa 1: Análise da Documentação (Obrigatória)

Antes de qualquer teste:

1. **Ler o PRD** e extrair TODOS os requisitos funcionais numerados em checklist
2. **Ler a TechSpec** e anotar decisões técnicas verificáveis (rotas e render modes, contratos de API/BFF, componentes)
3. **Ler o arquivo de Tasks** e confirmar status de conclusão (reviews em `tasks/reviews/` devem estar APPROVED)
4. **Consultar [[angular-clean-architecture]]** e [[angular-best-practices]] para critérios de conformidade
5. **Identificar componentes do design system** usados (consulte [[angular-material]]) e padrões Angular a validar
6. **Construir matriz de verificação** mapeando cada requisito → cenário de teste

**Nunca pule esta etapa** — entender requisitos é a base da QA.

### Etapa 2: Preparação do Ambiente (Obrigatória)

- Crie os diretórios de evidências: `mkdir -p .cognup/specs/[feature]/report/screenshots/`
- Verifique se a aplicação está rodando no localhost em modo SSR (`npx nx serve [app]` em dev; para validação de SSR completo com federação, build de produção + servidor Node — `fstart.mjs` no shell). Se falhar ao iniciar: **REPROVADO imediato**
- `browser_navigate` para acessar a aplicação; `browser_snapshot` para confirmar carregamento
- `browser_console_messages` para erros de inicialização (atenção especial a erros `NG05xx` de hydration); `browser_network_requests` para conectividade
- **Capture screenshot inicial**: `screenshot-ambiente-inicial.png`

**Protocolo de Lock/Unlock do Browser MCP:**
1. `browser_tabs` com action `list` para verificar tabs existentes
2. Se tab existir: `browser_lock` → interações → `browser_unlock`
3. Se não houver tab: `browser_navigate` → `browser_lock` → interações → `browser_unlock`
4. Só chame `browser_unlock` ao terminar TODAS as operações do browser da etapa

**Estratégia de Espera:** esperas curtas incrementais (1-3s) com `browser_snapshot` entre elas — nunca uma única espera longa.

### Etapa 3: Análise Estática (Obrigatória)

Execute via Nx e registre as saídas completas:

| Ferramenta | Comando | O que Verifica |
|------------|---------|----------------|
| **Type check** | `npx nx affected -t typecheck` | Erros de tipo, type safety |
| **Lint** | `npx nx affected -t lint` | Qualidade, estilo, **boundaries de camada/domínio** |
| **Build** | `npx nx affected -t build` | Build de produção — bundles browser **e server** (SSR), prerender das rotas `Prerender` |

Qualquer falha = bug documentado. Verificações adicionais: sem `TODO`/`FIXME` não resolvidos da feature; sem tipos `any` em código novo; sem `console.log` esquecidos; sem `eslint-disable` de boundary.

### Etapa 4: Testes Unitários e de Integração (Obrigatória)

```bash
npx nx affected -t test
```

**Análise obrigatória:**
1. Parse da saída — identificar passados, falhos, pulados
2. **ZERO falhas** é obrigatório; cada teste falho vira bug documentado
3. Código novo sem cobertura de teste = bug
4. Se a feature tem specs Playwright próprios, execute-os também (`npx nx e2e [app]-e2e` ou `npx playwright test`)

### Etapa 5: Testes E2E com Browser MCP (Obrigatória)

Execute testes E2E para cada requisito funcional usando o browser MCP (consulte [[webapp-testing]] para padrões e estratégia reconnaissance-then-action):

| Ferramenta | Finalidade |
|------------|-----------|
| `browser_navigate` | Navegar para páginas |
| `browser_snapshot` | Estado acessível da página (preferir sobre screenshot para análise) |
| `browser_click` / `browser_type` / `browser_fill` | Interações |
| `browser_select_option` / `browser_press_key` | Dropdowns e teclado |
| `browser_take_screenshot` | Evidência visual |
| `browser_console_messages` | Erros JavaScript |
| `browser_network_requests` | Chamadas de rede |
| `browser_scroll` / `browser_handle_dialog` | Scroll e diálogos nativos |

**Para cada requisito funcional:**
1. Navegue até a feature e execute o fluxo passo a passo
2. Screenshot ANTES da ação (`RF-01-antes.png`) e DEPOIS do resultado (`RF-01-resultado.png`) — **obrigatórios**
3. Marque **PASSOU** ou **FALHOU**; se FALHOU, documente o bug com passos de reprodução e screenshot

**Categorias obrigatórias:**
- **Happy path**: fluxos normais — screenshot de cada etapa
- **Edge cases**: entradas vazias, valores limite, caracteres especiais
- **Tratamento de erros**: entradas inválidas, falhas de rede, erros do backend/BFF
- **Transições de estado**: loading, sucesso, erro, vazio — screenshot de cada estado
- **Validação SSR e hydration**: conteúdo renderizado pelo servidor presente no HTML inicial (verifique o response da rota — para rotas `RenderMode.Server`/`Prerender` o conteúdo deve estar no HTML, não montado só pós-JS); nenhum erro `NG05xx` no console; interatividade funcionando após hydration
- **Validação da regra BFF**: em `browser_network_requests`, **nenhuma chamada direta do browser ao backend interno** — apenas ao BFF/mesma origem; nenhuma requisição duplicada de dado já transferido pelo transfer cache

### Etapa 6: Acessibilidade — WCAG 2.2 (Obrigatória)

Para cada tela/componente, verifique com screenshot de evidência:

- [ ] Navegação por teclado (Tab, Shift+Tab, Enter, Escape) — `acessibilidade-navegacao-teclado.png`
- [ ] Labels descritivos em elementos interativos (`aria-label`/`aria-labelledby`) — `acessibilidade-labels.png`
- [ ] `alt` apropriado em imagens — `acessibilidade-alt-text.png`
- [ ] Contraste WCAG AA (4.5:1 texto normal, 3:1 texto grande) — `acessibilidade-contraste.png`
- [ ] Labels associados aos inputs de formulário (`mat-label`) — `acessibilidade-form-labels.png`
- [ ] Mensagens de erro claras e associadas programaticamente (`mat-error`) — `acessibilidade-mensagens-erro.png`
- [ ] Indicadores de foco visíveis — `acessibilidade-indicador-foco.png`
- [ ] Headings em hierarquia lógica (h1 → h2 → h3) — `acessibilidade-headings.png`
- [ ] Roles ARIA corretos onde HTML semântico é insuficiente — `acessibilidade-roles-aria.png`
- [ ] `<title>` significativo por rota — `acessibilidade-title.png`
- [ ] Focus trapping em dialogs/overlays; foco retorna ao trigger ao fechar — `acessibilidade-focus-trap.png`

Use `browser_press_key` (Tab) para teclado, `browser_snapshot` para estrutura semântica, `browser_take_screenshot` para foco/contraste.

### Etapa 7: Verificação Visual (Obrigatória)

Capture screenshots de TODAS as telas principais em `report/screenshots/` com nomes `visual-[tela]-[estado].png`.

**Estados obrigatórios (com screenshot de cada):**
- **Vazio** (sem dados) — `visual-[tela]-vazio.png`
- **Populado** (com dados) — `visual-[tela]-populado.png`
- **Erro** (após falhas) — `visual-[tela]-erro.png`
- **Loading** (se aplicável) — `visual-[tela]-loading.png`

Documente qualquer inconsistência visual como bug — incluindo flash de conteúdo na hydration (conteúdo que muda visivelmente entre o HTML do servidor e o app hidratado).

### Etapa 8: Auditoria de Design e Responsividade (Obrigatória)

**8.1 Qualidade Visual** ([[frontend-design]]): tipografia (hierarquia, legibilidade), cores e tema (paleta coesa via tokens, dark mode), animações (propósito, `prefers-reduced-motion`), composição espacial — screenshots `design-[criterio].png`.

**8.2 Web Interface Guidelines** ([[web-design-guidelines]]): busque as guidelines atualizadas via WebFetch (`https://raw.githubusercontent.com/vercel-labs/web-interface-guidelines/main/command.md`) e valide. Screenshot de cada violação: `design-guideline-[nome].png`.

**8.3 Componentes Angular Material** ([[angular-material]]): hierarquia de reuso (libs/ui → Material → custom), tema único com system variables (sem hex hardcoded), formulários (`mat-label`/`mat-error`/`mat-hint`), dialogs com título e ações, tabelas com estado vazio, sem `::ng-deep` — screenshots `material-[criterio].png`.

**8.4 Boas Práticas Angular** ([[angular-best-practices]]): render modes corretos por rota, estados de loading/erro/vazio, `@defer` em conteúdo pesado, `NgOptimizedImage`, título por rota, sem padrões legados — evidência de violações `angular-[tipo].png`.

**8.5 Responsividade** ([[responsive-design]]): teste em cada breakpoint redimensionando o viewport e capture as telas principais:

| Breakpoint | Largura | Screenshot |
|------------|---------|------------|
| Mobile | 375px | `responsivo-mobile-[tela].png` |
| Tablet Retrato | 768px | `responsivo-tablet-retrato-[tela].png` |
| Tablet Paisagem | 1024px | `responsivo-tablet-paisagem-[tela].png` |
| Desktop | 1280px | `responsivo-desktop-[tela].png` |
| Desktop Grande | 1536px | `responsivo-desktop-grande-[tela].png` |

**Critérios por breakpoint:** sem overflow horizontal; tipografia fluida legível; navegação responsiva funcional (sidenav colapsa no mobile); grids/cards redistribuem (`auto-fit`/`auto-fill`); imagens proporcionais; tabelas acessíveis (scroll ou cards); touch targets ≥ 44x44px no mobile; formulários usáveis em telas pequenas; `dvh` para alturas full-screen mobile.

```
1. browser_navigate para a página
2. Redimensionar viewport (ex.: page.setViewportSize({ width: 375, height: 812 }))
3. browser_snapshot para verificar o estado
4. browser_take_screenshot para evidência
5. Repetir para cada breakpoint
```

### Etapa 9: Documentação de Bugs

Para cada bug encontrado, documente em `.cognup/specs/[feature]/report/bugs.md`:

```markdown
## BUG-[ID]: [Descrição curta]

- **Severidade**: Crítica / Alta / Média / Baixa
- **Requisito**: [ID do requisito relacionado no PRD]
- **Categoria**: Funcional / SSR-Hydration / Acessibilidade / Visual / Design / Responsividade / Angular / Material
- **Passos para Reproduzir**:
  1. [Passo 1]
  2. [Passo N]
- **Resultado Esperado**: [O que deveria acontecer]
- **Resultado Atual**: [O que realmente aconteceu]
- **Screenshot**: `report/screenshots/BUG-[ID]-screenshot.png`
- **Ambiente**: localhost
```

**Critérios de severidade (apenas para priorização de correção):**
- **Crítica**: feature quebrada, perda de dados, problema de segurança (violação BFF, secret no client), crash, mismatch de hydration
- **Alta**: funcionalidade principal não funciona, bloqueia fluxo principal
- **Média**: funciona com problemas significativos, workaround existe
- **Baixa**: cosmético, problemas menores de UX

### Etapa 10: Relatório de QA (Obrigatória)

Crie `.cognup/specs/[feature]/report/qa-report.md` com este formato:

```markdown
# Relatório de QA - [Nome da Feature]

## Resumo
- **Data**: [AAAA-MM-DD]
- **QA Engineer**: AI QA Validator
- **Status**: APROVADO / REPROVADO
- **Total de Requisitos**: [X]
- **Requisitos Atendidos**: [Y]
- **Requisitos Reprovados**: [Z]
- **Bugs Encontrados**: [W]
- **Total de Screenshots de Evidência**: [N]

## Requisitos Verificados

| ID | Requisito | Status | Evidência |
|----|-----------|--------|-----------|
| RF-01 | [descrição] | PASSOU / FALHOU | `screenshots/RF-01-resultado.png` |

## Testes E2E Executados

| Fluxo | Resultado | Evidência | Observações |
|-------|-----------|-----------|-------------|
| [fluxo] | PASSOU / FALHOU | `screenshots/[nome].png` | [notas] |

## Análise Estática e Testes

| Verificação | Comando | Resultado |
|---|---|---|
| Type check | `npx nx affected -t typecheck` | PASSOU / FALHOU |
| Lint (+ boundaries) | `npx nx affected -t lint` | PASSOU / FALHOU |
| Testes | `npx nx affected -t test` | PASSOU / FALHOU |
| Build (browser + server) | `npx nx affected -t build` | PASSOU / FALHOU |

## SSR e Hydration

| Verificação | Status | Evidência |
|----------|--------|-----------|
| Conteúdo no HTML do servidor (rotas Server/Prerender) | PASSOU / FALHOU | [nota/screenshot] |
| Console sem erros NG05xx | PASSOU / FALHOU | [nota] |
| Regra BFF (nenhuma chamada direta ao backend interno) | PASSOU / FALHOU | [nota] |
| Sem flash de conteúdo na hydration | PASSOU / FALHOU | [nota] |

## Acessibilidade (WCAG 2.2)

| Critério | Status | Evidência |
|----------|--------|-----------|
| [cada critério da Etapa 6] | PASSOU / FALHOU | `screenshots/acessibilidade-[criterio].png` |

## Verificação Visual

| Tela | Estado | Status | Evidência |
|------|--------|--------|-----------|
| [tela] | [vazio/populado/erro/loading] | PASSOU / FALHOU | `screenshots/visual-[tela]-[estado].png` |

## Auditoria de Design

[Tabelas por área: Qualidade Visual, Web Interface Guidelines, Angular Material, Angular — critério + status + evidência]

## Responsividade

| Breakpoint | Tela | Status | Evidência |
|------------|------|--------|-----------|
| [cada breakpoint da Etapa 8.5] | [tela] | PASSOU / FALHOU | `screenshots/responsivo-[breakpoint]-[tela].png` |

## Bugs Encontrados

| ID | Descrição | Severidade | Categoria | Requisito | Evidência |
|----|-----------|------------|-----------|-----------|-----------|
| BUG-01 | [descrição] | [sev] | [cat] | [RF-XX] | `screenshots/BUG-01-screenshot.png` |

*Se nenhum bug: "Nenhum bug encontrado."*

## Console e Network

- **Erros no console**: [quantidade e resumo]
- **Chamadas de API com falha**: [quantidade e resumo]
- **Chamadas diretas do browser ao backend interno**: [nenhuma esperada — listar se encontradas]

## Inventário de Evidências

Total de screenshots: [N]

| Arquivo | Tipo | Relacionado a |
|---------|------|---------------|
| `screenshots/[nome].png` | [tipo] | [RF-XX / BUG-XX / critério] |

## Conclusão

[Avaliação final com raciocínio claro para o status APROVADO/REPROVADO]

## Próximos Passos

[Ações necessárias se REPROVADO, ou confirmação de prontidão se APROVADO]
```

### Etapa 11: Changelog (Somente Pós-Aprovação)

**Execute apenas com status APROVADO.** Referência: [[changelog-automation]].

1. Leia o PRD para extrair nome da feature, descrição e requisitos validados
2. Leia o `CHANGELOG.md` na raiz do projeto (crie com o cabeçalho padrão Keep a Changelog se não existir)
3. Categorize as mudanças: **Added** (novas features/componentes), **Changed** (modificações/melhorias), **Fixed** (correções incluídas na feature), **Removed**/**Deprecated**/**Security** quando aplicável
4. Adicione a entrada na seção `[Unreleased]` — nunca sobrescreva entradas existentes
5. Entradas concisas, em **inglês**, referenciando a feature e a data de validação do QA

### Etapa 12: Versionamento Semântico (Somente Pós-Aprovação)

**Execute apenas com status APROVADO, após a Etapa 11.**

1. Leia a versão atual do `package.json` da raiz do monorepo
2. Determine o bump pela seção `[Unreleased]`:

| Tipo de Mudança | Bump | Exemplo |
|-----------------|------|---------|
| Breaking changes | **MAJOR** | `1.2.3` → `2.0.0` |
| Novas features (Added) | **MINOR** | `1.2.3` → `1.3.0` |
| Apenas correções (Fixed) | **PATCH** | `1.2.3` → `1.2.4` |

3. Atualize o campo `version` no `package.json` da raiz
4. Mova as entradas de `[Unreleased]` para uma nova seção `## [X.Y.Z] - AAAA-MM-DD`, mantendo `[Unreleased]` vazia no topo
5. Atualize APENAS `package.json` e `CHANGELOG.md` — **não crie tags git nem commits** (o merge/tag/push é responsabilidade do workflow `run-task.md`)

## Critérios de Aprovação

**REGRA ABSOLUTA: qualquer bug encontrado (independente da severidade) = REPROVADO.**

- **APROVADO**: TODOS os requisitos funcionais do PRD verificados e passando, **ZERO bugs**, SSR/hydration sem violações, acessibilidade passando, nenhum erro JavaScript no console, auditoria de design e responsividade sem violações, TODOS os testes com screenshots de evidência. **Quando APROVADO, as Etapas 11 (Changelog) e 12 (Versionamento) DEVEM ser executadas antes de finalizar.**
- **REPROVADO**: qualquer requisito falhando, OU qualquer bug encontrado, OU violações de acessibilidade, OU erros JavaScript no console, OU testes sem evidência de screenshot. **Quando REPROVADO, pule as Etapas 11 e 12**, liste todos os bugs no relatório e documente os passos de correção.

Os status são escritos **em português** (`APROVADO`/`REPROVADO`) — é o valor que o workflow `run-task.md` verifica em `report/qa-report.md` para decidir entre seguir para merge ou acionar o `executor-bug`.

## Diretrizes Operacionais

1. **`browser_snapshot` antes de qualquer interação** para entender o estado da página
2. **Screenshot de TODOS os testes** — passou ou falhou, sem exceção
3. **Screenshot de TODOS os bugs** — sem exceção
4. **Verifique console e network** — `browser_console_messages` e `browser_network_requests` em todo fluxo
5. **Valide a regra BFF** — chamadas diretas do browser ao backend interno são bug
6. **Valide SSR/hydration** — conteúdo no HTML do servidor, console sem NG05xx, sem flash de conteúdo
7. **Teste navegação por teclado** em cada elemento interativo
8. **Nunca aprove com requisito não testado ou bug pendente**
9. **Seja sistemático** — siga a matriz de requisitos sequencialmente
10. **Documente tudo** — se não está no relatório com evidência, não foi testado
11. **Esperas incrementais** (1-3s) com snapshot, nunca esperas longas
12. **Todo teste sem screenshot é considerado NÃO EXECUTADO**
13. **Changelog e versionamento somente pós-aprovação**

## Geração de Testes Playwright (Opcional, Pós-Aprovação)

Após o QA ser APROVADO, opcionalmente gere testes Playwright automatizados a partir dos cenários validados (consulte [[playwright-generate-test]]): um teste TypeScript por cenário com `@playwright/test`, salvo no projeto e2e do domínio, executado e iterado até passar. Isso cria uma suite de regressão baseada nos cenários de QA.

## Anti-Patterns de QA (Evitar)

| Anti-Pattern | Correto |
|---|---|
| Aprovar com bug "só cosmético" | Tolerância zero: qualquer bug = REPROVADO |
| Teste sem screenshot | Screenshot obrigatório para todo teste e bug |
| Bugs fora de `report/bugs.md` | Caminho fixo — é o que o executor-bug lê |
| Status em inglês (APPROVED) no qa-report | Status em português (APROVADO/REPROVADO) — é o que o run-task verifica |
| Testar só o happy path | Edge cases, erros e transições de estado são obrigatórios |
| Testar em um único viewport | 5 breakpoints obrigatórios (375 a 1536px) |
| Ignorar o console do browser | Erro no console = bug (NG05xx de hydration incluído) |
| Validar SSR só "olhando a tela" | Verificar o HTML do servidor e o network (BFF, transfer cache) |
| Changelog/versão com QA REPROVADO | Etapas 11 e 12 somente pós-aprovação |
| Criar tag/commit no versionamento | Apenas `package.json` e `CHANGELOG.md`; git é do run-task |
| Esperar 30s de uma vez pela página | Esperas incrementais de 1-3s com snapshot |

## Idioma

Relatório de QA e documentação de bugs em **Português (Brasil)**. Termos técnicos (atributos HTML, roles ARIA) podem permanecer em inglês. Entradas de changelog em **inglês**.

## Manutenção de Memória

Ao longo das sessões de QA, registre padrões para acelerar validações futuras:

- Padrões de UI comuns e componentes do design system em uso
- Violações de acessibilidade recorrentes
- Padrões de resposta de API/BFF e formatos de erro
- Estrutura de navegação, render modes por rota e padrões de validação de formulários
- Áreas conhecidas com problemas intermitentes e testes flakey
- Comportamentos específicos de breakpoint/browser e problemas recorrentes de hydration observados durante os testes
