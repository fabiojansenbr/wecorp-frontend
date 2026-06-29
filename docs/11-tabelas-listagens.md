# 11 — Tabelas e Listagens

> Todas as listagens do weCorp seguem **um único padrão**: um `DataTable` **server-side** sobre TanStack Table 8, com paginação, filtros, ordenação, ações em massa, export e filtros salvos. O fetch inicial é feito no **Server Component** com `<Suspense>`; a interatividade (filtros, seleção) fica num componente client orquestrado por hook.
> Regras não-negociáveis em [`../CLAUDE.md`](../CLAUDE.md) §7. Arquitetura em [`./01-arquitetura.md`](./01-arquitetura.md); consumo de API em [`./08-consumo-api-dados.md`](./08-consumo-api-dados.md).

---

## 1. Princípio: server-side, sempre

A paginação/filtro/ordenação acontecem **no backend** — o frontend nunca baixa a tabela inteira para filtrar em memória. O `DataTable` opera em **modo manual** (`manualPagination`, `manualSorting`, `manualFiltering`): ele apenas reflete o que veio da API e empurra os parâmetros de volta para a URL.

```text
URL (?page&perPage&sort&order&search...)   ← fonte de verdade dos parâmetros
        ↓ (Server Component lê searchParams)
action getX(filters) → api (@/infra) → backend            ← devolve { data, meta }
        ↓
<DataTable data={data} meta={meta} columns={columns} />   ← render + controles
        ↓ (usuário muda filtro/página)
useUrlSyncedFilters → atualiza a URL → re-render do Server Component
```

---

## 2. Paginação (lendo o `meta` do envelope)

A API devolve o envelope paginado `{ data, meta }` (ver [`./08-consumo-api-dados.md`](./08-consumo-api-dados.md)):

```json
{ "data": [ /* ... */ ],
  "meta": { "total": 1234, "page": 1, "perPage": 25, "totalPages": 50 } }
```

- Parâmetros enviados: `page` (1-based) e `perPage`.
- O rodapé da tabela usa `meta.total`/`meta.totalPages`/`meta.page` — **nunca** `data.length`.
- O `pageCount` do TanStack Table vem de `meta.totalPages` (modo `manualPagination`).

---

## 3. Filtros

- **Um input por critério** (sem um "filtro mágico" que mistura campos). Ex.: `/usuarios` filtra por `id`, `nome`, `login`, `e-mail`, `grupo`, `status`, `empresa`.
- **Debounce ~400ms** em inputs de texto antes de empurrar para a URL, evitando uma requisição por tecla.
- Cada filtro tem um schema Zod (coerção/validação dos valores da URL) e é tipado.

```typescript
// _business/schemas/users-filters/index.ts
import { z } from 'zod';

export const usersFiltersSchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  perPage: z.coerce.number().int().positive().max(100).default(25),
  search: z.string().trim().optional(),
  grupoId: z.coerce.number().int().min(1).max(10).optional(),
  status: z.enum(['1', '0']).optional(),
  sort: z.string().optional(),
  order: z.enum(['asc', 'desc']).optional(),
});

export type UsersFiltersProps = z.infer<typeof usersFiltersSchema>;
```

---

## 4. Sincronização de filtros na URL: `useUrlSyncedFilters`

A querystring é a **fonte de verdade** dos filtros (compartilhável, navegável, sobrevive a refresh). Em vez de `nuqs` (citado no spec antigo), usamos o hook próprio **`useUrlSyncedFilters`** (em `src/hooks/`), que lê/escreve `searchParams` de forma tipada e com debounce.

```typescript
// src/hooks/use-url-synced-filters/index.ts (assinatura)
export function useUrlSyncedFilters<TSchema extends z.ZodTypeAny>(args: {
  schema: TSchema;
  debounceMs?: number;            // default 400
}): {
  filters: z.infer<TSchema>;      // valores atuais (já validados)
  setFilter: (key: keyof z.infer<TSchema>, value: unknown) => void;
  setFilters: (patch: Partial<z.infer<TSchema>>) => void;
  resetFilters: () => void;
};
```

```typescript
// uso no client
'use client';
import { useUrlSyncedFilters } from '@/hooks';
import { usersFiltersSchema } from '@/app/(modules)/(users)/_business';

const { filters, setFilter, resetFilters } = useUrlSyncedFilters({ schema: usersFiltersSchema });
// setFilter('search', value) → atualiza ?search=... (com debounce) → Server Component re-fetcha
```

> Mudar página/ordenação/filtro **muda a URL**; o Server Component pai relê `searchParams` (Promise no Next 16 — `await`) e refaz o fetch. Não há estado de tabela duplicado em Zustand.

---

## 5. Ordenação, seleção e ações em massa

- **Ordenação:** clique no cabeçalho alterna `sort`/`order` na URL (modo `manualSorting`). Colunas marcam `enableSorting`.
- **Seleção:** coluna de checkbox (`rowSelection` do TanStack Table); o cabeçalho seleciona a página atual.
- **Ações em massa:** uma barra aparece quando há linhas selecionadas (ex.: "Exportar selecionados", "Bloquear", "Excluir"). Cada ação chama uma `action` via hook (com `toast` + `revalidatePath`). Ações em massa também respeitam a ACL (ver §8).

---

## 6. Export (CSV / XLSX / PDF) e filtros salvos

- **Export** respeita os filtros ativos: o pedido de export envia os **mesmos** parâmetros da listagem para o backend, que gera o arquivo (CSV/XLSX/PDF). Evita exportar a página visível apenas.
- **Filtros salvos por usuário:** o usuário pode salvar um conjunto de filtros nomeado e reaplicá-lo. Persistência preferencial via backend (preferências do usuário); como fallback simples pode-se usar `localStorage` por usuário — **nunca** para dados sensíveis ou de sessão.

---

## 7. Composição: página (Server) + tabela (Client)

### 7.1 Página de listagem (Server Component)

Faz o fetch inicial com os filtros da URL e envolve a tabela em `<Suspense>` com `TableFallback`. O `<Card>` e o cabeçalho aparecem imediatamente; só a tabela mostra skeleton.

```tsx
// (modules)/(users)/(web)/(pages)/users/page.tsx (Server Component)
import { Suspense } from 'react';
import { Card, TableFallback } from '@/components';
import { PermissionGateServer } from '@/components';
import { getUsers, usersFiltersSchema } from '@/app/(modules)/(users)/_business';
import { ModulesPermissionEnum, ActionsPermissionEnum } from '@/shared';
import { UsersTable } from '../_components/users-table';
import { columns } from '../_components/columns';

async function UsersTableContent({ raw }: { raw: Record<string, string> }) {
  const filters = usersFiltersSchema.parse(raw);          // valida/coage a URL
  const { data, meta } = await getUsers(filters);          // action → api → backend
  return <UsersTable data={data} meta={meta} columns={columns} />;
}

export default async function UsersPage(props: {
  searchParams: Promise<Record<string, string>>;
}) {
  const raw = await props.searchParams;                    // Next 16: Promise
  return (
    <PermissionGateServer requirement={{ module: ModulesPermissionEnum.ADMIN, action: ActionsPermissionEnum.LIST }}>
      <Card title="Usuários">
        <Suspense fallback={<TableFallback columns={columns} />}>
          <UsersTableContent raw={raw} />
        </Suspense>
      </Card>
    </PermissionGateServer>
  );
}
```

### 7.2 Componente de tabela (Client Component)

Controla filtros/ordenação/seleção via `useUrlSyncedFilters` e renderiza o TanStack Table em modo manual.

```tsx
// (modules)/(users)/(web)/_components/users-table/index.tsx
'use client';
import {
  useReactTable, getCoreRowModel, flexRender, type ColumnDef,
} from '@tanstack/react-table';
import { useUrlSyncedFilters } from '@/hooks';
import { DataTablePagination, DataTableToolbar } from '@/components';
import { usersFiltersSchema } from '@/app/(modules)/(users)/_business';
import type { User } from '../../../_business';

interface UsersTableProps {
  data: User[];
  meta: { total: number; page: number; perPage: number; totalPages: number };
  columns: ColumnDef<User>[];
}

export function UsersTable({ data, meta, columns }: UsersTableProps) {
  const { filters, setFilter } = useUrlSyncedFilters({ schema: usersFiltersSchema });

  const table = useReactTable({
    data,
    columns,
    getCoreRowModel: getCoreRowModel(),
    manualPagination: true,
    manualSorting: true,
    manualFiltering: true,
    pageCount: meta.totalPages,
    state: {
      pagination: { pageIndex: meta.page - 1, pageSize: meta.perPage },
    },
  });

  return (
    <div className="space-y-3">
      <DataTableToolbar filters={filters} onChange={setFilter} />  {/* 1 input por critério, debounce 400ms */}
      <table className="w-full">
        <thead>
          {table.getHeaderGroups().map((hg) => (
            <tr key={hg.id}>
              {hg.headers.map((h) => (
                <th key={h.id} onClick={h.column.getToggleSortingHandler()}>
                  {flexRender(h.column.columnDef.header, h.getContext())}
                </th>
              ))}
            </tr>
          ))}
        </thead>
        <tbody>
          {table.getRowModel().rows.map((row) => (
            <tr key={row.id}>
              {row.getVisibleCells().map((cell) => (
                <td key={cell.id}>{flexRender(cell.column.columnDef.cell, cell.getContext())}</td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
      <DataTablePagination meta={meta} onPageChange={(p) => setFilter('page', p)} />
    </div>
  );
}
```

### 7.3 Colunas tipadas

Colunas são `ColumnDef<T>[]` tipadas pelo modelo do domínio. Células de display reaproveitam formatadores (`<Money>`, `<Date>`, `<Cpf>`…). A coluna de ações da linha respeita a ACL (§8).

```tsx
// (modules)/(users)/(web)/_components/columns.tsx
'use client';
import type { ColumnDef } from '@tanstack/react-table';
import type { User } from '../../_business';
import { RowActions } from './row-actions';

export const columns: ColumnDef<User>[] = [
  { accessorKey: 'nome', header: 'Nome', enableSorting: true },
  { accessorKey: 'empresa.nome', header: 'Empresa' },
  { accessorKey: 'grupo.nome', header: 'Grupo' },
  { accessorKey: 'login', header: 'Login' },
  { accessorKey: 'status', header: 'Status', cell: ({ row }) => <StatusBadge value={row.original.status} /> },
  { id: 'actions', header: '', cell: ({ row }) => <RowActions user={row.original} /> },
];
```

---

## 8. Integração com permissões (ACL)

As ações de linha e de massa são filtradas pela ACL dos **10 grupos** — um usuário só vê "Editar"/"Excluir" se tiver o `nivel` exigido para aquela ação. Use `<PermissionGateClient>` na célula de ações (ou `hasAccess` no hook que monta o menu da linha).

```tsx
// row-actions.tsx
'use client';
import { PermissionGateClient } from '@/components';
import { ModulesPermissionEnum, ActionsPermissionEnum } from '@/shared';

export function RowActions({ user }: { user: User }) {
  return (
    <PermissionGateClient requirement={{ module: ModulesPermissionEnum.ADMIN, action: ActionsPermissionEnum.UPDATE }}>
      <EditButton id={user.id} />
    </PermissionGateClient>
  );
}
```

> Strings de módulo/ação **MUST NOT** ser hardcoded — sempre enums de `@/shared`. A `PermissionGateServer` da página (§7.1) protege o acesso à listagem inteira; os gates de linha refinam por ação. Ver [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md).

---

## 9. Estados: empty, loading e error

| Estado | Como | Onde |
|--------|------|------|
| **Loading** | `<Suspense fallback={<TableFallback columns={columns} />}>` no fetch inicial; skeleton com a mesma grade de colunas. | Página (Server). |
| **Empty** | Quando `data.length === 0`: `<EmptyState>` com mensagem PT-BR e CTA (ex.: "Nenhum usuário encontrado para os filtros."). | Tabela (Client). |
| **Error** | `error.tsx` do segmento, ou tratamento via `resolveErrorMessage` quando a falha ocorre numa ação (toast). | Segmento / hook. |

> Condicionais no JSX usam `<If condition={} elseRender={}>` — não `&&`/ternário inline (CLAUDE §7). Estados de borda fazem parte da Definition of Done.

---

## 10. Ver também

- [01 — Arquitetura](./01-arquitetura.md) · [05 — Roteamento e navegação](./05-roteamento-navegacao.md)
- [08 — Consumo de API e dados](./08-consumo-api-dados.md) · [12 — Estado e dados](./12-estado-dados.md)
- [07 — Auth, sessão e permissões](./07-auth-sessao-permissoes.md) · [04 — Design system e UI](./04-design-system-ui.md)
- [10 — Formulários e validação](./10-formularios-validacao.md)
</content>
