# 12 — Estado e dados

> Cada tipo de estado tem **um** lugar certo. Dados server → RSC/actions (cache Next + Suspense); dados client com cache → **TanStack Query**; UI efêmera → **Zustand** ou `useState` no hook; sessão/auth → **servidor** (cookies httpOnly + `getServerSession`, **nunca** Zustand); filtros persistentes → **URL** (`useUrlSyncedFilters`).
> **Regra de ouro:** Zustand é **só UI**. JWT/sessão/dados sensíveis **MUST NOT** entrar nele.
>
> Relacionados: [`../CLAUDE.md`](../CLAUDE.md) §4–§6 · [`./01-arquitetura.md`](./01-arquitetura.md) · [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md) · [`./08-consumo-api-dados.md`](./08-consumo-api-dados.md) · [`./11-tabelas-listagens.md`](./11-tabelas-listagens.md)

---

## 1. Matriz de estado

| Tipo de estado | Onde vive | Ferramenta | Exemplos |
|----------------|-----------|------------|----------|
| **Dados server** | Servidor | RSC + actions (`api`); cache Next / Suspense | Listagens, detalhe, fetch inicial de página |
| **Dados client com cache** | Client | **TanStack Query** | Dialog que carrega dados, autocomplete, refetch pós-ação |
| **UI efêmera** | Client | **Zustand** ou `useState` no hook | Sidebar aberta, tab ativa, modal aberto, step de wizard |
| **Sessão / auth** | **Servidor** | Cookies httpOnly + `getServerSession()` | `userId`, `grupoId`, `permissoes`, tenant |
| **Filtros persistentes** | **URL** | `useUrlSyncedFilters` (querystring) | `page`, `perPage`, `search`, `status` |
| **Estado de formulário** | Client | React Hook Form + Zod | Campos, validação, dirty state |

> Cada linha é exclusiva: o mesmo dado **não** deve ser duplicado em duas colunas (ex.: sessão no servidor **e** em Zustand; filtro na URL **e** em `useState`).

---

## 2. Árvore de decisão (qual usar)

```text
O dado é sessão/JWT/permissão?
  └─ SIM → SERVIDOR (cookie httpOnly + getServerSession). NUNCA Zustand. (§5)

É dado vindo do backend?
  ├─ Renderiza na carga inicial da página/listagem?
  │     └─ SIM → Server Component + action + Suspense (cache Next). (doc 08 §4)
  └─ É lido sob demanda no client (dialog, autocomplete, refetch)?
        └─ SIM → TanStack Query. (§4 / doc 08 §5)

Deve sobreviver a reload / ser compartilhável por link?
  └─ SIM → URL (useUrlSyncedFilters). (§6)

É só UI efêmera (aberto/fechado, aba ativa, step)?
  ├─ Local a um componente/hook? → useState no hook.
  └─ Compartilhada entre componentes distantes? → Zustand (store em src/stores/). (§3)
```

---

## 3. Zustand — APENAS UI

Stores em `src/stores/`, um por preocupação de UI. **MUST NOT** guardar JWT, sessão, permissões ou dados de domínio do backend.

```typescript
// src/stores/sidebar.store.ts
import { create } from 'zustand';

interface SidebarState {
  isOpen: boolean;
  toggle: () => void;
  setOpen: (open: boolean) => void;
}

export const useSidebarStore = create<SidebarState>((set) => ({
  isOpen: true,
  toggle: () => set((s) => ({ isOpen: !s.isOpen })),
  setOpen: (isOpen) => set({ isOpen }),
}));
```

| Pode ir no Zustand | NÃO pode ir no Zustand |
|--------------------|------------------------|
| Sidebar aberta/recolhida | JWT / access / refresh token |
| Tab/seção ativa, step de wizard | Sessão (`userId`, `grupoId`, `permissoes`) |
| Estado de UI global (tema temporário, command palette) | Dados de domínio do backend (empresas, análises) |
| Flags efêmeras de layout | Filtros de listagem (vão na URL) |

> Para UI **local** a um único componente/hook, prefira `useState` no hook (`useXxx`) — não crie store para algo que não é compartilhado.

---

## 4. TanStack Query — dados client com cache

Para leituras que acontecem **no client**: dialogs sob demanda, autocomplete, refetch após mutação, cache compartilhado entre componentes. O fetch **inicial** da página fica no Server Component (doc 08 §4). O `QueryClientProvider` é montado em `src/providers/`.

```tsx
// src/providers/query-provider.tsx ('use client')
'use client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useState } from 'react';

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [client] = useState(() => new QueryClient({
    defaultOptions: { queries: { staleTime: 30_000, retry: 1, refetchOnWindowFocus: false } },
  }));
  return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
}
```

```typescript
// hook de leitura client (queryFn usa a MESMA action que o RSC)
'use client';
import { useQuery } from '@tanstack/react-query';
import { getPessoaById } from '@/app/(modules)/(pessoas)/_business';

export function usePessoa(id: string, enabled: boolean) {
  return useQuery({ queryKey: ['pessoas', id], queryFn: () => getPessoaById(id), enabled });
}
```

> Convenção de `queryKey`: `[recurso, ...identificadores]` (ex.: `['empresas', 'lookup', term]`). É a chave de cache **e** de invalidação (§7).

---

## 5. Sessão / auth — sempre no servidor

A sessão é resolvida **uma vez** no servidor por `getServerSession()` (com `cache` do React) e passada ao client por contexto read-only. Detalhe completo em [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md).

```tsx
// componente client lê a sessão por contexto — NUNCA de um store
'use client';
import { useSession } from '@/providers';

export function UserMenu() {
  const session = useSession(); // read-only, hidratado uma vez
  return <span>{session?.name}</span>;
}
```

**Por que não Zustand para sessão:**

- Token em cookie **httpOnly** não é legível por JS — espelhá-lo em Zustand exigiria expô-lo ao client (vetor de XSS).
- O servidor é a fonte única; duplicar gera divergência (sessão "fantasma" após expirar/refresh).
- `proxy.ts` faz refresh na borda; o client não gerencia ciclo de vida de token.

> **MUST NOT:** `useAuthStore`, `useUserStore` com JWT, ou qualquer store espelhando `permissoes`. Permissões vêm de `session.permissoes` e são avaliadas por `hasAccess` ([`./07`](./07-auth-sessao-permissoes.md) §8).

---

## 6. Filtros persistentes na URL

Filtros de listagem vivem na **querystring**, não em estado local — assim a página é compartilhável, sobrevive a reload e o Server Component os lê de `searchParams` (doc 08 §7). O hook `useUrlSyncedFilters` sincroniza um schema Zod com a URL.

```typescript
// src/hooks/use-url-synced-filters/index.ts ('use client') — assinatura
'use client';
export function useUrlSyncedFilters<T extends z.ZodTypeAny>(schema: T): {
  filters: z.infer<T>;
  setFilter: (key: keyof z.infer<T>, value: string) => void; // atualiza querystring (replace)
  reset: () => void;
};
```

```typescript
// uso em uma listagem
const { filters, setFilter } = useUrlSyncedFilters(empresasFilterSchema);
// setFilter('statusEmpresa', '1') → /empresas?statusEmpresa=1 (debounce em inputs textuais)
```

> Detalhe de tabela/paginação/export em [`./11-tabelas-listagens.md`](./11-tabelas-listagens.md). Substitui o `nuqs` citado no spec antigo (ver [`./02`](./02-stack-tecnologico.md) §1).

---

## 7. Cache e invalidação

| Camada | Como invalida após mutação |
|--------|---------------------------|
| **Server (cache Next)** | `revalidatePath('/rota')` ou `revalidateTag('tag')` no hook. |
| **Client (TanStack Query)** | `queryClient.invalidateQueries({ queryKey: ['recurso'] })`. |
| **Ambos** (lista server + dialog client) | Revalidar os dois. |

```typescript
// no hook, após criar/editar
revalidatePath('/empresas');                                  // listagem renderizada por RSC
queryClient.invalidateQueries({ queryKey: ['empresas'] });    // caches client (lookup, dialogs)
```

> Regra: dado de RSC → `revalidatePath/Tag`; dado de Query → `invalidateQueries`. Evite manter a mesma leitura nas duas camadas; quando inevitável (lista server + autocomplete client), invalide ambas. Detalhe em [`./08`](./08-consumo-api-dados.md) §8.

---

## 8. Centro de notificações (header bell)

Notificações são **dados do backend** (tabela `notifications`), persistidas, com contador no sino do header (spec 1.4 "Notificações"). Por serem leitura client viva, usam **TanStack Query** com atualização periódica — **nunca** Zustand (não são UI efêmera nem cabe duplicá-las).

```typescript
// _business/hooks/use-notifications/index.ts ('use client')
'use client';
import { useQuery } from '@tanstack/react-query';
import { getNotifications } from '@/app/(modules)/(comunicacao)/_business';

export function useNotifications() {
  return useQuery({
    queryKey: ['notifications'],
    queryFn: getNotifications,                 // GET /api/notifications → { data, meta }
    refetchInterval: 30_000,                   // polling ~30s (spec 1.4)
    refetchOnWindowFocus: true,
  });
}
```

| Aspecto | Decisão |
|---------|---------|
| Fonte | Backend (`/api/notifications`), persistidas. |
| Transporte | **Polling ~30s** (default) ou **SSE/socket.io** quando disponível (ver [`./02`](./02-stack-tecnologico.md)). |
| Estado | TanStack Query (`['notifications']`); contador derivado dos dados. |
| Marcar como lida | Mutação → `invalidateQueries({ queryKey: ['notifications'] })`. |
| UI (sino aberto/fechado) | `useState` no hook do dropdown ou store de UI — isso **sim** é Zustand/`useState`. |

> Se/quando migrar para SSE, troque `refetchInterval` por um stream que faz `queryClient.setQueryData(['notifications'], ...)` — o resto da UI não muda.

---

## 9. Anti-padrões

| Evitar | Usar |
|--------|------|
| JWT/sessão/permissões em Zustand | Servidor: cookie httpOnly + `getServerSession` (§5) |
| Dados de domínio do backend em Zustand | RSC (inicial) ou TanStack Query (client) |
| Filtros de listagem em `useState` | URL via `useUrlSyncedFilters` (§6) |
| `useState` global "manual" (prop drilling) p/ UI compartilhada | Zustand store em `src/stores/` |
| Store para estado local de 1 componente | `useState` no hook `useXxx` |
| Refetch manual com `useEffect` + `fetch` | TanStack Query (`queryKey` + `invalidateQueries`) |
| Notificações em estado local | Query `['notifications']` com polling/SSE (§8) |
| Espelhar lista server em Query "por garantia" | Uma origem por dado; invalidar a camada certa (§7) |

---

## 10. Ver também

- [`./01-arquitetura.md`](./01-arquitetura.md) — camadas e fluxo de dados.
- [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md) — sessão no servidor, por que não Zustand, `hasAccess`.
- [`./08-consumo-api-dados.md`](./08-consumo-api-dados.md) — `api`, envelope, fetch RSC vs Query, cache/invalidação.
- [`./11-tabelas-listagens.md`](./11-tabelas-listagens.md) — DataTable, paginação, filtros na URL.
- [`./10-formularios-validacao.md`](./10-formularios-validacao.md) — estado de formulário (RHF + Zod).
