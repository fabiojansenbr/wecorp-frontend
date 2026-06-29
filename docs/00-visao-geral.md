# 00 — Visão Geral do Frontend weCorp

> Ponto de partida da documentação. Aqui você entende **o que é o frontend weCorp**, **para quem serve**, **quais módulos/telas existem** e **como ele se posiciona** na reescrita do legado.
> Regras operacionais (MUST/MUST NOT, fluxo do agente, Definition of Done) estão em [`../CLAUDE.md`](../CLAUDE.md).
> Índice completo: [`./README.md`](./README.md).

---

## 1. O que é o frontend weCorp

O **frontend weCorp** é a interface web do **weCorp** — um **SaaS B2B** para o mercado imobiliário brasileiro que concentra, em uma plataforma, o ciclo de locação de imóveis: da análise do proponente à emissão de seguros, vistorias, assinatura eletrônica de contratos e gestão financeira (repasses, faturas, comissões).

É a **reescrita do zero** da camada de apresentação do sistema legado **AlugueSeguro/weCorp** (CakePHP 2.x, views `.ctp` renderizadas no servidor) para uma **aplicação Next.js 16 (App Router)** desacoplada, que consome a **API REST do backend weCorp** (NestJS 11 + Prisma + PostgreSQL).

| Aspecto | Legado | Alvo |
|---------|--------|------|
| Renderização | Views `.ctp` (server-rendered, acopladas ao CakePHP) | Next.js App Router (React Server Components + Client) |
| Linguagem | PHP + jQuery | TypeScript strict + React 19 |
| Estilo | CSS/Bootstrap legado | Tailwind CSS 4 + shadcn/ui (new-york) |
| Estado/dados | jQuery/AJAX direto | Camadas UI→hooks→actions→`api`; TanStack Query/Table |
| Auth | Sessão CakePHP | Cookies httpOnly + `proxy.ts` (Next 16) contra a API |
| Acoplamento | Monolito | Frontend separado consumindo API versionada |

O frontend é simultaneamente:

- **Multi-tenant** — várias empresas operam na mesma aplicação, com isolamento garantido pelo backend (escopo) e refletido na UI (dados e permissões do usuário logado).
- **White-label por cliente** — **cada cliente pode ter um domínio próprio e uma marca exclusiva** (logo, cores, particularidades) **já na tela de login**, antes de autenticar. A marca é resolvida por **host**, de forma **data-driven** (qualquer empresa via `dominio_corporativo`), com tema por variáveis CSS — **sem rebuild**. As 8 marcas SaaS são os *defaults*; o cliente refina. Ver [`./06-multitenancy-whitelabel.md`](./06-multitenancy-whitelabel.md).
- **Mobile-first** — layouts pensados primeiro para telas pequenas, expandindo para desktop.

## 2. Quem usa (público-alvo)

- **Imobiliárias e administradoras** — carteira de imóveis, contratação de seguros e análises, acompanhamento de vistorias e repasses.
- **Corretoras e seguradoras parceiras** — produtos de seguro (fiança, incêndio, capitalização), cotações/emissões.
- **Operação interna weCorp** — análise cadastral, suporte, gestão da plataforma.
- **Proponentes (inquilinos/fiadores)** — via **portal público** (sem login): proposta de fiança, preenchimento de análise, validação de documentos, upload. Incorporado ao escopo na fase **F20** (ver [`./17-paridade-portal-publico.md`](./17-paridade-portal-publico.md)).

> **Nível de paridade:** o alvo é **equivalência funcional** com o legado — mesmas funcionalidades e regras de negócio, com liberdade para modernizar UX, layout e tecnologia (não é cópia tela-a-tela). Ver [`./17-paridade-portal-publico.md`](./17-paridade-portal-publico.md).

## 3. Objetivos (do spec `frontend.md`)

| # | Objetivo | Métrica |
|---|----------|---------|
| 1 | Performance percebida | TTI < 1.5s em rotas internas; LCP < 2s |
| 2 | Type-safety ponta a ponta | Zero `any` em produção |
| 3 | Acessibilidade | WCAG 2.1 AA em todos os fluxos |
| 4 | Consistência visual | 100% dos componentes via shadcn/ui |
| 5 | Independência de marca | Trocar de tenant sem rebuild |
| 6 | Paridade funcional | Mesmas funcionalidades do legado (modernizadas) |

## 4. Módulos do produto (telas)

Cada módulo é um conjunto de telas em `src/app/(modules)/(dominio)/` e consome o domínio correspondente do backend. Detalhe em [`./09-modulos-telas.md`](./09-modulos-telas.md); o contrato/telas detalhados estão no `frontend.md` (parte indicada).

| Módulo | O que faz | Parte no `frontend.md` |
|--------|-----------|------------------------|
| **Empresas** (multi-tenant) | Cadastro de empresas, módulos contratados, white-label | Parte 2 |
| **Usuários e Autenticação** | Login, usuários, 10 grupos, ACL | Parte 3 |
| **Dashboard** | Visão por grupo, indicadores | Parte 4 |
| **Análise Cadastral** (CORE) | Ficha, score, parecer, aprovação de locatários/fiadores | Parte 5 |
| **Pessoas** | Locatários, fiadores, sócios (cadastro compartilhado) | Parte 6 |
| **Vistorias** | Inspeção de imóveis, lock concorrente, modo demo | Parte 7 |
| **Seguros** | Fiança, Incêndio, Capitalização (cotação/emissão) | Parte 8 |
| **Imóveis** | Carteira de imóveis | Parte 9 |
| **Financeiro** | Contas a pagar/receber, repasses, faturas, comissões | Parte 10 |
| **CRM** | Leads, visitas, propostas, follow-ups | Parte 11 |
| **Assinatura Eletrônica (Sign)** | Envio/assinatura (Docusign/ClickSign/Autentique) | Parte 12 |
| **Sinistros** | Abertura e acompanhamento | Parte 13 |
| **Garantidora** | Gestão da garantidora (cobertura de fiança) | Parte 14 |
| **Prospects e Cotações** | Prospects, cotações WBS, pagamentos | Parte 15 |
| **Marketplace de Vistorias** | Rede de vistoriadores | Parte 16 |
| **Editor de Contratos** | Templates e renderização de documentos | Parte 17 |
| **Comunicação** | SMS/WhatsApp, notificações | Parte 18 |
| **Suporte e CMS** | Tickets, notícias, páginas | Parte 19 |
| **Configurações Globais** | Lookups, módulos/ACL, metadados | Parte 20 |
| **Relatórios** | Relatórios por domínio | (telas nas partes 10/22) |
| **Portal Público do Proponente** | Proposta/análise/validação/upload sem login | F20 (novo escopo) |

## 5. Como o frontend se posiciona na arquitetura

```text
Navegador (marca X) → Next.js (App Router, proxy.ts, RSC + Client)
                         │  cookies httpOnly (sessão)
                         ▼
                    API REST weCorp (NestJS 11)
                         ▼
                    PostgreSQL 16 (multi-tenant por escopo)
```

- O **backend** é a fonte da verdade de dados, regras e isolamento de tenant. O frontend **nunca** reimplementa regra de negócio crítica; ele consome o contrato e cuida de UX, validação de borda (espelhada via Zod), navegação, permissões de exibição e estado de UI.
- O **contrato** (envelope `{data}`/`{data,meta}`/`{error}`, paginação, códigos de erro) está em [`./08-consumo-api-dados.md`](./08-consumo-api-dados.md) e em `backend/docs/08-contrato-api.md`.

## 6. Princípios que guiam o frontend

1. **Camadas** (UI → hooks → actions → `api` → fetcher) — [`./01-arquitetura.md`](./01-arquitetura.md).
2. **Server-first** — RSC por padrão; client só com interatividade.
3. **Sessão no servidor** — cookies httpOnly; nada sensível no client/Zustand.
4. **White-label sem rebuild** — variáveis CSS por host.
5. **Permissões dos 10 grupos** — `PermissionGate` + `hasAccess` testado.
6. **Type-safety e PT-BR** — código em inglês, UI/validação em PT-BR.
7. **Acessível e mobile-first**.

## 7. Ver também

- [01 — Arquitetura](./01-arquitetura.md) · [02 — Stack](./02-stack-tecnologico.md) · [09 — Módulos e telas](./09-modulos-telas.md)
- [06 — Multi-tenancy/white-label](./06-multitenancy-whitelabel.md) · [07 — Auth/permissões](./07-auth-sessao-permissoes.md)
- [15 — Roadmap e fases](./15-roadmap-fases.md) · [17 — Paridade e portal público](./17-paridade-portal-publico.md)
