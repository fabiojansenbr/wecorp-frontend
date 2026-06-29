# 05 — Roteamento e Navegação

> Como o frontend weCorp organiza rotas com o **App Router (Next.js 16)**: route groups, layouts aninhados, `params`/`searchParams` como `Promise`, configuração de rotas derivada da hierarquia de módulos/ACL, menu lateral por grupo, breadcrumbs e estados de `loading`/`error`/`not-found`.
> Regras não-negociáveis em [`../CLAUDE.md`](../CLAUDE.md) §4. Arquitetura geral em [`./01-arquitetura.md`](./01-arquitetura.md).

---

## 1. App Router em uma frase

O roteamento é **baseado no sistema de arquivos**: cada `page.tsx` dentro de `src/app/` vira uma URL; `layout.tsx` define a moldura (UI persistente) compartilhada pelas rotas abaixo dele. No Next.js 16 os Server Components são padrão e os parâmetros de rota chegam como `Promise` (ver §4).

Os três arquivos especiais por segmento:

| Arquivo | Papel |
|---------|-------|
| `page.tsx` | A rota em si (gera a URL). |
| `layout.tsx` | Moldura persistente (não remonta ao navegar entre filhos). |
| `loading.tsx` / `error.tsx` / `not-found.tsx` | Estados automáticos do segmento (ver §8). |

---

## 2. Route groups: organização sem afetar a URL

Pastas no formato `(nome)/` são **route groups**: organizam o código **sem** aparecer na URL. O weCorp usa três grupos de topo, por contexto de autenticação:

| Route group | Autenticação | Conteúdo | Layout |
|-------------|--------------|----------|--------|
| `(auth)` | **Não** autenticado | Login, recuperar senha, aceite de EULA. | Layout enxuto (sem sidebar). |
| `(modules)` | **Autenticado** (painel) | Um group por domínio (`(analises)`, `(vistorias)`, `(seguros)`…). | Layout autenticado (Sidebar/Topbar/Breadcrumbs + TenantContext). |
| `(public)` | **Não** autenticado (portal F20) | Portal público do proponente: proposta de fiança, análise, validação, upload. | Layout público (branding do tenant, sem menu). |

```text
src/app/
├── layout.tsx                      # ROOT: fonts, Providers, globals.css, <html>/<body>
├── globals.css                     # Tailwind 4 + tokens CSS (white-label)
├── not-found.tsx                   # 404 global
├── (auth)/
│   └── (web)/(pages)/
│       ├── sign-in/page.tsx        # /sign-in
│       └── recover-account/page.tsx# /recover-account
├── (public)/                       # PORTAL PÚBLICO (F20, sem login)
│   ├── layout.tsx                  # branding por host; sem Sidebar
│   └── (web)/(pages)/
│       └── proposta-fianca/[token]/page.tsx   # /proposta-fianca/:token
└── (modules)/                      # PAINEL AUTENTICADO
    ├── layout.tsx                  # Sidebar + Topbar + Breadcrumbs + TenantProvider
    └── (analises)/
        ├── _business/              # actions/hooks/schemas/helpers (ver doc 01)
        └── (web)/(pages)/
            ├── analises/page.tsx               # /analises
            ├── analises/loading.tsx            # fallback de /analises
            ├── analises/error.tsx              # erro de /analises
            └── analises/[id]/page.tsx          # /analises/:id
```

> Como `(auth)`, `(modules)`, `(public)`, `(web)` e `(pages)` são todos route groups, **nenhum** deles entra na URL. A URL é formada apenas pelos segmentos "normais" (`analises`, `[id]`…).

---

## 3. Tabela arquivo → URL

| Arquivo | URL | Acesso |
|---------|-----|--------|
| `(auth)/(web)/(pages)/sign-in/page.tsx` | `/sign-in` | Público |
| `(auth)/(web)/(pages)/recover-account/page.tsx` | `/recover-account` | Público |
| `(modules)/(dashboard)/(web)/(pages)/dashboard/page.tsx` | `/dashboard` | Autenticado |
| `(modules)/(analises)/(web)/(pages)/analises/page.tsx` | `/analises` | Autenticado + ACL |
| `(modules)/(analises)/(web)/(pages)/analises/[id]/page.tsx` | `/analises/:id` | Autenticado + ACL |
| `(modules)/(vistorias)/(web)/(pages)/vistorias/page.tsx` | `/vistorias` | Autenticado + ACL |
| `(modules)/(config)/(web)/(pages)/config/grupos/page.tsx` | `/config/grupos` | Autenticado + ACL (grupo 1) |
| `(modules)/(config)/(web)/(pages)/config/grupos/[id]/permissoes/page.tsx` | `/config/grupos/:id/permissoes` | Autenticado + ACL (grupo 1) |
| `(public)/(web)/(pages)/proposta-fianca/[token]/page.tsx` | `/proposta-fianca/:token` | Público (token) |

Convenções de segmento dinâmico: `[id]` (um parâmetro), `[...path]` (catch-all), `[[...path]]` (catch-all opcional).

### 3.1 `basePath /app` e domínio do cliente

A aplicação roda sob **`/app`** (`basePath: '/app'` no `next.config.ts`) — assim, no domínio próprio do cliente (`www.cliente1.com.br/app`) a raiz fica livre para o site institucional. As URLs acima são **relativas ao basePath** (ex.: `/analises` resolve para `/app/analises`). O mesmo bundle serve qualquer host; a marca é resolvida por host (data-driven). Fallback: subdomínio da plataforma (`cliente1.wecorp.com.br/app`). Ver [`./06-multitenancy-whitelabel.md`](./06-multitenancy-whitelabel.md) §7 e [`./14-deploy-ambiente.md`](./14-deploy-ambiente.md).

### 3.2 Branding por host nos layouts (login pré-auth, sem FOUC)

O **root layout** e os layouts `(auth)`/`(public)` resolvem o branding **por host no servidor** e injetam as variáveis CSS no `<html>` + `generateMetadata` (title/description/OG/**favicon**) por host. Resultado: a **tela de login já aparece tematizada** com a marca do cliente, sem flash. Detalhe em [`./06-multitenancy-whitelabel.md`](./06-multitenancy-whitelabel.md) §5.

---

## 4. `params` e `searchParams` como `Promise`

No Next.js 16, `params` e `searchParams` de páginas/layouts são **`Promise`** e exigem `await`. Isso vale para Server Components (o padrão).

```tsx
// (modules)/(analises)/(web)/(pages)/analises/[id]/page.tsx — Server Component
export default async function AnaliseDetailPage(props: {
  params: Promise<{ id: string }>;
  searchParams: Promise<Record<string, string | string[] | undefined>>;
}) {
  const { id } = await props.params;            // segmento dinâmico
  const filters = await props.searchParams;     // querystring (?tab=...&page=...)
  // ... fetch inicial via api (@/infra) e composição
}
```

- Em **Client Components**, leia o mesmo `Promise` com o hook `use()` do React 19 (`const { id } = use(props.params)`), ou prefira deixar o fetch no Server Component pai.
- Filtros de listagem que vêm da URL são lidos de `searchParams` e sincronizados pelo hook `useUrlSyncedFilters` (em vez de `nuqs`). Ver [`./11-tabelas-listagens.md`](./11-tabelas-listagens.md).

---

## 5. Layouts aninhados

Os layouts compõem em cascata: o **root** envolve tudo; cada group de autenticação aplica sua moldura. Layouts **não remontam** ao navegar entre páginas-filhas — ideal para Sidebar/Topbar.

### 5.1 Root layout (`app/layout.tsx`)

Único `<html>`/`<body>` da aplicação. Carrega fontes, `globals.css` e os `Providers` (Theme, QueryClient, Toaster). **Server Component**; os `Providers` é que são `"use client"`.

```tsx
// app/layout.tsx (Server Component)
import { Providers } from '@/providers';
import './globals.css';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="pt-BR" suppressHydrationWarning>
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

### 5.2 Layout autenticado (`(modules)/layout.tsx`)

Moldura do painel: **Sidebar** (menu por grupo), **Topbar** (busca, sino de notificações, troca de tema, avatar), **Breadcrumbs** e o **TenantProvider** (white-label). Resolve a sessão no servidor e injeta o contexto de tenant/permissões uma única vez.

```tsx
// (modules)/layout.tsx (Server Component)
import { getServerSession } from '@/shared';
import { Sidebar, Topbar, Breadcrumbs } from '@/components';
import { TenantProvider } from '@/providers';
import { getSidebarForUser } from '@/configs/routes';

export default async function ModulesLayout({ children }: { children: React.ReactNode }) {
  const session = await getServerSession();              // cookies httpOnly, no servidor
  const menu = getSidebarForUser(session);              // filtrado por ACL (ver §6)

  return (
    <TenantProvider tenant={session.tenant} user={session.user}>
      <div className="flex min-h-screen">
        <Sidebar groups={menu} />
        <div className="flex flex-1 flex-col">
          <Topbar user={session.user} />
          <Breadcrumbs />
          <main className="flex-1 p-4">{children}</main>
        </div>
      </div>
    </TenantProvider>
  );
}
```

> A sessão é resolvida **só no servidor** (cookies httpOnly + `getServerSession()` com `cache` do React) e passada ao client por contexto **uma vez**. Nunca espelhar JWT no Zustand. Ver [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md).

### 5.3 Layout público (`(public)/layout.tsx`)

Moldura mínima do portal do proponente: aplica o **branding por host** (logo/cores via variáveis CSS), **sem** Sidebar/Topbar e sem sessão autenticada. Detalhes do portal em [`./17-paridade-portal-publico.md`](./17-paridade-portal-publico.md).

---

## 6. Configuração de rotas derivada de módulos/ACL

Os metadados de rota e os itens do menu lateral vivem em `src/configs/routes/` e são **derivados da hierarquia de módulos do backend**, não escritos à mão tela a tela. A hierarquia (mesma do legado, ver `frontend.md` Parte 3):

```text
modules → modulecontrollers → modulecontrolleractions
                                      ▲
                grupos (1–10) ──< permissiongroups (nivel 1–3) >──┘
```

| Nível | Entidade | Papel na navegação |
|-------|----------|--------------------|
| `modules` | Módulo (ex.: Seguros, Vistorias, Financeiro) | Seção do menu / ícone. |
| `modulecontrollers` | Controlador dentro do módulo | Item ou subitem do menu (rota base). |
| `modulecontrolleractions` | Ação (listar, criar, editar…) | Mapeia para `actionId` usado no `PermissionGate`. |
| `permissiongroups` | Pivot grupo×ação com `nivel` (1–3) | Define **se** e **com que nível** o grupo vê a ação. |

Cada item de rota declara o requisito de permissão (módulo/ação/nível). O menu final é o **conjunto de itens cujo requisito o usuário satisfaz** — itens sem permissão simplesmente não são renderizados.

```typescript
// src/configs/routes/index.ts (esboço)
import { ModulesPermissionEnum, ActionsPermissionEnum } from '@/shared';

export interface RouteItem {
  label: string;                    // PT-BR (rótulo de menu/breadcrumb)
  href: string;                     // URL
  icon?: string;                    // ícone lucide
  requirement?: {                   // requisito de ACL (derivado de modulecontrolleractions)
    module: ModulesPermissionEnum;
    action: ActionsPermissionEnum;
    nivel?: 1 | 2 | 3;
  };
  children?: RouteItem[];
}

export const routes: RouteItem[] = [
  { label: 'Dashboard', href: '/dashboard', icon: 'layout-dashboard' },
  {
    label: 'Análise Cadastral',
    href: '/analises',
    icon: 'briefcase',
    requirement: { module: ModulesPermissionEnum.ASSESSORIA, action: ActionsPermissionEnum.LIST },
  },
  // ...derivados da hierarquia de módulos
];
```

```typescript
// filtragem por permissão do usuário (servidor)
import { hasAccess } from '@/shared';

export function getSidebarForUser(session: Session): RouteItem[] {
  const filter = (items: RouteItem[]): RouteItem[] =>
    items
      .filter((i) => !i.requirement || hasAccess(session, i.requirement))
      .map((i) => ({ ...i, children: i.children ? filter(i.children) : undefined }));
  return filter(routes);
}
```

> `hasAccess(session, requirement)` é a **mesma** função pura usada pelos `PermissionGate` — com cobertura de testes obrigatória. Strings de módulo/ação **MUST NOT** ser hardcoded; sempre via enums de `@/shared`. Ver [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md).

---

## 7. Menu lateral por grupo e breadcrumbs

### 7.1 Sidebar e os 10 grupos

O menu lateral é o mesmo componente para todos, mas seu **conteúdo** muda conforme o `grupo_id` do usuário (10 grupos), porque a ACL filtra os itens (§6). Os módulos de topo seguem a tabela `Module` do legado:

| `id` | Módulo | Ícone (legado → lucide) |
|------|--------|-------------------------|
| 1 | Assessoria (Análise Cadastral) | `briefcase` |
| 2 | Admin | `settings` |
| 3 | Relatórios | `bar-chart-3` |
| 5 | Seguros | `shield` |
| 6 | Vistorias | `building-2` |
| 9 | Downloads | `lightbulb` |
| 11 | Financeiro | `credit-card` |
| 12 | Sign (variant) | `pen-line` |
| 13 | Assinatura Eletrônica | `signature` |
| 14 | Marketplace | `plug` |

Exemplos do efeito do grupo: grupos **6–10** (imobiliária cliente) veem um menu reduzido (dashboard "Indexadmin", análises, vistorias, seguros do próprio escopo); o grupo **5** (integrador) vê o dashboard de API/WBS; o grupo **1** (SuperAdmin) vê inclusive `Admin`/`config` (ACL, grupos, lookups). O isolamento real de dados é garantido pelo backend — o menu só reflete o que o usuário pode acessar.

### 7.2 Breadcrumbs

Derivados do caminho de rota cruzado com os `label` da config de rotas (§6). Renderizados no layout autenticado, abaixo da Topbar.

```text
Início / Análise Cadastral / Ficha #1234
   /         /analises          /analises/1234
```

O último segmento usa o rótulo do recurso (ex.: nome/identificador da ficha) quando disponível; os intermediários usam os `label` da config.

---

## 8. Estados: loading, error e not-found

O App Router gera fronteiras automáticas por segmento. Cada um é opcional e fica ao lado do `page.tsx`.

| Arquivo | Quando dispara | Tipo |
|---------|----------------|------|
| `loading.tsx` | Suspense boundary enquanto o segmento (ou seu fetch) carrega. | Server ou Client |
| `error.tsx` | Exceção não tratada na renderização do segmento. **Deve** ser `"use client"` e recebe `{ error, reset }`. | Client |
| `not-found.tsx` | `notFound()` chamado, ou rota inexistente (404). | Server |

```tsx
// (modules)/(analises)/(web)/(pages)/analises/loading.tsx
import { TableFallback } from '@/components';
import { columns } from '../_components/columns';

export default function Loading() {
  return <TableFallback columns={columns} />;   // skeleton da tabela
}
```

```tsx
// (modules)/(analises)/(web)/(pages)/analises/error.tsx
'use client';

export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div className="p-6 text-center">
      <p className="text-muted-foreground">Não foi possível carregar as análises.</p>
      <button onClick={reset} className="mt-2 underline">Tentar novamente</button>
    </div>
  );
}
```

```tsx
// (modules)/(analises)/(web)/(pages)/analises/[id]/page.tsx (trecho)
import { notFound } from 'next/navigation';

const ficha = await getFicha(id);
if (!ficha) notFound();   // → renderiza o not-found.tsx mais próximo
```

> Para listagens, prefira `<Suspense fallback={<TableFallback/>}>` **dentro** da página (granularidade fina) em vez de depender só do `loading.tsx` do segmento — assim o cabeçalho/`<Card>` aparece imediatamente e só a tabela mostra skeleton. Ver [`./11-tabelas-listagens.md`](./11-tabelas-listagens.md).

---

## 9. Ver também

- [01 — Arquitetura](./01-arquitetura.md) · [02 — Stack](./02-stack-tecnologico.md)
- [06 — Multi-tenancy e white-label](./06-multitenancy-whitelabel.md) · [07 — Auth, sessão e permissões](./07-auth-sessao-permissoes.md)
- [08 — Consumo de API e dados](./08-consumo-api-dados.md) · [11 — Tabelas e listagens](./11-tabelas-listagens.md)
- [09 — Módulos e telas](./09-modulos-telas.md) · [14 — Configs (routes/env)](./14-deploy-ambiente.md)
</content>
</invoke>
