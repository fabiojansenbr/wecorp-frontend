# 04 — Design System e UI

> Como construímos a interface do **frontend weCorp**: shadcn/ui (new-york), Tailwind 4 com tokens via variáveis CSS, dark mode, convenções de UI, providers globais, **mobile-first** e **acessibilidade WCAG 2.1 AA**.
> Regras não-negociáveis em [`../CLAUDE.md`](../CLAUDE.md) §7. Convenções de código em [03 — Convenções](./03-convencoes-codigo.md); white-label (a marca troca os tokens) em [06 — Multi-tenancy e white-label](./06-multitenancy-whitelabel.md).

---

## 1. shadcn/ui (new-york)

Todos os componentes visuais vêm do **shadcn/ui**, estilo **new-york** (Radix UI + CVA + `cn()`). O objetivo do produto é **100% dos componentes via shadcn/ui** ([00 — Visão geral](./00-visao-geral.md), objetivo 4) — não criamos componentes visuais do zero quando há equivalente shadcn.

- Componentes ficam em `src/components/` e são reexportados por um **barrel** `index.ts`. Fora de `components/`, importa-se sempre de `@/components`.
- shadcn gera componentes **RSC-compatíveis**; só recebem `"use client"` quando têm interatividade (ex.: `Dialog`, `Select`, `Tabs`).
- Componentes custom do produto (ex.: `<If>`, `<DataTable>`, `<Card>` de página) convivem no mesmo barrel.

```jsonc
// components.json — configuração do shadcn
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "src/app/globals.css",
    "baseColor": "slate",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "utils": "@/lib/utils"
  },
  "iconLibrary": "lucide-react"
}
```

```typescript
// src/lib/utils.ts — função cn() (clsx + tailwind-merge)
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

> `cn()` é o **único** mecanismo para compor classes condicionais. Nunca concatene strings de classe à mão.

---

## 2. Tailwind CSS 4

Tailwind 4 dispensa o `tailwind.config.js` para tokens: tudo vive no CSS via `@import 'tailwindcss'`, `@theme` e variáveis CSS em `:root`/`.dark`.

```css
/* src/app/globals.css */
@import 'tailwindcss';
@import 'tw-animate-css';

/* Mapeia tokens (variáveis CSS) para utilitários Tailwind (bg-primary, text-foreground...) */
@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);
  --color-secondary: var(--secondary);
  --color-muted: var(--muted);
  --color-destructive: var(--destructive);
  --color-border: var(--border);
  --radius: var(--radius);
  --font-sans: var(--font-sans);
}

:root {
  --background: oklch(1 0 0);
  --foreground: oklch(0.15 0 0);
  --primary: oklch(0.55 0.2 255);
  --primary-foreground: oklch(0.99 0 0);
  --secondary: oklch(0.96 0.01 255);
  --muted: oklch(0.96 0.01 255);
  --destructive: oklch(0.58 0.24 27);
  --border: oklch(0.92 0 0);
  --radius: 0.5rem;
}

.dark {
  --background: oklch(0.15 0 0);
  --foreground: oklch(0.98 0 0);
  --primary: oklch(0.62 0.19 255);
  --primary-foreground: oklch(0.15 0 0);
  --secondary: oklch(0.25 0.02 255);
  --muted: oklch(0.25 0.02 255);
  --border: oklch(0.3 0 0);
}
```

**Cores SEMPRE via variáveis CSS / tokens** — **MUST NOT** usar hex inline em componentes:

```tsx
// ✅ token Tailwind ligado à variável CSS
<div className="bg-primary text-primary-foreground rounded-[--radius]">...</div>

// ❌ hex inline — quebra o dark mode e o white-label
<div style={{ backgroundColor: '#2563eb' }}>...</div>
<div className="bg-[#2563eb]">...</div>
```

> Por que isso importa: o **white-label** troca os valores das variáveis por host **sem rebuild** (a marca redefine `--primary`, `--radius`, `--font-sans`, `--logo-url`). Qualquer hex inline escapa do tema e quebra a marca. Ver [06 — Multi-tenancy e white-label](./06-multitenancy-whitelabel.md).

---

## 3. Dark mode (next-themes)

O tema claro/escuro é controlado pelo **next-themes** via classe `.dark` no `<html>`. O `ThemeProvider` usa `attribute="class"`; os tokens de `.dark` (§2) fazem o resto.

```tsx
// uso: alternador de tema (client)
'use client';

import { useTheme } from 'next-themes';
import { Button } from '@/components';
import { Moon, Sun } from 'lucide-react';

export function ThemeToggle() {
  const { setTheme, resolvedTheme } = useTheme();
  return (
    <Button
      variant="ghost"
      size="icon"
      aria-label="Alternar tema"
      onClick={() => setTheme(resolvedTheme === 'dark' ? 'light' : 'dark')}
    >
      <Sun className="hidden dark:block" />
      <Moon className="block dark:hidden" />
    </Button>
  );
}
```

> Dark mode e white-label são ortogonais: a marca define os tokens base; o dark mode os sobrescreve sob `.dark`. Ambos operam pelas mesmas variáveis CSS.

---

## 4. Convenções de UI

| Regra | Padrão | Por quê |
|-------|--------|---------|
| Página | Conteúdo envolvido em `<Card>` | Consistência de layout entre telas |
| Condicional no JSX | `<If condition={} elseRender={}>` | Legibilidade; **MUST NOT** `&&`/ternário inline |
| Filtros de busca | **Um input por critério** | Clareza; cada filtro mapeia a um parâmetro |
| Listagens | TanStack Table (`<DataTable>`) + paginação | Padrão único server-side ([11](./11-tabelas-listagens.md)) |
| Filtros na URL | Hook `useUrlSyncedFilters` | Estado compartilhável/recarregável |
| Feedback de ação | `sonner` (toast) | Não-bloqueante e acessível |
| Após mutação server | `revalidatePath('/rota')` no hook | Reflete o novo estado sem refetch manual |

### 4.1 Página em `<Card>`

```tsx
// (web)/(pages)/analises/page.tsx (Server Component)
import { Card } from '@/components';

export default async function AnalisesPage() {
  return (
    <Card title="Análises Cadastrais">
      {/* filtros, tabela com Suspense, ações */}
    </Card>
  );
}
```

### 4.2 Condicional com `<If>` (não `&&`/ternário)

O componente `<If>` torna o JSX declarativo e evita os bugs clássicos de `0 && ...` ou ternários aninhados.

```tsx
import { If } from '@/components';

// ✅
<If condition={isLoading} elseRender={<ResourceTable data={data} />}>
  <TableFallback />
</If>

// ❌ MUST NOT
{isLoading ? <TableFallback /> : <ResourceTable data={data} />}
{!isLoading && <ResourceTable data={data} />}
```

### 4.3 Feedback com sonner

```tsx
import { toast } from '@/shared';

toast.success('Cadastro realizado com sucesso');
toast.error('Não foi possível concluir, tente novamente ou contate o suporte.');
```

> O `toast` é chamado **no hook**, nunca na UI ou na action ([03 — Convenções](./03-convencoes-codigo.md) §3).

---

## 5. Providers globais (`src/providers/`)

Os providers client globais ficam em `src/providers/` e são montados **uma vez** no root layout: tema, cache do TanStack Query e o `<Toaster>` do sonner.

```tsx
// src/providers/index.tsx
'use client';

import { ThemeProvider } from 'next-themes';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { Toaster } from 'sonner';
import { useState } from 'react';

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () => new QueryClient({ defaultOptions: { queries: { staleTime: 30_000 } } }),
  );

  return (
    <ThemeProvider attribute="class" defaultTheme="light" enableSystem>
      <QueryClientProvider client={queryClient}>
        {children}
        <Toaster position="bottom-right" richColors closeButton />
      </QueryClientProvider>
    </ThemeProvider>
  );
}
```

```tsx
// src/app/layout.tsx (root, Server Component)
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

> `lang="pt-BR"` é requisito de acessibilidade (§7). `suppressHydrationWarning` é necessário porque o next-themes ajusta a classe do `<html>` no client. A sessão **não** é hidratada aqui — vem do servidor por prop/contexto ([07](./07-auth-sessao-permissoes.md)); o Zustand não espelha JWT.

---

## 6. Mobile-first

Layouts são pensados **primeiro para telas pequenas** e expandidos para desktop com os breakpoints do Tailwind. Estilo base = mobile; prefixos (`sm:`, `md:`, `lg:`) **adicionam** complexidade para telas maiores.

| Breakpoint | Largura mínima | Uso típico |
|------------|----------------|------------|
| (base) | 0 | Mobile (coluna única, ações em sheet) |
| `sm` | 640px | Mobile landscape / tablets pequenos |
| `md` | 768px | Tablet (sidebar colapsável aparece) |
| `lg` | 1024px | Desktop (sidebar fixa, multi-coluna) |
| `xl` | 1280px | Desktop largo |

Padrões obrigatórios:

- **Layout primeiro para telas pequenas**, depois prefixos para ampliar.
- **Sidebar colapsável**: drawer/sheet no mobile; fixa a partir de `lg`.
- **Tabelas → cards no mobile**: em telas estreitas, a listagem vira lista de cards (uma linha = um card) em vez de scroll horizontal.

```tsx
// grid responsivo: 1 coluna no mobile, 2 no tablet, 3 no desktop
<div className="grid grid-cols-1 gap-4 md:grid-cols-2 lg:grid-cols-3">...</div>

// tabela só a partir de md; cards abaixo disso
<div className="hidden md:block">
  <DataTable columns={columns} data={data} />
</div>
<div className="space-y-3 md:hidden">
  {data.map((row) => (
    <ResourceCard key={row.id} resource={row} />
  ))}
</div>
```

---

## 7. Acessibilidade (WCAG 2.1 AA)

Acessibilidade **AA é obrigatória em todos os fluxos** ([CLAUDE.md](../CLAUDE.md) §7, objetivo 3). O uso de Radix (via shadcn) já entrega grande parte de teclado/ARIA, mas há responsabilidades nossas:

| Critério | Como atendemos |
|----------|----------------|
| **Labels** | Todo input tem `<label>` associado (`htmlFor`/`id`); ícone-botão usa `aria-label` |
| **Foco visível** | Manter o anel de foco do shadcn (`focus-visible:ring`); nunca `outline: none` sem alternativa |
| **Contraste** | Tokens de cor com contraste ≥ 4.5:1 (texto) / 3:1 (UI); validar em claro e escuro |
| **Navegação por teclado** | Tudo operável sem mouse; ordem de tab lógica; `Esc` fecha dialogs |
| **ARIA via Radix** | `Dialog`, `Select`, `Tabs`, `Tooltip` do shadcn já trazem roles/`aria-*` corretos — não reescrever à mão |
| **Idioma** | `<html lang="pt-BR">` |
| **Estados** | Loading/empty/error anunciados (ex.: `aria-busy`, `role="alert"` em erros) |
| **Imagens** | `alt` descritivo; ícones decorativos com `aria-hidden` |

```tsx
// input com label associado + erro acessível
<label htmlFor="email">E-mail</label>
<input id="email" type="email" aria-invalid={!!error} aria-describedby="email-error" />
<If condition={!!error}>
  <p id="email-error" role="alert" className="text-destructive text-sm">
    {error}
  </p>
</If>
```

> Preferir componentes Radix/shadcn a `div` com `onClick`: um `<Button>`/`<DialogTrigger>` já é focável, anunciável e operável por teclado. Detalhes de validação acessível de formulários em [10 — Formulários e validação](./10-formularios-validacao.md).

## Ver também

- [03 — Convenções de código](./03-convencoes-codigo.md) · [02 — Stack tecnológico](./02-stack-tecnologico.md)
- [06 — Multi-tenancy e white-label](./06-multitenancy-whitelabel.md) · [10 — Formulários e validação](./10-formularios-validacao.md)
- [11 — Tabelas e listagens](./11-tabelas-listagens.md) · [CLAUDE.md](../CLAUDE.md)
</content>
