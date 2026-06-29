# 07 — Autenticação, sessão e permissões

> Sessão **100% no servidor**: token em **cookie httpOnly** (nunca `localStorage`/`document.cookie`), `proxy.ts` na borda (refresh proativo + redirects), `getServerSession()` como fonte única.
> Autorização pelos **10 grupos** do legado + **ACL granular** (permissions/modules), via enums em `@/shared`, `<PermissionGate>` e `hasAccess()` **testado**.
> Coerente com o backend: [`backend/docs/07`](../../backend/docs/07-auth-permissoes.md) (JWT HS256, claims, RBAC) e o envelope de [`backend/docs/08`](../../backend/docs/08-contrato-api.md).
>
> Relacionados: [`../CLAUDE.md`](../CLAUDE.md) §6 · [`./02-stack-tecnologico.md`](./02-stack-tecnologico.md) · [`./06-multitenancy-whitelabel.md`](./06-multitenancy-whitelabel.md) · [`./08-consumo-api-dados.md`](./08-consumo-api-dados.md)

---

## 1. Princípios (MUST / MUST NOT)

| Regra | Detalhe |
|-------|---------|
| **MUST** — token em cookie | `httpOnly + secure + sameSite=lax + path=/`. Escrita centralizada em `src/shared/cookies/`. |
| **MUST NOT** — token no client | Proibido `localStorage`, `sessionStorage`, `document.cookie` para access/refresh token. |
| **MUST** — sessão no servidor | Fonte única `getServerSession()` (com `cache` do React); passada ao client **uma vez** (prop RSC ou contexto hidratado). |
| **MUST NOT** — JWT no Zustand | Zustand é **apenas UI**; nunca espelha JWT, sessão ou dados sensíveis. Ver [`./12-estado-dados.md`](./12-estado-dados.md). |
| **MUST** — autorização por enum | Strings de role/módulo/ação **sempre** via `ModulesPermissionEnum`/`ActionsPermissionEnum`/`RolesPermissionEnum`. Nunca hardcodar. |
| **MUST** — `hasAccess` testado | Função pura com **cobertura de testes obrigatória** (todos os branches). Ver [`./13-testes.md`](./13-testes.md). |

> O backend é a autoridade de segurança: cada request carrega o JWT e o backend **reaplica** RBAC/ACL e escopo de tenant ([`backend/docs/06`](../../backend/docs/06-multi-tenancy.md)). O frontend usa permissões para **exibição** (esconder menu/botão/rota) — nunca como única barreira.

---

## 2. Cookies de sessão

Dois cookies, ambos httpOnly, escritos **só no servidor** (Server Action / Route Handler / `proxy.ts`). Os nomes vêm de `env` (`NEXT_PUBLIC_TOKEN_KEY` / `NEXT_PUBLIC_REFRESH_TOKEN_KEY`) — ver [`./14-deploy-ambiente.md`](./14-deploy-ambiente.md).

| Cookie | Conteúdo | TTL | Flags |
|--------|----------|-----|-------|
| access token | JWT HS256 (claims `userId/empresaId/saasId/grupoId`) | curto (~15 min) | `httpOnly secure sameSite=lax path=/` |
| refresh token | refresh opaco/JWT (rotation) | longo (~7 dias) | `httpOnly secure sameSite=lax path=/` |

```typescript
// src/shared/cookies/index.ts — única porta de escrita/leitura de cookies de sessão
import { cookies } from 'next/headers';
import { env } from '@/configs/env';

const isProd = process.env.NODE_ENV === 'production';
const base = { httpOnly: true, secure: isProd, sameSite: 'lax', path: '/' } as const;

export async function setSessionCookies(tokens: { accessToken: string; refreshToken: string; expiresIn: number }) {
  const jar = await cookies();
  jar.set(env.NEXT_PUBLIC_TOKEN_KEY, tokens.accessToken, { ...base, maxAge: tokens.expiresIn });
  jar.set(env.NEXT_PUBLIC_REFRESH_TOKEN_KEY, tokens.refreshToken, { ...base, maxAge: 60 * 60 * 24 * 7 });
}

export async function getAccessToken() {
  const jar = await cookies();
  return jar.get(env.NEXT_PUBLIC_TOKEN_KEY)?.value ?? null;
}

export async function clearSessionCookies() {
  const jar = await cookies();
  jar.delete(env.NEXT_PUBLIC_TOKEN_KEY);
  jar.delete(env.NEXT_PUBLIC_REFRESH_TOKEN_KEY);
}
```

> **`secure` em dev:** relaxado quando `NODE_ENV !== 'production'` para permitir HTTP local. Em produção é sempre `true`.

---

## 3. `proxy.ts` (middleware Next 16)

No Next 16 o middleware chama-se **`proxy.ts`** (raiz de `src/`). Ele é a **borda** da aplicação e cuida de: rotas públicas, refresh proativo, redirects e limpeza de sessão inválida. White-label (detecção de marca por host) também passa por aqui — ver [`./06-multitenancy-whitelabel.md`](./06-multitenancy-whitelabel.md).

```typescript
// src/proxy.ts
import { NextRequest, NextResponse } from 'next/server';
import { env } from '@/configs/env';
import { isAccessExpiringSoon, refreshTokens } from '@/shared/jwt';

const PUBLIC_ROUTES = ['/sign-in', '/recover-account', '/reset-password', '/termos', '/termos-eula'];

function isPublic(pathname: string) {
  return PUBLIC_ROUTES.some((r) => pathname === r || pathname.startsWith(`${r}/`));
}

export async function proxy(req: NextRequest) {
  const { pathname } = req.nextUrl;
  const access = req.cookies.get(env.NEXT_PUBLIC_TOKEN_KEY)?.value;
  const refresh = req.cookies.get(env.NEXT_PUBLIC_REFRESH_TOKEN_KEY)?.value;
  const res = NextResponse.next();

  // 1) sem sessão em rota privada → login
  if (!access && !refresh && !isPublic(pathname)) {
    return NextResponse.redirect(new URL('/sign-in', req.url));
  }

  // 2) já logado tentando rota pública → home
  if (access && isPublic(pathname)) {
    return NextResponse.redirect(new URL('/dashboard', req.url));
  }

  // 3) refresh PROATIVO: access perto de expirar e há refresh válido
  if (refresh && (!access || isAccessExpiringSoon(access))) {
    const renewed = await refreshTokens(refresh); // POST /api/auth/refresh (rotation)
    if (renewed) {
      res.cookies.set(env.NEXT_PUBLIC_TOKEN_KEY, renewed.accessToken, { httpOnly: true, sameSite: 'lax', path: '/' });
      res.cookies.set(env.NEXT_PUBLIC_REFRESH_TOKEN_KEY, renewed.refreshToken, { httpOnly: true, sameSite: 'lax', path: '/' });
    } else {
      // 4) sessão inválida → limpa cookies e manda pro login
      const redirect = NextResponse.redirect(new URL('/sign-in', req.url));
      redirect.cookies.delete(env.NEXT_PUBLIC_TOKEN_KEY);
      redirect.cookies.delete(env.NEXT_PUBLIC_REFRESH_TOKEN_KEY);
      return redirect;
    }
  }

  return res;
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|logos|api/health).*)'],
};
```

| Responsabilidade | Comportamento |
|------------------|---------------|
| Rotas públicas | `PUBLIC_ROUTES` — login, recuperação/reset de senha, termos. |
| Refresh proativo | Renova o access token **antes** de expirar (`isAccessExpiringSoon`), aproveitando o `Refresh-Token` (rotation no backend). |
| Redirect sem token | Rota privada sem sessão → `/sign-in`. |
| Redirect logado | Token válido em rota pública → `/dashboard`. |
| Sessão inválida | Refresh falhou → limpa cookies e redireciona ao login. |

> **Rotation:** cada refresh emite um novo par e revoga o anterior no backend ([`backend/docs/07`](../../backend/docs/07-auth-permissoes.md) §4.4). O `proxy.ts` só repassa o refresh atual e grava o novo par retornado.

---

## 4. Sessão: fonte única no servidor

`getServerSession()` decodifica o JWT do cookie e devolve o `Session` tipado. Envolto em `cache` do React, executa **uma vez por request** mesmo se vários Server Components o chamarem.

```typescript
// src/shared/session/get-server-session.ts
import { cache } from 'react';
import { getAccessToken } from '@/shared/cookies';
import { decodeJwt } from '@/shared/jwt';

export interface Session {
  userId: string;
  empresaId: string;
  saasId: string;     // marca white-label
  grupoId: number;    // 1..10
  name: string;
  permissoes: Array<{ actionId: string; nivel: number }>; // ACL do login (ver §6)
}

export const getServerSession = cache(async (): Promise<Session | null> => {
  const token = await getAccessToken();
  if (!token) return null;
  const payload = decodeJwt(token);
  if (!payload) return null;
  return mapToSession(payload);
});
```

A sessão é passada ao client **uma única vez** via prop de RSC ou contexto hidratado no layout — nunca refeita no browser e **nunca** colocada em Zustand:

```tsx
// (modules)/layout.tsx (Server Component)
import { getServerSession } from '@/shared/session';
import { SessionProvider } from '@/providers';

export default async function ModulesLayout({ children }: { children: React.ReactNode }) {
  const session = await getServerSession(); // proxy.ts já garantiu autenticação
  return <SessionProvider value={session}>{children}</SessionProvider>;
}
```

> Componentes client leem a sessão por `useSession()` (contexto). Esse contexto é **read-only** e hidratado uma vez — não há escrita de JWT no client.

---

## 5. Fluxos de autenticação

Endpoints sob `/api/auth/*` no backend (ver spec parte 3.6 e [`backend/docs/07`](../../backend/docs/07-auth-permissoes.md) §4). No frontend, login/logout são **Server Actions** que gravam/limpam cookies httpOnly.

### 5.1 Login por formulário (multi-tenant)

```typescript
// (auth)/_business/actions/sign-in/index.ts
import { api } from '@/infra';
import type { SignInProps, SignInResponse } from './interfaces';

export async function signIn(props: SignInProps) {
  return api.post<SignInResponse>('/auth/login', props); // { username, password, tenant? }
}
```

```typescript
// (auth)/_business/hooks/use-form-sign-in/index.ts ('use client')
export function useFormSignIn() {
  const router = useRouter();
  const form = useForm<SignInSchemaProps>({ resolver: zodResolver(signInSchema) });

  async function onSubmit(data: SignInSchemaProps) {
    try {
      const { data: result } = await signInAndPersist(data); // Server Action: signIn + setSessionCookies
      if (result.needsEula) return router.push('/termos-eula'); // grupo 2 sem EULA
      router.replace('/dashboard');
    } catch (error) {
      toast.error(resolveErrorMessage(error)); // 401 → INVALID_CREDENTIALS (mensagem genérica)
    }
  }
  return { form, onSubmit: form.handleSubmit(onSubmit) };
}
```

A resposta do `/api/auth/login` traz `user`, `accessToken`, `refreshToken`, `expiresIn`, `empresa`, `permissoes`, `theme` e `needsEula` (espelha [`backend/docs/07`](../../backend/docs/07-auth-permissoes.md) §4.1). A Server Action grava os tokens (`setSessionCookies`) e o tema é aplicado por host/`saasId` ([`./06`](./06-multitenancy-whitelabel.md)).

### 5.2 Recuperação de senha

| Passo | Endpoint | Tela |
|-------|----------|------|
| Solicitar | `POST /api/auth/recover` (e-mail/username) | `/recover-account` |
| Redefinir | `POST /api/auth/reset-password` (token + nova senha) | `/reset-password?token=...` |

Mensagens **genéricas** (anti-enumeração): "Se o cadastro existir, enviaremos as instruções." Erros tratados por `error.code`, nunca por texto.

### 5.3 Logout, EULA e refresh

| Fluxo | Ação | Efeito no frontend |
|-------|------|--------------------|
| Logout | `POST /api/auth/logout` → `clearSessionCookies()` | Revoga refresh; limpa cookies; redireciona a `/sign-in`. |
| Aceitar EULA | `POST /api/auth/aceitar-eula` | Libera grupo **2** após primeiro login (`needsEula=false`). |
| Refresh | `POST /api/auth/refresh` (rotation) | Disparado pelo `proxy.ts` (§3); cliente não chama direto. |

---

## 6. RBAC — os 10 grupos

Mantidos **idênticos ao legado** (spec parte 3.1). O `grupoId` (1..10) vem na claim do JWT e na sessão. Centralizados em `RolesPermissionEnum`.

| grupoId | Papel | Dashboard |
|---------|-------|-----------|
| 1 | SuperAdmin | Operacional completo |
| 2 | Admin (Imobiliária master) — exige EULA | Operacional |
| 3 | Operacional Senior | Operacional |
| 4 | Operacional | Operacional |
| 5 | Integrador (API partner) | API dashboard |
| 6 | Usuário Master Imobiliária Parceira | Indexadmin |
| 7 | Corretor | Indexadmin |
| 8 | Assistente | Indexadmin simulado |
| 9 | Operacional/Admin Análise | Operacional |
| 10 | Parceiro Variante | Indexadmin |

```typescript
// src/shared/permissions/roles.enum.ts
export enum RolesPermissionEnum {
  SuperAdmin = 1,
  AdminImobiliaria = 2,
  OperacionalSenior = 3,
  Operacional = 4,
  Integrador = 5,
  MasterParceira = 6,
  Corretor = 7,
  Assistente = 8,
  AdminAnalise = 9,
  ParceiroVariante = 10,
}
```

---

## 7. ACL granular (permissions / modules)

Além do papel (grupo), o weCorp tem ACL fina herdada do legado. Hierarquia (spec 3.3–3.5 e [`backend/docs/07`](../../backend/docs/07-auth-permissoes.md) §3):

```text
grupos (1..10)
   └──< permissiongroups (pivot) >── modulecontrolleractions
                │  nivel (1..3)            │
                │                          └── modulecontrollers ── modules (menu)
                └── permission_id ──> permissions (features_id legacy)
```

| Modelo | Papel | Enum no frontend |
|--------|-------|------------------|
| `Module` | Item de menu / agrupador de alto nível | `ModulesPermissionEnum` |
| `Modulecontrolleraction` | Ação concreta (`controller/action`, ex.: `empresas/edit`) | `ActionsPermissionEnum` |
| `Permissiongroup` (pivot) | Liga grupo → ação com `nivel` (1..3) | — (vem em `session.permissoes`) |

O `nivel` (1=leitura, 2=edição/criar, 3=total) é o que o `<PermissionGate>` compara. O backend devolve o mapa `permissoes: [{ actionId, nivel }]` no login e em `me` — o frontend só consome.

```typescript
// src/shared/permissions/modules.enum.ts
export enum ModulesPermissionEnum {
  Assessoria = 1,
  Admin = 2,
  Relatorios = 3,
  Seguros = 5,
  Vistorias = 6,
  Downloads = 9,
  Financeiro = 11,
  SignVariant = 12,
  AssinaturaEletronica = 13,
  Marketplace = 14,
}

// src/shared/permissions/actions.enum.ts — actionId no formato controller/action
export enum ActionsPermissionEnum {
  EmpresasView = 'empresas/view',
  EmpresasEdit = 'empresas/edit',
  AnalisesView = 'analises/view',
  AnalisesExecutar = 'analises/executa',
  // ...
}

export enum LevelPermissionEnum {
  Read = 1,
  Write = 2,
  Full = 3,
}
```

---

## 8. `hasAccess` — função pura (testes obrigatórios)

`hasAccess(session, requirement)` é a **única** regra de decisão de permissão no frontend. Pura (sem I/O), determinística e com **cobertura total de testes** ([`./13-testes.md`](./13-testes.md)).

```typescript
// src/shared/permissions/has-access.ts
import type { Session } from '@/shared/session';
import { RolesPermissionEnum, LevelPermissionEnum } from '@/shared/permissions';

export interface PermissionRequirement {
  roles?: RolesPermissionEnum[];        // qualquer um destes grupos
  action?: string;                       // ActionsPermissionEnum
  level?: LevelPermissionEnum;           // nivel mínimo (default Read)
}

export function hasAccess(session: Session | null, req: PermissionRequirement): boolean {
  if (!session) return false;

  // 1) por grupo (RBAC)
  if (req.roles?.length && !req.roles.includes(session.grupoId)) return false;

  // 2) por ação + nível (ACL fina)
  if (req.action) {
    const minLevel = req.level ?? LevelPermissionEnum.Read;
    const granted = session.permissoes.find((p) => p.actionId === req.action);
    if (!granted || granted.nivel < minLevel) return false;
  }

  return true;
}
```

Branches que os testes **DEVEM** cobrir: sessão nula; grupo permitido / negado; sem `roles` mas com `action`; ação ausente; ação presente com `nivel` insuficiente / suficiente; combinação roles+action.

---

## 9. Permission gates

Mesma regra (`hasAccess`) em dois invólucros — servidor e client. Nunca chamar `hasAccess` com strings cruas: sempre os enums.

### 9.1 `<PermissionGateServer>` (RSC)

Para esconder seções/rotas no servidor (sem enviar HTML protegido ao client).

```tsx
// src/components/permission/PermissionGateServer.tsx (Server Component)
import { getServerSession } from '@/shared/session';
import { hasAccess, type PermissionRequirement } from '@/shared/permissions';

export async function PermissionGateServer({
  requirement, children, fallback = null,
}: { requirement: PermissionRequirement; children: React.ReactNode; fallback?: React.ReactNode }) {
  const session = await getServerSession();
  return hasAccess(session, requirement) ? <>{children}</> : <>{fallback}</>;
}
```

```tsx
// uso em page.tsx (Server)
<PermissionGateServer requirement={{ roles: [RolesPermissionEnum.SuperAdmin, RolesPermissionEnum.AdminImobiliaria] }}>
  <NovaEmpresaButton />
</PermissionGateServer>
```

### 9.2 `<PermissionGateClient>` + `usePermission()`

Para casos dinâmicos no client (mostrar/ocultar botão num dialog, item de menu).

```tsx
// src/components/permission/PermissionGateClient.tsx ('use client')
'use client';
import { useSession } from '@/providers';
import { hasAccess, type PermissionRequirement } from '@/shared/permissions';

export function PermissionGateClient({
  requirement, children, fallback = null,
}: { requirement: PermissionRequirement; children: React.ReactNode; fallback?: React.ReactNode }) {
  const session = useSession();
  return hasAccess(session, requirement) ? <>{children}</> : <>{fallback}</>;
}
```

```tsx
// src/shared/permissions/use-permission.ts ('use client')
'use client';
import { useSession } from '@/providers';
import { hasAccess, type PermissionRequirement } from '@/shared/permissions';

export function usePermission() {
  const session = useSession();
  return (requirement: PermissionRequirement) => hasAccess(session, requirement);
}
```

```tsx
// uso em componente client
const can = usePermission();

<If condition={can({ action: ActionsPermissionEnum.EmpresasEdit, level: LevelPermissionEnum.Write })}>
  <Button onClick={openEdit}>Editar</Button>
</If>
```

> Gate é **conveniência de UX**, não segurança: o backend revalida toda ação. Para rotas inteiras, prefira o gate **no servidor** (não envia markup protegido ao browser).

---

## 10. Anti-padrões

| Evitar | Usar |
|--------|------|
| Token em `localStorage`/`document.cookie` | Cookie httpOnly via `src/shared/cookies/` |
| Espelhar JWT/sessão no Zustand | `getServerSession()` + contexto read-only |
| `getServerSession()` chamado várias vezes sem `cache` | `cache(...)` do React (uma vez por request) |
| `if (grupoId === 1)` espalhado | `hasAccess` + `RolesPermissionEnum` |
| `hasAccess(session, { action: 'empresas/edit' })` | `ActionsPermissionEnum.EmpresasEdit` |
| Confiar só no gate client p/ proteger dados | Gate **server** + backend revalida |
| Decidir auth por texto da mensagem de erro | `error.code` (`INVALID_CREDENTIALS`/`UNAUTHENTICATED`/`FORBIDDEN`) |

---

## 11. Ver também

- [`../CLAUDE.md`](../CLAUDE.md) §5–§6 — princípios de sessão e permissões.
- [`./06-multitenancy-whitelabel.md`](./06-multitenancy-whitelabel.md) — detecção de marca no `proxy.ts`, tema por `saasId`.
- [`./08-consumo-api-dados.md`](./08-consumo-api-dados.md) — `api` resolve o token (cookie) no servidor; envelope e códigos de erro.
- [`./12-estado-dados.md`](./12-estado-dados.md) — por que sessão **não** vai para Zustand.
- [`./13-testes.md`](./13-testes.md) — cobertura obrigatória de `hasAccess`.
- Backend: [`backend/docs/07`](../../backend/docs/07-auth-permissoes.md) (JWT/claims/RBAC/ACL) · [`backend/docs/08`](../../backend/docs/08-contrato-api.md) (envelope, headers).
