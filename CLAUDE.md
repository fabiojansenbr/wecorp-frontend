# CLAUDE.md — Constituição do Frontend weCorp

> **Este arquivo é a fonte de verdade operacional para qualquer agente (humano ou IA) que trabalhe no frontend weCorp.**
> Regras marcadas com **MUST / MUST NOT** são não-negociáveis. Em conflito, esta constituição vence.
> Detalhes de implementação vivem em [`docs/`](./docs/README.md) — sempre consulte a doc do tema antes de codar.

---

## 1. Identidade e missão

O **frontend weCorp** é a reescrita da interface do sistema legado **AlugueSeguro/weCorp** (CakePHP 2.x, views `.ctp` server-rendered)
para uma aplicação **Next.js 16 (App Router) + React 19 + TypeScript strict**, **mobile-first** e **white-label**, que consome a API do **backend weCorp** (NestJS 11).

- **Domínio:** SaaS B2B **multi-tenant** e **white-label** (8 marcas) para o mercado imobiliário:
  Seguro Fiança, Incêndio, Capitalização, Análise Cadastral, Vistorias, Financeiro, CRM, Assinatura Eletrônica.
- **Fonte do contrato de API, telas e domínio:** `C:\projetos\wecorp\frontend.md` (22 partes) — é a verdade de endpoints, telas e modelos.
- **Base de decisões técnicas:** `C:\projetos\codefirst\api\frontend.md` (adaptado nos docs deste repo). **Onde o spec citar stack antigo (Next 14, NextAuth, ky, nuqs, Tailwind 3), prevalece a base moderna** (Next 16, cookies httpOnly + `proxy.ts`, fetcher próprio, Tailwind 4). Ver [`docs/02`](./docs/02-stack-tecnologico.md).
- **Backend pareado:** `C:\projetos\work\wecorp\backend` (NestJS) — contrato em `backend/docs/08-contrato-api.md`.
- **Gestão:** Linear, team **Hubtech**, projeto **wecorp-frontend** — 21 fases (F00–**F20**), issues **HUB-225 a HUB-310** + portal público **HUB-319 a HUB-323**.
- **Escopo:** painel admin **+ portal público do proponente** (F20). Alvo = **equivalência funcional** com o legado (modernizar UX/tech é permitido; replicar bug não). Ver [`docs/17`](./docs/17-paridade-portal-publico.md).

## 2. Princípios não-negociáveis

1. **Arquitetura em camadas é obrigatória.** UI → hooks (`useXxx`) → actions → `api` (`@/infra`) → fetcher → backend. (§4)
2. **Server Components por padrão.** `"use client"` **MUST** ser justificado por interatividade (estado, handlers, browser API). (§4)
3. **Type-safety ponta a ponta.** `strict: true`. **MUST NOT** usar `any` em código de produção.
4. **Sessão só no servidor.** Token em **cookie httpOnly**; **MUST NOT** usar `localStorage`/`sessionStorage`/`document.cookie` para token, nem espelhar JWT no Zustand. (§6)
5. **Isolamento de tenant + permissões.** Toda tela/ação respeita o tenant (white-label) e a ACL dos **10 grupos** via `PermissionGate` + `hasAccess` (testado). (§5)
6. **Acessibilidade e mobile-first.** WCAG 2.1 AA; layouts pensados primeiro para telas pequenas. (§7)
7. **Sem `fetch` direto.** Sempre `import { api } from "@/infra"`. **Actions sem `try/catch`** — o hook trata erro com `toast` + `resolveErrorMessage`.
8. **Validação na borda do form.** Todo formulário usa **RHF + Zod** com mensagens **PT-BR**; erro tratado por `code`/`status`, **nunca** por texto da mensagem.
9. **Rastreabilidade até a issue.** Todo trabalho mapeia para uma issue `HUB-N` — branch, commits e PR carregam o ID. Trabalho sem issue **MUST NOT** ir para a branch principal. (§8)

## 3. Stack (resumo — detalhe em [docs/02](./docs/02-stack-tecnologico.md))

| Camada | Tecnologia | Versão |
|--------|-----------|--------|
| Runtime | Node.js | 24+ LTS |
| Framework | Next.js (App Router, Turbopack) | 16.x |
| UI | React (Server Components) | 19.x |
| Linguagem | TypeScript (strict) | 5.x |
| Estilos | Tailwind CSS | 4.x |
| Design system | shadcn/ui (new-york) + Radix + CVA | — |
| Ícones | lucide-react | — |
| Formulários | react-hook-form | 7.x |
| Validação | Zod (+ zod-br p/ CPF/CNPJ) | 4.x |
| Server state | TanStack Query | 5.x |
| Tabelas | TanStack Table | 8.x |
| UI state | Zustand (**UI only**) | 5.x |
| Toasts | sonner | — |
| Gráficos | recharts | — |
| Máscaras | react-imask | — |
| Datas | date-fns (+ -tz) | — |
| Testes | Vitest + Testing Library + Playwright (e2e) | — |
| Package manager | **Yarn** | — |

## 4. Regras de arquitetura — [docs/01](./docs/01-arquitetura.md)

- **UI (componentes)** — compõem markup; **MUST NOT** conter `useState`/`useEffect`/`useForm` diretos nem `fetch`.
- **Hooks (`useXxx`)** — orquestram estado, submit, loading, `toast`, `revalidatePath`.
- **Actions** — só chamam `api`; **um objeto `props` tipado**; **sem `try/catch`**.
- **`@/infra` (api + fetcher)** — único ponto de HTTP; token resolvido **no servidor** (cookies via `next/headers`).
- **Estrutura:** `src/app/(modules)/(dominio)/` com `_business/` (actions/hooks/schemas/helpers) + `(web)/` (pages/_components). Barrels com alias `@/`; **MUST NOT** usar `../../..`.
- **Server vs Client:** listagens/fetch inicial/permission gate no servidor; formulários/dialogs/tabs no client. `params`/`searchParams` são **`Promise`** (Next 16) — usar `await`.

## 5. Multi-tenant, white-label e permissões — [docs/06](./docs/06-multitenancy-whitelabel.md) · [docs/07](./docs/07-auth-sessao-permissoes.md)

- **White-label:** marca detectada por host no `proxy.ts`; tema aplicado por **variáveis CSS** (`--primary`, `--logo-url`, `--radius`, `--font-sans`). 8 marcas + localhost. Cores **MUST** vir de variáveis CSS — nunca hex inline.
- **Permissões (10 grupos):** enums em `@/shared` (`ModulesPermissionEnum`, `ActionsPermissionEnum`, `RolesPermissionEnum`); `<PermissionGateServer>`/`<PermissionGateClient>`; `hasAccess(session, requirement)` **com cobertura de testes obrigatória**. **MUST NOT** hardcodar strings de role/módulo/ação.

## 6. Auth e sessão — [docs/07](./docs/07-auth-sessao-permissoes.md)

- Token em **cookie httpOnly + secure + sameSite=lax**; escrita centralizada em `src/shared/cookies/`.
- `proxy.ts` (middleware Next 16): rotas públicas, refresh proativo, redirect (sem token→login / token em rota pública→home), limpeza em sessão inválida.
- Sessão: fonte única `getServerSession()` (com `cache` do React); passada ao client uma vez por prop/contexto. **MUST NOT** duplicar sessão no Zustand.

## 7. UI / Design system / Acessibilidade — [docs/04](./docs/04-design-system-ui.md)

- shadcn/ui **new-york**; `cn()` em `@/lib/utils`; componentes em `@/components` com barrel.
- Página envolvida em `<Card>`; condicional no JSX via `<If condition={} elseRender={}>` — **MUST NOT** usar `&&`/ternário inline.
- Listagens via TanStack Table + paginação; filtros sincronizados na URL; feedback via `sonner`; `revalidatePath` após mutação server.
- **Mobile-first** e **WCAG 2.1 AA** em todos os fluxos.

## 8. Como um agente executa uma issue (fluxo padrão)

> Cada issue do Linear já traz: User Story, Telas/fluxos, contrato consumido, passo a passo, critérios de aceite. Siga-a; esta seção é o processo ao redor.

0. **Declarar a issue.** Antes de qualquer ação, o agente anuncia em qual issue está trabalhando: `> Trabalhando em **HUB-N — <título>** (Fase Fxx)`. Essa é a âncora do trabalho — todo o restante (branch, commits, PR) carrega o mesmo `HUB-N`.
1. **Pegar a issue** no Linear (projeto wecorp-frontend), respeitando dependências de fase (ver [docs/15](./docs/15-roadmap-fases.md)).
2. **Branch (vínculo primário):** `feat/HUB-<n>-<slug>` a partir da branch principal atualizada. O ID na branch é o que liga o trabalho à issue — a integração Linear↔GitHub move a issue para *In Progress* automaticamente (fallback: mover à mão).
3. **Ler a doc do tema** em `docs/` + a parte correspondente do `frontend.md` + o contrato do backend.
4. **Implementar na ordem das camadas:** schema Zod → action → hook → componente/página (Server por padrão) → permission gate → registrar rota/menu.
5. **Commits:** toda mensagem **MUST** começar com o ID — `HUB-<n>: <descrição imperativa>` (ex.: `HUB-228: cria tela de listagem de empresas`).
6. **Testar:** Vitest (schemas, `hasAccess`, mappers) + componente (Testing Library) + e2e crítico (Playwright).
7. **Qualidade:** `yarn lint && yarn build` verdes. Sem `any`, sem `console.log`, sem segredo, sem hex inline, sem string de permissão hardcoded.
8. **PR:** título `HUB-<n>: <título da issue>`; descrição com o que foi feito, como testar e `Fixes HUB-<n>` (magic word — fecha a issue no merge). A integração move a issue para *In Review* (fallback: mover à mão).
9. **Definition of Done** (§10) satisfeita → merge → *Done* (automático no merge da branch principal).

> **Automação Linear↔GitHub:** configurada por time em *Settings → Team (Hubtech) → Workflows & automations → Pull request and commit automations*. Padrão: PR aberto → *In Progress*; merge → *Done*. Se a integração não estiver ativa, os passos de status são manuais.

## 9. Comandos

```bash
yarn install
cp .env.example .env
yarn dev            # desenvolvimento (Turbopack)
yarn lint           # ESLint
yarn test           # Vitest
yarn test:e2e       # Playwright
yarn build && yarn start   # produção (standalone)
```

## 10. Definition of Done (checklist universal)

- [ ] Implementado nas camadas corretas; UI sem `fetch`/estado direto; action sem `try/catch`; Server Component por padrão.
- [ ] Schema Zod (PT-BR) + RHF; erros tratados por `code`/`status` com `resolveErrorMessage`.
- [ ] Permissão aplicada (`PermissionGate` + `hasAccess`); tenant/white-label respeitado; nada hardcoded.
- [ ] Acessível (WCAG AA) e responsivo (mobile-first); estados loading/empty/error cobertos.
- [ ] `revalidatePath` após mutações; envelope `{data}`/`{data,meta}`/`{error}` consumido corretamente.
- [ ] Testes: schemas + `hasAccess` (todos os branches) + componente; e2e dos fluxos críticos.
- [ ] `yarn lint && yarn build` sem erros; zero `any`; sem `console.log`; cookies httpOnly para token.
- [ ] Critérios de aceite da issue atendidos.
- [ ] **Paridade funcional** verificada contra a tela/fluxo legado correspondente (ver [`docs/17`](./docs/17-paridade-portal-publico.md)); divergências intencionais documentadas.
- [ ] Branch `feat/HUB-N-<slug>`; todos os commits prefixados com `HUB-N:`.
- [ ] PR com título `HUB-N: <título>` referenciando a issue (com `Fixes HUB-N` na descrição).

## 11. Consulta ao legado (base de conhecimento)

> Alvo é **equivalência funcional** (§1). Antes de reimplementar uma tela/fluxo, consulte **como o legado fazia**.

- **Fonte legada:** CakePHP 2.x em `C:\projetos\wecorp\app` — **telas** em views `.ctp` (`app/View/**`, `app/Plugin/*/View/**`) + a lógica de controller/model correspondente. É **read-only** — **MUST NOT** escrever lá.
- **Base de conhecimento (on-demand):** [`docs/legado/`](./docs/legado/README.md) — **um dossiê por domínio/módulo de tela**, com fluxos, campos/validações, máscaras, comportamento por grupo e gotchas.
- **Ordem de consulta ao pegar uma issue:**
  1. [`docs/legado/README.md`](./docs/legado/README.md) → abra **só o dossiê do módulo** da issue.
  2. Matriz de rastreabilidade do backend ([`../backend/docs/parity-matrix.md`](../backend/docs/parity-matrix.md)) → controller/view legados exatos.
  3. Dúvida profunda → **`/legacy-lookup "<pergunta>"`** (explora o legado em subagente; resposta concisa; pode persistir como dossiê).
  4. Só então `Grep`/`Read` nas `.ctp`/controllers.
- **MUST NOT ler/indexar:** `**/Vendor/**`, `lib/Cake/**`, `*~`, `*-bkp.*`, `*.zip`, `frontend.{html,md}`.
- **Anti-context-rot:** referência **on-demand** — não auto-carregada; **um dossiê por tarefa**.

## 12. Índice da documentação

Comece por [`docs/README.md`](./docs/README.md). Mapa rápido:

| Tema | Doc |
|------|-----|
| Visão geral, domínio, white-label, mobile-first | [docs/00](./docs/00-visao-geral.md) |
| Arquitetura (camadas, App Router, pastas) | [docs/01](./docs/01-arquitetura.md) |
| Stack tecnológico (+ reconciliação de versões) | [docs/02](./docs/02-stack-tecnologico.md) |
| Convenções de código | [docs/03](./docs/03-convencoes-codigo.md) |
| Design system e UI | [docs/04](./docs/04-design-system-ui.md) |
| Roteamento e navegação | [docs/05](./docs/05-roteamento-navegacao.md) |
| Multi-tenancy e white-label | [docs/06](./docs/06-multitenancy-whitelabel.md) |
| Auth, sessão e permissões | [docs/07](./docs/07-auth-sessao-permissoes.md) |
| Consumo de API e data fetching | [docs/08](./docs/08-consumo-api-dados.md) |
| Módulos e telas (bíblia) | [docs/09](./docs/09-modulos-telas.md) |
| Formulários e validação | [docs/10](./docs/10-formularios-validacao.md) |
| Tabelas e listagens | [docs/11](./docs/11-tabelas-listagens.md) |
| Estado e dados | [docs/12](./docs/12-estado-dados.md) |
| Testes | [docs/13](./docs/13-testes.md) |
| Deploy e ambiente | [docs/14](./docs/14-deploy-ambiente.md) |
| Roadmap e fases (Linear) | [docs/15](./docs/15-roadmap-fases.md) |
| Glossário | [docs/16](./docs/16-glossario.md) |
| Paridade e portal público | [docs/17](./docs/17-paridade-portal-publico.md) |
| Base de conhecimento legado (dossiês de telas) | [docs/legado/README.md](./docs/legado/README.md) |
