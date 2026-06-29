# 06 — Multi-tenancy e White-label

> O frontend weCorp serve **8 marcas** a partir do mesmo código, cada uma de um domínio próprio, com identidade visual customizada — **sem rebuild**. Várias empresas operam na mesma aplicação; o **isolamento de dados é garantido pelo backend**, e o frontend apenas **reflete** os dados e permissões do usuário logado.
> Regras não-negociáveis em [`../CLAUDE.md`](../CLAUDE.md) §5. Escopo de tenant no backend em [`../../backend/docs/06-multi-tenancy.md`](../../backend/docs/06-multi-tenancy.md).

---

## 1. Dois conceitos distintos

| Conceito | Pergunta que responde | Onde mora |
|----------|-----------------------|-----------|
| **Multi-tenancy** | *Quais dados este usuário pode ver?* | **Backend** (escopo por `empresaId`/`saasId`). O frontend só consome o que a API devolve. |
| **White-label** | *Com que cara (logo, cor, fonte) a aplicação aparece?* | **Frontend** (marca detectada por host → variáveis CSS). |

> Princípio não-negociável: o frontend **nunca** reimplementa o isolamento de tenant. Ele não decide "filtrar dados da empresa X" — quem faz isso é o backend, com filtro de escopo explícito em todo `where`. O frontend confia no contrato e exibe o que recebe (ver `backend/docs/06`). Vazar dado entre tenants é responsabilidade do servidor; o frontend não pode "consertar" nem "burlar" isso.

---

## 2. Multi-tenancy do lado do frontend

O weCorp é um SaaS B2B multi-tenant: muitas empresas (imobiliárias, corretoras, administradoras, seguradoras) usam a mesma instância. O que o frontend faz:

- **Resolve a sessão no servidor** (cookies httpOnly + `getServerSession()`), obtendo `user`, `empresa`, `grupo_id` e `permissoes`.
- **Reflete** dados e ações conforme essa sessão: listagens trazem só o escopo do usuário (porque o backend escopou), e o menu/ações são filtrados pela ACL dos **10 grupos** (`PermissionGate` + `hasAccess`). Ver [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md).
- **Não** envia `empresa_id`/`saas_id` no body/query para "escolher" tenant — o escopo vem **sempre** do token, no servidor.

> Relação com o backend: o token carrega `{ userId, empresaId, saasId, grupoId }`; o backend aplica defesa em profundidade (service explícito + AsyncLocalStorage + extensão Prisma). Cross-tenant de leitura no backend retorna **404** (anti-enumeração). O frontend só precisa tratar o envelope `{data}`/`{data,meta}`/`{error}`. Detalhes: [`../../backend/docs/06-multi-tenancy.md`](../../backend/docs/06-multi-tenancy.md).

---

## 3. White-label: detecção da marca por host

A marca é detectada pelo **HOST** da requisição no `proxy.ts` (o middleware do Next 16 — substitui `middleware.ts`). O proxy roda na borda, identifica o tenant e o **propaga** via cookie/header para os layouts/Server Components.

```typescript
// src/proxy.ts (trecho — detecção de marca por host)
import { NextRequest, NextResponse } from 'next/server';
import { detectTenantByHost } from '@/configs/tenants';

export function proxy(req: NextRequest) {
  const tenant = detectTenantByHost(req.headers.get('host')); // → { id, theme, ... }

  const res = NextResponse.next();
  res.headers.set('x-tenant', tenant.id);                     // disponível ao layout
  res.cookies.set('tenant', tenant.id, { sameSite: 'lax', path: '/' });
  return res;
}
```

```typescript
// src/configs/tenants/index.ts (esboço)
export function detectTenantByHost(host: string | null): Tenant {
  const h = (host ?? '').toLowerCase();
  return tenants.find((t) => t.domains.some((d) => h.endsWith(d))) ?? tenants.localhost;
}
```

> A detecção por host é o caso normal. Há ainda **override por `empresa_id` do usuário logado**: se a empresa tiver `url_logotipo`/tema próprio (campo do modelo `Empresa`), o branding da empresa prevalece sobre o branding genérico da marca após o login. A marca define o *default*; a empresa pode refinar.

---

## 4. Aplicação do tema por variáveis CSS

A identidade visual é aplicada **exclusivamente por variáveis CSS** no `<html>` — **nunca** por hex hardcoded em componentes. Trocar de marca = trocar os valores das variáveis; o código de UI não muda. É isso que viabiliza **trocar de tenant sem rebuild**.

Tokens de marca:

| Variável | Papel |
|----------|-------|
| `--color-primary` | Cor primária da marca. |
| `--color-primary-foreground` | Cor do texto/ícone sobre a primária. |
| `--logo-url` | URL do logotipo (usado em `Sidebar`/`Topbar`/portal). |
| `--brand-name` | Nome exibível da marca. |
| `--radius` | Raio de borda padrão (shadcn). |
| `--font-sans` | Família de fonte da marca. |

### 4.1 `globals.css` — valores default

```css
/* app/globals.css */
@import 'tailwindcss';
@import 'tw-animate-css';

:root {
  --color-primary: 220 90% 50%;          /* default weCorp */
  --color-primary-foreground: 0 0% 100%;
  --color-secondary: 220 14% 96%;
  --radius: 0.5rem;
  --font-sans: 'Inter', system-ui, sans-serif;
  --logo-url: url('/logos/wecorp.svg');
  --brand-name: 'weCorp';
}
```

### 4.2 Aplicação no `<html>` via TenantProvider

O layout autenticado (e o público) injeta os tokens do tenant resolvido como `style` inline no elemento raiz. Como são variáveis CSS, todo o shadcn/Tailwind passa a usar a cor da marca automaticamente.

```tsx
// src/providers/tenant-provider.tsx ('use client')
'use client';
import { createContext, useContext } from 'react';

const TenantContext = createContext<Tenant | null>(null);
export const useTenant = () => useContext(TenantContext);

export function TenantProvider({ tenant, children }: { tenant: Tenant; children: React.ReactNode }) {
  return (
    <TenantContext.Provider value={tenant}>
      <div
        style={{
          ['--color-primary' as string]: tenant.theme.primary,
          ['--logo-url' as string]: `url('${tenant.theme.logo}')`,
          ['--brand-name' as string]: `'${tenant.name}'`,
          ['--radius' as string]: tenant.theme.radius,
          ['--font-sans' as string]: tenant.theme.fontSans,
        }}
      >
        {children}
      </div>
    </TenantContext.Provider>
  );
}
```

> `TenantContext` é provido no layout autenticado (e no layout público), uma única vez. Componentes que precisam do nome/logo da marca usam `useTenant()`; componentes que só precisam da cor usam as variáveis CSS diretamente (`bg-primary`, `text-primary-foreground` etc.). **Regra:** cor **sempre** via variável CSS — nunca hex inline.

---

## 5. As 8 marcas (+ localhost dev)

Mapa de marca → domínio → logo → cor primária. Fonte: `frontend.md` Parte 1.5.

| Marca | Domínio | Logo | Cor primária |
|-------|---------|------|--------------|
| weCorp | `*.wecorp.com.br` | `/logos/wecorp.svg` | `#1e40af` |
| Hubstate | `*.hubstate.com.br` | `/logos/hubstate.svg` | `#0f766e` |
| AlugueSeguro | `*.alugueseguro.com.br` | `/logos/alugueseguro.svg` | `#0ea5e9` |
| HubbPro | `*.hubbpro.com.br` | `/logos/hubbpro.svg` | `#7c3aed` |
| Buskaza | `*.buskaza.com.br` | `/logos/buskaza.svg` | `#f59e0b` |
| Completto | `*.completto.com.br` | `/logos/completto.svg` | `#dc2626` |
| EsteiraTech | `*.esteiratech.com.br` | `/logos/esteira.svg` | `#16a34a` |
| Upgrade Garantia | `*.upgradegarantia.com.br` | `/logos/upgrade.svg` | `#ea580c` |
| Localhost (dev) | `localhost:*` | `/logos/hubstate.svg` | `#0f766e` |

> As 8 marcas correspondem aos `saasId` que particionam o universo white-label no backend (ver `backend/docs/06` §1). `localhost` é apenas conveniência de desenvolvimento e cai no tema da Hubstate.

---

## 6. Troca de tenant sem rebuild

A independência de marca (objetivo nº 5 do produto) vem de duas propriedades combinadas:

1. **Mesmo bundle, vários hosts.** O `proxy.ts` resolve o tenant em runtime por host; não há build por marca.
2. **Tema 100% em variáveis CSS.** Mudar logo/cor/fonte é mudar valores de variáveis injetadas no `<html>` — nenhum componente referencia uma marca específica.

Consequências práticas:

- Adicionar uma marca = adicionar uma entrada em `src/configs/tenants/` + assets em `public/logos/`. Sem alterar componentes.
- `next.config.ts` → `images.remotePatterns` deve cobrir os domínios reais de produção das marcas, se as logos vierem de URL remota.
- Logos locais ficam em `public/logos/` e são referenciadas por `--logo-url`.

---

## 7. Armadilhas comuns

| Armadilha | Por que falha | Correção |
|-----------|---------------|----------|
| Hex de cor inline no componente | Quebra white-label; a marca não troca a cor. | Usar `bg-primary`/variável CSS. |
| Confiar no frontend para "filtrar" tenant | O isolamento é do backend; filtro no client é burlável. | Consumir o escopo que a API já devolve. |
| Enviar `empresa_id`/`saas_id` no body para escolher tenant | Cliente forja escopo. | Escopo vem do token, no servidor. |
| Espelhar sessão/JWT no Zustand | Vaza dado sensível ao client; duplica fonte de verdade. | `getServerSession()` no servidor; Zustand só UI. |
| Detectar marca no client (`window.location`) | Tarde demais (flash) e fora do `proxy.ts`. | Detectar no `proxy.ts` por host; injetar no layout. |
| Hardcodar logo da weCorp | Quebra nas outras 7 marcas. | `--logo-url` via TenantProvider. |

---

## 8. Ver também

- [01 — Arquitetura](./01-arquitetura.md) · [02 — Stack](./02-stack-tecnologico.md)
- [05 — Roteamento e navegação](./05-roteamento-navegacao.md) · [07 — Auth, sessão e permissões](./07-auth-sessao-permissoes.md)
- [04 — Design system e UI](./04-design-system-ui.md) · [08 — Consumo de API e dados](./08-consumo-api-dados.md)
- Backend: [`../../backend/docs/06-multi-tenancy.md`](../../backend/docs/06-multi-tenancy.md)
</content>
