# Template de Especificação Técnica (Frontend — Next.js)

## Resumo Executivo

[Forneça uma breve visão técnica da abordagem de solução. Resuma as decisões de UI/UX, estratégia de renderização (SSR/SSG/ISR) e a integração com APIs de backend em 1-2 parágrafos.]

## Arquitetura Frontend

### Visão Geral dos Componentes

[Breve descrição dos componentes principais e suas responsabilidades:

- Páginas (App Router) e layouts envolvidos
- Componentes React novos ou modificados — **liste cada um**
- Relacionamentos e hierarquia entre componentes
- Fluxo de dados (props, contextos, stores)]

### Estratégia de Renderização

[Defina a estratégia de renderização para cada rota/página:

- **Server Components (RSC)**: componentes que buscam dados no servidor
- **SSR (`dynamic = 'force-dynamic'`)**: páginas que precisam de dados frescos a cada requisição
- **SSG / ISR**: páginas que podem ser pré-renderizadas ou revalidadas
- **Client Components (`'use client'`)**: componentes que precisam de interatividade no browser]

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

type NomeAction = {
  execute(input: InputType): Promise<OutputType>;
};
```

]

### Modelos de Dados

[Defina estruturas de dados essenciais:

- Entidades de domínio exibidas na UI
- Tipos de estado (local, global, server state)
- Schemas de validação (Zod, etc.)]

### Integração com Backend (SSR)

[A integração com o backend é feita via **Server-Side Rendering (SSR) do Next.js** — Server Components, Server Actions e Route Handlers atuam como camada de integração:

- **Server Components**: fazem `fetch` direto às APIs do backend durante a renderização no servidor
- **Server Actions**: processam mutações (formulários, ações do usuário) no servidor
- **Route Handlers (`app/api/`)**: expõem endpoints auxiliares quando necessário (webhooks, integrações externas)
- Estratégia de autenticação/autorização nos requests ao backend
- Tratamento de erros e estados de fallback]

### Contratos de API (Backend)

[Liste **todas** as APIs do backend que esta feature consome, com seus contratos completos. Esses contratos serão a base para construção de mocks nos testes.

Para cada endpoint, documente:

#### `METHOD /caminho/do/endpoint`

- **Descrição**: o que o endpoint faz
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

- **Testes unitários/integração**: usar os contratos para criar fixtures/mocks nos handlers (ex: MSW — Mock Service Worker, ou mocks diretos no `fetch`)
- **Testes E2E (Playwright)**: definir se o backend será real ou mockado; se mockado, descrever a estratégia (ex: MSW em modo browser, API routes de mock no Next.js, ou servidor de mock externo)
- **Fixtures de dados**: listar os cenários de dados necessários com base nos contratos (caso de sucesso, erros de validação, recurso não encontrado, não autorizado, etc.)
- **Localização dos mocks**: onde os arquivos de mock/fixtures serão armazenados no projeto]

## Pontos de Integração

[Inclua apenas se a funcionalidade requer integrações externas além do backend principal:

- Serviços de terceiros (ex: pagamento, analytics, CDN)
- SDKs ou bibliotecas externas
- Requisitos de autenticação (OAuth, tokens)
- Abordagem de tratamento de erros]

## Abordagem de Testes

### Testes Unitários

[Descreva estratégia de testes unitários:

- Componentes React a testar (renderização, props, estados)
- Hooks customizados a testar
- Funções utilitárias e helpers
- **Mocks de API**: usar as fixtures baseadas nos contratos da seção "Contratos de API" para mockar respostas de sucesso e erro
- Cenários de teste críticos]

### Testes de Integração

[Se necessário, descreva testes de integração:

- Fluxos de componentes compostos a testar juntos
- Integração de formulários com validação
- **Mocks de API**: usar os contratos documentados para interceptar chamadas ao backend e retornar dados controlados (sucesso, erro de validação, timeout, etc.)
- Requisitos de dados de teste]

### Testes E2E (Playwright)

[Descreva os testes end-to-end usando **Playwright** contra a aplicação Next.js rodando com SSR:

- **Escopo**: testar fluxos completos do usuário no browser, validando a integração frontend ↔ backend via SSR
- **Cenários críticos**: listar os fluxos de usuário que devem ser cobertos (ex: login, navegação, envio de formulário, exibição de dados do servidor)
- **Validações**: verificar conteúdo renderizado pelo servidor (SEO, dados dinâmicos), interações no client, navegação entre rotas
- **Backend real ou mockado**: definir a abordagem conforme descrito na seção "Estratégia de Mocks" — se mockado, garantir que os mocks respeitam os contratos de API documentados
- **Cenários de erro**: testar comportamento do frontend quando o backend retorna erros (usar os cenários de erro dos contratos)
- **Ambiente**: Next.js rodando em modo SSR (`next start` ou `next dev`)
- **Estrutura dos testes**: descrever organização dos arquivos de teste e page objects se aplicável]

## Sequenciamento de Desenvolvimento

### Ordem de Construção

[Defina sequência de implementação:

1. Tipos e interfaces base (por que primeiro)
2. Componentes de UI (design system / componentes base)
3. Server Components e integração com backend via SSR
4. Interatividade no client (Client Components)
5. Testes unitários e de integração
6. Testes E2E com Playwright]

### Dependências Técnicas

[Liste quaisquer dependências bloqueantes:

- Endpoints de backend que precisam estar disponíveis
- Bibliotecas ou pacotes necessários
- Variáveis de ambiente requeridas]

## Performance e UX

[Defina considerações de performance e experiência:

- Core Web Vitals alvos (LCP, FID, CLS)
- Estratégias de carregamento (Suspense, loading states, skeleton)
- Otimização de imagens (next/image)
- Code splitting e lazy loading
- Acessibilidade (a11y) — requisitos mínimos]

## Considerações Técnicas

### Decisões Principais

[Documente decisões técnicas importantes:

- Escolha de abordagem e justificativa
- Trade-offs considerados (ex: SSR vs SSG para determinada rota)
- Alternativas rejeitadas e por quê]

### Riscos Conhecidos

[Identifique riscos técnicos:

- Desafios potenciais (performance, compatibilidade, complexidade de estado)
- Abordagens de mitigação
- Áreas precisando pesquisa ou prova de conceito]

### Conformidade com Padrões

[Pesquise as rules na pasta @.claude/rules que se encaixam e se apliquem nesta techspec e liste-as abaixo:]

### Arquivos Relevantes e Dependentes

[Liste aqui arquivos relevantes e dependentes — componentes, páginas, hooks, utils, tipos, configs, etc.]