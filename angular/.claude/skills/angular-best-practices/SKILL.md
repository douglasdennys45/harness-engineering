---
name: angular-best-practices
description: Boas praticas de Angular moderno (v20+) - standalone components, signals (input/output/model/computed/effect), zoneless change detection, control flow (@if/@for/@switch), @defer e incremental hydration, SSR com @angular/ssr (render modes, transfer cache, event replay, afterNextRender), httpResource/HttpClient, Signal Forms e Reactive Forms, roteamento (lazy loading, guards funcionais, resolvers), NgOptimizedImage, error handling, acessibilidade com CDK, debug de hydration mismatch e performance. Use ao escrever qualquer codigo Angular.
---

# Angular Best Practices — Angular Moderno com SSR

Referencia de boas praticas para qualquer codigo Angular do projeto. Baseline: Angular 20+ (zoneless estavel, incremental hydration estavel, signals como primitivo reativo padrao).

## 1. Componentes

- **Standalone sempre** — `standalone: true` e implicito; NgModules nao sao usados em codigo novo.
- **`ChangeDetectionStrategy.OnPush` em todo componente** (padrao no zoneless). Estado muda via signals, nunca via mutacao de objeto observado.
- **Componentes pequenos e compostos**: ate ~300 linhas; extraia quando ultrapassar. Logica de negocio nunca vive no componente (vai para use case/entidade).
- `inject()` em vez de injecao por construtor; `private readonly` para dependencias.
- Template inline apenas para componentes triviais (< ~15 linhas de template); acima disso, arquivo `.html` separado.
- `host: {}` no decorator em vez de `@HostBinding`/`@HostListener`.

```ts
@Component({
  selector: 'app-invoice-table',
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [MatTableModule, CurrencyPipe],
  templateUrl: './invoice-table.component.html',
})
export class InvoiceTableComponent {
  readonly invoices = input.required<InvoiceDto[]>();   // signal input
  readonly selected = output<InvoiceDto>();              // output
  protected readonly total = computed(() =>
    this.invoices().reduce((sum, i) => sum + i.amount, 0),
  );
}
```

## 2. Signals — Primitivo Reativo Padrao

| API | Uso |
|---|---|
| `signal()` | Estado local mutavel |
| `computed()` | Estado derivado — **nunca armazene o que pode ser calculado** |
| `input()` / `input.required()` | Props do componente (substitui `@Input`) |
| `output()` | Eventos (substitui `@Output`) |
| `model()` | Two-way binding |
| `linkedSignal()` | Estado local que reseta quando uma fonte muda |
| `effect()` | Efeitos colaterais reativos — **ultimo recurso**, nunca para derivar estado |
| `toSignal()` / `toObservable()` | Interop com RxJS nas bordas |
| `resource()` / `httpResource()` | Dados assincronos como signal (loading/error/value) |

Regras:

- **Derive, nao sincronize**: `computed()` em vez de `effect()` + `set()`.
- Nao chame `set()`/`update()` dentro de `computed()` (proibido) nem de `effect()` sem justificativa forte.
- RxJS continua valido para streams de eventos complexos (debounce, retry, combinacao); converta para signal na borda do template com `toSignal()`.
- Stores de dominio sao services com signals privados + `computed` publicos:

```ts
@Injectable({ providedIn: 'root' })
export class BillingStore {
  private readonly state = signal<BillingState>({ invoices: [], status: 'idle' });
  readonly invoices = computed(() => this.state().invoices);
  readonly isLoading = computed(() => this.state().status === 'loading');

  setInvoices(invoices: InvoiceDto[]): void {
    this.state.update((s) => ({ ...s, invoices, status: 'loaded' }));
  }
}
```

## 3. Zoneless

- Apps novos nascem zoneless (`provideZonelessChangeDetection()` / padrao no Angular 21+). Sem `zone.js` no `polyfills`.
- Change detection dispara por: signals lidos no template, `AsyncPipe`, `markForCheck`, eventos do template. Codigo que muda estado fora desses canais (callbacks de libs de terceiros, `setTimeout` cru) deve escrever em um **signal**.
- Nunca dependa de `NgZone.onStable`/`run()`. Testes usam `TestBed` com zoneless (padrao) e `await fixture.whenStable()`.

## 4. Control Flow e Templates

- `@if` / `@for` / `@switch` — nunca `*ngIf`/`*ngFor` em codigo novo.
- `@for` exige `track` — use identidade estavel (`track item.id`), nunca `track $index` em listas dinamicas.
- `@empty` no `@for` para estado vazio; estados de **loading, erro e vazio** sao obrigatorios em toda tela com dados.
- Pipes puros para transformacao no template; funcoes chamadas no template devem ser triviais (getter de signal) — nada de trabalho pesado por render.

## 5. `@defer` e Incremental Hydration

- Conteudo pesado ou abaixo da dobra: `@defer (on viewport)` com `@placeholder` (dimensoes estaveis — evita CLS) e `@error`.
- Com SSR + `withIncrementalHydration()`, use gatilhos de hydration: `@defer (hydrate on viewport)` — o servidor renderiza o conteudo, o JS so carrega/hidrata quando necessario.
- Componentes federados (Native Federation) consumidos no shell sempre atras de `@defer` com fallback proprio.

```html
@defer (hydrate on viewport) {
  <app-invoice-chart [data]="chartData()" />
} @placeholder {
  <div class="chart-skeleton" aria-hidden="true"></div>
} @error {
  <app-widget-error label="Nao foi possivel carregar o grafico" />
}
```

## 6. SSR e Hydration

### Render mode por rota (`app.routes.server.ts`)

| Modo | Quando |
|---|---|
| `RenderMode.Server` | Dados frescos por request, conteudo personalizado, SEO dinamico |
| `RenderMode.Prerender` | Conteudo estavel em build time (landing, planos, docs) — use `getPrerenderParams` para rotas parametrizadas |
| `RenderMode.Client` | Telas 100% interativas sem valor de SEO (dashboards internos, simuladores) |

### Regras de codigo SSR-safe

- `provideClientHydration(withEventReplay(), withIncrementalHydration())` em todo app SSR. Event replay captura cliques antes da hydration — nao remova.
- **Render deterministico**: nada de `Math.random()`, `Date.now()`/`new Date()` livre, ou browser APIs (`window`, `document`, `localStorage`) no caminho de render. Mismatch de hydration e bug Critico.
- Codigo browser-only: `afterNextRender(() => { ... })` (roda so no browser, apos o primeiro render) ou `afterEveryRender` quando recorrente. `isPlatformBrowser(inject(PLATFORM_ID))` apenas quando a API acima nao serve.
- DOM nunca manipulado manualmente antes da hydration (quebra a reconciliacao). `ngSkipHydration` e valvula de escape pontual para componentes que manipulam DOM — nunca solucao padrao.
- Estado de autenticacao via **cookies** (visiveis no servidor), nunca `localStorage` (invisivel no SSR — causa flicker de conteudo logado/deslogado).

### Data fetching

- Leitura inicial: `HttpClient`/`httpResource` durante o SSR — o **transfer cache de hydration** serializa a resposta no HTML e o client nao repete a requisicao. Por isso repositories usam `HttpClient`, nunca `fetch` manual.
- `provideHttpClient(withFetch(), ...)` — backend fetch no servidor.
- Evite waterfalls: dispare requests independentes em paralelo (`Promise.all` no use case / multiplos `httpResource`).
- Mutacoes e leituras pos-hydration: via BFF do dominio (regra da skill [[angular-clean-architecture]]).

### Debug de hydration mismatch

1. Erro `NG0500`/`NG0501` no console indica mismatch — leia o componente apontado.
2. Causas comuns: data/random no render, HTML invalido (`<div>` dentro de `<p>`, `<a>` aninhado), conteudo dependente de browser API, condicao baseada em `localStorage`.
3. Corrija a **causa** (render deterministico, HTML valido, `afterNextRender`); nunca esconda com `ngSkipHydration` generalizado.

## 7. HTTP e Dados Assincronos

- `httpResource()` para leituras reativas simples (URL derivada de signals, status loading/error/value integrado).
- `HttpClient` em repositories (camada infra) para controle fino; interceptors funcionais (`withInterceptors([...])`) para auth, base URL e erros.
- Erros HTTP tratados na camada infra e convertidos em erros de dominio tipados — o componente exibe estado de erro, nunca inspeciona `HttpErrorResponse`.
- Timeouts e retry declarados no interceptor/repository, nao no componente.

## 8. Formularios

- **Reactive Forms tipados** (`FormGroup<...>`, `nonNullable`) como padrao estavel; template-driven apenas para casos triviais. (Signal Forms: adotar quando estabilizar.)
- Validacao dupla: no form (UX imediata) **e** no servidor/BFF (fonte de verdade) — validacao so no client e bug de seguranca.
- Erros associados programaticamente ao campo (`aria-describedby` + `mat-error`); estado de submissao (`pending`) desabilita o submit e mostra feedback.
- Formularios em SSR: valores iniciais deterministas; sem acesso a browser APIs na inicializacao.

## 9. Roteamento

- Lazy loading por padrao: `loadChildren`/`loadComponent` em toda feature.
- Guards e resolvers **funcionais** (`CanActivateFn`, `ResolveFn`) com `inject()`.
- Guards SSR-safe: decisao baseada em cookies/estado injetavel, nunca `window.location` ou `localStorage`.
- `withComponentInputBinding()` — params de rota chegam como `input()` no componente.
- Titulo por rota (`title:` na config ou `TitleStrategy`) — toda pagina tem `<title>` significativo.
- `RouterLink`/`RouterLinkActive` para navegacao interna; scroll restoration habilitado (`withInMemoryScrolling`).

## 10. Performance e Core Web Vitals

- **LCP**: imagem principal com `NgOptimizedImage` (`ngSrc`) + `priority`; fontes com `font-display: swap` e preload.
- **CLS**: `width`/`height` (ou `fill`) em toda imagem; placeholders de `@defer` com dimensoes estaveis; sem inserção de conteudo acima do fold pos-load.
- **INP**: handlers leves; trabalho pesado em `@defer`, `requestIdleCallback` (via `afterNextRender`) ou Web Worker.
- Bundle: lazy route por feature; `@defer` para libs pesadas (charts, editores); analise com `ng build --stats-json` + esbuild bundle analyzer; budgets configurados no `angular.json`/`project.json`.
- Zoneless + OnPush + signals = change detection minima; nao introduza subscribes manuais que setam estado fora de signals.

## 11. Error Handling

- `provideBrowserGlobalErrorListeners()` + `ErrorHandler` customizado para report centralizado.
- Erros de rota: rota wildcard `**` com pagina 404; `withNavigationErrorHandler` para falhas de lazy load (inclui falha de carga de remote federado — mostre fallback, nao tela branca).
- Toda tela com dados tem os 4 estados: loading (skeleton), erro (com acao de retry), vazio (com orientacao) e populado.
- Nunca `catch` vazio; erros de dominio tipados sobem ate a camada que sabe exibi-los.

## 12. Acessibilidade

- HTML semantico primeiro; Angular CDK `a11y` para o resto: `FocusTrap`/`FocusMonitor` em overlays, `LiveAnnouncer` para mudancas dinamicas, `cdkTrapFocus`.
- Componentes Material ja cobrem roles/keyboard — nao reimplemente botao/dialog/menu na mao.
- `:focus-visible` estilizado; navegacao por teclado completa; contraste WCAG AA.
- Rotas anunciadas: title por rota + foco gerenciado na navegacao (skip links no shell).

## 13. Testes

- **Vitest** como test runner (padrao Angular 21+); Angular Testing Library para componentes (queries por role/label, nunca por classe CSS).
- `TestBed` com providers de stub pelos **tokens** (contratos), nunca mock da implementacao concreta.
- MSW para testes de integracao da camada infra.
- Playwright para E2E contra o app em SSR — validar o HTML do servidor (conteudo presente antes da hydration) e a interatividade pos-hydration.
- Zoneless nos testes: `await fixture.whenStable()` em vez de `fakeAsync`/`tick` onde possivel.

## 14. Anti-patterns

| Anti-pattern | Correto |
|---|---|
| `*ngIf`/`*ngFor` em codigo novo | `@if`/`@for` com `track` |
| `@Input()`/`@Output()` decorators | `input()`/`output()` signals |
| `effect()` para derivar estado | `computed()`/`linkedSignal()` |
| Subscribe manual + variavel de classe | `toSignal()`/`httpResource`/`AsyncPipe` |
| `window`/`localStorage` no construtor | `afterNextRender()` ou cookies |
| `new Date()`/`Math.random()` no render | Valor vindo do servidor/input; formatacao deterministica |
| `ngSkipHydration` para calar mismatch | Corrigir o render nao-deterministico |
| `fetch` manual em repository | `HttpClient` (transfer cache + interceptors) |
| Componente sem estados de loading/erro/vazio | 4 estados obrigatorios |
| `track $index` em lista dinamica | `track item.id` |
| Logica pesada em funcao chamada no template | `computed()` no componente |
| `any` em codigo novo | Tipos explicitos; strict mode |
| NgModules em codigo novo | Standalone components |
| Zone.js em app novo | Zoneless |
