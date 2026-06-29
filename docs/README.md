# Documentação — Frontend weCorp

Documentação técnica e de domínio para a **criação/migração do frontend weCorp** (CakePHP views `.ctp` → **Next.js 16 + React 19 + TypeScript strict**, mobile-first, white-label).
Serve como **memória do projeto**: o quê fazer, como fazer e por quê, em todas as fases.

> A constituição operacional (regras MUST/MUST NOT, fluxo do agente, Definition of Done) está em [`../CLAUDE.md`](../CLAUDE.md).
> Contrato de API, telas e domínio: `C:\projetos\wecorp\frontend.md`. Decisões técnicas base: `C:\projetos\codefirst\api\frontend.md` (a base moderna prevalece sobre o stack antigo citado no spec — ver [02](./02-stack-tecnologico.md)).
> Backend pareado: `C:\projetos\work\wecorp\backend` (contrato em `backend/docs/08-contrato-api.md`).
> Gestão no Linear: projeto **wecorp-frontend** (21 fases F00–**F20**, issues **HUB-225 a HUB-310** + portal público **HUB-319 a HUB-323**).
> Escopo inclui o **portal público do proponente** (F20); alvo = **equivalência funcional** com o legado (ver doc 17).

## Como usar

1. Antes de iniciar uma issue, leia [`../CLAUDE.md`](../CLAUDE.md) (§8 fluxo) e a doc do tema relevante abaixo.
2. Para entender uma tela/módulo, vá a [09 — Módulos e telas](./09-modulos-telas.md) e à parte correspondente do `frontend.md`.
3. Para "como a aplicação funciona por baixo", veja arquitetura/auth/consumo de API/estado (01, 07, 08, 12).

## Índice

| # | Documento | Conteúdo |
|---|-----------|----------|
| 00 | [Visão geral](./00-visao-geral.md) | Produto, multi-tenant, white-label, módulos, objetivos, mobile-first |
| 01 | [Arquitetura](./01-arquitetura.md) | Camadas, App Router, route groups, estrutura de pastas, Server/Client |
| 02 | [Stack tecnológico](./02-stack-tecnologico.md) | Tecnologias, versões, reconciliação spec×base moderna |
| 03 | [Convenções de código](./03-convencoes-codigo.md) | Nomenclatura, barrels, padrões action/hook/schema, anti-padrões |
| 04 | [Design system e UI](./04-design-system-ui.md) | shadcn/ui new-york, Tailwind 4, tokens, `cn()`, `<Card>`/`<If>`, a11y |
| 05 | [Roteamento e navegação](./05-roteamento-navegacao.md) | App Router, layouts, rotas, sidebar/menu por ACL, params Promise |
| 06 | [Multi-tenancy e white-label](./06-multitenancy-whitelabel.md) | Detecção por host, variáveis CSS, 8 marcas, tenant context |
| 07 | [Auth, sessão e permissões](./07-auth-sessao-permissoes.md) | Cookies httpOnly, `proxy.ts`, sessão, 10 grupos, PermissionGate |
| 08 | [Consumo de API e data fetching](./08-consumo-api-dados.md) | fetcher/api, envelope, erros por code, TanStack Query, revalidate |
| 09 | [Módulos e telas](./09-modulos-telas.md) | Referência por domínio: telas, fluxos, contratos consumidos |
| 10 | [Formulários e validação](./10-formularios-validacao.md) | RHF + Zod PT-BR, máscaras BR, seções colapsáveis, autosave, dirty-state |
| 11 | [Tabelas e listagens](./11-tabelas-listagens.md) | DataTable server-side, TanStack Table, filtros/URL, export |
| 12 | [Estado e dados](./12-estado-dados.md) | Matriz de estado (server/client/UI/sessão/URL) |
| 13 | [Testes](./13-testes.md) | Vitest + Testing Library + Playwright, cobertura obrigatória |
| 14 | [Deploy e ambiente](./14-deploy-ambiente.md) | env (Zod), Docker standalone, scripts, NEXT_PUBLIC |
| 15 | [Roadmap e fases](./15-roadmap-fases.md) | 21 fases, ordem de dependências, mapeamento Linear |
| 16 | [Glossário](./16-glossario.md) | Termos de domínio e de frontend |
| 17 | [Paridade e portal público](./17-paridade-portal-publico.md) | Equivalência funcional, telas do portal público (F20) |
