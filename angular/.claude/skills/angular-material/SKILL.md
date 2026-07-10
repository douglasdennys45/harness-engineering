---
name: angular-material
description: Gerencia o design system do projeto baseado em Angular Material (M3) e CDK - theming com design tokens (system variables), tipografia e densidade, composicao de componentes (tabelas, dialogs, forms, navegacao), wrappers do design system em libs/ui, dark mode, acessibilidade dos componentes e regras de customizacao (nunca sobrescrever internals com ::ng-deep). Use ao adicionar, compor ou estilizar componentes UI, ao criar wrappers no libs/ui, ao configurar tema/tokens, ou ao revisar consistencia visual de componentes Material.
---

# Angular Material — Design System do Projeto

Angular Material (Material 3) + CDK e o design system base. Componentes do `libs/ui` (wrappers e composicoes) vem **antes** de usar Material cru nas features; Material cru vem **antes** de qualquer componente custom.

## Hierarquia de Reuso (obrigatoria)

1. **`libs/ui`** — componentes do design system do projeto (wrappers tematizados, empty states, data tables compostas). Verifique o barrel `libs/ui/src/index.ts` antes de criar qualquer componente.
2. **Angular Material / CDK** — componente pronto e acessivel para o padrao (dialog, menu, table, stepper, autocomplete...).
3. **CDK sem estilo** (`cdk-listbox`, `cdk-menu`, `cdkDrag`, overlays) — quando o visual e proprio mas o comportamento/a11y e padrao.
4. **Custom** — apenas quando 1-3 nao cobrem; nasce em `libs/ui` se for reutilizavel, com foco/keyboard/ARIA completos.

Nunca reimplemente na mao componentes com semantica de acessibilidade complexa (dialog, menu, combobox, tabs) — use Material/CDK.

## Tema e Design Tokens

Tema unico definido em `libs/ui/src/theme/_theme.scss` e importado pelos apps — **nenhum app define tema proprio** (consistencia visual entre microfrontends).

```scss
// libs/ui/src/theme/_theme.scss
@use '@angular/material' as mat;

html {
  color-scheme: light dark;                    // dark mode automatico
  @include mat.theme((
    color: (
      primary: mat.$azure-palette,             // trocar pela paleta da marca
      tertiary: mat.$blue-palette,
    ),
    typography: (
      plain-family: Inter,
      brand-family: Inter,
    ),
    density: 0,
  ));
}
```

Regras:

- **Estilize com system variables** (`var(--mat-sys-primary)`, `--mat-sys-surface-container`, `--mat-sys-on-surface`, `--mat-sys-outline` etc.) — nunca hex hardcoded em componentes.
- Customizacao de componente via `overrides` mixins do proprio Material (`mat.card-overrides(...)`) ou tokens — **nunca `::ng-deep`**, nunca seletores de classes internas (`.mdc-*`, `.mat-mdc-*`), que quebram a cada upgrade.
- Dark mode via `color-scheme: light dark` + system variables; componentes custom usam os mesmos tokens para herdar o tema.
- Densidade e tipografia definidas no tema, nao por componente.

## Regras de Composicao

### Formularios

- `mat-form-field` com `appearance` unica no projeto (definida no tema via `MAT_FORM_FIELD_DEFAULT_OPTIONS`).
- Sempre: `mat-label` (nunca placeholder como label), `mat-error` para cada validacao, `mat-hint` para orientacao.
- Erros: exibidos apos touch/submit; associados ao campo (Material ja faz `aria-describedby`).
- Botao de submit com estado pending (desabilitado + spinner/progress) durante a acao.

### Tabelas e Listas de Dados

- `mat-table` com `trackBy`; sort/paginacao via `MatSort`/`MatPaginator` quando ha mais de ~20 itens.
- Estado vazio obrigatorio (componente `empty-state` do `libs/ui`), nunca tabela em branco.
- Em telas pequenas: tabela vira cards ou ganha scroll horizontal explicito (ver responsive-design) — nunca overflow silencioso.

### Overlays (Dialog, Bottom Sheet, Menu, Snackbar)

- `MatDialog` sempre com titulo (`mat-dialog-title`) e acoes explicitas; fechamento por Escape e backdrop preservado.
- Conteudo de dialog e um componente proprio (nao template inline gigante); dados via `MAT_DIALOG_DATA` tipado.
- Snackbar para feedback transitorio (sucesso/erro de acao); nunca para erros que exigem decisao.
- Overlays sao browser-only por natureza — dispare-os de handlers de evento, nunca durante o render SSR.

### Navegacao

- Shell: `mat-sidenav` (colapsavel em mobile) + `mat-toolbar`; itens de navegacao com `routerLinkActive`.
- `mat-tabs` para navegacao interna de pagina; tabs com rota (`routerLink`) quando o estado deve ser navegavel.

### Icones

- `mat-icon` com Material Symbols; `aria-hidden="true"` quando decorativo, `aria-label` no botao quando o icone e o unico conteudo (`mat-icon-button`).

## SSR

- Componentes Material sao SSR-safe; overlays/dialogs so abrem no browser (handler de evento).
- Nao meça DOM (`getBoundingClientRect`) na inicializacao — use `afterNextRender`.
- Animacoes: `provideAnimationsAsync()` (nao bloqueia SSR nem o bundle inicial).

## Checklist de Componente Novo no libs/ui

```
[ ] Verificado que Material/CDK nao cobre o caso
[ ] Standalone, OnPush, signals (input/output)
[ ] Estilizado 100% com system variables do tema (funciona em light e dark)
[ ] Estados: default, hover, focus-visible, disabled, loading (quando aplicavel)
[ ] Acessivel: role/keyboard/ARIA completos, testado com Tab e screen reader
[ ] Responsivo (container queries quando vive em contextos variados)
[ ] Teste de componente (Angular Testing Library) cobrindo comportamento
[ ] Exportado no barrel index.ts e documentado
```

## Anti-patterns

| Anti-pattern | Correto |
|---|---|
| `::ng-deep` / seletores `.mdc-*` | `overrides` mixins ou system variables |
| Hex hardcoded em componente | `var(--mat-sys-*)` do tema |
| Tema definido por app | Tema unico em `libs/ui/theme`, importado pelos apps |
| Placeholder como label | `mat-label` sempre |
| Dialog sem titulo/acoes | `mat-dialog-title` + acoes explicitas |
| Tabela sem estado vazio | `empty-state` do libs/ui |
| Reimplementar dropdown/dialog na mao | Material ou CDK |
| Feature importando Material cru para padrao ja wrapado | Componente do `libs/ui` primeiro |
| Abrir dialog/snackbar durante render SSR | Apenas em handlers de evento no browser |
