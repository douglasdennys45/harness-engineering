---
name: nextjs-clean-architecture
description: Aplica as regras de arquitetura do monorepo Next.js com Turborepo, Multi-Zones (microfrontends) e Clean Architecture por dominio. Cobre estrutura do monorepo (apps/shell como zona principal/roteador, apps/[domain] como zonas independentes, packages/ compartilhados), camadas por dominio (domain/, application/, infra/, presentation/, di/), regras de dependencia entre camadas e entre dominios, nomenclatura (kebab-case, PascalCase, sufixos DTO/Mapper/Repository), Multi-Zones (basePath por zona, rewrites no shell, hard navigation entre zonas, env vars), shared packages (ui, shared-types, shared-events, http-client, utils), comunicacao entre dominios (URL, cookies, backend, BroadcastChannel), roteamento (basePath, rewrites, auth guard no shell), estrategia de testes por camada (Vitest, MSW, Testing Library, Playwright), CI/CD com deploy independente por dominio, e enforcement via ESLint boundaries. Use sempre que criar/editar codigo em apps/ ou packages/, ao criar um novo dominio/zona, ao definir use cases, entidades, repositories, DTOs, mappers, stores ou componentes, ao configurar next.config.js com basePath/rewrites, ao revisar imports entre camadas ou dominios, ao escrever testes, ou ao discutir arquitetura de projetos Next.js com monorepo.
---

# Next.js Monorepo — Clean Architecture e Microfrontends (Multi-Zones)

Skill baseada no documento de referencia de arquitetura para projetos Next.js com monorepo (Turborepo), Multi-Zones e Clean Architecture. Todas as regras aqui devem ser seguidas ao escrever, revisar ou refatorar codigo do monorepo.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer arquivo em `apps/` ou `packages/`
- Ao criar um novo dominio (zona)
- Ao definir entidades, use cases, repositories, DTOs, mappers ou componentes
- Ao configurar Multi-Zones (basePath, rewrites), roteamento ou pipelines de deploy
- Ao revisar imports entre camadas ou entre dominios

## 0. Por que Multi-Zones (e nao Module Federation)

Module Federation foi descartado como estrategia de microfrontends neste projeto pelas limitacoes concretas do ecossistema Next.js:

| Limitacao do Module Federation | Consequencia |
|---|---|
| `@module-federation/nextjs-mf` foi descontinuado e nao recebe manutencao | Risco de ficar preso em versoes antigas do Next.js |
| Sem suporte a App Router e React Server Components | Obriga o projeto inteiro a permanecer no Pages Router |
| Remotes exigem `ssr: false` na pratica | Perde SSR/streaming justamente na camada composta |
| Dependencias singleton exigem versoes sincronizadas entre todos os apps | Acopla upgrades de React/libs — anula a independencia dos times |
| Plugin depende de webpack | Incompativel com Turbopack e com a evolucao do build do Next.js |

**Multi-Zones** e a abordagem oficial do Next.js para microfrontends: cada dominio e um app Next.js completo e independente, composto por **rota** (rewrites no shell) em vez de por componente em runtime. Nao ha runtime compartilhado — cada zona faz build, deploy e upgrade de dependencias de forma totalmente independente, com SSR/RSC funcionando normalmente. Compartilhamento de codigo acontece em **build-time** via `packages/` (Turborepo + pnpm workspaces).

Trade-off aceito: navegacao entre zonas e um full page load (hard navigation). Por isso os dominios devem ser divididos por areas de navegacao pouco frequente entre si (auth, clients, billing), nunca por componentes de uma mesma tela.

## 1. Estrutura do Monorepo

```
root/
├── apps/
│   ├── shell/                    # Zona principal (layout, nav, auth guard, roteia via rewrites)
│   ├── auth/                     # Dominio: autenticacao e autorizacao
│   ├── client/                   # Dominio: gestao de clientes
│   ├── billing/                  # Dominio: cobranca, faturas, planos
│   └── [domain]/                 # Novos dominios seguem o mesmo padrao
│
├── packages/
│   ├── ui/                       # Design system compartilhado
│   ├── shared-types/             # Contratos e tipos entre dominios
│   ├── shared-events/            # Eventos cross-zone (BroadcastChannel tipado)
│   ├── http-client/              # Adapter HTTP base (axios/fetch configurado)
│   ├── utils/                    # Helpers genericos sem logica de negocio
│   ├── eslint-config/            # Regras ESLint compartilhadas
│   └── tsconfig/                 # Configuracoes TypeScript base
│
├── turbo.json
├── pnpm-workspace.yaml
├── package.json
└── .github/workflows/            # ci.yml + deploy-[domain].yml
```

### Regras

- **R1.1** — Cada dominio de negocio e um app Next.js independente (uma zona) dentro de `apps/`.
- **R1.2** — Codigo compartilhado entre dominios vive exclusivamente em `packages/`.
- **R1.3** — O `shell` e a unica zona principal: recebe todo o trafego do dominio publico e roteia para as demais zonas via `rewrites`. Nenhuma outra zona roteia para outra.
- **R1.4** — Cada app deve ser executavel de forma isolada (`pnpm dev --filter=billing`).
- **R1.5** — Usar `pnpm` como package manager com workspaces.
- **R1.6** — Usar Turborepo para orquestracao de builds, com cache habilitado.

## 2. Nomenclatura

### Diretorios e Arquivos

| Tipo | Convencao | Exemplo |
|---|---|---|
| Diretorios | `kebab-case` | `value-objects/`, `use-cases/` |
| Componentes React | `PascalCase.tsx` | `InvoiceTable.tsx` |
| Hooks | `camelCase.ts` com prefixo `use` | `useCreateInvoice.ts` |
| Use Cases (classe) | `PascalCase.ts` (verbo + substantivo) | `CreateInvoice.ts` |
| Entidades | `PascalCase.ts` | `Invoice.ts` |
| DTOs | `PascalCase.ts` com sufixo `DTO` | `CreateInvoiceDTO.ts` |
| Mappers | `PascalCase.ts` com sufixo `Mapper` | `InvoiceMapper.ts` |
| Repositories (interface) | `PascalCase.ts` com sufixo `Repository` | `InvoiceRepository.ts` |
| Repositories (impl) | `PascalCase.ts` com prefixo `Http` | `HttpInvoiceRepository.ts` |
| Store | `kebab-case.ts` com sufixo `-store` | `billing-store.ts` |
| Tipos/Interfaces | `PascalCase.ts` | `BillingTypes.ts` |
| Constantes | `UPPER_SNAKE_CASE` | `MAX_RETRY_ATTEMPTS` |

### Branches Git

```
feat/[domain]/[descricao]       → feat/billing/add-invoice-export
fix/[domain]/[descricao]        → fix/auth/token-refresh-loop
chore/[domain]/[descricao]      → chore/shell/upgrade-next-15
refactor/[domain]/[descricao]   → refactor/client/extract-use-cases
```

## 3. Clean Architecture por Dominio

Cada app em `apps/` segue a mesma estrutura interna:

```
apps/[domain]/src/
├── domain/                      # Nucleo — zero dependencias externas
│   ├── entities/                # Entidades com regras de negocio
│   ├── value-objects/           # Objetos de valor imutaveis
│   ├── errors/                  # Erros de dominio tipados
│   └── repositories/            # Interfaces (ports) — so contratos
│
├── application/                 # Orquestracao — depende so do domain
│   ├── use-cases/               # Casos de uso (1 classe = 1 acao)
│   ├── dtos/                    # Input/Output dos use cases
│   └── mappers/                 # Conversao Entity <-> DTO
│
├── infra/                       # Implementacoes concretas
│   ├── repositories/            # Implementam interfaces do domain
│   ├── gateways/                # Integracoes externas (Stripe, etc)
│   ├── http/                    # Configuracao HTTP do dominio
│   └── store/                   # Estado local (Zustand/etc)
│
├── presentation/                # React/Next.js — camada mais externa
│   ├── pages/ ou app/           # Rotas do Next.js (sob o basePath da zona)
│   ├── components/              # Componentes do dominio
│   ├── hooks/                   # Conectam UI aos use cases
│   ├── providers/               # Context providers do dominio
│   └── layouts/                 # Layouts do dominio
│
└── di/                          # Injecao de dependencia
    └── container.ts             # Composicao: instancia adapters e injeta nos use cases
```

### Regras

- **R3.1** — `domain/` nao importa NENHUMA lib externa (nem React, nem Axios, nem Zustand). Somente TypeScript puro.
- **R3.2** — `application/` importa somente de `domain/`. Use cases recebem interfaces via construtor (DI).
- **R3.3** — `infra/` implementa as interfaces definidas em `domain/repositories/`.
- **R3.4** — `presentation/` nunca importa de `domain/` diretamente. Acessa dados via DTOs retornados pelos use cases.
- **R3.5** — `di/container.ts` e o unico arquivo que conhece todas as camadas. Ele instancia adapters e injeta nos use cases.
- **R3.6** — Cada use case e uma classe com um unico metodo publico `execute()`.
- **R3.7** — Validacoes de negocio vivem na entidade, nao no componente.
- **R3.8** — Value Objects sao imutaveis. Toda mutacao retorna uma nova instancia.
- **R3.9** — DTOs sao objetos simples (interfaces). Nao possuem metodos nem logica.
- **R3.10** — Mappers sao classes com metodos estaticos. Nao possuem estado.

## 4. Dependencia entre Camadas

```
PERMITIDO (→ = "pode importar de")

  presentation → application (via hooks → use cases)
  presentation → di (para obter instancias)
  infra        → domain (para implementar interfaces)
  infra        → application (para usar DTOs se necessario)
  application  → domain (para usar entidades e interfaces)
  di           → todas as camadas (e o compositor)

PROIBIDO (✗)

  domain       ✗ application, infra, presentation
  application  ✗ infra, presentation
  infra        ✗ presentation
  presentation ✗ domain diretamente (use DTOs)
```

| Camada | Importa de | Nunca importa de |
|---|---|---|
| `domain/` | Nada (TS puro) | `application/`, `infra/`, `presentation/` |
| `application/` | `domain/` | `infra/`, `presentation/` |
| `infra/` | `domain/`, `application/` | `presentation/` |
| `presentation/` | `application/`, `di/` | `domain/` direto |
| `di/` | Todas | — |

## 5. Dependencia entre Dominios

```
PERMITIDO
  zona    → packages/* (libs compartilhadas, build-time)
  shell   → packages/* (o shell tambem e um app Next.js normal)

PROIBIDO
  zona    ✗ outra zona (NUNCA importacao direta)
  zona    ✗ shell (NUNCA depender da zona principal)
  shell   ✗ codigo interno de qualquer zona (a composicao e via rewrites, nao via import)
  domain logic em packages/ (packages nao contem logica de negocio)
```

- **R5.1** — Dominios NUNCA importam diretamente uns dos outros. O shell tambem nao importa codigo das zonas — ele apenas roteia requests via `rewrites`.
- **R5.2** — Se billing precisa de dados do auth, a comunicacao ocorre via URL, cookies ou backend (ver secao 8). Nunca via import ou estado em memoria.
- **R5.3** — Tipos compartilhados entre dominios vivem em `packages/shared-types/`, nao dentro de nenhum dominio.
- **R5.4** — `packages/` contem apenas codigo utilitario, UI e contratos. Logica de dominio so existe dentro de `apps/[domain]/src/domain/`.
- **R5.5** — UI compartilhada entre zonas vive em `packages/ui` e e resolvida em build-time (workspace). Nao existe compartilhamento de componentes em runtime.

## 6. Multi-Zones

### Zona principal (Shell)

O shell recebe todo o trafego do dominio publico (ex: `app.company.com`) e encaminha cada prefixo de rota para a zona correspondente:

```js
// apps/shell/next.config.js
module.exports = {
  async rewrites() {
    return [
      { source: '/auth',           destination: `${process.env.AUTH_URL}/auth` },
      { source: '/auth/:path*',    destination: `${process.env.AUTH_URL}/auth/:path*` },
      { source: '/clients',        destination: `${process.env.CLIENT_URL}/clients` },
      { source: '/clients/:path*', destination: `${process.env.CLIENT_URL}/clients/:path*` },
      { source: '/billing',        destination: `${process.env.BILLING_URL}/billing` },
      { source: '/billing/:path*', destination: `${process.env.BILLING_URL}/billing/:path*` },
    ];
  },
};
```

Como cada zona define `basePath`, seus assets (`/billing/_next/...`) e API routes ja chegam prefixados e sao roteados pelos mesmos rewrites — sem conflito entre zonas.

### Zona (dominio)

```js
// apps/billing/next.config.js
module.exports = {
  basePath: '/billing',   // igual ao prefixo de rota do dominio — obrigatorio e unico
};
```

### Navegacao entre zonas

```tsx
// Dentro da mesma zona: next/link normalmente
<Link href="/billing/invoices">Faturas</Link>

// Entre zonas: ancora nativa (hard navigation — apps diferentes)
<a href="/clients">Clientes</a>
```

### Regras

- **R6.1** — Cada zona define um `basePath` unico, identico ao seu prefixo de rota. Nenhuma rota pode existir fora do `basePath` da zona.
- **R6.2** — O shell e o unico ponto de entrada publico. As URLs internas das zonas vem de variaveis de ambiente nos `rewrites`, nunca hardcoded.
- **R6.3** — Navegacao entre zonas usa `<a href>` (hard navigation). `next/link` entre zonas e proibido — quebra prefetch e client-side routing.
- **R6.4** — Zonas nao compartilham runtime: nada de estado em memoria, contexts ou singletons atravessando zonas. Cada zona carrega seu proprio bundle de React.
- **R6.5** — Cada zona renderiza paginas completas com SSR/RSC normalmente. Nao ha restricao de `ssr: false`.
- **R6.6** — Cada zona pode fazer upgrade de dependencias (React, Next.js, libs) de forma independente. Nao ha exigencia de versoes sincronizadas — mas mantenha `packages/*` compativeis com todas as zonas que os consomem.
- **R6.7** — Divida zonas por area de navegacao (usuario transita pouco entre elas), nunca por componentes de uma mesma tela. Se duas "zonas" precisam compor a mesma pagina, elas sao um unico dominio.

## 7. Shared Packages

### `packages/ui` — Design System

```
packages/ui/src/
├── components/
│   ├── Button/
│   │   ├── Button.tsx
│   │   ├── Button.styles.ts
│   │   ├── Button.test.tsx
│   │   └── index.ts
│   ├── Input/
│   └── Modal/
├── tokens/
│   ├── colors.ts
│   ├── spacing.ts
│   └── typography.ts
└── index.ts
```

### `packages/shared-types` — Contratos

```ts
// packages/shared-types/src/user.ts
export interface AuthenticatedUser {
  id: string;
  email: string;
  role: 'admin' | 'user' | 'viewer';
  name: string;
}

// packages/shared-types/src/events.ts
export type CrossZoneEvents = {
  'auth:logout': void;
  'billing:plan-upgraded': { planId: string; userId: string };
  'client:selected': { clientId: string };
};
```

### `packages/shared-events` — Eventos Cross-Zone (BroadcastChannel)

Zonas nao compartilham runtime, entao um event bus em memoria (`EventTarget`) so funciona dentro de uma unica pagina. Para eventos entre zonas abertas em abas/janelas diferentes da mesma origem, usa-se `BroadcastChannel`:

```ts
// packages/shared-events/src/cross-zone-bus.ts
import type { CrossZoneEvents } from '@company/shared-types';

class CrossZoneBus {
  private channel = new BroadcastChannel('app-events');

  emit<K extends keyof CrossZoneEvents>(event: K, data: CrossZoneEvents[K]): void {
    this.channel.postMessage({ event, data });
  }

  on<K extends keyof CrossZoneEvents>(
    event: K,
    callback: (data: CrossZoneEvents[K]) => void,
  ): () => void {
    const handler = (e: MessageEvent) => {
      if (e.data?.event === event) callback(e.data.data);
    };
    this.channel.addEventListener('message', handler);
    return () => this.channel.removeEventListener('message', handler);
  }
}

export const crossZoneBus = new CrossZoneBus();
```

Escopo e limites: `BroadcastChannel` alcanca outras abas/janelas da mesma origem, **nao** a propria pagina emissora, e os eventos **nao sobrevivem a navegacao** (hard navigation entre zonas recomeca o runtime). Estado que precisa atravessar navegacao usa URL, cookie ou backend (secao 8). Uso tipico: propagar `auth:logout` para todas as abas abertas.

### Regras

- **R7.1** — `packages/ui` contem apenas componentes visuais. Zero logica de negocio.
- **R7.2** — `packages/shared-types` contem apenas `type` e `interface`. Nenhuma implementacao.
- **R7.3** — Novos eventos devem ser adicionados ao type `CrossZoneEvents` antes de serem emitidos.
- **R7.4** — `packages/utils` contem apenas funcoes puras e genericas (formatDate, debounce, etc).
- **R7.5** — Nenhum package pode importar de `apps/`. A dependencia e unidirecional: apps → packages.
- **R7.6** — Todo package exporta via barrel file (`index.ts`). Imports internos de package nao sao permitidos (ex: `@company/ui/src/Button` e proibido).

## 8. Estado e Comunicacao entre Dominios

Nao existe runtime compartilhado entre zonas. Todo estado que atravessa dominios precisa viver em um meio que sobreviva a hard navigation.

### Hierarquia de preferencia

1. **URL/Query Params** — estado navegavel e compartilhavel (`/billing/invoices?clientId=123`). Sobrevive a navegacao entre zonas e e a forma padrao de uma zona "passar dados" para outra.
2. **Cookies** — sessao/identidade (usuario autenticado, tema, idioma). Lidos por qualquer zona no server (SSR/RSC) e no client, pois todas servem sob o mesmo dominio.
3. **Backend/API** — fonte de verdade para dados de negocio. Se billing precisa de dados do cliente selecionado, busca na API com o id vindo da URL/cookie.
4. **BroadcastChannel (`shared-events`)** — apenas para notificar outras abas/janelas abertas (ex: logout global). Nunca como transporte de dados de negocio.

### Regras

- **R8.1** — Cada dominio gerencia seu proprio estado interno (Zustand/etc). Stores nunca atravessam zonas.
- **R8.2** — O unico estado global permitido e: usuario autenticado, tema e idioma — e ele vive em cookies, nao em stores em memoria.
- **R8.3** — Comunicacao entre dominios usa URL, cookies ou backend. Nunca imports diretos, nunca estado em memoria compartilhado.
- **R8.4** — Eventos via BroadcastChannel sao fire-and-forget e restritos a notificacoes entre abas. Se precisa de dados, busque na API.
- **R8.5** — Nao armazene estado derivavel. Se pode ser calculado a partir de outro estado, calcule.

## 9. Roteamento

```
shell       → / (redirect), /dashboard
auth        → /auth/login, /auth/signup, /auth/forgot-password
client      → /clients, /clients/:id, /clients/:id/edit
billing     → /billing, /billing/invoices, /billing/plans
```

- **R9.1** — Cada dominio usa `basePath` no `next.config.js` correspondente ao seu prefixo de rota.
- **R9.2** — O shell configura `rewrites` para rotear cada prefixo para a zona correta (secao 6).
- **R9.3** — Links entre zonas usam `<a href>`, nao `next/link` (apps diferentes — hard navigation).
- **R9.4** — Links internos do dominio usam `next/link` normalmente.
- **R9.5** — Rotas protegidas sao guardadas pelo shell (auth guard no middleware da zona principal), com validacao de sessao tambem em cada zona (defesa em profundidade — a zona nao confia que so recebe trafego do shell).

## 10. Testes

### Estrategia por Camada

| Camada | Tipo de Teste | Ferramenta | Cobertura Minima |
|---|---|---|---|
| `domain/` | Unitario | Vitest | 90% |
| `application/` | Unitario | Vitest | 85% |
| `infra/` | Integracao | Vitest + MSW | 75% |
| `presentation/` | Componente | Testing Library | 70% |
| E2E (criticos) | End-to-end | Playwright | Fluxos principais |

### Estrutura

Testes vivem em `__tests__/` ao lado do codigo testado (ex: `domain/entities/__tests__/Invoice.test.ts`), exceto componentes, que colocam o `.test.tsx` ao lado do componente.

- `domain/` — testa regras de negocio puras, sem mocks.
- `application/` — mocka o repository via interface.
- `infra/` — usa MSW para mockar chamadas HTTP. Nunca mock do axios/fetch.
- `presentation/` — Testing Library, testando comportamento do usuario.

### Regras

- **R10.1** — Testes de `domain/` nao usam mocks. Entidades sao testadas diretamente.
- **R10.2** — Testes de `application/` mockam repositories pela interface, nunca a implementacao.
- **R10.3** — Testes de `infra/` usam MSW para interceptar chamadas HTTP. Nunca mock do axios/fetch.
- **R10.4** — Testes de `presentation/` testam comportamento do usuario, nao implementacao.
- **R10.5** — Cada dominio roda seus testes de forma independente (`pnpm test --filter=billing`).
- **R10.6** — Testes E2E cobrem apenas fluxos criticos cross-domain (login → criar fatura → pagar), exercitando a navegacao real entre zonas via shell.
- **R10.7** — Nao teste getters, setters ou codigo trivial. Teste logica e comportamento.

## 11. CI/CD e Deploy

Pipeline (`.github/workflows/ci.yml`): detecta apps/packages alterados → lint e type-check nos afetados → testes unitarios nos afetados → build dos afetados → E2E (se fluxos criticos foram tocados) → deploy independente por dominio.

- **R11.1** — Cada dominio tem deploy independente. Mudanca no billing nao faz redeploy do auth.
- **R11.2** — Turborepo `--filter` detecta o que mudou. So roda CI nos apps afetados.
- **R11.3** — Mudancas em `packages/` disparam CI de todos os apps que usam o package.
- **R11.4** — Mudancas no `shell` disparam E2E completo (e a zona principal que roteia tudo).
- **R11.5** — Cada zona tem sua propria URL interna de deploy (auth.internal.company.com, billing.internal.company.com, etc). O usuario final so ve o dominio do shell.
- **R11.6** — As env vars com as URLs das zonas (`AUTH_URL`, `BILLING_URL`, ...) sao configuracao do shell. Como os rewrites as resolvem por request, deploy de uma zona nao exige redeploy do shell — apenas atualizacao da env var se a URL mudar.
- **R11.7** — Feature flags por dominio para releases graduais. Nao fazer big-bang deploy.

## 12. ESLint e Enforcement

Boundaries sao enforcados via `no-restricted-imports` em `packages/eslint-config/rules/architecture.js`:

```js
module.exports = {
  rules: {
    'no-restricted-imports': ['error', {
      patterns: [
        {
          group: ['react', 'react-dom', 'next/*', 'axios', 'zustand', '@tanstack/*'],
          message: 'A camada domain/ nao pode importar libs externas.',
        },
      ],
    }],
  },

  overrides: [
    {
      files: ['**/domain/**/*.ts'],
      rules: {
        'no-restricted-imports': ['error', {
          patterns: [
            { group: ['**/application/**'], message: 'domain/ nao importa de application/' },
            { group: ['**/infra/**'], message: 'domain/ nao importa de infra/' },
            { group: ['**/presentation/**'], message: 'domain/ nao importa de presentation/' },
            { group: ['**/di/**'], message: 'domain/ nao importa de di/' },
          ],
        }],
      },
    },
    {
      files: ['**/application/**/*.ts'],
      rules: {
        'no-restricted-imports': ['error', {
          patterns: [
            { group: ['**/infra/**'], message: 'application/ nao importa de infra/' },
            { group: ['**/presentation/**'], message: 'application/ nao importa de presentation/' },
          ],
        }],
      },
    },
    {
      files: ['**/infra/**/*.ts'],
      rules: {
        'no-restricted-imports': ['error', {
          patterns: [
            { group: ['**/presentation/**'], message: 'infra/ nao importa de presentation/' },
          ],
        }],
      },
    },
    {
      files: ['apps/auth/**', 'apps/client/**', 'apps/billing/**'],
      rules: {
        'no-restricted-imports': ['error', {
          patterns: [
            { group: ['apps/shell/**'], message: 'Zona nao importa do shell' },
            { group: ['apps/auth/**', 'apps/client/**', 'apps/billing/**'],
              message: 'Zonas nao importam umas das outras' },
          ],
        }],
      },
    },
    {
      files: ['apps/shell/**'],
      rules: {
        'no-restricted-imports': ['error', {
          patterns: [
            { group: ['apps/auth/**', 'apps/client/**', 'apps/billing/**'],
              message: 'Shell nao importa codigo das zonas — composicao e via rewrites' },
          ],
        }],
      },
    },
  ],
};
```

- **R12.1** — ESLint com regras de boundary roda em todo PR. PR nao merge se violar boundaries.
- **R12.2** — TypeScript strict mode habilitado em todos os apps e packages.
- **R12.3** — `noImplicitAny: true` — nenhum `any` implicito permitido.
- **R12.4** — Imports devem usar path aliases (`@domain/`, `@application/`, `@infra/`, `@presentation/`).

## 13. Checklist de Novo Dominio

```
[ ] 1.  Criar app em apps/[domain]/ seguindo a estrutura padrao
[ ] 2.  Configurar next.config.js com basePath igual ao prefixo de rota do dominio
[ ] 3.  Configurar tsconfig.json com path aliases das camadas
[ ] 4.  Criar di/container.ts com as dependencias do dominio
[ ] 5.  Definir entidades e interfaces de repository em domain/
[ ] 6.  Implementar pelo menos 1 use case em application/
[ ] 7.  Implementar repositories em infra/
[ ] 8.  Criar paginas/rotas em presentation/ sob o basePath da zona
[ ] 9.  Adicionar rewrites no shell para o basePath do novo dominio (+ env var [DOMAIN]_URL)
[ ] 10. Garantir validacao de sessao na zona (alem do auth guard do shell)
[ ] 11. Adicionar eventos do dominio em packages/shared-types CrossZoneEvents (se houver)
[ ] 12. Configurar pipeline de deploy separado
[ ] 13. Adicionar testes unitarios para domain/ e application/
[ ] 14. Documentar rotas do dominio no README do dominio
[ ] 15. Validar que o app roda isolado (pnpm dev --filter=[domain])
[ ] 16. Validar a navegacao shell → zona → shell (hard navigation, assets sob o basePath)
```

## 14. Anti-patterns

| Anti-pattern | Por que e ruim | O que fazer |
|---|---|---|
| Importar de zona para zona | Acoplamento direto, quebra independencia | Comunicar via URL, cookies ou backend |
| Shell importar codigo de uma zona | Recria o acoplamento que Multi-Zones elimina | Compor via rewrites; UI comum em packages/ui |
| `next/link` entre zonas | Prefetch e client routing quebram entre apps | Usar `<a href>` (hard navigation) |
| Compartilhar componente entre zonas em runtime | Reintroduz os problemas do Module Federation | Extrair para packages/ui (build-time) |
| Estado em memoria cross-zone (store/context global) | Nao sobrevive a hard navigation entre zonas | URL, cookies ou backend |
| Dividir uma mesma tela em duas zonas | Navegacao vira full page load a cada interacao | Zonas = areas de navegacao; mesma tela = mesmo dominio |
| Logica de negocio em componente | Nao testavel, nao reutilizavel | Mover para domain/entities ou application/use-cases |
| Logica de negocio em packages/ | Package vira dominio oculto | Manter logica dentro de apps/[domain]/domain/ |
| Store global por dominio | Estado espalhado, dificil de debugar | Estado global so para user, tema, idioma (via cookies) |
| `any` em DTOs | Perde type safety entre camadas | Tipar DTOs como interfaces explicitas |
| Use case com dependencia de React | Application fica acoplada ao framework | Use case so usa TypeScript puro |
| Repository sem interface | Nao pode trocar implementacao, nao pode testar | Sempre definir interface em domain/ |
| Componente consumindo API direto | Pula todas as camadas, nao testavel | Componente → hook → use case → repository |
| Testes que testam implementacao | Quebram a cada refactor | Testar comportamento e contratos |
| Deploy de todos os apps juntos | Anula o beneficio de microfrontends | Deploy independente por dominio |

## Fluxo de Dados — Resumo

```
Componente (presentation)
    → hook (presentation/hooks)
        → use case (application) [obtido via di/container.ts]
            → interface de repository (domain/repositories)
                → HttpRepository (infra) → API
            ← Entity
        ← DTO (via Mapper)
    ← estado para a UI
```

Nunca pule camadas: componente nao chama API direto, hook nao instancia repository, use case nao conhece implementacao concreta.
