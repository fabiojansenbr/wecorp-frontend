# 08 — Consumo de API e data fetching

> Único ponto de HTTP: **`api`** em `@/infra` (get/post/put/patch/delete). **MUST NOT** `fetch` direto.
> Token resolvido **no servidor** (cookie httpOnly via `next/headers`). Envelope **igual ao backend**: `{data}` / `{data,meta}` / `{error:{code,message,details[]}}`.
> Erro tratado por `error.code`/`status` — **nunca** por texto. Fetch inicial em Server Component + Suspense; cache client com TanStack Query.
>
> Contrato canônico: [`backend/docs/08`](../../backend/docs/08-contrato-api.md). Relacionados: [`./01-arquitetura.md`](./01-arquitetura.md) · [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md) · [`./11-tabelas-listagens.md`](./11-tabelas-listagens.md) · [`./12-estado-dados.md`](./12-estado-dados.md)

---

## 1. Camada `@/infra` (fetcher + `api`)

O fluxo de dados é em camadas (ver [`./01`](./01-arquitetura.md)). `@/infra` é a **última camada antes do backend**:

```text
hook → action → api (@/infra) → fetcher → backend REST
```

| Peça | Responsabilidade |
|------|------------------|
| `fetcher` | Encapsula `fetch`; monta URL base (`env.NEXT_PUBLIC_API_URL`), headers, serialização JSON/FormData, parse do envelope e **normalização de erro**. |
| `api` | Cliente fino com `get/post/put/patch/delete`. Resolve o **token no servidor** e desembrulha `{data}`. |

```typescript
// src/infra/http/fetcher.ts
import { getAccessToken } from '@/shared/cookies';
import { env } from '@/configs/env';

export interface ApiError extends Error {
  status?: number;
  code?: string;        // SCREAMING_SNAKE_CASE do backend
  details?: Array<{ field: string; code: string; message: string }>;
}

export async function fetcher<T>(path: string, init: RequestInit = {}): Promise<{ data: T; meta?: ApiMeta }> {
  const token = await getAccessToken(); // cookie httpOnly — só no servidor

  const res = await fetch(`${env.NEXT_PUBLIC_API_URL}${path}`, {
    ...init,
    headers: {
      Accept: 'application/json',
      ...(init.body instanceof FormData ? {} : { 'Content-Type': 'application/json' }),
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...init.headers,
    },
  });

  const body = await res.json().catch(() => null);

  if (!res.ok) {
    const error = new Error(body?.error?.message ?? 'Erro') as ApiError;
    error.status = res.status;
    error.code = body?.error?.code;          // ex.: VALIDATION_ERROR, FORBIDDEN
    error.details = body?.error?.details;
    if (res.status === 401 || res.status === 403) error.digest = 'AUTH'; // pode sinalizar redirect
    throw error;
  }

  return { data: body.data, meta: body.meta };
}
```

```typescript
// src/infra/http/index.ts
import { fetcher } from './fetcher';

export const api = {
  get:    <T>(path: string, init?: RequestInit) => fetcher<T>(path, { ...init, method: 'GET' }),
  post:   <T, B = unknown>(path: string, body?: B, init?: RequestInit) =>
            fetcher<T>(path, { ...init, method: 'POST', body: serialize(body) }),
  put:    <T, B = unknown>(path: string, body?: B, init?: RequestInit) =>
            fetcher<T>(path, { ...init, method: 'PUT', body: serialize(body) }),
  patch:  <T, B = unknown>(path: string, body?: B, init?: RequestInit) =>
            fetcher<T>(path, { ...init, method: 'PATCH', body: serialize(body) }),
  delete: <T>(path: string, init?: RequestInit) => fetcher<T>(path, { ...init, method: 'DELETE' }),
};

function serialize(body: unknown) {
  if (body === undefined) return undefined;
  return body instanceof FormData ? body : JSON.stringify(body);
}
```

> **Regra (MUST):** nunca `fetch` direto no código de aplicação — sempre `import { api } from '@/infra'`. O token vive em cookie httpOnly e só é legível no servidor; por isso o JavaScript do browser **não** o lê (ver [`./07`](./07-auth-sessao-permissoes.md)).

---

## 2. Envelope de resposta

Idêntico ao backend ([`backend/docs/08`](../../backend/docs/08-contrato-api.md) §2 e spec 1.7). **Três** formatos, sempre — nunca array/objeto cru.

### 2.1 Recurso único — `{ data }`

```jsonc
// 200 OK  GET /api/empresas/uuid
{ "data": { "id": "uuid", "nome": "Imob X", "documento": "12345678000190", "statusEmpresa": 1 } }
```

### 2.2 Lista paginada — `{ data, meta }`

```jsonc
// 200 OK  GET /api/empresas?page=1&perPage=25
{
  "data": [ { "id": "uuid", "nome": "Imob X" } ],
  "meta": { "total": 1234, "page": 1, "perPage": 25, "totalPages": 50 }
}
```

```typescript
// src/infra/http/interfaces.ts
export interface ApiMeta { total: number; page: number; perPage: number; totalPages: number; }
```

### 2.3 Erro — `{ error: { code, message, details[] } }`

```jsonc
// 422 Unprocessable Entity
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Erro de validação",
    "details": [ { "field": "documento", "code": "invalid_cpf", "message": "CPF inválido" } ]
  }
}
```

`details` só aparece em validação (422); ausente em 401/403/404 genéricos (mensagens genéricas, anti-enumeração).

---

## 3. Tratamento de erro

**MUST** chavear por `error.code` ou `status` — **nunca** pelo texto da `message` (ele muda, é localizado e não é estável).

| HTTP | `code` | Significado no frontend |
|------|--------|-------------------------|
| 400 | `BAD_REQUEST` | Requisição malformada. |
| 401 | `INVALID_CREDENTIALS` / `UNAUTHENTICATED` | Login inválido / sessão expirada → tratado pelo `proxy.ts` (refresh) ou redireciona ao login. |
| 403 | `FORBIDDEN` | Sem papel/permissão → toast + esconder ação. |
| 404 | `NOT_FOUND` | Inexistente ou fora do escopo do tenant. |
| 409 | `CONFLICT` / `IDEMPOTENCY_KEY_REUSED` | Conflito de estado. |
| 422 | `VALIDATION_ERROR` | Mapear `details[]` para campos do form (RHF `setError`). |
| 429 | `RATE_LIMITED` | "Muitas tentativas, aguarde." |
| 500 | `INTERNAL_ERROR` | Mensagem padrão genérica. |

```typescript
// src/shared/utils/errors/resolveErrorMessage.ts
import type { ApiError } from '@/infra';

const DEFAULT = 'Não foi possível concluir, tente novamente ou contate o suporte.';

const BY_CODE: Record<string, string> = {
  INVALID_CREDENTIALS: 'Usuário ou senha inválidos.',
  FORBIDDEN: 'Você não tem permissão para esta ação.',
  NOT_FOUND: 'Registro não encontrado.',
  CONFLICT: 'Este registro já existe ou está em conflito.',
  RATE_LIMITED: 'Muitas tentativas. Aguarde um instante e tente novamente.',
  VALIDATION_ERROR: 'Verifique os campos destacados.',
};

export function resolveErrorMessage(error: unknown): string {
  const e = error as ApiError;
  if (e?.code && BY_CODE[e.code]) return BY_CODE[e.code];
  if (e?.status === 401) return 'Sessão expirada. Faça login novamente.';
  return DEFAULT;
}
```

Mapeamento de erros de validação (422) para o formulário, no hook:

```typescript
// dentro do catch do hook (RHF)
catch (error) {
  const e = error as ApiError;
  if (e.code === 'VALIDATION_ERROR' && e.details) {
    e.details.forEach((d) => form.setError(d.field as never, { message: d.message }));
    return;
  }
  toast.error(resolveErrorMessage(error));
}
```

---

## 4. Data fetching no Server Component

Padrão para listagens/telas de leitura: **fetch inicial no servidor** + `<Suspense>`. Reduz JS no client e melhora TTI/LCP (objetivos do produto).

```typescript
// _business/actions/get-empresas/index.ts
import { api } from '@/infra';
import type { Empresa } from '../../schemas/empresa';

export async function getEmpresas(query: Record<string, string>) {
  const qs = new URLSearchParams(query).toString();
  return api.get<Empresa[]>(`/empresas${qs ? `?${qs}` : ''}`); // → { data, meta }
}
```

```tsx
// (web)/(pages)/empresas/page.tsx (Server Component)
import { Suspense } from 'react';
import { Card, TableFallback } from '@/components';
import { PermissionGateServer } from '@/components/permission';
import { ActionsPermissionEnum } from '@/shared/permissions';
import { getEmpresas } from '@/app/(modules)/(empresas)/_business';

async function EmpresasTable({ filters }: { filters: Record<string, string> }) {
  const { data, meta } = await getEmpresas(filters);     // token resolvido no servidor
  return <EmpresasDataTable rows={data} meta={meta} />;
}

export default async function EmpresasPage(props: { searchParams: Promise<Record<string, string>> }) {
  const filters = await props.searchParams;              // Next 16: searchParams é Promise
  return (
    <PermissionGateServer requirement={{ action: ActionsPermissionEnum.EmpresasView }}>
      <Card title="Empresas">
        <Suspense fallback={<TableFallback />}>
          <EmpresasTable filters={filters} />
        </Suspense>
      </Card>
    </PermissionGateServer>
  );
}
```

| Cache (Next) | Quando | Como |
|--------------|--------|------|
| Estático (revalidável) | Dados pouco voláteis (lookups) | `api.get(path, { next: { revalidate: 3600 } })` |
| Dinâmico (no-store) | Listagens por usuário/tenant | default em rotas com cookies; ou `{ cache: 'no-store' }` |
| Tag-based | Invalidação seletiva | `{ next: { tags: ['empresas'] } }` + `revalidateTag('empresas')` |

---

## 5. TanStack Query no client

Use Query **quando a leitura acontece no client**: dialogs que carregam dados sob demanda, refetch após ação, cache entre componentes, lookups de autocomplete. Para o fetch inicial da página, prefira o Server Component (§4). Ver matriz em [`./12-estado-dados.md`](./12-estado-dados.md).

```typescript
// _business/hooks/use-empresa-lookup/index.ts ('use client')
'use client';
import { useQuery } from '@tanstack/react-query';
import { getEmpresasLookup } from '@/app/(modules)/(empresas)/_business';

export function useEmpresaLookup(term: string) {
  return useQuery({
    queryKey: ['empresas', 'lookup', term],
    queryFn: () => getEmpresasLookup(term),
    enabled: term.length >= 2,
    staleTime: 60_000,
  });
}
```

A mesma `action` (que usa `api`) serve ao Server Component e ao `queryFn` — não há `fetch` paralelo. O `QueryClientProvider` fica em `src/providers/` ([`./12`](./12-estado-dados.md)).

---

## 6. Mutações: Server Actions + `revalidatePath`

Mutações chamam a `action` (sem `try/catch`) pelo hook, que trata erro e revalida. Após alterar dados do servidor: `revalidatePath('/rota')` (RSC) ou `queryClient.invalidateQueries` (cache client) — ver §8.

```typescript
// _business/actions/create-empresa/index.ts
import { api } from '@/infra';
import type { EmpresaSchemaProps } from '../../schemas/empresa';

export async function createEmpresa(props: EmpresaSchemaProps) {
  return api.post('/empresas', props); // sem try/catch
}
```

```typescript
// _business/hooks/use-form-create-empresa/index.ts ('use client')
'use client';
export function useFormCreateEmpresa() {
  const [isOpen, setIsOpen] = useState(false);
  const form = useForm<EmpresaSchemaProps>({ resolver: zodResolver(empresaSchema) });

  async function onSubmit(data: EmpresaSchemaProps) {
    try {
      await createEmpresa(data);
      setIsOpen(false);
      form.reset();
      revalidatePath('/empresas');          // atualiza a listagem (RSC)
      toast.success('Empresa cadastrada com sucesso');
    } catch (error) {
      const e = error as ApiError;
      if (e.code === 'VALIDATION_ERROR' && e.details) {
        e.details.forEach((d) => form.setError(d.field as never, { message: d.message }));
        return;
      }
      toast.error(resolveErrorMessage(error));
    }
  }

  return { form, isOpen, setIsOpen, onSubmit: form.handleSubmit(onSubmit) };
}
```

---

## 7. Paginação, filtros e querystring

Listagens server-side seguem o contrato único do backend ([`backend/docs/08`](../../backend/docs/08-contrato-api.md) §3):

| Param | Tipo | Default | Uso |
|-------|------|---------|-----|
| `page` | int ≥ 1 | `1` | Página (1-based). |
| `perPage` | int 1..100 | `25` | Itens por página. |
| `sort` | string | — | Campo (allowlist no backend). |
| `order` | `asc`/`desc` | `asc` | Direção. |
| `search` | string | — | Busca textual. |
| `<filtro>` | string \| string[] | — | Filtro específico do recurso. |

```text
GET /api/empresas?page=1&perPage=25&sort=nome&order=asc&search=hubstate&statusEmpresa=1
```

Os filtros vivem **na URL** (sincronizados via `useUrlSyncedFilters`), não em estado local — assim a página é compartilhável e o Server Component os lê de `searchParams`. Detalhe de UI/tabela em [`./11-tabelas-listagens.md`](./11-tabelas-listagens.md); estado de filtros em [`./12-estado-dados.md`](./12-estado-dados.md).

---

## 8. Cache e invalidação

| Origem do dado | Como invalidar |
|----------------|----------------|
| Server Component (cache Next) | `revalidatePath('/rota')` ou `revalidateTag('tag')` no hook após mutação. |
| TanStack Query (cache client) | `queryClient.invalidateQueries({ queryKey: ['empresas'] })`. |
| Ambos (lista server + dialog client) | Revalidar os dois: `revalidatePath` + `invalidateQueries`. |

> Regra: dado renderizado por RSC → `revalidatePath/Tag`; dado lido por Query no client → `invalidateQueries`. Não duplicar a mesma leitura nas duas camadas sem necessidade.

---

## 9. Headers, autenticação e multi-tenant

`api`/`fetcher` anexam automaticamente:

| Header | Origem | Observação |
|--------|--------|------------|
| `Authorization: Bearer <jwt>` | cookie httpOnly (servidor) | Resolvido por `getAccessToken()`. |
| `X-Tenant` | host/`saasId` da sessão | Dica de roteamento/tema; o **escopo de segurança vem do JWT**, não do header ([`./06`](./06-multitenancy-whitelabel.md)). |
| `X-Request-Id` | gerado/propagado | Correlação de logs (opcional). |
| `X-Idempotency-Key` | gerado em POST não-repetível | Quando a operação não pode duplicar efeito ([`backend/docs/08`](../../backend/docs/08-contrato-api.md) §7). |

O **refresh** do access token é feito proativamente pelo `proxy.ts` (rotation) — a camada `api` apenas envia o token vigente; não implementa retry de refresh por conta própria. Ver [`./07`](./07-auth-sessao-permissoes.md) §3.

---

## 10. BFF / proxy (quando agregar contrato)

Em geral o frontend fala **direto** com a API REST via `api`. Use Route Handlers (BFF) **apenas** quando for necessário:

- **Agregar** múltiplas chamadas em um payload só para a tela (evitar waterfalls no client);
- **Adaptar** um contrato legado/diferente ao formato do envelope;
- **Esconder** um endpoint/segredo que não deve ir ao client.

Mesmo no BFF, a regra de negócio crítica permanece no backend; o Route Handler valida entrada com Zod e repassa via `api`. Ver [`./01`](./01-arquitetura.md) §6.

---

## 11. Anti-padrões

| Evitar | Usar |
|--------|------|
| `fetch('...')` direto | `api` de `@/infra` |
| `try/catch` na action | Tratamento no hook (`toast` + `resolveErrorMessage`) |
| Chavear erro por `error.message` (texto) | `error.code` / `error.status` |
| Ler token no client p/ montar header | Token resolvido no servidor (cookie httpOnly) |
| Retornar array/objeto cru | Envelope `{data}` / `{data,meta}` |
| Filtros em `useState` | Querystring + `useUrlSyncedFilters` |
| Fetch client duplicando o fetch server | RSC para inicial; Query só para client (dialogs/refetch) |
| `process.env.X` espalhado | `env` validado em `@/configs/env` |

---

## 12. Ver também

- [`./01-arquitetura.md`](./01-arquitetura.md) — camadas UI→hooks→actions→`api`→fetcher.
- [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md) — token em cookie, `proxy.ts`, refresh.
- [`./11-tabelas-listagens.md`](./11-tabelas-listagens.md) — DataTable, paginação, export.
- [`./12-estado-dados.md`](./12-estado-dados.md) — matriz server/client/Query/Zustand.
- [`./10-formularios-validacao.md`](./10-formularios-validacao.md) — RHF + Zod + mapeamento de `details[]`.
- Backend: [`backend/docs/08`](../../backend/docs/08-contrato-api.md) — envelope, paginação, headers, rate limit, idempotência, códigos de erro.
