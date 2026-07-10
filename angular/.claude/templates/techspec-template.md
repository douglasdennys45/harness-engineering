# Template de Especificação Técnica (Frontend — Angular com SSR)

## Resumo Executivo

[Forneça uma breve visão técnica da abordagem de solução. Resuma as decisões de UI/UX, o(s) domínio(s)/microfrontend(s) afetado(s), a estratégia de renderização (render modes) e a integração com APIs de backend em 1-2 parágrafos.]

## Arquitetura Frontend

### Domínio e Composição

[Defina onde a feature vive no monorepo:

- Domínio(s) afetado(s) (`apps/[domain]`) e libs compartilhadas tocadas (`libs/*`)
- Se cria domínio novo: seguir o Checklist de Novo Domínio da skill angular-clean-architecture
- Artefatos expostos via Native Federation (rotas em `exposed/routes.ts`, componentes cross-domain) e onde são consumidos no shell
- Comunicação com outros domínios (inputs via shell, eventos no Event Bus tipado)]

### Visão Geral dos Componentes

[Breve descrição dos componentes principais e suas responsabilidades:

- Páginas (componentes de rota) e layouts envolvidos
- Componentes novos ou modificados — **liste cada um** (e o que vem do libs/ui vs Material vs custom)
- Relacionamentos e hierarquia entre componentes
- Fluxo de dados (inputs, signals stores, use cases)]

### Estratégia de Renderização (Render Modes)

[Defina o render mode de cada rota em `app.routes.server.ts`, com justificativa:

- **`RenderMode.Server` (SSR)**: rotas que precisam de dados frescos a cada requisição / SEO dinâmico
- **`RenderMode.Prerender` (SSG)**: rotas com conteúdo estável pré-renderizável em build time
- **`RenderMode.Client` (CSR)**: telas 100% interativas sem valor de SEO
- Pontos de **incremental hydration** (`@defer (hydrate on ...)`) e conteúdo diferido]

## Design de Implementação

### Interfaces e Tipos

[Defina os tipos TypeScript principais (≤20 linhas por exemplo):

```typescript
// Exemplo de tipos
interface NomeEntidade {
  id: string;
  campo: string;
  // ...
}

interface NomeRepository {
  execute(input: InputType): Promise<OutputType>;
}
```

]

### Modelos de Dados

[Defina estruturas de dados essenciais:

- Entidades de domínio e value objects (camada domain)
- DTOs dos use cases e mappers (camada application)
- Tipos de estado (signals store local, estado global mínimo, server state)
- Schemas de validação (Zod ou validators tipados)]

### Integração com Backend (SSR + BFF)

[A integração com o backend segue a regra da skill angular-clean-architecture — **o browser nunca chama o backend interno diretamente**:

- **Leituras iniciais**: `HttpClient` nos repositories, executado durante o SSR — resultado transferido ao client via hydration transfer cache
- **Mutações e leituras pós-hydration**: endpoints **BFF** no `server.ts` do domínio (listar cada endpoint BFF novo)
- **Webhooks/streaming**: rotas Express no `server.ts` quando necessário
- Estratégia de autenticação/autorização nos requests ao backend (cookies, tokens server-side)
- Tratamento de erros e estados de fallback]

### Contratos de API (Backend)

[Liste **todas** as APIs do backend que esta feature consome, com seus contratos completos. Esses contratos serão a base para construção de mocks nos testes.

Para cada endpoint, documente:

#### `METHOD /caminho/do/endpoint`

- **Descrição**: o que o endpoint faz
- **Consumido por**: [SSR (repository) / BFF (server.ts)]
- **Headers obrigatórios**: `Authorization: Bearer <token>`, etc.
- **Request body** (se aplicável):

```json
{
  "campo": "tipo — descrição",
  "outrocampo": "tipo — descrição"
}
```

- **Response (sucesso — 200/201)**:

```json
{
  "id": "string",
  "campo": "tipo — descrição"
}
```

- **Response (erro — 4xx/5xx)**:

```json
{
  "error": "string — código ou mensagem de erro",
  "message": "string — descrição legível"
}
```

- **Cenários de erro relevantes**: listar códigos HTTP e quando ocorrem (ex: 401 token expirado, 404 recurso não encontrado, 422 validação)

> Repita o bloco acima para cada endpoint consumido pela feature.]

### Estratégia de Mocks

[Defina como os contratos acima serão usados para mockar as APIs nos testes:

- **Testes unitários**: stubs dos repositories pelos **InjectionTokens** (contratos), com fixtures baseadas nos contratos de API
- **Testes de integração**: MSW (Mock Service Worker) interceptando as chamadas HTTP dos repositories e do BFF
- **Testes E2E (Playwright)**: definir se o backend será real ou mockado; se mockado, descrever a estratégia (MSW no servidor de teste, rotas de mock no BFF, ou servidor de mock externo)
- **Fixtures de dados**: listar os cenários de dados necessários com base nos contratos (sucesso, erros de validação, recurso não encontrado, não autorizado, etc.)
- **Localização dos mocks**: onde os arquivos de mock/fixtures serão armazenados no projeto]

## Pontos de Integração

[Inclua apenas se a funcionalidade requer integrações externas além do backend principal:

- Serviços de terceiros (ex: pagamento, analytics, CDN)
- SDKs ou bibliotecas externas (avaliar impacto no bundle e compatibilidade com SSR)
- Requisitos de autenticação (OAuth, tokens)
- Abordagem de tratamento de erros]

## Abordagem de Testes

### Testes Unitários

[Descreva estratégia de testes unitários (Vitest):

- Entidades e use cases a testar (regras de negócio — sem mocks no domain, stubs por token no application)
- Componentes a testar com Angular Testing Library (renderização, interações, estados de loading/erro/vazio)
- Pipes, utils e schemas de validação
- **Mocks de API**: fixtures baseadas nos contratos da seção "Contratos de API"
- Cenários de teste críticos]

### Testes de Integração

[Se necessário, descreva testes de integração:

- Fluxos de componentes compostos a testar juntos (formulário com validação + submissão ao BFF)
- Camada infra: repositories contra MSW (sucesso, erro de validação, timeout)
- Requisitos de dados de teste]

### Testes E2E (Playwright)

[Descreva os testes end-to-end usando **Playwright** contra a aplicação Angular rodando com SSR:

- **Escopo**: testar fluxos completos do usuário no browser, validando a integração frontend ↔ backend via SSR/BFF
- **Cenários críticos**: listar os fluxos de usuário que devem ser cobertos (ex: login, navegação, envio de formulário, exibição de dados do servidor)
- **Validações SSR**: conteúdo presente no HTML renderizado pelo servidor (rotas Server/Prerender), interatividade pós-hydration, console sem erros NG05xx, network sem chamadas diretas ao backend interno
- **Backend real ou mockado**: definir a abordagem conforme a seção "Estratégia de Mocks"
- **Cenários de erro**: testar comportamento do frontend quando o backend/BFF retorna erros (usar os cenários de erro dos contratos)
- **Ambiente**: app rodando em modo SSR (`nx serve` em dev; build de produção + servidor Node para validação completa com federação)
- **Estrutura dos testes**: descrever organização dos arquivos de teste e page objects se aplicável]

## Sequenciamento de Desenvolvimento

### Ordem de Construção

[Defina sequência de implementação:

1. Contratos e tipos base — entidades, interfaces de repository + tokens, DTOs/mappers, fixtures (por que primeiro)
2. Camada servidor — repositories (HttpClient), use cases, endpoints BFF
3. Componentes de UI (libs/ui / Material / custom) com estados de loading/erro/vazio
4. Composição de páginas — rotas do domínio, render modes, exposição federada e registro no shell
5. Testes unitários e de integração (acompanham cada etapa)
6. Testes E2E com Playwright]

### Dependências Técnicas

[Liste quaisquer dependências bloqueantes:

- Endpoints de backend que precisam estar disponíveis
- Bibliotecas ou pacotes necessários
- Variáveis de ambiente / entradas no federation.manifest.json requeridas]

## Performance e UX

[Defina considerações de performance e experiência:

- Core Web Vitals alvos (LCP, INP, CLS)
- Estratégias de carregamento (@defer, incremental hydration, skeletons, loading states)
- Otimização de imagens (NgOptimizedImage) e fontes
- Code splitting (lazy routes) e impacto no bundle (budgets)
- Acessibilidade (a11y) — requisitos mínimos]

## Considerações Técnicas

### Decisões Principais

[Documente decisões técnicas importantes:

- Escolha de abordagem e justificativa
- Trade-offs considerados (ex: Server vs Prerender vs Client para determinada rota; expor componente federado vs compor só por rota)
- Alternativas rejeitadas e por quê]

### Riscos Conhecidos

[Identifique riscos técnicos:

- Desafios potenciais (hydration, performance, compatibilidade SSR de libs, complexidade de estado)
- Abordagens de mitigação
- Áreas precisando pesquisa ou prova de conceito]

### Conformidade com Padrões

[Liste as skills do projeto consultadas e como esta techspec se conforma a elas (angular-clean-architecture, angular-best-practices, angular-material, responsive-design, etc.), além das rules em @.claude/rules se existirem. Indique também quais skills devem ser ativadas em cada fase da implementação (UI: frontend-design + angular-material; responsividade: responsive-design; arquitetura: angular-best-practices + angular-clean-architecture; E2E: playwright-generate-test). Desvios exigem justificativa e alternativa conforme:]

### Arquivos Relevantes e Dependentes

[Liste aqui arquivos relevantes e dependentes — componentes, páginas, use cases, repositories, configs de federação/SSR, etc.]
