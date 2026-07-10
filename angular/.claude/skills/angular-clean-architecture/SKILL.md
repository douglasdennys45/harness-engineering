---
name: angular-clean-architecture
description: Aplica as regras de arquitetura do monorepo Angular com Nx, Native Federation (microfrontends com SSR) e Clean Architecture por dominio. Cobre estrutura do monorepo (apps/shell como unico host dinamico, apps/[domain] como remotes com @angular/ssr, libs/ compartilhadas), camadas por dominio (domain/, application/, infra/, presentation/), regras de dependencia entre camadas e entre dominios, nomenclatura (kebab-case, PascalCase, sufixos Dto/Mapper/Repository/Store), Native Federation (exposes via presentation/exposed/, singletons de @angular/*, federation.manifest.json), SSR com hydration (transfer cache, render modes por rota, incremental hydration com @defer), regra BFF (browser nunca chama o backend interno diretamente), shared libs (ui, shared-types, shared-events, http-client, util), comunicacao entre dominios (inputs, Event Bus tipado, estado global minimo com signals), roteamento (prefixo por dominio, auth guard no shell), estrategia de testes por camada (Vitest, MSW, Angular Testing Library, Playwright), CI/CD com deploy independente por dominio, e enforcement via @nx/enforce-module-boundaries com tags. Use sempre que criar/editar codigo em apps/ ou libs/, ao criar um novo dominio/microfrontend, ao definir use cases, entidades, repositories, DTOs, mappers, stores ou componentes expostos, ao configurar federation.config.js ou app.routes.server.ts, ao revisar imports entre camadas ou dominios, ao escrever testes, ou ao discutir arquitetura de projetos Angular com monorepo e SSR.
---

# Angular Monorepo — Clean Architecture e Microfrontends com SSR

Skill baseada no documento de referencia de arquitetura para projetos Angular com monorepo (Nx), Native Federation (`@angular-architects/native-federation`) e Clean Architecture. Todas as regras aqui devem ser seguidas ao escrever, revisar ou refatorar codigo do monorepo.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer arquivo em `apps/` ou `libs/`
- Ao criar um novo dominio (microfrontend)
- Ao definir entidades, use cases, repositories, DTOs, mappers ou componentes
- Ao configurar Native Federation, SSR, roteamento ou pipelines de deploy
- Ao revisar imports entre camadas ou entre dominios

## 1. Estrutura do Monorepo

```
root/
├── apps/
│   ├── shell/                    # Host dinamico unico (layout, nav, auth guard, roteamento) — com @angular/ssr
│   ├── auth/                     # Dominio: autenticacao e autorizacao — remote com @angular/ssr
│   ├── client/                   # Dominio: gestao de clientes — remote com @angular/ssr
│   ├── billing/                  # Dominio: cobranca, faturas, planos — remote com @angular/ssr
│   └── [domain]/                 # Novos dominios seguem o mesmo padrao
│
├── libs/
│   ├── ui/                       # Design system compartilhado (Angular Material + tokens)
│   ├── shared-types/             # Contratos e tipos entre dominios
│   ├── shared-events/            # Event bus tipado
│   ├── http-client/              # Interceptors e configuracao HTTP base (auth, transfer cache, erros)
│   └── util/                     # Helpers genericos sem logica de negocio
│
├── nx.json                       # Orquestracao, cache e tags de boundary
├── pnpm-workspace.yaml
├── package.json                  # Single version policy — uma versao de cada dependencia
└── .github/workflows/            # ci.yml + deploy-[domain].yml
```

### Regras

- **R1.1** — Cada dominio de negocio e um app Angular independente dentro de `apps/`, com `@angular/ssr` habilitado.
- **R1.2** — Codigo compartilhado entre dominios vive exclusivamente em `libs/`.
- **R1.3** — O `shell` e o unico host (`--type dynamic-host`). Nenhum outro app pode ser host.
- **R1.4** — Cada app deve ser executavel de forma isolada (`npx nx serve billing`).
- **R1.5** — Usar `pnpm` como package manager e **single version policy**: todas as dependencias no `package.json` da raiz — dois runtimes de Angular no mesmo documento quebram DI e change detection.
- **R1.6** — Usar Nx para orquestracao de builds, com cache e `affected` habilitados.
- **R1.7** — Todos os apps na mesma major do Angular e todos **zoneless** (padrao do Angular 21+). Nunca misturar apps zoneless com apps Zone.js no mesmo documento.

## 2. Nomenclatura

### Diretorios e Arquivos

| Tipo | Convencao | Exemplo |
|---|---|---|
| Diretorios | `kebab-case` | `value-objects/`, `use-cases/` |
| Componentes | `kebab-case.component.ts` (classe `PascalCaseComponent`) | `invoice-table.component.ts` → `InvoiceTableComponent` |
| Use Cases (classe) | `kebab-case.use-case.ts` (verbo + substantivo) | `create-invoice.use-case.ts` → `CreateInvoice` |
| Entidades | `kebab-case.entity.ts` | `invoice.entity.ts` → `Invoice` |
| DTOs | `kebab-case.dto.ts` com sufixo `Dto` | `create-invoice.dto.ts` → `CreateInvoiceDto` |
| Mappers | `kebab-case.mapper.ts` com sufixo `Mapper` | `invoice.mapper.ts` → `InvoiceMapper` |
| Repositories (interface) | `kebab-case.repository.ts` com sufixo `Repository` | `invoice.repository.ts` → `InvoiceRepository` |
| Repositories (impl) | prefixo `http-` / `Http` | `http-invoice.repository.ts` → `HttpInvoiceRepository` |
| Injection Tokens | `UPPER_SNAKE_CASE` | `INVOICE_REPOSITORY` |
| Stores (signals) | `kebab-case.store.ts` com sufixo `Store` | `billing.store.ts` → `BillingStore` |
| Guards / Interceptors / Resolvers | sufixo funcional | `auth.guard.ts`, `transfer-cache.interceptor.ts` |
| Tipos/Interfaces | `PascalCase` em `kebab-case.types.ts` | `billing.types.ts` |
| Constantes | `UPPER_SNAKE_CASE` | `MAX_RETRY_ATTEMPTS` |

### Branches Git

```
feat/[domain]/[descricao]       → feat/billing/add-invoice-export
fix/[domain]/[descricao]        → fix/auth/token-refresh-loop
chore/[domain]/[descricao]      → chore/shell/upgrade-angular-22
refactor/[domain]/[descricao]   → refactor/client/extract-use-cases
```

## 3. Clean Architecture por Dominio

Cada app em `apps/` segue a mesma estrutura interna:

```
apps/[domain]/src/app/
├── domain/                      # Nucleo — zero dependencias externas
│   ├── entities/                # Entidades com regras de negocio
│   ├── value-objects/           # Objetos de valor imutaveis
│   ├── errors/                  # Erros de dominio tipados
│   └── repositories/            # Interfaces (ports) + InjectionTokens — so contratos
│
├── application/                 # Orquestracao — depende so do domain
│   ├── use-cases/               # Casos de uso (1 classe = 1 acao)
│   ├── dtos/                    # Input/Output dos use cases
│   └── mappers/                 # Conversao Entity <-> DTO
│
├── infra/                       # Implementacoes concretas
│   ├── repositories/            # Implementam interfaces do domain (HttpClient)
│   ├── gateways/                # Integracoes externas (Stripe, etc)
│   ├── interceptors/            # HTTP do dominio (auth header, base URL)
│   └── store/                   # Estado local (signals store)
│
├── presentation/                # Angular UI — camada mais externa
│   ├── pages/                   # Componentes de rota do dominio
│   ├── components/              # Componentes do dominio
│   ├── exposed/                 # Componentes/rotas expostos via Native Federation
│   ├── layouts/                 # Layouts do dominio
│   └── [domain].routes.ts       # Rotas do dominio (exportadas para o shell)
│
├── app.config.ts                # Providers do app (composition root do dominio)
├── app.config.server.ts         # Providers de SSR
└── app.routes.server.ts         # Render mode por rota (Server/Prerender/Client)
```

### Regras

- **R3.1** — `domain/` nao importa NENHUMA lib externa (nem `@angular/*`, nem RxJS). Somente TypeScript puro — exceto o `InjectionToken` do contrato, que vive junto da interface em `domain/repositories/` (unica dependencia de `@angular/core` permitida, por pragmatismo de DI).
- **R3.2** — `application/` importa somente de `domain/`. Use cases recebem interfaces via `inject(TOKEN)` no construtor.
- **R3.3** — `infra/` implementa as interfaces definidas em `domain/repositories/` usando `HttpClient` (nunca `fetch`/axios direto — o transfer cache do SSR depende do `HttpClient`).
- **R3.4** — `presentation/` nunca importa de `domain/` diretamente. Acessa dados via DTOs retornados pelos use cases.
- **R3.5** — O **composition root** e o `app.config.ts`: e o unico lugar que liga interface → implementacao (`{ provide: INVOICE_REPOSITORY, useClass: HttpInvoiceRepository }`).
- **R3.6** — Cada use case e uma classe `@Injectable({ providedIn: 'root' })` com um unico metodo publico `execute()`.
- **R3.7** — Validacoes de negocio vivem na entidade, nao no componente.
- **R3.8** — Value Objects sao imutaveis. Toda mutacao retorna uma nova instancia.
- **R3.9** — DTOs sao objetos simples (interfaces). Nao possuem metodos nem logica.
- **R3.10** — Mappers sao classes com metodos estaticos. Nao possuem estado.

### Exemplo — contrato + DI

```ts
// domain/repositories/invoice.repository.ts
import { InjectionToken } from '@angular/core';
import { Invoice } from '../entities/invoice.entity';

export interface InvoiceRepository {
  findAll(): Promise<Invoice[]>;
  create(invoice: Invoice): Promise<Invoice>;
}

export const INVOICE_REPOSITORY = new InjectionToken<InvoiceRepository>('InvoiceRepository');
```

```ts
// application/use-cases/create-invoice.use-case.ts
@Injectable({ providedIn: 'root' })
export class CreateInvoice {
  private readonly repository = inject(INVOICE_REPOSITORY);

  async execute(input: CreateInvoiceDto): Promise<InvoiceDto> {
    const invoice = Invoice.create(input);           // validacao na entidade
    const created = await this.repository.create(invoice);
    return InvoiceMapper.toDto(created);
  }
}
```

```ts
// app.config.ts (composition root)
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(billingRoutes),
    provideHttpClient(withFetch(), withInterceptors([authInterceptor])),
    provideClientHydration(withEventReplay(), withIncrementalHydration()),
    { provide: INVOICE_REPOSITORY, useClass: HttpInvoiceRepository },
  ],
};
```

## 4. Dependencia entre Camadas

```
PERMITIDO (→ = "pode importar de")

  presentation → application (componentes injetam use cases)
  infra        → domain (para implementar interfaces)
  infra        → application (para usar DTOs se necessario)
  application  → domain (para usar entidades e interfaces)
  app.config   → todas as camadas (e o compositor)

PROIBIDO (✗)

  domain       ✗ application, infra, presentation
  application  ✗ infra, presentation
  infra        ✗ presentation
  presentation ✗ domain diretamente (use DTOs) e ✗ infra diretamente
```

| Camada | Importa de | Nunca importa de |
|---|---|---|
| `domain/` | Nada (TS puro + InjectionToken) | `application/`, `infra/`, `presentation/` |
| `application/` | `domain/` | `infra/`, `presentation/` |
| `infra/` | `domain/`, `application/` | `presentation/` |
| `presentation/` | `application/` | `domain/` direto, `infra/` direto |
| `app.config.ts` | Todas | — |

## 5. Dependencia entre Dominios

```
PERMITIDO
  shell   → qualquer remote (consome rotas e componentes expostos)
  remote  → libs/* (bibliotecas compartilhadas)
  remote  → shared-events (comunicacao desacoplada)

PROIBIDO
  remote  ✗ outro remote (NUNCA importacao direta)
  remote  ✗ shell (NUNCA depender do host)
  logica de dominio em libs/ (libs nao contem logica de negocio)
```

- **R5.1** — Dominios NUNCA importam diretamente uns dos outros.
- **R5.2** — Se billing precisa de dados do auth, a comunicacao ocorre via Event Bus ou inputs passados pelo shell.
- **R5.3** — Tipos compartilhados entre dominios vivem em `libs/shared-types/`, nao dentro de nenhum dominio.
- **R5.4** — `libs/` contem apenas codigo utilitario, componentes visuais e contratos. Logica de dominio so existe dentro de `apps/[domain]/src/app/domain/`.

## 6. Native Federation

Federacao via `@angular-architects/native-federation` (import maps + ES modules — compativel com o builder esbuild). **A ordem de setup importa: primeiro `ng add @angular/ssr`, depois `ng add @angular-architects/native-federation`** — o schematic detecta o SSR e integra os dois automaticamente.

### Host (Shell)

```js
// apps/shell/federation.config.js
const { withNativeFederation, shareAll } = require('@angular-architects/native-federation/config');

module.exports = withNativeFederation({
  shared: {
    ...shareAll({ singleton: true, strictVersion: true, requiredVersion: 'auto' }),
  },
  skip: ['rxjs/ajax', 'rxjs/fetch', 'rxjs/testing', 'rxjs/webSocket'],
});
```

```ts
// apps/shell — carregamento dos remotes por manifest
// public/federation.manifest.json (por ambiente, nunca hardcoded)
{
  "auth":    "https://auth.company.com/remoteEntry.json",
  "client":  "https://client.company.com/remoteEntry.json",
  "billing": "https://billing.company.com/remoteEntry.json"
}
```

```ts
// apps/shell/src/app/app.routes.ts — composicao por rota (padrao)
export const appRoutes: Routes = [
  {
    path: 'billing',
    canActivate: [authGuard],                          // guard no shell
    loadChildren: () =>
      loadRemoteModule('billing', './routes').then((m) => m.billingRoutes),
  },
];
```

### Remote

```js
// apps/[domain]/federation.config.js
module.exports = withNativeFederation({
  name: '[domain]',
  exposes: {
    // SOMENTE artefatos da pasta exposed/ (rotas do dominio + componentes cross-domain)
    './routes': './apps/[domain]/src/app/presentation/exposed/routes.ts',
    './InvoiceWidget': './apps/[domain]/src/app/presentation/exposed/invoice-widget.component.ts',
  },
  shared: { ...shareAll({ singleton: true, strictVersion: true, requiredVersion: 'auto' }) },
});
```

### Regras

- **R6.1** — Somente artefatos em `presentation/exposed/` podem ser listados em `exposes`.
- **R6.2** — Componentes expostos devem ser standalone e auto-contidos (nao dependem de providers do app pai; declaram seus proprios providers).
- **R6.3** — Componentes expostos consumidos no shell usam `@defer` com `@placeholder` e `@error` proprios — isso da incremental hydration de graca: o servidor renderiza, o JS so hidrata quando necessario.
- **R6.4** — **SSR habilitado no shell sempre** — o shell renderiza os remotes no lado do servidor mesmo que o remote nao ative SSR proprio. Em `ng serve` (dev), o carregamento de remote acontece so no client: use o `fallback` do `loadRemoteModule()` para evitar erro no servidor. Em producao, inicie pelo `fstart.mjs` (nao o `server.mjs` padrao do CLI).
- **R6.5** — URLs dos remotes vem do `federation.manifest.json` por ambiente, nunca hardcoded.
- **R6.6** — Toda dependencia compartilhada e `singleton: true` com `strictVersion` — especialmente `@angular/*` e RxJS. Versoes divergentes entre apps sao build quebrado, nao warning.
- **R6.7** — Nunca exponha paginas inteiras se so precisa de um componente. Exponha o minimo necessario.

## 7. SSR, Hydration e a Regra BFF

### Render mode por rota

Cada dominio declara a estrategia de renderizacao por rota em `app.routes.server.ts` — decisao registrada na techspec:

```ts
// apps/billing/src/app/app.routes.server.ts
import { RenderMode, ServerRoute } from '@angular/ssr';

export const serverRoutes: ServerRoute[] = [
  { path: 'billing/plans', renderMode: RenderMode.Prerender },   // SSG: conteudo estavel
  { path: 'billing/invoices/**', renderMode: RenderMode.Server }, // SSR: dados frescos por request
  { path: 'billing/simulator', renderMode: RenderMode.Client },   // CSR: 100% interativo, sem SEO
];
```

### Regra de Integracao Frontend-Backend (BFF Obrigatorio)

**O browser NUNCA chama o backend interno diretamente.** Toda comunicacao passa pela camada servidor:

- **Leitura (SSR + transfer cache)**: `HttpClient` executado durante a renderizacao no servidor; o resultado e serializado no HTML via hydration transfer cache — o client nao repete a requisicao.
- **Mutacoes e leituras pos-hydration (BFF)**: o browser chama endpoints do **BFF do dominio** — rotas Express registradas no `server.ts` do proprio app Angular SSR — e o BFF chama o backend interno. Tokens/secrets vivem apenas no servidor.
- **Webhooks/streaming**: rotas Express no `server.ts` do dominio.

```ts
// apps/billing/src/server.ts — BFF do dominio
app.post('/billing/api/invoices', async (req, res) => {
  const response = await backendApi.createInvoice(req.body, serverAuthToken(req));
  res.status(response.status).json(response.data);
});
```

Violacoes (componente chamando URL do backend interno, SDK de backend no bundle do browser, secret exposto no client) sao classificadas como **Critico** no review e como **bug** no QA.

### Regras

- **R7.1** — `provideClientHydration(withEventReplay(), withIncrementalHydration())` em todos os apps com SSR.
- **R7.2** — Render deterministico: nada de `Math.random()`, `new Date()` sem controle ou APIs do browser no caminho de render — mismatch de hydration e Critico.
- **R7.3** — Codigo que so roda no browser fica atras de `afterNextRender()` ou guardado por `isPlatformBrowser` — nunca no construtor/render de componentes SSR.
- **R7.4** — Conteudo pesado abaixo da dobra usa `@defer (on viewport)` para incremental hydration.
- **R7.5** — Leituras iniciais via `HttpClient` (transfer cache automatico); nunca `fetch` manual nos repositories.

## 8. Shared Libs

### `libs/ui` — Design System

```
libs/ui/src/
├── components/                  # Wrappers e composicoes sobre Angular Material
│   ├── button/
│   ├── data-table/
│   └── empty-state/
├── theme/
│   ├── _theme.scss              # Tema Material 3 (design tokens)
│   ├── _typography.scss
│   └── _density.scss
└── index.ts
```

### `libs/shared-types` — Contratos

```ts
// libs/shared-types/src/user.ts
export interface AuthenticatedUser {
  id: string;
  email: string;
  role: 'admin' | 'user' | 'viewer';
  name: string;
}

// libs/shared-types/src/events.ts
export type DomainEvents = {
  'auth:login': { user: AuthenticatedUser };
  'auth:logout': void;
  'billing:plan-upgraded': { planId: string; userId: string };
  'client:selected': { clientId: string };
};
```

### `libs/shared-events` — Event Bus Tipado

```ts
// libs/shared-events/src/event-bus.ts
import type { DomainEvents } from '@company/shared-types';

class EventBus {
  private target = new EventTarget();

  emit<K extends keyof DomainEvents>(event: K, data: DomainEvents[K]): void {
    this.target.dispatchEvent(new CustomEvent(event, { detail: data }));
  }

  on<K extends keyof DomainEvents>(
    event: K,
    callback: (data: DomainEvents[K]) => void,
  ): () => void {
    const handler = (e: Event) => callback((e as CustomEvent).detail);
    this.target.addEventListener(event, handler);
    return () => this.target.removeEventListener(event, handler);
  }
}

export const eventBus = new EventBus();
```

> O Event Bus usa `EventTarget` — so existe no browser. Consumo em componentes: registrar em `afterNextRender()` e converter para signal quando util. Nunca emitir/ouvir durante SSR.

### Regras

- **R8.1** — `libs/ui` contem apenas componentes visuais. Zero logica de negocio.
- **R8.2** — `libs/shared-types` contem apenas `type` e `interface`. Nenhuma implementacao.
- **R8.3** — Novos eventos devem ser adicionados ao type `DomainEvents` antes de serem emitidos.
- **R8.4** — `libs/util` contem apenas funcoes puras e genericas (formatDate, debounce, etc).
- **R8.5** — Nenhuma lib pode importar de `apps/`. A dependencia e unidirecional: apps → libs.
- **R8.6** — Toda lib exporta via barrel file (`index.ts`). Deep imports (`@company/ui/src/...`) sao proibidos.

## 9. Estado e Comunicacao entre Dominios

### Hierarquia de preferencia

1. **Inputs** — quando o shell passa dados diretamente ao componente exposto
2. **Event Bus** — quando dominios precisam reagir a eventos de outros
3. **Store global (signals)** — apenas para estado verdadeiramente global (usuario logado, tema, idioma)
4. **URL/Query Params** — para estado navegavel

### Regras

- **R9.1** — Cada dominio gerencia seu proprio estado interno com signals stores (`signal`/`computed` em services). Nao existe store global por dominio.
- **R9.2** — O unico estado global permitido e: usuario autenticado, tema e idioma. Tudo mais e local.
- **R9.3** — Comunicacao entre dominios usa o Event Bus tipado. Nunca chamadas diretas.
- **R9.4** — Eventos sao fire-and-forget. Se precisa de resposta, use inputs/outputs via shell.
- **R9.5** — Nao armazene estado derivavel: se pode ser calculado, use `computed()`.

## 10. Roteamento

```
shell       → / (redirect), /dashboard
auth        → /auth/login, /auth/signup, /auth/forgot-password
client      → /clients, /clients/:id, /clients/:id/edit
billing     → /billing, /billing/invoices, /billing/plans
```

- **R10.1** — Cada dominio e dono de um prefixo de rota; suas rotas internas ja nascem com o prefixo (`billing/invoices`), permitindo rodar isolado e federado sem reescrita.
- **R10.2** — O shell compoe as rotas via `loadRemoteModule('domain', './routes')` no `loadChildren`.
- **R10.3** — Links internos do dominio e navegacao composta no shell usam `routerLink` normalmente (a federacao unifica o router). Navegacao para um dominio **fora** da composicao do shell (deploy standalone) usa `window.location`.
- **R10.4** — Rotas protegidas sao guardadas pelo shell (`canActivate` antes do `loadChildren`), nao pelo dominio individual. Guards devem ser SSR-safe (sem acesso direto a `window`/`localStorage`; tokens via cookies).
- **R10.5** — Cada rota tem render mode declarado em `app.routes.server.ts` (secao 7).

## 11. Testes

### Estrategia por Camada

| Camada | Tipo de Teste | Ferramenta | Cobertura Minima |
|---|---|---|---|
| `domain/` | Unitario | Vitest | 90% |
| `application/` | Unitario | Vitest | 85% |
| `infra/` | Integracao | Vitest + MSW | 75% |
| `presentation/` | Componente | Angular Testing Library | 70% |
| E2E (criticos) | End-to-end | Playwright | Fluxos principais |

### Estrutura

Testes vivem em `__tests__/` ao lado do codigo testado (ex: `domain/entities/__tests__/invoice.entity.spec.ts`), exceto componentes, que colocam o `.spec.ts` ao lado do componente.

- `domain/` — testa regras de negocio puras, sem mocks e sem TestBed.
- `application/` — mocka o repository pela interface (provide do token com stub via TestBed).
- `infra/` — usa MSW para mockar chamadas HTTP. Nunca mock do `HttpClient` em si.
- `presentation/` — Angular Testing Library, testando comportamento do usuario.

### Regras

- **R11.1** — Testes de `domain/` nao usam mocks nem TestBed. Entidades sao testadas diretamente.
- **R11.2** — Testes de `application/` mockam repositories pela interface/token, nunca a implementacao.
- **R11.3** — Testes de `infra/` usam MSW para interceptar chamadas HTTP.
- **R11.4** — Testes de `presentation/` testam comportamento do usuario, nao implementacao.
- **R11.5** — Cada dominio roda seus testes de forma independente (`npx nx test billing`).
- **R11.6** — Testes E2E cobrem apenas fluxos criticos cross-domain (login → criar fatura → pagar), contra o app rodando em SSR — validando inclusive o HTML renderizado pelo servidor.
- **R11.7** — Nao teste getters, setters ou codigo trivial. Teste logica e comportamento.

## 12. CI/CD e Deploy

Pipeline (`.github/workflows/ci.yml`): `nx affected` detecta apps/libs alterados → lint e type-check nos afetados → testes unitarios nos afetados → build (browser + server bundles) dos afetados → E2E (se fluxos criticos foram tocados) → deploy independente por dominio.

- **R12.1** — Cada dominio tem deploy independente (Node server proprio — SSR exige runtime). Mudanca no billing nao faz redeploy do auth.
- **R12.2** — `npx nx affected -t lint,test,build` — so roda CI nos projetos afetados.
- **R12.3** — Mudancas em `libs/` disparam CI de todos os apps que usam a lib.
- **R12.4** — Mudancas no `shell` disparam E2E completo (e o orquestrador).
- **R12.5** — Cada app tem sua propria URL de deploy (auth.company.com, billing.company.com, etc); o `federation.manifest.json` do shell e atualizado a cada deploy.
- **R12.6** — Producao roda o servidor federado (`fstart.mjs`), nunca `ng serve`.
- **R12.7** — Feature flags por dominio para releases graduais. Nao fazer big-bang deploy.

## 13. Enforcement — Nx Module Boundaries

Boundaries sao enforcados via tags no `project.json` de cada projeto + `@nx/enforce-module-boundaries` no ESLint da raiz:

```jsonc
// apps/billing/project.json
{ "tags": ["scope:billing", "type:app"] }
// apps/shell/project.json
{ "tags": ["scope:shell", "type:app"] }
// libs/ui/project.json
{ "tags": ["scope:shared", "type:ui"] }
```

```js
// eslint.config.js (raiz)
{
  rules: {
    '@nx/enforce-module-boundaries': ['error', {
      depConstraints: [
        // shell consome qualquer dominio; dominios nunca se importam
        { sourceTag: 'scope:shell',   onlyDependOnLibsWithTags: ['scope:shared', 'scope:auth', 'scope:client', 'scope:billing'] },
        { sourceTag: 'scope:billing', onlyDependOnLibsWithTags: ['scope:shared'] },
        { sourceTag: 'scope:auth',    onlyDependOnLibsWithTags: ['scope:shared'] },
        { sourceTag: 'scope:client',  onlyDependOnLibsWithTags: ['scope:shared'] },
        { sourceTag: 'scope:shared',  onlyDependOnLibsWithTags: ['scope:shared'] },
      ],
    }],
  },
}
```

Boundaries **entre camadas** (domain/application/infra/presentation) sao enforcados com `no-restricted-imports` por override de path, no mesmo espirito:

```js
{
  files: ['**/domain/**/*.ts'],
  rules: {
    'no-restricted-imports': ['error', {
      patterns: [
        { group: ['@angular/*', 'rxjs*'], message: 'domain/ nao importa libs externas (excecao: InjectionToken).' },
        { group: ['**/application/**', '**/infra/**', '**/presentation/**'], message: 'domain/ e o nucleo — nao importa das outras camadas.' },
      ],
    }],
  },
},
{
  files: ['**/application/**/*.ts'],
  rules: {
    'no-restricted-imports': ['error', {
      patterns: [{ group: ['**/infra/**', '**/presentation/**'], message: 'application/ so importa de domain/.' }],
    }],
  },
},
{
  files: ['**/presentation/**/*.ts'],
  rules: {
    'no-restricted-imports': ['error', {
      patterns: [{ group: ['**/domain/**', '**/infra/**'], message: 'presentation/ acessa dados via use cases e DTOs.' }],
    }],
  },
}
```

- **R13.1** — ESLint com boundaries roda em todo PR. PR nao merge se violar boundaries.
- **R13.2** — TypeScript strict mode habilitado em todos os apps e libs.
- **R13.3** — `noImplicitAny: true` — nenhum `any` implicito permitido.
- **R13.4** — Imports usam path aliases (`@domain/`, `@application/`, `@infra/`, `@presentation/` dentro do app; `@company/*` para libs).

## 14. Checklist de Novo Dominio

```
[ ] 1.  Gerar app em apps/[domain]/ (npx nx g @nx/angular:app) seguindo a estrutura padrao
[ ] 2.  ng add @angular/ssr --project [domain] (SEMPRE antes da federacao)
[ ] 3.  ng add @angular-architects/native-federation --project [domain] --type remote
[ ] 4.  Configurar tsconfig com path aliases das camadas e tags no project.json
[ ] 5.  Definir entidades e interfaces de repository (+ tokens) em domain/
[ ] 6.  Implementar pelo menos 1 use case em application/
[ ] 7.  Implementar repositories em infra/ (HttpClient)
[ ] 8.  Criar [domain].routes.ts e exposed/routes.ts; declarar render modes em app.routes.server.ts
[ ] 9.  Criar endpoints BFF no server.ts para as mutacoes do dominio
[ ] 10. Registrar o remote no federation.manifest.json do shell + rota com loadRemoteModule
[ ] 11. Adicionar eventos do dominio em libs/shared-types DomainEvents
[ ] 12. Adicionar depConstraints do novo scope no eslint.config.js
[ ] 13. Configurar pipeline de deploy separado (Node server)
[ ] 14. Adicionar testes unitarios para domain/ e application/
[ ] 15. Documentar rotas, render modes e artefatos expostos no README do dominio
[ ] 16. Validar que o app roda isolado (npx nx serve [domain]) e federado no shell
```

## 15. Anti-patterns

| Anti-pattern | Por que e ruim | O que fazer |
|---|---|---|
| Importar de remote para remote | Acoplamento direto, quebra independencia | Usar Event Bus ou inputs via shell |
| Logica de negocio em componente | Nao testavel, nao reutilizavel | Mover para domain/entities ou application/use-cases |
| Logica de negocio em libs/ | Lib vira dominio oculto | Manter logica dentro de apps/[domain]/domain/ |
| Store global por dominio | Estado espalhado, dificil de debugar | Estado global so para user, tema, idioma |
| Expor componentes internos via federacao | Over-sharing, acoplamento | So expor o que esta em exposed/ |
| `any` em DTOs | Perde type safety entre camadas | Tipar DTOs como interfaces explicitas |
| Use case importando Angular UI/Router | Application acoplada ao framework | Use case so usa TypeScript puro + inject de contratos |
| Repository sem interface/token | Nao pode trocar implementacao, nao pode testar | Sempre definir interface + InjectionToken em domain/ |
| Componente consumindo API direto | Pula todas as camadas, nao testavel | Componente → use case → repository (token) → HttpClient |
| Browser chamando backend interno | Vaza contratos/secrets, quebra a regra BFF | Leitura via SSR + transfer cache; mutacao via BFF do dominio |
| `fetch` manual no repository | Perde transfer cache e interceptors | Sempre HttpClient |
| Zone.js num app, zoneless em outro | Change detection inconsistente no documento federado | Todos zoneless |
| Versoes divergentes de @angular/* entre apps | Dois runtimes no documento — DI e CD quebram | Single version policy na raiz + strictVersion |
| Testes que testam implementacao | Quebram a cada refactor | Testar comportamento e contratos |
| Deploy de todos os apps juntos | Anula o beneficio de microfrontends | Deploy independente por dominio |

## Fluxo de Dados — Resumo

```
Componente (presentation)
    → use case (application) [injetado via DI]
        → interface de repository (domain/repositories, via InjectionToken)
            → HttpRepository (infra) → HttpClient
                → [no servidor] backend interno direto (SSR + transfer cache)
                → [no browser]  BFF do dominio (server.ts) → backend interno
            ← Entity
        ← DTO (via Mapper)
    ← signal/estado para a UI
```

Nunca pule camadas: componente nao chama API direto, componente nao instancia repository, use case nao conhece implementacao concreta.
