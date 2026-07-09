---
name: nextjs-clean-architecture
description: Aplica as regras de arquitetura do monorepo Next.js com Turborepo, Module Federation (microfrontends) e Clean Architecture por dominio. Cobre estrutura do monorepo (apps/shell como unico host, apps/[domain] como remotes, packages/ compartilhados), camadas por dominio (domain/, application/, infra/, presentation/, di/), regras de dependencia entre camadas e entre dominios, nomenclatura (kebab-case, PascalCase, sufixos DTO/Mapper/Repository), Module Federation (exposes via presentation/exposed/, singletons, env vars), shared packages (ui, shared-types, shared-events, http-client, utils), comunicacao entre dominios (props, Event Bus tipado, estado global minimo), roteamento (basePath, rewrites, auth guard no shell), estrategia de testes por camada (Vitest, MSW, Testing Library, Playwright), CI/CD com deploy independente por dominio, e enforcement via ESLint boundaries. Use sempre que criar/editar codigo em apps/ ou packages/, ao criar um novo dominio/microfrontend, ao definir use cases, entidades, repositories, DTOs, mappers, stores ou componentes expostos, ao configurar next.config.js com NextFederationPlugin, ao revisar imports entre camadas ou dominios, ao escrever testes, ou ao discutir arquitetura de projetos Next.js com monorepo.
---

# Next.js Monorepo — Clean Architecture e Microfrontends

Skill baseada no documento de referencia de arquitetura para projetos Next.js com monorepo (Turborepo), Module Federation e Clean Architecture. Todas as regras aqui devem ser seguidas ao escrever, revisar ou refatorar codigo do monorepo.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer arquivo em `apps/` ou `packages/`
- Ao criar um novo dominio (microfrontend)
- Ao definir entidades, use cases, repositories, DTOs, mappers ou componentes
- Ao configurar Module Federation, roteamento ou pipelines de deploy
- Ao revisar imports entre camadas ou entre dominios

## 1. Estrutura do Monorepo

```
root/
├── apps/
│   ├── shell/                    # Host principal (layout, nav, auth guard, roteamento)
│   ├── auth/                     # Dominio: autenticacao e autorizacao
│   ├── client/                   # Dominio: gestao de clientes
│   ├── billing/                  # Dominio: cobranca, faturas, planos
│   └── [domain]/                 # Novos dominios seguem o mesmo padrao
│
├── packages/
│   ├── ui/                       # Design system compartilhado
│   ├── shared-types/             # Contratos e tipos entre dominios
│   ├── shared-events/            # Event bus tipado
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

- **R1.1** — Cada dominio de negocio e um app Next.js independente dentro de `apps/`.
- **R1.2** — Codigo compartilhado entre dominios vive exclusivamente em `packages/`.
- **R1.3** — O `shell` e o unico host. Nenhum outro app pode ser host.
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
│   ├── pages/ ou app/           # Rotas do Next.js
│   ├── components/              # Componentes do dominio
│   ├── exposed/                 # Componentes expostos via Module Federation
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
  shell   → qualquer remote (consome componentes expostos)
  remote  → packages/* (libs compartilhadas)
  remote  → shared-events (comunicacao desacoplada)

PROIBIDO
  remote  ✗ outro remote (NUNCA importacao direta)
  remote  ✗ shell (NUNCA depender do host)
  domain logic em packages/ (packages nao contem logica de negocio)
```

- **R5.1** — Dominios NUNCA importam diretamente uns dos outros.
- **R5.2** — Se billing precisa de dados do auth, a comunicacao ocorre via Event Bus ou props passadas pelo shell.
- **R5.3** — Tipos compartilhados entre dominios vivem em `packages/shared-types/`, nao dentro de nenhum dominio.
- **R5.4** — `packages/` contem apenas codigo utilitario e contratos. Logica de dominio so existe dentro de `apps/[domain]/src/domain/`.

## 6. Module Federation

### Host (Shell)

```js
// apps/shell/next.config.js
const { NextFederationPlugin } = require('@module-federation/nextjs-mf');

module.exports = {
  webpack(config) {
    config.plugins.push(
      new NextFederationPlugin({
        name: 'shell',
        remotes: {
          auth:    `auth@${process.env.AUTH_URL}/_next/static/chunks/remoteEntry.js`,
          client:  `client@${process.env.CLIENT_URL}/_next/static/chunks/remoteEntry.js`,
          billing: `billing@${process.env.BILLING_URL}/_next/static/chunks/remoteEntry.js`,
        },
        shared: {
          react: { singleton: true, requiredVersion: '^18.0.0' },
          'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
          '@company/ui': { singleton: true },
          zustand: { singleton: true },
        },
      })
    );
    return config;
  },
};
```

### Remote

```js
// apps/[domain]/next.config.js
new NextFederationPlugin({
  name: '[domain]',
  filename: 'static/chunks/remoteEntry.js',
  exposes: {
    // SOMENTE componentes da pasta exposed/
    './ComponentName': './src/presentation/exposed/ComponentName',
  },
  shared: { /* mesma config do host */ },
});
```

### Regras

- **R6.1** — Somente componentes em `presentation/exposed/` podem ser listados em `exposes`.
- **R6.2** — Componentes expostos devem ser auto-contidos (nao dependem de providers do dominio pai).
- **R6.3** — Componentes expostos devem ter fallback de loading e error boundary proprios.
- **R6.4** — O consumo de remotes no host usa `next/dynamic` com `ssr: false` como padrao.
- **R6.5** — URLs dos remotes vem de variaveis de ambiente, nunca hardcoded.
- **R6.6** — Toda dependencia singleton deve ter a mesma `requiredVersion` em todos os apps.
- **R6.7** — Nunca exponha paginas inteiras se so precisa de um componente. Exponha o minimo necessario.

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
export type DomainEvents = {
  'auth:login': { user: AuthenticatedUser };
  'auth:logout': void;
  'billing:plan-upgraded': { planId: string; userId: string };
  'client:selected': { clientId: string };
};
```

### `packages/shared-events` — Event Bus Tipado

```ts
// packages/shared-events/src/event-bus.ts
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

### Regras

- **R7.1** — `packages/ui` contem apenas componentes visuais. Zero logica de negocio.
- **R7.2** — `packages/shared-types` contem apenas `type` e `interface`. Nenhuma implementacao.
- **R7.3** — Novos eventos devem ser adicionados ao type `DomainEvents` antes de serem emitidos.
- **R7.4** — `packages/utils` contem apenas funcoes puras e genericas (formatDate, debounce, etc).
- **R7.5** — Nenhum package pode importar de `apps/`. A dependencia e unidirecional: apps → packages.
- **R7.6** — Todo package exporta via barrel file (`index.ts`). Imports internos de package nao sao permitidos (ex: `@company/ui/src/Button` e proibido).

## 8. Estado e Comunicacao entre Dominios

### Hierarquia de preferencia

1. **Props** — quando o shell passa dados diretamente ao remote
2. **Event Bus** — quando dominios precisam reagir a eventos de outros
3. **Shared Store (global)** — apenas para estado verdadeiramente global (usuario logado, tema)
4. **URL/Query Params** — para estado navegavel

### Regras

- **R8.1** — Cada dominio gerencia seu proprio estado interno. Nao existe store global por dominio.
- **R8.2** — O unico estado global permitido e: usuario autenticado, tema e idioma. Tudo mais e local.
- **R8.3** — Comunicacao entre dominios usa o Event Bus tipado. Nunca chamadas diretas.
- **R8.4** — Eventos sao fire-and-forget. Se precisa de resposta, use request/response via props ou callbacks.
- **R8.5** — Nao armazene estado derivavel. Se pode ser calculado a partir de outro estado, calcule.

## 9. Roteamento

```
shell       → / (redirect), /dashboard
auth        → /auth/login, /auth/signup, /auth/forgot-password
client      → /clients, /clients/:id, /clients/:id/edit
billing     → /billing, /billing/invoices, /billing/plans
```

- **R9.1** — Cada dominio usa `basePath` no `next.config.js` correspondente ao seu prefixo de rota.
- **R9.2** — O shell configura `rewrites` para rotear para o dominio correto.
- **R9.3** — Links entre dominios usam `<a href>` ou `window.location`, nao `next/link` (apps diferentes).
- **R9.4** — Links internos do dominio usam `next/link` normalmente.
- **R9.5** — Rotas protegidas sao guardadas pelo shell (auth guard), nao pelo dominio individual.

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
- **R10.6** — Testes E2E cobrem apenas fluxos criticos cross-domain (login → criar fatura → pagar).
- **R10.7** — Nao teste getters, setters ou codigo trivial. Teste logica e comportamento.

## 11. CI/CD e Deploy

Pipeline (`.github/workflows/ci.yml`): detecta apps/packages alterados → lint e type-check nos afetados → testes unitarios nos afetados → build dos afetados → E2E (se fluxos criticos foram tocados) → deploy independente por dominio.

- **R11.1** — Cada dominio tem deploy independente. Mudanca no billing nao faz redeploy do auth.
- **R11.2** — Turborepo `--filter` detecta o que mudou. So roda CI nos apps afetados.
- **R11.3** — Mudancas em `packages/` disparam CI de todos os apps que usam o package.
- **R11.4** — Mudancas no `shell` disparam E2E completo (e o orquestrador).
- **R11.5** — Cada app tem sua propria URL de deploy (auth.company.com, billing.company.com, etc).
- **R11.6** — Variaveis de ambiente de URLs dos remotes sao atualizadas no shell a cada deploy.
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
            { group: ['apps/shell/**'], message: 'Remote nao importa do shell' },
            { group: ['apps/auth/**', 'apps/client/**', 'apps/billing/**'],
              message: 'Remotes nao importam uns dos outros' },
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
[ ] 2.  Configurar next.config.js com basePath e NextFederationPlugin
[ ] 3.  Configurar tsconfig.json com path aliases das camadas
[ ] 4.  Criar di/container.ts com as dependencias do dominio
[ ] 5.  Definir entidades e interfaces de repository em domain/
[ ] 6.  Implementar pelo menos 1 use case em application/
[ ] 7.  Implementar repositories em infra/
[ ] 8.  Criar componentes expostos em presentation/exposed/
[ ] 9.  Registrar remote no shell (next.config.js + env vars)
[ ] 10. Adicionar rewrite no shell para o basePath do novo dominio
[ ] 11. Adicionar eventos do dominio em packages/shared-types DomainEvents
[ ] 12. Configurar pipeline de deploy separado
[ ] 13. Adicionar testes unitarios para domain/ e application/
[ ] 14. Documentar rotas e componentes expostos no README do dominio
[ ] 15. Validar que o app roda isolado (pnpm dev --filter=[domain])
```

## 14. Anti-patterns

| Anti-pattern | Por que e ruim | O que fazer |
|---|---|---|
| Importar de remote para remote | Acoplamento direto, quebra independencia | Usar Event Bus ou props via shell |
| Logica de negocio em componente | Nao testavel, nao reutilizavel | Mover para domain/entities ou application/use-cases |
| Logica de negocio em packages/ | Package vira dominio oculto | Manter logica dentro de apps/[domain]/domain/ |
| Store global por dominio | Estado espalhado, dificil de debugar | Estado global so para user, tema, idioma |
| Expor componentes internos via MF | Over-sharing, acoplamento | So expor o que esta em exposed/ |
| `any` em DTOs | Perde type safety entre camadas | Tipar DTOs como interfaces explicitas |
| Use case com dependencia de React | Application fica acoplada ao framework | Use case so usa TypeScript puro |
| Repository sem interface | Nao pode trocar implementacao, nao pode testar | Sempre definir interface em domain/ |
| Componente consumindo API direto | Pula todas as camadas, nao testavel | Componente → hook → use case → repository |
| Shared state para dados de um dominio | Acoplamento desnecessario | Cada dominio gerencia seu estado |
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
