# 15 — Roadmap e Fases (Linear)

> Plano de execução da reescrita do **frontend weCorp** (CakePHP 2.x, views `.ctp` → **Next.js 16 + React 19 + TypeScript strict + Tailwind 4**), organizado em **21 fases** (F00–F20).
> Constituição operacional: [`../CLAUDE.md`](../CLAUDE.md). Visão geral/módulos: [`./00-visao-geral.md`](./00-visao-geral.md). Arquitetura: [`./01-arquitetura.md`](./01-arquitetura.md). Backend pareado: [`backend/docs/15-roadmap-fases.md`](../../backend/docs/15-roadmap-fases.md).

## Como ler este documento

- **Gestão no Linear** — team **Hubtech**, projeto **wecorp-frontend**: <https://linear.app/hubtechsistemas/project/wecorp-frontend-e29a8ee4dd97>.
- Cada **fase é um milestone** do Linear e agrupa um intervalo de issues `HUB-N` (de **HUB-225** a **HUB-310**, mais o portal público **HUB-319..323** na fase **F20**).
- Cada **issue já contém** User Story, telas/fluxos, contrato consumido, passo a passo e critérios de aceite — comece pela issue, não por suposições. Esta doc dá o **mapa macro e a ordem de dependências**; a issue dá o detalhe.
- Cada fase abaixo traz: **Objetivo**, **O que entrega** e **Dependências** (fases do frontend e/ou do backend que precisam estar prontas antes).
- **Regra de dependência transversal:** todo domínio depende do **backend correspondente expor o contrato** (envelope, endpoints, escopo). A fundação `F00–F03` vem primeiro; nenhuma tela de domínio é segura sem auth, RBAC e design system.
- Fluxo de execução de uma issue: ver [`../CLAUDE.md` §8](../CLAUDE.md). Definition of Done: [`../CLAUDE.md` §10](../CLAUDE.md).

## Mapa de fases × issues (visão rápida)

| Fase | Nome | Issues | Tipo |
|------|------|--------|------|
| **F00** | Fundação | HUB-225 … HUB-231 | Fundação |
| **F01** | Autenticação e Multi-tenant white-label | HUB-232 … HUB-237 | Fundação |
| **F02** | Design System e Componentes | HUB-238 … HUB-245 | Fundação |
| **F03** | Permissões e RBAC | HUB-246 … HUB-248 | Fundação |
| **F04** | Empresas e Usuários | HUB-249 … HUB-254 | Base |
| **F05** | Dashboard | HUB-255 … HUB-257 | Domínio |
| **F06** | Análise Cadastral e Pessoas | HUB-258 … HUB-264 | Domínio |
| **F07** | Vistorias | HUB-265 … HUB-268 | Domínio |
| **F08** | Imóveis | HUB-269 … HUB-271 | Domínio |
| **F09** | Seguros | HUB-272 … HUB-277 | Domínio |
| **F10** | Financeiro | HUB-278 … HUB-283 | Domínio |
| **F11** | CRM | HUB-284 … HUB-287 | Domínio |
| **F12** | Assinatura Eletrônica | HUB-288 … HUB-290 | Domínio |
| **F13** | Sinistros e Garantidora | HUB-291 … HUB-292 | Domínio |
| **F14** | Prospects/Cotações e Marketplace | HUB-293 … HUB-295 | Domínio |
| **F15** | Editor de Contratos | HUB-296 … HUB-298 | Domínio |
| **F16** | Comunicação, Suporte e CMS | HUB-299 … HUB-301 | Domínio/Transversal |
| **F17** | Configurações Globais | HUB-302 … HUB-304 | Base |
| **F18** | Relatórios | HUB-305 … HUB-306 | Domínio |
| **F19** | Qualidade, Testes e Deploy | HUB-307 … HUB-310 | Transversal |
| **F20** | Portal Público do Proponente | HUB-319 … HUB-323 | Domínio (público) |

> **Portal público (F20):** milestone adicionado na auditoria de paridade. Consome os **endpoints públicos do backend F24** (HUB-311..318). Ver [`./17-paridade-portal-publico.md`](./17-paridade-portal-publico.md) e [`backend/docs/parity-matrix.md`](../../backend/docs/parity-matrix.md).

---

## F00 — Fundação `HUB-225..231`

- **Objetivo:** subir o esqueleto Next.js 16 com toda a infraestrutura transversal antes de qualquer tela de domínio.
- **O que entrega:** projeto Next 16 (App Router, Turbopack) + React 19 + TypeScript strict; estrutura de pastas (`src/app/(modules|auth|public)`, `infra`, `shared`, `configs`, `providers`); camada `@/infra` (fetcher + cliente `api`, token no servidor); `src/configs/env.ts` validado por Zod; `Providers` (theme, toaster); ESLint 9 + Husky pre-commit; Vitest + Playwright + MSW configurados; Docker standalone base. Ver [`./01-arquitetura.md`](./01-arquitetura.md) e [`./14-deploy-ambiente.md`](./14-deploy-ambiente.md).
- **Dependências:** nenhuma. Ponto de partida.

## F01 — Autenticação e Multi-tenant white-label `HUB-232..237`

- **Objetivo:** sessão segura por cookies httpOnly, `proxy.ts` e white-label por host — a borda da aplicação.
- **O que entrega:** login/recuperar senha (`(auth)`); cookies httpOnly (`src/shared/cookies/`); `proxy.ts` (rotas públicas, refresh proativo, redirects, limpeza de sessão); `getServerSession()` com cache; detecção de marca por host + tema por variáveis CSS (8 marcas + localhost); `TENANT_API_MAP`. Ver [`./06-multitenancy-whitelabel.md`](./06-multitenancy-whitelabel.md) e [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md).
- **Dependências:** F00; **backend F03** (login/JWT, claims `userId`/`empresaId`/`grupoId`) e **F04** (white-label da empresa).

## F02 — Design System e Componentes `HUB-238..245`

- **Objetivo:** o vocabulário visual reutilizável — shadcn/ui new-york + tokens white-label + componentes weCorp.
- **O que entrega:** shadcn/ui (new-york) + `cn()`; `globals.css` (Tailwind 4 + tokens por marca); componentes base (`Card`, `Table` com TanStack Table, `If`, `Dialog`, form fields, máscaras react-imask, toasts sonner); layout do painel (sidebar/topbar) mobile-first; estados loading/empty/error. Ver [`./04-design-system-ui.md`](./04-design-system-ui.md).
- **Dependências:** F00; F01 (white-label fornece os tokens de tema).

## F03 — Permissões e RBAC `HUB-246..248`

- **Objetivo:** o modelo de permissões dos **10 grupos** refletido na UI.
- **O que entrega:** enums em `@/shared` (`ModulesPermissionEnum`, `ActionsPermissionEnum`, `RolesPermissionEnum`); `hasAccess(session, requirement)` (função pura, **cobertura de todos os branches obrigatória**); `<PermissionGateServer>`/`<PermissionGateClient>`; gating de menu/rota. Ver [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md) e [`./13-testes.md`](./13-testes.md).
- **Dependências:** F01 (sessão); **backend F03** (ACL dos 10 grupos no JWT/contrato).

## F04 — Empresas e Usuários `HUB-249..254`

- **Objetivo:** gestão do tenant — empresas (módulos contratados, branding) e usuários/grupos.
- **O que entrega:** CRUD de empresas; módulos/fornecedores contratados; branding white-label (logo, domínio, tema); CRUD de usuários, atribuição de grupo, recuperação de senha. Telas no `frontend.md` partes 2 e 3.
- **Dependências:** F03; **backend F04** (Empresas) e **F03** (Usuários/permissões).

## F05 — Dashboard `HUB-255..257`

- **Objetivo:** a home pós-login — indicadores por grupo/perfil.
- **O que entrega:** dashboard por grupo; KPIs (análises, vistorias, fiança, CAP, sign por status); gráficos recharts; atalhos por permissão. `frontend.md` parte 4.
- **Dependências:** F02, F03; **backend F22** (KPIs/dashboard) — pode começar com dados parciais dos domínios já disponíveis.

## F06 — Análise Cadastral e Pessoas `HUB-258..264`

- **Objetivo:** o coração do produto — fichas de análise cadastral e o cadastro de pessoas compartilhado.
- **O que entrega:** CRUD de pessoas (PF/PJ, endereços, documentos, qualificações, relacionamentos); ficha de análise (Expressa/Completa) com a máquina de estado refletida na UI; itens/pareceres por pessoa; visualização de score/PDF; estados/transições. `frontend.md` partes 5 e 6.
- **Dependências:** F03–F04; **backend F05** (Pessoas) e **F06** (Análise Cadastral); consultas reais de bureau dependem do backend F21.

## F07 — Vistorias `HUB-265..268`

- **Objetivo:** inspeção de imóveis com edição concorrente (lock) e modo demo.
- **O que entrega:** CRUD de vistorias; máquina de estado na UI; cômodos/itens, fotos, aditivos; **lock** de edição concorrente; histórico de status; laudo PDF (preview). `frontend.md` parte 7.
- **Dependências:** F06 (pessoas), F08 (imóveis); **backend F07** (Vistorias).

## F08 — Imóveis `HUB-269..271`

- **Objetivo:** carteira de imóveis usada por vistorias, seguros e contratos.
- **O que entrega:** CRUD de imóveis; tipos/situação; endereços; vínculo a locador/locatário. `frontend.md` parte 9.
- **Dependências:** F06 (pessoas); **backend F08** (Imóveis).

## F09 — Seguros `HUB-272..277`

- **Objetivo:** os produtos de seguro — Fiança, Incêndio e Capitalização (cotação/emissão).
- **O que entrega:** Seguro Fiança com a máquina de estado refletida; CAP; cotação/emissão de Incêndio; parcelas, coberturas, repasses; apólices e endossos. `frontend.md` parte 8.
- **Dependências:** F06, F08, F10 (vínculo financeiro); **backend F09** (Seguros); cotação/emissão real depende do backend F21 (WBS/seguradoras).

## F10 — Financeiro `HUB-278..283`

- **Objetivo:** contas a pagar/receber, parcelas, boletos, repasses e conciliação.
- **O que entrega:** plano de contas; contas a pagar/receber; parcelas com status; boletos; movimentos/conciliação; repasse ao produtor; IOF/comissões (exibição). `frontend.md` parte 10.
- **Dependências:** F04; **backend F10** (Financeiro).

## F11 — CRM `HUB-284..287`

- **Objetivo:** gestão comercial — leads, visitas, propostas, follow-ups.
- **O que entrega:** funil de leads; agenda/status de visitas; propostas; follow-ups. `frontend.md` parte 11.
- **Dependências:** F04, F06 (pessoas); **backend F11** (CRM).

## F12 — Assinatura Eletrônica `HUB-288..290`

- **Objetivo:** envelopes e signatários integrados a provedores externos.
- **O que entrega:** envio/assinatura (Docusign/ClickSign/Autentique); vínculo polimórfico (vistoria/contrato/ficha); status do envelope/signatário; download dos assinados. `frontend.md` parte 12.
- **Dependências:** F06; **backend F12** (Sign) e F21 (provedores/webhooks reais).

## F13 — Sinistros e Garantidora `HUB-291..292`

- **Objetivo:** abertura/acompanhamento de sinistros e o painel da garantidora.
- **O que entrega:** registro/acompanhamento de sinistros e indenização; painel da garantidora (análise, faturas/parcelas, vínculo a pessoas/imóveis). `frontend.md` partes 13 e 14.
- **Dependências:** F09 (apólices), F06, F10; **backend F13** (Sinistros) e **F14** (Garantidora).

## F14 — Prospects/Cotações e Marketplace `HUB-293..295`

- **Objetivo:** captação de prospects/cotações (WBS) e o marketplace de vistorias.
- **O que entrega:** prospects; cotações com status; pagamentos; marketplace (pacotes/preços, oferta/aceite, rede de vistoriadores). `frontend.md` partes 15 e 16.
- **Dependências:** F06, F07, F10; **backend F15** (Prospects/Cotações) e **F16** (Marketplace) + F21 (WBS).

## F15 — Editor de Contratos `HUB-296..298`

- **Objetivo:** templates de contrato e renderização com dados do domínio.
- **O que entrega:** editor de templates (TipTap) com variáveis; preview/render para PDF; vínculo a imóveis/pessoas/seguros. `frontend.md` parte 17.
- **Dependências:** F06, F08; **backend F17** (Editor de Contratos) + F21 (PDF).

## F16 — Comunicação, Suporte e CMS `HUB-299..301`

- **Objetivo:** notificações (SMS/WhatsApp/e-mail), helpdesk e gestão de conteúdo das marcas.
- **O que entrega:** envio/templates de mensagens; tickets de suporte; CMS (páginas/banners/notícias por tenant). `frontend.md` partes 18 e 19.
- **Dependências:** F01, F04; **backend F18** (Comunicação) e **F19** (Suporte/CMS).

## F17 — Configurações Globais `HUB-302..304`

- **Objetivo:** gestão das dezenas de lookups/metadados e da ACL (`/config`).
- **O que entrega:** CRUD genérico de lookups (status, tipos, categorias, relacionamentos); gestão de módulos/ACL; metadados. `frontend.md` parte 20.
- **Dependências:** F03; **backend F20** (Configurações Globais e Lookups). Habilita formulários dos domínios que consomem lookups.

## F18 — Relatórios `HUB-305..306`

- **Objetivo:** relatórios por domínio e exportações.
- **O que entrega:** relatórios financeiros e operacionais; filtros; exportação. Telas no `frontend.md` partes 10/22.
- **Dependências:** domínios geradores de dados (F06–F12); **backend F22** (Dashboard e Relatórios) e F10 (relatórios financeiros).

## F19 — Qualidade, Testes e Deploy `HUB-307..310`

- **Objetivo:** endurecer, testar a fundo e publicar o frontend.
- **O que entrega:** suíte Vitest + Testing Library (HUB-307); MSW + e2e Playwright dos fluxos críticos (HUB-308); Docker standalone + `env.ts` Zod + `.env.example` (HUB-309); CI/CD GitHub Actions + Husky (HUB-310). Ver [`./13-testes.md`](./13-testes.md) e [`./14-deploy-ambiente.md`](./14-deploy-ambiente.md).
- **Dependências:** **transversal e final** — boas práticas aplicadas continuamente, validação e deploy fechados aqui.

## F20 — Portal Público do Proponente `HUB-319..323`

- **Objetivo:** as superfícies **públicas** (sem login, white-label, mobile-first) que o legado expunha ao proponente.
- **O que entrega:** shell público white-label + autocadastro + termos/EULA (HUB-319); formulário público multi-step de proposta de Seguro Fiança (HUB-320); formulário público de Análise Cadastral preenchido pelo pretendente (HUB-321); páginas públicas de validação de autenticidade — análise/vistoria (HUB-322); upload público de documentos token-gated, componente reutilizável (HUB-323). Ver [`./17-paridade-portal-publico.md`](./17-paridade-portal-publico.md).
- **Dependências:** F00–F02 (fundação, design system, white-label); **backend F24** (HUB-311..318 — token público, proposta fiança, garantidora proponente, análise pública, validação, endereços/CEP, reset de senha, upload token-gated).

---

## Ordem recomendada de execução

1. **Fundação (sequencial, sem atalhos):** `F00 → F01 → F02 → F03`. Sem infra HTTP, sessão/white-label, design system e RBAC, nenhuma tela de domínio é segura nem consistente.
2. **Base:** `F04` (Empresas e Usuários) e `F17` (Configurações/Lookups) — alicerce de dados e dos formulários de domínio. `F05` (Dashboard) entra cedo como home, evoluindo com os KPIs.
3. **Domínios (paralelizáveis após a base):** com F00–F04 e F17 prontos, os módulos `F06`+ podem ser tocados por times distintos em paralelo, respeitando dependências locais — `F07` (Vistorias) precisa de `F08` (Imóveis); `F09` (Seguros) precisa de `F10` (Financeiro); `F13` precisa de `F09`. **Cada domínio só inicia quando o backend correspondente expõe o contrato.**
4. **Agregação:** `F18` (Relatórios) depois que os domínios geram dados.
5. **Público:** `F20` (Portal Público) após a fundação e quando o **backend F24** expuser os endpoints públicos.
6. **Transversal/final:** `F19` (Qualidade, Testes e Deploy) aplicada continuamente e fechada por último.

### Diagrama de dependências

```mermaid
flowchart TD
    F00[F00 Fundação] --> F01[F01 Auth + White-label]
    F00 --> F02[F02 Design System]
    F01 --> F02
    F01 --> F03[F03 Permissões/RBAC]

    F03 --> F04[F04 Empresas e Usuários]
    F03 --> F17[F17 Configurações/Lookups]
    F02 --> F05[F05 Dashboard]
    F03 --> F05

    F04 --> F06[F06 Análise Cadastral e Pessoas]
    F17 --> F06
    F06 --> F08[F08 Imóveis]
    F08 --> F07[F07 Vistorias]
    F06 --> F07
    F04 --> F10[F10 Financeiro]
    F08 --> F09[F09 Seguros]
    F10 --> F09
    F06 --> F11[F11 CRM]
    F06 --> F12[F12 Sign]
    F09 --> F13[F13 Sinistros/Garantidora]
    F06 --> F14[F14 Prospects/Marketplace]
    F07 --> F14
    F06 --> F15[F15 Editor de Contratos]
    F08 --> F15
    F01 --> F16[F16 Comunicação/Suporte/CMS]

    F06 --> F18[F18 Relatórios]
    F09 --> F18
    F10 --> F18

    F00 --> F20[F20 Portal Público]
    F02 --> F20
    BE24[Backend F24 públicos] -.HUB-311..318.-> F20

    F18 --> F19[F19 Qualidade/Testes/Deploy]
    F20 --> F19

    classDef fund fill:#1f2937,stroke:#60a5fa,color:#fff;
    classDef base fill:#374151,stroke:#34d399,color:#fff;
    classDef pub fill:#3b1f2b,stroke:#f472b6,color:#fff;
    classDef cross fill:#3f2d1a,stroke:#f59e0b,color:#fff;
    class F00,F01,F02,F03 fund;
    class F04,F05,F17 base;
    class F20 pub;
    class F19 cross;
```

> Legenda: setas cheias = dependência de pré-requisito do frontend; setas tracejadas = dependência do backend (contrato/endpoints). Azul = fundação, verde = base, rosa = público, laranja = transversal. Cada domínio também depende do backend correspondente expor o contrato (não desenhado para todos, para não poluir).

## Ver também

- [`../CLAUDE.md`](../CLAUDE.md) — constituição, fluxo do agente (§8), Definition of Done (§10).
- [`./00-visao-geral.md`](./00-visao-geral.md) — módulos/telas e posicionamento.
- [`./13-testes.md`](./13-testes.md) · [`./14-deploy-ambiente.md`](./14-deploy-ambiente.md) — F19.
- [`./17-paridade-portal-publico.md`](./17-paridade-portal-publico.md) — F20 e paridade.
- [`backend/docs/15-roadmap-fases.md`](../../backend/docs/15-roadmap-fases.md) — fases pareadas do backend.
</content>
</invoke>
