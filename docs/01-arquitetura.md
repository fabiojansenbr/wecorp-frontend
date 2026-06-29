# 01 — Arquitetura do Frontend weCorp

> Como o frontend é organizado: **camadas**, **App Router**, **route groups**, **estrutura de pastas** e **Server vs Client**.
> Regras não-negociáveis em [`../CLAUDE.md`](../CLAUDE.md) §4. Stack em [`./02-stack-tecnologico.md`](./02-stack-tecnologico.md).

---

## 1. Princípio central: arquitetura em camadas

O fluxo de dados é **unidirecional** e cada camada tem uma responsabilidade única:

```text
(web) page.tsx / componente            ← markup, composição, Suspense
        ↓
useXxx (hook)                          ← estado, submit, loading, toast, revalidate
        ↓
action (função async)                  ← chama api; um props tipado; SEM try/catch
        ↓
api (@/infra)                          ← HTTP; token resolvido no SERVIDOR (cookies)
        ↓
fetcher                                ← fetch, headers, parse, normalização de erro
        ↓
Backend REST (NestJS)
```

**Regras (MUST):**

- **UI** compõe markup; **não** contém `useState`/`useEffect`/`useForm` diretos nem `fetch`.
- **Hooks (`useXxx`)** orquestram estado, submit, loading e tratamento de erro (`toast` + `resolveErrorMessage`).
- **Actions** apenas chamam `api`; recebem **um objeto `props` tipado**; **sem `try/catch`** (o hook trata).
- **Nunca** `fetch` direto — sempre `import { api } from "@/infra"`.
- Erro tratado por `error.code`/`status`, **nunca** por texto da mensagem do backend.

> Por que separar: testabilidade (lógica nos hooks/actions, não na UI), reuso (a mesma action serve vários hooks), e segurança (token só no servidor, na camada `api`).

## 2. App Router (Next.js 16)

- **Server Components por padrão.** `"use client"` só quando há interatividade (estado, handlers, dialogs, browser API). Ver §5.
- **`params` e `searchParams` são `Promise`** — sempre `await`:
  ```tsx
  export default async function Page(props: { searchParams: Promise<Record<string, string>> }) {
    const filters = await props.searchParams;
    // ...
  }
  ```
- **Turbopack** em dev. Alias `@/*` → `src/*` (nunca `../../..`).
- **Idioma:** código em **inglês**; UI e mensagens de validação em **PT-BR**.

## 3. Organização modular (route groups)

A aplicação é organizada **por domínio** em route groups. Route groups `(nome)/` **não aparecem na URL** — servem para organização.

```text
src/app/
├── layout.tsx                      # root: fonts, Providers, globals.css
├── globals.css                     # Tailwind 4 + tokens CSS (white-label)
├── (auth)/                         # login, recuperar senha (não autenticado)
├── (public)/                       # PORTAL PÚBLICO do proponente (F20, sem login)
└── (modules)/                      # painel autenticado, um group por domínio
    └── (analises)/
        ├── _business/
        │   ├── actions/            # 1 pasta por action (createX, getX...)
        │   ├── hooks/              # 1 pasta por hook (useFormX, useTableX...)
        │   ├── schemas/            # Zod (compartilha o domínio do backend)
        │   ├── helpers/            # mappers, constants, máscaras
        │   └── index.ts            # barrel público do módulo
        └── (web)/
            ├── (pages)/            # rotas App Router → URLs
            └── _components/        # componentes da tela (Form, Dialog, Table...)
```

Exemplos de URL:

| Arquivo | URL |
|---------|-----|
| `(modules)/(analises)/(web)/(pages)/analises/page.tsx` | `/analises` |
| `(auth)/(web)/(pages)/sign-in/page.tsx` | `/sign-in` |
| `(public)/(web)/(pages)/proposta-fianca/[token]/page.tsx` | `/proposta-fianca/:token` |

### Estrutura de um artefato

Toda pasta de componente, hook, action ou schema segue:

```text
NomeDoArtefato/
├── index.ts(x)       # implementação
└── interfaces.ts     # Props, Response, types
```

## 4. Estrutura de pastas (raiz `src/`)

```text
src/
├── app/                  # rotas (App Router) — ver §3
├── components/           # design system (shadcn + custom) + barrel
├── configs/              # routes (sidebar/menu), env (Zod) — ver doc 14
├── hooks/                # hooks compartilhados (ex.: useUrlSyncedFilters)
├── infra/                # fetcher + cliente api (HTTP)
├── lib/                  # cn(), utilitários
├── providers/            # Theme, QueryClient, (Socket)
├── shared/               # types, enums, utils, jwt, cookies, errors, toast, revalidate
├── stores/               # Zustand (UI only)
└── proxy.ts              # middleware Next 16 (auth, refresh, white-label)
```

> A fronteira de import é o **barrel público** de cada área: `@/shared`, `@/components`, `@/infra`, e o `_business/index.ts` de cada módulo. Ver [`./03-convencoes-codigo.md`](./03-convencoes-codigo.md).

## 5. Server vs Client

| Server Component (padrão) | `"use client"` |
|---------------------------|----------------|
| Páginas de listagem com Suspense | Formulários interativos (RHF) |
| Fetch inicial de dados (via `api`) | Dialogs, dropdowns, tabs client-side |
| Permission gate no servidor (`PermissionGateServer`) | Hooks de estado (`useXxx`) |
| Layout estático | Zustand, socket.io, máscaras |

- O fetch inicial acontece no **Server Component** (com `<Suspense>` e fallback). Mutações e interatividade ficam em componentes client orquestrados por hooks.
- Após mutação que altera dados do servidor: `revalidatePath('/rota')` (no hook). Ver [`./08-consumo-api-dados.md`](./08-consumo-api-dados.md).

## 6. BFF e camada de borda

- O Next atua como **borda**: o `proxy.ts` cuida de sessão (refresh, redirects) e white-label (detecção de marca por host). Ver [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md) e [`./06-multitenancy-whitelabel.md`](./06-multitenancy-whitelabel.md).
- A camada `api` (`@/infra`) resolve o token **no servidor** (cookies via `next/headers`) e fala com a API REST do backend. O browser **não lê** o token de sessão.
- Quando necessário agregar/adaptar contrato, faz-se em Route Handlers/Server Actions validando com Zod — mas a regra de negócio crítica permanece no backend.

## 7. Exemplo end-to-end (criar recurso)

```typescript
// _business/schemas/resource/index.ts
export const resourceSchema = z.object({ nome: z.string().min(1, 'Campo obrigatório') });
export type ResourceSchemaProps = z.infer<typeof resourceSchema>;

// _business/actions/create-resource/index.ts
import { api } from '@/infra';
export async function createResource(props: ResourceSchemaProps) {
  return api.post('/resources', props);          // sem try/catch
}

// _business/hooks/use-form-create-resource/index.ts ('use client')
export function useFormCreateResource() {
  const form = useForm<ResourceSchemaProps>({ resolver: zodResolver(resourceSchema) });
  async function onSubmit(data: ResourceSchemaProps) {
    try {
      await createResource(data);
      revalidatePath('/resources');
      toast.success('Cadastro realizado com sucesso');
    } catch (e) { toast.error(resolveErrorMessage(e)); }
  }
  return { form, onSubmit: form.handleSubmit(onSubmit) };
}

// (web)/_components/DialogCreateResource ('use client') → usa o hook, compõe markup
// (web)/(pages)/resources/page.tsx (Server) → fetch inicial + Suspense + PermissionGate
```

## 8. Ver também

- [02 — Stack](./02-stack-tecnologico.md) · [03 — Convenções](./03-convencoes-codigo.md) · [04 — Design system](./04-design-system-ui.md)
- [07 — Auth/permissões](./07-auth-sessao-permissoes.md) · [08 — Consumo de API](./08-consumo-api-dados.md) · [12 — Estado](./12-estado-dados.md)
