# 02 — Stack Tecnológico

> O que usamos, em que versão e **por quê**. Inclui a **reconciliação** entre o stack antigo citado no spec e a base técnica moderna que prevalece.
> Constituição: [`../CLAUDE.md`](../CLAUDE.md) §3. Arquitetura: [`./01-arquitetura.md`](./01-arquitetura.md).

---

## 1. Reconciliação: spec × base moderna (LEIA PRIMEIRO)

Há **duas fontes** com pesos diferentes:

- **`C:\projetos\wecorp\frontend.md`** — fonte da verdade de **domínio, telas e contrato de API**. **Mas** o seu capítulo de stack reflete um estado **antigo** do projeto.
- **`C:\projetos\codefirst\api\frontend.md`** — **base de decisões técnicas** (stack moderna). **É ela que prevalece** para escolhas de tecnologia — exatamente como o backend adaptou `tecnologias.md`.

| Tema | Spec antigo (`wecorp/frontend.md`) | **Adotado (base moderna)** | Motivo |
|------|-----------------------------------|----------------------------|--------|
| Next.js | 14.2+ | **16.x** (App Router, Turbopack) | Versão atual; `params`/`searchParams` como Promise; `proxy.ts` |
| React | (implícito 18) | **19.x** (Server Components) | RSC por padrão |
| Tailwind | 3.4+ | **4.x** (`@tailwindcss/postcss`) | Tokens via `@theme`/variáveis CSS |
| Auth | NextAuth/Auth.js v5 | **Cookies httpOnly + `proxy.ts`** próprios | Menos dependência; sessão no servidor; controle do refresh |
| HTTP client | `ky` | **fetcher próprio em `@/infra`** | Token no servidor, normalização de erro padronizada |
| URL state | `nuqs` | **`useUrlSyncedFilters`** (hook próprio) | Padrão da base; sem dependência extra |
| Middleware | `middleware.ts` | **`proxy.ts`** (renome no Next 16) | Convenção Next 16 |
| i18n | `next-intl` | **PT-BR direto** (UI/validação) | App single-locale PT-BR; revisitar só se surgir multi-idioma |

> Regra prática: **decisão de tecnologia → base moderna**; **o que a tela faz / qual endpoint consome / qual modelo → spec**. Em dúvida sobre uma versão, vale a tabela abaixo.

## 2. Stack oficial (o que usar)

### Core

| Camada | Tecnologia | Versão | Observação |
|--------|------------|--------|------------|
| Runtime | Node.js | 24+ LTS | dev e build |
| Package manager | **Yarn** | — | lockfile `yarn.lock` |
| Framework | Next.js | **16.x** | App Router, Turbopack em dev |
| UI | React | **19.x** | Server Components por padrão |
| Linguagem | TypeScript | 5.x | `strict: true`, zero `any` |
| Estilos | Tailwind CSS | **4.x** | via `@tailwindcss/postcss` |
| Animações | tw-animate-css | — | utilitários de animação |
| Design system | shadcn/ui | **new-york** | Radix UI + CVA + `cn()` |
| Ícones | lucide-react | — | padrão shadcn |

### Dados e formulários

| Função | Tecnologia | Versão | Observação |
|--------|-----------|--------|------------|
| Formulários | react-hook-form | 7.x | lógica em hooks `useFormXxx` |
| Validação | Zod | 4.x | schemas + `z.infer`; mensagens PT-BR |
| Resolver | @hookform/resolvers | — | `zodResolver` |
| Validação BR | zod-br | — | CPF/CNPJ/PIS |
| Máscaras | react-imask | — | CPF/CNPJ/CEP/telefone/moeda |
| Server state (client) | TanStack Query | 5.x | cache em dialogs/refetch |
| Tabelas | TanStack Table | 8.x | listagens server-side |
| UI state | Zustand | 5.x | **apenas UI** (não sessão/JWT) |
| Datas | date-fns (+ -tz) | — | formatação/timezone BR |

### UX e apoio

| Função | Tecnologia | Observação |
|--------|-----------|------------|
| Tema | next-themes | light/dark via classe `.dark` |
| Toasts | sonner | feedback de ações |
| Gráficos | recharts | dashboards |
| PDF (preview) | react-pdf | visualização de documentos |
| Drag & drop | dnd-kit | reordenação (ex.: itens de vistoria/contrato) |
| Rich text | TipTap | campos HTML (editor de contratos) |
| Tempo real | socket.io-client | quando necessário (notificações) |
| Utilitários CSS | clsx, tailwind-merge, CVA | função `cn()` em `@/lib/utils` |

### Qualidade e build

| Função | Tecnologia | Versão |
|--------|-----------|--------|
| Testes unit/componente | Vitest + Testing Library | — |
| E2E | Playwright | — |
| Mock HTTP (test/dev) | MSW | — |
| Lint | ESLint | 9.x (`eslint-config-next`) |
| Git hooks | Husky | 9.x (pre-commit) |
| Container | Docker | build `standalone` |

## 3. Racional das escolhas-chave

- **Server Components + Suspense:** fetch no servidor reduz JS no cliente e melhora TTI/LCP (objetivos do spec). Interatividade isolada em ilhas client.
- **Cookies httpOnly + `proxy.ts`:** sessão fora do alcance de JS do browser (mitiga XSS); refresh proativo na borda. Ver [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md).
- **Zod compartilhado:** o mesmo schema valida o form e descreve o contrato — espelha o domínio do backend e mantém type-safety.
- **TanStack Query/Table:** padrões maduros para cache client e listagens server-side (paginação/sort/filtro). Ver [`./11-tabelas-listagens.md`](./11-tabelas-listagens.md) e [`./12-estado-dados.md`](./12-estado-dados.md).
- **Tailwind 4 + variáveis CSS:** white-label sem rebuild — a marca troca os tokens, não o código. Ver [`./06-multitenancy-whitelabel.md`](./06-multitenancy-whitelabel.md).

## 4. Versionamento

- Versões **fixadas** no `package.json` de cada release; subir major exige nota nesta doc.
- Node 24+ e Yarn obrigatórios; CI usa as mesmas versões do `package.json`/`.nvmrc`.

## 5. Ver também

- [01 — Arquitetura](./01-arquitetura.md) · [04 — Design system](./04-design-system-ui.md) · [10 — Formulários](./10-formularios-validacao.md)
- [14 — Deploy e ambiente](./14-deploy-ambiente.md) — env, Docker, scripts.
