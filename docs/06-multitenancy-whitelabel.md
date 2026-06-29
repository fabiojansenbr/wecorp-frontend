# 06 — Multi-tenancy e White-label

> O frontend weCorp serve **muitos clientes a partir do mesmo código**, cada um podendo ter **domínio próprio e marca exclusiva** — com logo, cores e particularidades **já na tela de login** (antes de autenticar), **sem rebuild**. Várias empresas operam na mesma aplicação; o **isolamento de dados é garantido pelo backend**, e o frontend apenas **reflete** os dados e permissões do usuário logado.
> Regras não-negociáveis em [`../CLAUDE.md`](../CLAUDE.md) §5. Escopo de tenant no backend em [`../../backend/docs/06-multi-tenancy.md`](../../backend/docs/06-multi-tenancy.md).

---

## 1. Dois conceitos distintos

| Conceito | Pergunta que responde | Onde mora |
|----------|-----------------------|-----------|
| **Multi-tenancy** | *Quais dados este usuário pode ver?* | **Backend** (escopo por `empresaId`/`saasId`). O frontend só consome o que a API devolve. |
| **White-label** | *Com que cara (logo, cor, fonte, domínio) a aplicação aparece?* | **Frontend** (marca resolvida por host → variáveis CSS), com os dados de marca vindos do **backend**. |

> Princípio não-negociável: o frontend **nunca** reimplementa o isolamento de tenant. Ele não decide "filtrar dados da empresa X" — quem faz isso é o backend, com filtro de escopo explícito em todo `where`. A resolução de marca por host é **apresentação**, não segurança: trocar o host muda a *cara*, nunca o *escopo de dados* (que vem sempre do token). Ver `backend/docs/06`.

---

## 2. Premissa: cada cliente com aplicação exclusiva

Requisito de produto inegociável: ao acessar **`www.cliente1.com.br/app`**, o usuário vê — **já no login** — o logo, as cores e as particularidades do cliente 1; em **`www.cliente2.com.br/app`**, as do cliente 2. Isso implica três propriedades:

1. **Branding por domínio próprio, por cliente** (não só pelas marcas SaaS) — resolvido de forma **data-driven** (qualquer empresa pode ter o seu).
2. **Branding pré-autenticação** — a marca aparece **antes** de existir sessão (na tela de login), resolvida só pelo **host**.
3. **Sem rebuild** — adicionar um cliente é cadastrar um domínio + apontar DNS; nenhuma alteração de código.

---

## 3. Resolução de tenant por host (data-driven)

A marca é resolvida pelo **HOST** da requisição, em duas etapas (borda + servidor), nesta **ordem de precedência**:

1. **Domínio próprio do cliente** — `host == Empresa.dominio_corporativo` (match exato, índice único no backend) → branding **da empresa** (cliente exclusivo). É o caso de `www.cliente1.com.br`.
2. **Marca SaaS (default)** — o sufixo do host casa uma das marcas SaaS (`*.wecorp.com.br`, `*.hubstate.com.br`, …) → branding **da marca** (§6).
3. **Fallback** — host não provisionado → marca padrão **ou** página de "domínio não configurado" (conforme a allowlist; ver §8).

> A antiga lista fixa de 8 marcas no código (`src/configs/tenants`) **não é mais a fonte única**: ela vira apenas o **seed de defaults das marcas SaaS** e o fallback de `localhost` em dev. A fonte da verdade de quem-tem-qual-domínio é o **backend** (`Empresa.dominio_corporativo`), consultado via endpoint público (§4).

```typescript
// src/proxy.ts (trecho — resolução por host, na borda)
import { NextRequest, NextResponse } from 'next/server';
import { resolveBrandingByHost } from '@/configs/tenants'; // consulta o endpoint público com cache TTL

export async function proxy(req: NextRequest) {
  const host = req.headers.get('host');
  const branding = await resolveBrandingByHost(host); // { saasId, empresaId?, theme, seo, login, features }

  const res = NextResponse.next();
  res.headers.set('x-tenant-host', host ?? '');         // o layout (RSC) re-resolve no servidor
  return res;
}
```

---

## 4. Endpoint público de branding (habilita o login tematizado)

Como o login acontece **sem sessão**, o tema precisa vir de um endpoint **público** (sem auth), resolvido pelo host:

```
GET /api/public/branding        (resolve pelo header Host; aceita ?host= para dev)
```

Resposta (payload completo da "exclusividade"):

```jsonc
{
  "saasId": "…",
  "empresaId": "…|null",                 // null quando é só a marca SaaS
  "theme":  { "primary", "primaryForeground", "secondary", "radius", "fontSans", "logoUrl", "brandName" },
  "seo":    { "title", "description", "ogImage", "faviconUrl" },
  "login":  { "background", "welcome", "supportEmail", "termsUrl" },
  "features": { "seguros": true, "analises": true, "vistorias": false /* … flags por empresa */ }
}
```

- **Público e cacheável:** `Cache-Control` curto + `s-maxage` (a resposta é por host, barata e não sensível). O `proxy.ts`/layout cacheiam por TTL na borda.
- **Sem PII** e **com allowlist de hosts** (§8).
- Backend: ver [`../../backend/docs/09-modulos-dominio.md`](../../backend/docs/09-modulos-dominio.md) (módulo Empresas) — resolução `dominio_corporativo`→empresa, com fallback para os defaults da marca SaaS.

---

## 5. Branding pré-autenticação (login) **sem FOUC**

A marca **MUST** aparecer já no primeiro paint do login — nada de "piscar" o tema genérico e depois trocar. Por isso o tema é resolvido **no servidor**, não no client:

- **Layout raiz / `(auth)` / `(public)` (Server Components):** buscam o branding por host (endpoint §4, com `cache` do React) e injetam:
  - as **variáveis CSS** no `<html>` (cor, fonte, raio, logo) — todo o shadcn/Tailwind passa a usar a cor da marca;
  - **`generateMetadata`** por host → `title`, `description`, Open Graph e **favicon** com a cara do cliente.
- A **tela de login** consome `login.*` (fundo, texto de boas-vindas, e-mail de suporte, termos) e `theme.logoUrl`.
- **Proibido** detectar marca no client (`window.location`) para tematizar — chega tarde (flash) e fora do fluxo. A detecção é por host, no servidor/borda.

```tsx
// src/app/layout.tsx (RSC) — injeta tema do host já no HTML do servidor
import { getBrandingByHost } from '@/configs/tenants/server';

export async function generateMetadata() {
  const b = await getBrandingByHost();
  return { title: b.seo.title, description: b.seo.description,
           icons: { icon: b.seo.faviconUrl }, openGraph: { images: [b.seo.ogImage] } };
}

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const b = await getBrandingByHost();
  const style = {
    ['--color-primary' as string]: b.theme.primary,
    ['--logo-url' as string]: `url('${b.theme.logoUrl}')`,
    ['--brand-name' as string]: `'${b.theme.brandName}'`,
    ['--radius' as string]: b.theme.radius,
    ['--font-sans' as string]: b.theme.fontSans,
  };
  return <html lang="pt-BR" style={style}><body>{children}</body></html>;
}
```

> Após o login, o `TenantProvider` recebe o mesmo payload (agora podendo refinar com dados da empresa do usuário) e mantém logo/cores no app autenticado. A regra permanece: **cor sempre via variável CSS — nunca hex inline**.

---

## 6. Tokens de tema (variáveis CSS) e marcas SaaS (defaults)

A identidade é aplicada **exclusivamente por variáveis CSS** no `<html>`. Tokens:

| Variável | Papel |
|----------|-------|
| `--color-primary` / `--color-primary-foreground` | Cor primária da marca e do texto sobre ela. |
| `--logo-url` | URL do logotipo (login, Sidebar/Topbar, portal). |
| `--brand-name` | Nome exibível. |
| `--radius` | Raio de borda padrão (shadcn). |
| `--font-sans` | Família de fonte da marca. |

`globals.css` define apenas os **defaults** (marca weCorp); o host sobrescreve em runtime via §5.

As **marcas SaaS** (defaults por sufixo de host) seguem o spec `frontend.md` 1.5. Elas são o ponto de partida; **cada empresa cliente pode ter domínio próprio** que sobrepõe esses defaults.

| Marca SaaS | Domínio (sufixo) | Logo | Cor primária |
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

> As marcas SaaS correspondem aos `saasId` que particionam o universo white-label no backend (`backend/docs/06` §1). Um **cliente** (`empresaId`) pertence a um `saasId` e pode, opcionalmente, ter `dominio_corporativo` próprio para exclusividade total.

---

## 7. Domínio próprio do cliente: onboarding e URL

- **URL:** a aplicação roda em **`/app`** do domínio do cliente (`www.cliente1.com.br/app`) — `basePath: '/app'` no `next.config.ts`. A raiz fica livre para o site institucional do cliente. Ver [`./05-roteamento-navegacao.md`](./05-roteamento-navegacao.md) e [`./14-deploy-ambiente.md`](./14-deploy-ambiente.md).
- **Fallback imediato:** subdomínio da plataforma (`cliente1.wecorp.com.br`) com **wildcard TLS** funciona enquanto o domínio próprio é configurado/propagado.
- **Onboarding (sem rebuild):** cadastrar `Empresa.dominio_corporativo` → cliente aponta `CNAME`/`A` para a plataforma → provisionar TLS (estratégia agnóstica em [`./14-deploy-ambiente.md`](./14-deploy-ambiente.md)) → marca no ar. Nenhuma alteração de código/deploy.

---

## 8. Troca/onboarding sem rebuild + segurança

**Sem rebuild** vem de: (1) mesmo bundle para todos os hosts; (2) tema 100% em variáveis CSS; (3) mapeamento host→empresa **data-driven** (banco), não código.

| Armadilha | Por que falha | Correção |
|-----------|---------------|----------|
| Hex de cor inline no componente | Quebra white-label. | `bg-primary`/variável CSS. |
| Lista fixa de domínios no código | Cada cliente novo exigiria deploy. | Resolver por `dominio_corporativo` (banco) via endpoint público. |
| Tematizar no client (`window.location`) | Flash (FOUC) e tarde demais. | Resolver por host no servidor/borda; injetar no `<html>`. |
| Confiar no host para "filtrar" dados | Host é apresentação, não segurança. | Escopo vem do token; isolamento no backend. |
| Aceitar qualquer Host | Host header injection / branding indevido. | **Allowlist** de hosts provisionados; host desconhecido → fallback/404. |
| Cookie de sessão sem `domain` correto | Login não persiste no domínio próprio. | Cookie escopado ao host do cliente. Ver [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md). |
| Hardcodar logo/título da weCorp | Quebra nos demais clientes. | `--logo-url`/`generateMetadata` por host. |

---

## 9. Ver também

- [05 — Roteamento e navegação](./05-roteamento-navegacao.md) (basePath `/app`, metadata por host) · [07 — Auth, sessão e permissões](./07-auth-sessao-permissoes.md) (login pré-auth, cookies por domínio)
- [14 — Deploy e ambiente](./14-deploy-ambiente.md) (domínios customizados + TLS) · [04 — Design system](./04-design-system-ui.md) · [08 — Consumo de API](./08-consumo-api-dados.md)
- Backend: [`../../backend/docs/06-multi-tenancy.md`](../../backend/docs/06-multi-tenancy.md) · [`../../backend/docs/09-modulos-dominio.md`](../../backend/docs/09-modulos-dominio.md)
