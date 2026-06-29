# 16 — Glossário

> Vocabulário do frontend weCorp em duas frentes: termos de **domínio** (mercado imobiliário, seguros, análise) e termos de **frontend** (Next.js, camadas, padrões da base). Serve a qualquer agente — humano ou IA — para entender o que aparece no código, nas issues e no `frontend.md`.
> Coerente com [`backend/docs/16-glossario.md`](../../backend/docs/16-glossario.md). Telas por módulo: [`./09-modulos-telas.md`](./09-modulos-telas.md). Arquitetura: [`./01-arquitetura.md`](./01-arquitetura.md). Auth/permissões: [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md).

## Como usar

- Termos agrupados em **Domínio** e **Frontend**; cada um traz **definição** curta + onde aparece.
- Os valores de **status (enums)** por domínio vivem em [`backend/docs/16-glossario.md`](../../backend/docs/16-glossario.md) e em [`./09-modulos-telas.md`](./09-modulos-telas.md).

---

## Domínio (negócio)

| Termo | Definição |
|-------|-----------|
| **PF** | Pessoa Física (CPF). Aparece em cadastro de pessoas, análise, seguros. |
| **PJ** | Pessoa Jurídica (CNPJ). Cadastro de pessoas, empresas, seguros. |
| **Locatário** | Inquilino — quem aluga o imóvel e é o principal analisado. |
| **Locador / Proprietário** | Dono que aluga o imóvel; parte em contratos, vistorias e Sign. |
| **Fiador** | Pessoa que garante o aluguel (alternativa à fiança/CAP). |
| **Sócio** | Sócio de PJ analisada; compõe a análise de pessoa jurídica (sem análise própria). |
| **Cônjuge** | Parceiro(a) do analisado; compõe renda/responsabilidade (condicional ao estado civil). |
| **Proponente** | Pretendente que preenche a proposta/análise pelo **portal público** (sem login, via token). |
| **Análise Cadastral** | Processo CORE: avaliar locatário/fiador/RH para aprovar uma locação (módulo F06). |
| **Score** | Nota/classificação de risco resultante das consultas e do parecer (`classificacao_risco`). |
| **Parecer** | Conclusão do operador sobre a pessoa/ficha (aprovado, com ressalvas, reprovado). |
| **Análise Expressa (E) / Completa (C)** | Tipos de ficha: rápida (conjunto reduzido) vs aprofundada (mais consultas). |
| **Bureau** | Serviço de proteção ao crédito consultado na análise (Serasa, Procob). |
| **Sindicância / RH** | Variação da análise para ficha de funcionário (`tipoficha_analises_id=2`). |
| **Fiança (Seguro Fiança)** | Garantia locatícia: a seguradora cobre o aluguel na inadimplência (F09). |
| **Seguro Incêndio** | Cotação/emissão de seguro contra incêndio do imóvel (produtos 130/138/140). |
| **Capitalização (CAP)** | Título de capitalização usado como garantia/produto financeiro (F09). |
| **Seguro Condomínio / Inspeção Predial** | Seguro de condomínio precedido de questionário de inspeção predial. |
| **Apólice** | Contrato de seguro emitido pela seguradora. |
| **Cobertura** | Risco/valor coberto na apólice. |
| **Proposta** | Formalização enviada à seguradora para emissão (também sentido comercial no CRM). |
| **Vistoria** | Inspeção do imóvel (Entrada/Saída/Periódica) com cômodos, itens, fotos e laudo (F07). |
| **Laudo** | Relatório final da vistoria em PDF (Expressa/Completa/Final). |
| **Aditivo** | Alteração/adição a um laudo de vistoria (ou a um contrato, no Editor). |
| **Lock concorrente** | Bloqueio que impede dois usuários editarem a mesma vistoria ao mesmo tempo (heartbeat 30s, expira em 5min). |
| **Modo demo** | Acesso público temporário a uma vistoria via token, com data limite. |
| **Sinistro** | Ocorrência coberta pelo seguro (incêndio, roubo, alagamento, inadimplência). |
| **Indenização** | Pagamento ao segurado quando o sinistro é procedente. |
| **Garantidora** | Produto de garantia locatícia próprio; gere faturas/parcelas entre imobiliária e seguradora (F13). |
| **Repasse** | Pagamento/comissão devido ao corretor/produtor (e transferências a terceiros no Financeiro). |
| **Fatura** | Documento financeiro que agrupa cobranças de um período. |
| **Parcela** | Cada vencimento de uma conta a pagar/receber ou de uma fatura. |
| **Comissão** | Parcela de remuneração sobre o negócio (aliquota de comissão no Financeiro). |
| **Plano de Contas** | Estrutura hierárquica de contas (despesa/receita, analítica/sintética). |
| **Boleto** | Instrumento de cobrança bancária (nosso número, linha digitável, URL). |
| **Comissão / IOF / ISS** | Tributos e remunerações que incidem sobre seguros/serviços no Financeiro. |
| **WBS O2** | Plataforma O2 (`o2wbs.net`) das seguradoras para cotação/emissão; origem de prospects/cotações. |
| **Marketplace de Vistorias** | Rede que conecta imobiliárias a empresas vistoriadoras, com proposta/aceite/avaliação (F14). |
| **Prospect** | Potencial cliente captado para cotação de seguro (via WBS). |
| **Cotação** | Simulação de preço de seguro junto à seguradora. |
| **CRECI** | Conselho Regional de Corretores de Imóveis — registro do corretor/empresa. |
| **EULA** | *End User License Agreement* — termo de uso aceito pelo usuário (obrigatório p/ grupo 2). |
| **White-label** | Cada marca é servida de domínio próprio com identidade visual customizada (8 marcas) — sem rebuild. |
| **Multi-tenant** | Várias empresas (tenants) na mesma aplicação, com isolamento de escopo garantido pelo backend. |
| **10 grupos / ACL** | Os 10 grupos de acesso (1–10) herdados do legado, base do RBAC; não simplificar. |

## Frontend (técnico)

| Termo | Definição |
|-------|-----------|
| **App Router** | Sistema de rotas do Next.js baseado em pastas em `src/app/`; Server Components por padrão. |
| **Route group** | Pasta `(nome)/` que organiza rotas **sem aparecer na URL** (ex.: `(modules)`, `(auth)`, `(public)`). |
| **Server Component (RSC)** | Componente renderizado no servidor (padrão); faz fetch inicial, sem JS no cliente. |
| **`"use client"`** | Diretiva que marca uma ilha client (estado, handlers, dialogs, browser API); só com justificativa. |
| **Camadas** | Fluxo unidirecional UI → hooks (`useXxx`) → actions → `api` (`@/infra`) → fetcher → backend. |
| **action** | Função async que apenas chama `api`; recebe um `props` tipado; **sem `try/catch`** (o hook trata). |
| **hook `useXxx`** | Orquestra estado, submit, loading, `toast` e `revalidatePath`; concentra a lógica (testável). |
| **`api` (`@/infra`)** | Único ponto de HTTP; token resolvido **no servidor** (cookies via `next/headers`). |
| **fetcher** | Camada base sob `api`: faz `fetch`, monta headers, faz parse e **normaliza erro** por `code`/`status`. |
| **barrel** | `index.ts` que reexporta a API pública de uma área (`@/shared`, `@/components`, `_business/index.ts`). |
| **`proxy.ts`** | Middleware do Next 16 (renome de `middleware.ts`): rotas públicas, refresh proativo, redirects, white-label por host. |
| **token httpOnly** | Token de sessão em cookie httpOnly+secure+sameSite=lax; **nunca** em `localStorage`/`Zustand`/`document.cookie`. |
| **envelope de resposta** | Formato padrão da API: `{data}` (item), `{data, meta}` (lista paginada) ou `{error}` (falha). |
| **`PermissionGate`** | Componente (`PermissionGateServer`/`PermissionGateClient`) que renderiza conteúdo conforme a ACL. |
| **`hasAccess`** | `hasAccess(session, requirement)` decide acesso por módulo/ação/role; **cobertura de testes obrigatória**. |
| **`RolesPermissionEnum` / `ModulesPermissionEnum` / `ActionsPermissionEnum`** | Enums em `@/shared`; evitam strings hardcoded de role/módulo/ação. |
| **`revalidatePath`** | Invalida o cache de uma rota após mutação no servidor (chamado no hook). |
| **Suspense** | Limite de carregamento do React para streaming/fallback (skeletons) em Server Components. |
| **DataTable** | Wrapper de **TanStack Table** para listagens server-side (paginação/sort/filtro/colunas). |
| **`useUrlSyncedFilters`** | Hook próprio que sincroniza filtros de tabela com a query string da URL (substitui `nuqs`). |
| **Zustand (UI-only)** | Store usada **apenas** para estado de UI (drawer aberto, tema…); **nunca** sessão/JWT. |
| **TanStack Query** | Cache de server state no client (dialogs, refetch, lookups). |
| **RHF + Zod** | react-hook-form + Zod: formulários com validação na borda, mensagens em PT-BR. |
| **`discriminatedUnion`** | Padrão Zod para PF/PJ (`discriminatedUnion('tipo_pessoa', [PF, PJ])`), compartilhado front/back. |
| **`cn()`** | Utilitário `clsx` + `tailwind-merge` em `@/lib/utils` para compor classes. |
| **`<If>`** | Componente de condicional no JSX (`<If condition elseRender>`); evita `&&`/ternário inline. |
| **shadcn/ui (new-york)** | Design system (Radix + CVA) usado em `@/components`. |
| **`getServerSession()`** | Fonte única da sessão no servidor (com `cache` do React); passada ao client uma vez. |
| **autosave / dirty-state** | Salvamento periódico de rascunho (ex.: análise a cada 30s) e indicador de alterações não salvas. |
| **`@PublicToken`** | Guard do backend para rotas públicas do portal do proponente (tenant/recurso vêm do token). |

---

## Ver também

- [`backend/docs/16-glossario.md`](../../backend/docs/16-glossario.md) — glossário de domínio do backend (enums de status detalhados).
- [`./09-modulos-telas.md`](./09-modulos-telas.md) — telas, fluxos e endpoints por módulo.
- [`./01-arquitetura.md`](./01-arquitetura.md) · [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md) · [`./08-consumo-api-dados.md`](./08-consumo-api-dados.md).
- [`../CLAUDE.md`](../CLAUDE.md) — constituição operacional.
</content>
</invoke>
