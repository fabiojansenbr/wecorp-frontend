# 09 — Módulos e Telas

> **A "bíblia de telas" do frontend weCorp.** Referência completa, por domínio de negócio, das **rotas/telas**, fluxos e estados (loading/empty/error), endpoints consumidos do backend, grupos/permissões e ponteiros para o spec e o Linear.
> Fonte de verdade de telas/contrato: `C:\projetos\wecorp\frontend.md` (Partes 2–20). O **domínio equivalente** (entidades, enums, endpoints) está em `backend/docs/09-modulos-dominio.md` — este documento é o espelho em telas. Arquitetura de camadas: [`./01-arquitetura.md`](./01-arquitetura.md). Auth/permissões: [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md).

Convenções usadas neste documento:

- Cada módulo vive em `src/app/(modules)/(dominio)/` com `_business/` (actions/hooks/schemas/helpers) + `(web)/` (pages/_components). Ver [`./01-arquitetura.md`](./01-arquitetura.md) §3.
- **Server Component por padrão** para listagens e fetch inicial (com `<Suspense>`); formulários, dialogs e interatividade são ilhas `"use client"` orquestradas por hooks `useXxx`.
- Todo fetch passa por `import { api } from "@/infra"` (token resolvido no servidor via cookies). Listagens seguem `?page=&perPage=&sort=&order=&search=&filters[...]` ([`./08-consumo-api-dados.md`](./08-consumo-api-dados.md)).
- O **prefixo HTTP** da API é `/api`; os paths abaixo já o incluem.
- Acesso por **10 grupos** (1–10) via `<PermissionGateServer>`/`<PermissionGateClient>` + `hasAccess(session, requirement)` — strings de role/módulo/ação **nunca** hardcoded; usar enums em `@/shared`. Ver [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md).
- "Fase Fxx" indica o épico do Linear (projeto **wecorp-frontend**, issues HUB-225–310 + portal público HUB-319–323).
- Após mutação que altera dados do servidor: `revalidatePath('/rota')` no hook + `toast` (sonner).

---

## Índice

| # | Módulo | Telas principais | Parte spec | Fase Linear |
|---|--------|------------------|-----------|-------------|
| 1 | [Empresas](#1-empresas-multi-tenant) | `/empresas`, `/empresas/new`, `/empresas/:id`, `/empresas/:id/modulos` | Parte 2 | F04 |
| 2 | [Usuários e Autenticação](#2-usuários-e-autenticação) | `/login`, `/termos-eula`, `/usuarios`, `/usuarios/new`, `/autocadastro` | Parte 3 | F01 / F04 |
| 3 | [Dashboard](#3-dashboard) | `/dashboard` (3 variantes por grupo) | Parte 4 | F05 |
| 4 | [Análise Cadastral (CORE)](#4-análise-cadastral-core) | `/analises`, `/analises/new`, `/analises/:id`, `/analises/:id/analise/:pessoaId`, `/analises/rh` | Parte 5 | F06 |
| 5 | [Pessoas](#5-pessoas-locatários-fiadores-sócios) | `/pessoas`, `/pessoas/new`, `/pessoas/:id` | Parte 6 | F06 |
| 6 | [Vistorias](#6-vistorias) | `/vistorias`, `/vistorias/new`, `/vistorias/:id`, `/vistorias/:id/revisao/:comodoId` | Parte 7 | F07 |
| 7 | [Seguros](#7-seguros-fiança-incêndio-cap-condomínio) | `/seguros/fianca`, `/seguros/cap`, `/seguros/incendio`, `/seguros/inspecao-predial`, `/seguros/condominio` | Parte 8 | F09 |
| 8 | [Imóveis](#8-imóveis) | `/imoveis`, `/imoveis/new`, `/imoveis/:id` | Parte 9 | F08 |
| 9 | [Financeiro](#9-financeiro) | `/financeiro/dashboard`, `/financeiro/contas-pagar`, `/financeiro/contas/:id`, `/financeiro/plano-contas` | Parte 10 | F10 |
| 10 | [CRM](#10-crm) | `/crm`, `/crm/leads`, `/crm/visitas`, `/crm/propostas` | Parte 11 | F11 |
| 11 | [Assinatura Eletrônica (Sign)](#11-assinatura-eletrônica-sign) | `/sign`, `/sign/new`, `/sign/:id`, `/sign/pesquisa` | Parte 12 | F12 |
| 12 | [Sinistros](#12-sinistros) | `/sinistros`, `/sinistros/gerencia`, `/sinistros/:id` | Parte 13 | F13 |
| 13 | [Garantidora](#13-garantidora) | `/garantidora`, `/garantidora/apolices`, `/garantidora/faturas`, `/garantidora/parcelas` | Parte 14 | F13 |
| 14 | [Prospects e Cotações (WBS)](#14-prospects-e-cotações-wbs) | `/prospects/cotacoes`, `/prospects/empresas-produtos`, `/prospects/integradores` | Parte 15 | F14 |
| 15 | [Marketplace de Vistorias](#15-marketplace-de-vistorias) | `/marketplace`, `/marketplace/rede`, `/marketplace/solicitar/:empresaId` | Parte 16 | F14 |
| 16 | [Editor de Contratos](#16-editor-de-contratos) | `/contratos/templates`, `/contratos/templates/new`, `/contratos/templates/:id/preview` | Parte 17 | F15 |
| 17 | [Comunicação](#17-comunicação) | `/comunicacao/sms/historico`, `/comunicacao/whatsapp/historico` (+ modais) | Parte 18 | F16 |
| 18 | [Suporte e CMS](#18-suporte-e-cms) | `/suporte`, `/suporte/tickets`, `/cms/paginas`, `/cms/categorias` | Parte 19 | F16 |
| 19 | [Configurações Globais e Lookups](#19-configurações-globais-e-lookups) | `/config/dominios`, `/config/grupos/:id/permissoes`, `/config/modules` | Parte 20 | F17 |
| 20 | [Relatórios](#20-relatórios) | `/financeiro/relatorios` + relatórios por domínio | Partes 10/22 | F18 |
| 21 | [Portal Público do Proponente](#21-portal-público-do-proponente) | `/proposta-fianca/:token`, `/analise/:token`, `/validar/...`, upload | F20 (novo) | F20 |

> Convenções de UI (Card, `<If>`, DataTable, máscaras), validação (RHF+Zod) e tabelas estão em [`./04-design-system-ui.md`](./04-design-system-ui.md), [`./10-formularios-validacao.md`](./10-formularios-validacao.md) e [`./11-tabelas-listagens.md`](./11-tabelas-listagens.md).

---

## 1. Empresas (Multi-tenant)

**Objetivo no produto.** Cadastrar e administrar as empresas que operam na plataforma (imobiliárias, corretoras, administradoras, seguradoras e fornecedoras), definindo módulos contratados, condições comerciais, white-label e flags de fornecimento (`fornece_seguros/analises/vistorias`). É a base do multi-tenant.

**Spec:** Parte 2 · **Fase Linear:** F04 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §1.

### Telas / rotas

| Rota | Tipo | O que faz |
|------|------|-----------|
| `/empresas` | Server (lista) | DataTable server-side: id, nome, documento, tipo (PF/PJ), status (badge), módulos (chips) e ações (ver, editar, módulos, logo, follow-ups). Filtros: nome, documento, status, fornecedor, tipo. Botão **Exportar** (XLSX/CSV/PDF) e **Nova empresa** (sob permissão). |
| `/empresas/new` | Client (form) | Formulário em **12 seções colapsáveis**: Identificação, CRECI, Contatos, Endereço (ViaCEP), Adm. de condomínios (condicional `realiza_adm`), Visão geral, Responsáveis, Comercial (flags `fornece_*`/`empresa_resp_*`), Customizações e Notificações, Conta indenização/comissão, Integrações (Docusign/ClickSign/integrador), Senha portal. |
| `/empresas/:id` | Server + ilhas | Visualização read-only das mesmas seções + abas **Usuários**, **Módulos**, **Filiais**, **Histórico de follow-ups**, **Anexos**. |
| `/empresas/:id/modulos` | Client | Grid de checkboxes de `Modulo` (Assessoria, Vistoria, Seguros, Financeiro, Sign, Marketplace, Relatórios…). |

### Fluxos e estados

- **Criar/editar:** schema Zod com `discriminatedUnion('tipo_pessoa', [PF, PJ])`; máscaras de CPF/CNPJ, CEP, telefone, moeda; ViaCEP preenche endereço; ao salvar → `revalidatePath('/empresas')` + toast.
- **Estados:** *loading* via `<Suspense>`/skeleton de tabela; *empty* com `<EmptyState>` ("Nenhuma empresa encontrada"); *error* tratado por `code`/`status` com `resolveErrorMessage`.
- **Segredos** (`senha_docusign`, contas bancárias) nunca renderizados em claro nem logados.

### Endpoints consumidos

`GET /api/empresas` · `GET /api/empresas/:id` · `POST /api/empresas` · `PUT /api/empresas/:id` · `DELETE /api/empresas/:id` · `POST /api/empresas/:id/logo` · `GET|PUT /api/empresas/:id/modulos` · `GET /api/empresas/lookup` · `GET /api/empresas/:id/dashboard`.

### Permissões

SuperAdmin (1) e Admin/imobiliária master (2); operação interna senior (3/9) para leitura. Grupos 6–8/10 não administram empresas. `PermissionGate` por módulo **Admin**.

---

## 2. Usuários e Autenticação

**Objetivo no produto.** Autenticar (login multi-tenant, SSO por token, EULA) e gerir usuários com o RBAC dos **10 grupos** herdados do legado. As telas de login/EULA vivem no route group `(auth)/` (não autenticado); a gestão de usuários no painel.

**Spec:** Parte 3 · **Fase Linear:** F01 (auth/login/sessão) e F04 (gestão de usuários) · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §2.

### Telas / rotas

| Rota | Tipo | O que faz |
|------|------|-----------|
| `/login` | Server + form client | Login multi-tenant: logo por host (`--logo-url`), campos usuário/senha, "lembrar-me", captcha opcional, box lateral da marca. 5+ layouts (`login_layout*`). Submete a `POST /api/auth/login`; em sucesso grava cookie httpOnly e redireciona. |
| `/termos` · `/termos-eula` | Server + ilha | Texto do EULA + checkbox "Li e aceito" + "Aceitar e continuar" (`POST /api/auth/aceitar-eula`). Bloqueante para grupo 2. |
| `/dashboard` | Server | Resolve a variante de dashboard pelo `grupo_id` (ver §3). |
| `/usuarios` | Server (lista) | DataTable: id, nome, empresa, grupo, login, status, último acesso, ações. Filtros (nome, login, e-mail, grupo, status, empresa); ação em massa ativar/desativar; Exportar. |
| `/usuarios/new` · `/usuarios/:id/edit` | Client (form) | Seções: Empresa e Grupo, Dados pessoais, Credenciais (username, senha, senha app vistoria), Endereço, Customizações. Validação cross-field grupo×empresa (6/7/8/10 não viram 1); `email_boas_vindas=1` dispara e-mail ao salvar. |
| `/autocadastro` | Public form | Auto-cadastro público quando `flag_permite_autocadastros=true`; cria usuário `status=1` (aprovação configurável). |

### Fluxos e estados

- **Login → refresh → logout:** o `proxy.ts` cuida de rotas públicas, refresh proativo do token e redirects (sem token → `/login`; token em rota pública → home). Sessão é fonte única via `getServerSession()` — **nunca** espelhada no Zustand.
- **Erro de login:** mensagem genérica (anti-enumeração) tratada por `error.code = INVALID_CREDENTIALS`.
- **Estados:** form com `disabled`/spinner durante submit; toasts de sucesso/erro; skeleton na lista.

### Endpoints consumidos

`POST /api/auth/login` · `GET /api/auth/login-token` · `POST /api/auth/login-api` · `POST /api/auth/refresh` · `POST /api/auth/logout` · `GET /api/auth/me` · `POST /api/auth/aceitar-eula` · `GET /api/usuarios` · `GET /api/usuarios/:id` · `POST /api/usuarios` · `PUT /api/usuarios/:id` · `DELETE /api/usuarios/:id` · `POST /api/autocadastro`.

### Permissões

Login/EULA: público/autenticado. Gestão de usuários: SuperAdmin (1) e Admin (2); o seletor de empresa/grupo é filtrado conforme o grupo do operador.

---

## 3. Dashboard

**Objetivo no produto.** Painel pós-login que muda conforme o `grupo_id`: operacional (cards + 6 gráficos), integrador (docs da API) ou boas-vindas (imobiliárias parceiras).

**Spec:** Parte 4 · **Fase Linear:** F05 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §3.

### Telas / rotas

`/dashboard` resolve no servidor uma das três variantes:

| Variante | Grupos | Conteúdo |
|----------|--------|----------|
| `OperacionalDashboard` | 1–4, 9 | 4 cards (vistorias em andamento/enviadas/processamento, análises em andamento), card de **repasses do mês** (total previsto + filtro de período + últimos repasses) e **grid de 6 gráficos** Recharts: fiança por seguradora (pizza), fiança por status (barras), CAP por status, incêndio cotados×emitidos, vistorias por status (pizza), sign cadastrados×concluídos. Auto-refresh 60s. |
| `IntegradorDashboard` | 5 | Card "Documentação da API" (link WBS), métricas de prospects/cotações, últimos retornos de integração. |
| `WelcomeImobHelp` | 6–8, 10 | Boas-vindas, atalhos (cards), tutoriais, status de módulos contratados (chips), link de atendimento. |

### Fluxos e estados

- Fetch inicial no Server Component; gráficos/cards client com refetch (TanStack Query) e `period` em estado de UI. *Empty* por card; *error* por boundary do card.
- Form modal "Solicitar apólice individual" → `POST /api/dashboard/solicita-apolice`.

### Endpoints consumidos

`GET /api/dashboard/metrics` · `GET /api/dashboard/charts?period=` · `GET /api/dashboard/repasses?from=&to=` · `POST /api/dashboard/solicita-apolice`. Períodos: `mes_atual`, `mes_anterior`, `3_meses`, `6_meses`, `12_meses`.

### Permissões

Todos os grupos autenticados (variante decidida pelo grupo). Cards/gráficos respeitam o escopo do tenant.

---

## 4. Análise Cadastral (CORE)

**Objetivo no produto.** **Módulo mais importante e complexo.** Cria fichas de análise de crédito de locatários/fiadores (e RH/sindicância), com pareceres por pessoa e por item, consulta a bureaus (Serasa/Procob), classificação de risco e geração de PDF. É o núcleo da operação de assessoria.

**Spec:** Parte 5 · **Fase Linear:** F06 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §4. Workflow/estados: spec §5.1 e §5.9.

### Workflow e status

Rascunho (1) → Em andamento (2) → Pendências (3) → Concluída (4) → PDF; ramo Cancelada (5); estado de fila Processar (6). O **parecer por pessoa** (`parecer_analisepessoa_statusid`) usa a mesma enumeração. Tipos: Expressa (E) × Completa (C); relacionamentos: Pretendente Locatário (4), Fiador (5), Cônjuge (6), Sócio (9), Funcionário/RH (18).

### Telas / rotas

| Rota | Tipo | O que faz |
|------|------|-----------|
| `/analises` | Server (lista) | DataTable: id, pessoa principal, documento, tipo (E/C), status (badge), empresa, data, ações. Filtros: nome, documento, status, empresa parceira, período. Dropdown **Nova análise** com 4 variantes (Locatário PF/PJ, Fiador PF/PJ). Exportar. |
| `/analises/new` | Client (form longo) | Formulário em **9+ cards colapsáveis**: Dados do Imóvel; Pessoa Principal (PF/PJ); Cônjuge (condicional ao estado civil); Sócios (só PJ, tabela dinâmica, sem análise própria); Locatários Solidários (checkbox `compoe_renda`); Patrimônios (Imóveis/Veículos/Outros, 3 abas); Referências Pessoais (mín. 2)/Comerciais (mín. 1)/Bancárias (mín. 1); Forma de pagamento e envio de link; Itens de Análise selecionados. **Salvar Rascunho** + **Salvar e Enviar para Análise**. **Autosave a cada 30s** em rascunho. |
| `/analises/:id` | Server + abas | Header com dados/status/ações (editar, enviar, gerar PDF, excluir). Abas: Dados do Imóvel, Pessoas (principal/cônjuge/sócios/solidários), Patrimônios, Referências, Financeiro (valor/fatura/status pagamento), Histórico. |
| `/analises/:id/edit/:tipo` | Client | Reusa o form de criação filtrando a pessoa (`principal`/`conjuge`/`solidario`/`socio`/`novo`). |
| `/analises/:id/analise/:pessoaId` | Client | **Tela do operador** em 3 colunas: esquerda = navegação entre pessoas; centro = cards de **Itens de Análise** (título/descrição, `modelos_parecer_id`, `parecer_obs`, `status_analises_id`, `retorno_consulta_externa` readonly, botão "Consultar bureau"); direita = **Parecer Final** (parecer, observações por grupo, `classificacao_risco_id` + percentual, `data_entrega`, status) com **Salvar Parecer** + **Concluir Análise**. Alerta de documento duplicado (`checaDocumento`). |
| `/analises/:id/pdf?tipo=0\|1\|2` | Server (stream) | Gera/visualiza PDF (0=auto, 1=Completa, 2=Expressa); download + preview; arquivo `analise_{tipo}_{documento}_{YYYYMMDD}.pdf`. |
| `/analises/rh` · `/analises/rh/new` · `/analises/rh/:id/edit` · `/analises/rh/:id/pdf` | Server/Client | Fluxo RH/Sindicância (`tipoficha_analises_id=2`): form simplificado (candidato, cargo/salário, escolaridade/CTPS, 3 empregos anteriores, pagamento, envio de link). |

### Fluxos e estados

- **Pricing:** `tabela_analise_empresas` define valor por empresa+tipo; fallback `tipo_analises.valor`. Persiste `valor_original/cobrar/pagar`. Ao cobrar, o backend gera Financeiro/Fatura/boleto — a UI apenas reflete (aba Financeiro).
- **Autosave:** rascunho salvo a cada 30s e em `onBlur` de seção; indicador "salvo/há alterações" (dirty-state).
- **Bureaus:** botão "Consultar" com loading no card; rate limit (5 req/min) tratado por `code` com toast informativo.
- **Estados:** *loading* skeleton da lista/cards; *empty* por aba; *error* por `code`/`status`.

### Endpoints consumidos

`GET|POST /api/analises` · `GET|PUT /api/analises/:id` · `POST /api/analises/:id/enviar-execucao` · `GET|PUT /api/analises/:id/analise/:pessoaId` · `GET /api/analises/:id/pdf` · `POST /api/analises/checa-documento` · `GET /api/analises/lookup|tipos|itens` · `GET|POST|PUT|DELETE /api/analises/:id/socios[...]` · `POST /api/analises/:id/consultar-serasa|consultar-procob` · família `/api/analises/rh[...]`.

### Permissões

Criação/origem por corretores e imobiliárias (grupos 5–8, 10); **execução do parecer** por operacional/análise (1, 3, 4, 9). A tela do operador exige permissão de ação no módulo **Assessoria**.

---

## 5. Pessoas (Locatários, Fiadores, Sócios)

**Objetivo no produto.** Cadastro unificado de PF/PJ usadas como locatários, fiadores, sócios, cônjuges e candidatos de RH, com qualificação e referências. Alimenta Análise, Seguros e CRM.

**Spec:** Parte 6 · **Fase Linear:** F06 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §5.

### Telas / rotas

| Rota | Tipo | O que faz |
|------|------|-----------|
| `/pessoas` | Server (lista) | DataTable: id, nome, documento, tipo (PF/PJ), status, empresa, criado em; ações ver/editar/excluir. Filtros: nome, documento, tipo, status, origem. |
| `/pessoas/new` · `/pessoas/:id/edit` | Client (form) | 8 seções: Identificação (toggle PF/PJ, validação CPF/CNPJ, RG, nascimento, e-mails), Contato, Endereço (ViaCEP), Qualificação (estado civil, sexo, moradia, vínculo…), Renda (profissão, empresa, cargo, renda + extras), Cônjuge (condicional), Origem, Status/parecer. |
| `/pessoas/:id` | Server + abas | Read-only + abas Qualificação, Referências (Pessoais/Comerciais/Bancárias), Fichas vinculadas, Histórico. |

### Fluxos e estados

- Busca por documento (`buscar-por-documento`) para reaproveitar cadastro; cônjuge condicional ao estado civil; referências como tabelas dinâmicas (`tipo=1|2|3`).
- *loading/empty/error* padrão; toasts + `revalidatePath('/pessoas')`.

### Endpoints consumidos

`GET|POST /api/pessoas` · `GET|PUT|DELETE /api/pessoas/:id` · `GET /api/pessoas/buscar-por-documento` · `GET /api/pessoas/lookup` · `GET|PUT /api/pessoas/:id/qualificacao` · `GET|POST|PUT|DELETE /api/pessoas/:id/referencias[...]` · lookups de status/origem/tipos.

### Permissões

Operacional (1, 3, 4, 9) e imobiliárias/corretores (6, 7, 8, 10) dentro do seu escopo de empresa.

---

## 6. Vistorias

**Objetivo no produto.** Inspeção física de imóveis: cômodos, itens, fotos, laudo PDF e aditivos. Inclui **lock concorrente** (evita edição simultânea) e **modo demo** (acesso público temporário).

**Spec:** Parte 7 · **Fase Linear:** F07 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §6.

### Telas / rotas

| Rota | Tipo | O que faz |
|------|------|-----------|
| `/vistorias` | Server (lista) | DataTable: id, código, data, imóvel, locatário, status (badge), tipo, valor, ações. Filtros amplos; cards de KPI (em andamento/enviadas/processamento); **Nova vistoria**. |
| `/vistorias/new` | Client (wizard) | 5 steps: Tipo e dados básicos (partes locatário/proprietário/fiador/testemunha, data/hora) → Endereço (ViaCEP) → Chaves (local/qtd/obs) → Tipo de entrega → Confirmação com cálculo de valor. |
| `/vistorias/:id` | Server + abas | Header + ações (editar, finalizar, enviar, gerar laudo). Abas: Dados Básicos, Cômodos e Itens (DnD), Pessoas, Fotos (galeria com drag), Chaves, Histórico de Status (timeline), Aditivos, Laudo (preview+download). |
| `/vistorias/:id/revisao/:comodoId` | Client | Revisão por cômodo: cômodos em tabs (DnD), itens com estado (badge colorido)/qtd/obs (DnD), galeria de fotos, "Adicionar item/cômodo". Upload multipart, girar/reordenar fotos. |
| `/vistorias/:id/laudo` | Server (stream) | 3 botões (Expressa/Completa/Final) + preview iframe + download. |
| `/vistorias/:id/termos` | Client | Aceite de termos (checkbox + "Aceitar e continuar"). |
| `/vistorias/:id/editar-locadores` | Dialog | Edição inline das partes via busca por documento. |
| `/vistorias/demo/:token` | Public | Modo demo sem login (ver §21 e doc 17). |

### Fluxos e estados

- **Lock concorrente:** ao abrir a edição, heartbeat `POST /wait-lock` a cada 30s; se `lock_user_id` de outro usuário e lock recente, exibe aviso "Vistoria sendo editada por X"; lock expira em 5min; admin pode "Forçar desbloqueio"; ao sair, `release-lock`. Outro editor recebe `409 LOCKED`.
- **Cálculo de preço:** `(área × preço_m²) + Σ(item × qtd) × multiplicador_pacote` via `calcular-valor`.
- **Estados:** *loading* skeleton; upload com progress; *error* por `code`; otimismo controlado no DnD com rollback em falha.

### Endpoints consumidos

`GET|POST /api/vistorias` · `GET|PUT|DELETE /api/vistorias/:id` · `agendar|finalizar|enviar|aceitar-termos` · `comodos[...]` (+ `ordem`) · `comodos/:id/itens[...]` · `fotos` (upload/ordem/girar/delete) · `laudo` · `aditivos[...]` · `wait-lock`/`release-lock` · `calcular-valor` · lookups.

### Permissões

Vistoriadores/operacional (1, 3, 4) e imobiliárias (6, 7, 8). Forçar desbloqueio: apenas admin (1/2). Modo demo: público por token.

---

## 7. Seguros (Fiança, Incêndio, CAP, Condomínio)

**Objetivo no produto.** Núcleo comercial: 8 sub-produtos de seguro com ciclos de vida próprios. As telas principais cobrem **Fiança**, **Capitalização (CAP)**, **Incêndio (API Tokio)**, **Inspeção Predial/Condomínio**; SPA, Condo Vida e Condo Incêndio Conteúdo reusam o mesmo padrão.

**Spec:** Parte 8 · **Fase Linear:** F09 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §8.

### 7.1 Seguro Fiança — `/seguros/fianca`

Status: Rascunho(1) → Em análise(2) → Pré-aprovado(3) → Aprovado(4) / Recusado(5) / Cancelado(6) / Aguardando documentação(7) → Emitido(8). Seguradoras: Porto Seguro, Liberty, Pottencial, SulAmérica, Tokio Marine.

| Rota | O que faz |
|------|-----------|
| `/seguros/fianca` | Listagem: locatário, status (badge), valor fiança, apólice, data. |
| `/seguros/fianca/new` | Pré-cadastro (PF/PJ, dados básicos, link ao cliente). |
| `/seguros/fianca/:id/proposta` | Proposta completa: imóvel, locatário, fiador, coberturas, parcelas. |
| `/seguros/fianca/:id/parecer` | Parecer do operador (modelo Porto Seguro ou genérico). |
| `/seguros/fianca/:id/visualizar-parcelas` | Lista/edita parcelas. |
| `/seguros/fianca/:id/pdf-proposta` · `/pdf-fianca` | PDFs. |

Endpoints: `GET /api/seguros/fianca[/:id]` · `pre-cadastro` · `:id/proposta|parecer|validar|enviar|status` · `:id/pdf?tipo=` · `:id/parcelas[...]` · `:id/cliente-proponente` · `buscar-pessoa` · `planos` · `cotacoes` · lookups. **Fluxo:** Pré-cadastro → Proposta → Envio à seguradora → Análise → Parecer → Apólice emitida. Cotações chegam da **WBS O2** (ver §14).

### 7.2 Capitalização (CAP) — `/seguros/cap`

Status: Rascunho(1) → Em análise(2) → Transmitir(3) → Transmitida(4) → Finalizada(5) / Cancelada(6). Telas: `/seguros/cap` (lista), `/new`, `/:id/edit`, `/:id/liberar-transmissao`. Endpoints: CRUD `/api/seguros/cap` · `:id/liberar-transmissao` · `:id/clonar` · `:id/retorno-sign` · `status`. Integra com Sign (signatários `Capsegurocontrolador`).

### 7.3 Seguro Incêndio (API) — `/seguros/incendio`

Produtos: 130 Incêndio · 138 Aluguel · 140 Danos Elétricos. Status: Rascunho(1) → Confirmado(2) → Cotado(3) → Contratado(4) → Emitido(5) / Cancelado(6). Telas: `/seguros/incendio` (lista), `/new` (wizard 3 passos), `/:id` (detalhe), `/:id/efetivar-contratacao`. Endpoints: CRUD `/api/seguros/incendio` · `confirmar-cotacao` · `efetivar-cotacao` · `efetivar-contratacao` · `enviar-cotacao` · `exportar-documentos` · lookups (seguradoras/coberturas).

### 7.4 Inspeção Predial e Condomínio — `/seguros/inspecao-predial`, `/seguros/condominio`

Questionário multi-step (8+ seções: construção, pavimentos, blocos, unidades, elevadores, sistemas de proteção, funcionários, coberturas, histórico de sinistros). Telas: lista, `/new` (multi-step), `/:id/pdf`. Endpoints: CRUD `/api/seguros/inspecao-predial` · `:id/validar` · `:id/pdf` · lookups; Condomínio (produto 7): CRUD `/api/seguros/condominio` · `lookup`.

### Fluxos e estados (transversais)

- Schemas Zod compartilhados (front+back) com `discriminatedUnion('tipo_pessoa', [PF, PJ])`.
- Token público (`token`/`codigo_formulario_web`) para o cliente preencher a proposta (ver §21 / doc 17).
- *loading* por wizard step; *error* de cotação/transmissão por `code`; badges de status por produto.

### Permissões

Corretores (5, 7) e operacional de seguros (1, 3, 4, 9); parecer/transmissão restritos ao operacional. Escopo por `empresaparceira_id`.

---

## 8. Imóveis

**Objetivo no produto.** Carteira de imóveis (residenciais/comerciais) com fotos, anexos, características e proprietário; base para vistorias, análises e CRM.

**Spec:** Parte 9 · **Fase Linear:** F08 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §7.

### Telas / rotas

| Rota | Tipo | O que faz |
|------|------|-----------|
| `/imoveis` | Server (lista) | Toggle grid/tabela: thumbnail, código, endereço, tipo, área, valor aluguel, status (badge). Filtros: código, endereço, bairro, cidade, UF, tipo, status. |
| `/imoveis/new` | Client (form) | 8 seções: Endereço (ViaCEP), Características (tipo/área/quartos/banheiros/vagas), Valores, Descrição, Características extras (chips), Fotos (DnD reorder), Anexos (upload múltiplo), Proprietário. |
| `/imoveis/:id` | Server + abas | Galeria (carousel) + abas Características, Anexos, Histórico, Vistorias vinculadas, Análises vinculadas. |

### Endpoints consumidos

`GET|POST /api/imoveis` · `GET|PUT|DELETE /api/imoveis/:id` · `fotos[...]` (upload/ordem/delete) · `anexos[...]` · `fotos/download`·`anexos/download` (ZIP) · lookups (tipos/caracteristicas/lookup).

### Permissões

Imobiliárias (6, 7, 8) e operacional (1, 3, 4). Estados *loading/empty/error* padrão; download ZIP com feedback.

---

## 9. Financeiro

**Objetivo no produto.** Plano de contas, contas a pagar/receber (parcelamento + boleto), movimentações/conciliação, repasses, holdings, planejamento orçamentário e relatórios.

**Spec:** Parte 10 · **Fase Linear:** F10 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §9.

### Telas / rotas

| Rota | Tipo | O que faz |
|------|------|-----------|
| `/financeiro/dashboard` | Server | Cards (vencido, a vencer 7/14/21 dias, pagos no mês, receitas previstas), gráfico Previsto×Realizado (12 meses, Recharts), lista de próximos vencimentos. |
| `/financeiro/contas-pagar` · `/financeiro/contas-receber` | Server (lista) | DataTable: descrição, plano de contas, cliente/fornecedor, valor, parcela, vencimento, status (badge). Filtros amplos; ação em massa (autorizar/baixar/excluir); Exportar. |
| `/financeiro/contas/new` | Client (form) | Seções: Tipo (Pagar/Receber), Dados básicos, Cliente/Fornecedor (autocomplete), Plano de contas (autocomplete hierárquico), Parcelamento (N parcelas), Anexos. |
| `/financeiro/contas/:id` | Server + abas | Header + ações (editar, baixar, autorizar). Abas: Parcelas (baixa individual), Movimentações (conciliação), Documentos, Histórico. |
| `/financeiro/plano-contas` | Client | Árvore expansível, CRUD inline (modal), filtro por tipo (D/R). |
| `/financeiro/relatorios` | Server | Central de relatórios (ver §20). |

### Fluxos e estados

- **Baixa/autorização:** individual, em lote (`baixar-lote`) e por repasse (`baixar-repasse`); autorização separa criação de aprovação. Boletos exibem `linha_digitavel`/`boleto_url`.
- **Status de parcela:** Aberto(1)/Recebido(2)/Pago(3)/Vencido(4)/Cancelado(5) com badges.

### Endpoints consumidos

`/api/financeiro/plano-contas[...]` (+`arvore`/`lookup`/`buscar`) · `bancos`·`bancos-empresa` · `contas-pagar`·`contas-receber` · `contas/:id[...]` (`baixar`/`baixar-repasse`/`confirmar-baixa`/`autorizar`/`baixar-lote`/`parcelas`/`boleto`/`documentos`) · `movimentacoes[...]`·`conciliar` · `holdings`/`iofs`/`aliquotas-comissao`/`planejamento` · `relatorios/*`.

### Permissões

Admin financeiro (1, 2) e operacional senior (3, 9); autorização de pagamento restrita.

---

## 10. CRM

**Objetivo no produto.** Gestão comercial de locação/venda: leads (funil Kanban), visitas, propostas/negócios com garantias, proprietários e tarefas.

**Spec:** Parte 11 · **Fase Linear:** F11 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §10.

### Telas / rotas

| Rota | Tipo | O que faz |
|------|------|-----------|
| `/crm` | Server | Dashboard: funil Kanban por status, métricas (leads/mês, conversão, valor de propostas). |
| `/crm/leads` | Server (lista) | Toggle Tabela/Kanban; filtros (nome, e-mail, documento, status, corretor, origem, período). |
| `/crm/leads/new` | Client (form) | Dados, imóvel de interesse, observações. |
| `/crm/leads/:id` | Server + abas | Abas com CRUD inline: Dados, Visitas, Propostas, Tarefas, Histórico, Proprietários. |
| `/crm/visitas` | Client | Calendário (mês/semana/dia), agendamento rápido, filtro corretor/status. |
| `/crm/visitas/new` | Dialog | Lead/imóvel/corretor (autocomplete), data/hora, tipo, obs; "Agendar" e "Agendar e Enviar Confirmação". |
| `/crm/propostas` | Server | Cards: lead, imóvel, valor, status (badge), validade; filtros. |
| `/crm/propostas/new` | Client (form) | Lead, imóvel, valor, condições, garantias (tabela dinâmica); "Gerar PDF da Proposta". |
| `/crm/propostas/:id/modal-garantia` | Dialog | Adicionar/editar garantias (fiador, caução, seguro fiança…). |

### Endpoints consumidos

`/api/crm/leads[...]` (+`lead-por-email`) · `visitas[...]`·`:id/validar` · `propostas[...]`·`:id/garantias` · `proprietarios[...]` · `tarefas[...]` · lookups (status-lead/proposta/visita).

### Permissões

Corretores e imobiliárias (6, 7, 8, 10) + operacional (1, 3, 4). Kanban e calendário são ilhas client com filtros sincronizados na URL.

---

## 11. Assinatura Eletrônica (Sign)

**Objetivo no produto.** Gerir envelopes e signatários integrados a **Docusign**, **ClickSign** e **Autentique**, sincronizando status por webhooks. Vinculação polimórfica a qualquer documento (vistoria, CAP, ficha…).

**Spec:** Parte 12 · **Fase Linear:** F12 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §11.

### Telas / rotas

| Rota | Tipo | O que faz |
|------|------|-----------|
| `/sign` | Server (lista) | DataTable: título, tipo (badge), plataforma (logo), signatários (chips), status (badge), data envio, ações. Filtros amplos. |
| `/sign/new` | Client (wizard) | 6 steps: Tipo de documento → Vincular entidade (autocomplete) → Documento (upload PDF) → Signatários (tabela dinâmica: nome/e-mail/documento/telefone/ordem/tipo) → Provedor → Revisão + "Processar e Enviar". |
| `/sign/:id` | Server + abas | Header (título/status/provedor) + abas Documentos (galeria), Signatários (status individual), Histórico (timeline), Download. |
| `/sign/pesquisa` | Dialog | Modal para buscar/vincular envelopes a partir de outras telas. |

### Fluxos e estados

- Status do envelope (0–5) e do signatário (0–6) atualizados por **webhook** do provedor (`POST /api/sign/webhook/:provedor`), que dispara notificação in-app ao criador. A UI reflete por refetch/socket.
- *loading* por step; *error* de processamento por `code`; download dos assinados via `:id/downloads`.

### Endpoints consumidos

`GET|POST|PUT /api/sign[/:id]` · `:id/processar`·`confirmar-envio` · `:id/signers[...]` · `:id/docs[...]` · `:id/downloads` · `webhook/:provedor` (recebido) · lookups (tipos-documento/provedores/tipos-signer).

### Permissões

Operacional (1, 3, 4) e imobiliárias master (2, 6). Webhook é endpoint de servidor (sem UI).

---

## 12. Sinistros

**Objetivo no produto.** Comunicar e gerir sinistros vinculados a apólices, com documentação, indenização e visão administrativa.

**Spec:** Parte 13 · **Fase Linear:** F13 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §12.

### Telas / rotas

| Rota | Tipo | O que faz |
|------|------|-----------|
| `/sinistros` | Server (lista) | Visão do corretor: apólice, tipo, status (badge), data, valor; "Comunicar sinistro". |
| `/sinistros/gerencia` | Server | Visão admin: filtros avançados + cards de KPI por status. |
| `/sinistros/:id` | Server + form | Todos os campos, upload de documentos (múltiplos), timeline de status; botões Salvar, Alterar Status, Finalizar. |

Status: `pendente_analise`, `pendente_documentacao`, `indenizacao_andamento`, `finalizado_indenizacao`, `finalizado_negociacao`, `cancelado`.

### Endpoints consumidos

`GET|POST|PUT|DELETE /api/sinistros` · `gerencia` · `lista` · `gravar` · `:id/visualizar` · `lookup`.

### Permissões

Corretores (5, 7) abrem; gerência por operacional/admin (1, 2, 3, 9).

---

## 13. Garantidora

**Objetivo no produto.** Espelha o Seguro Fiança para gestão de faturas/parcelas entre imobiliárias e seguradoras, com registro de apólices.

**Spec:** Parte 14 · **Fase Linear:** F13 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §13.

### Telas / rotas

| Rota | O que faz |
|------|-----------|
| `/garantidora` | Dashboard. |
| `/garantidora/apolices` · `/apolices/new` · `/apolices/:id/parecer` | Listagem/cadastro (espelho do seguro fiança) + parecer. |
| `/garantidora/faturas` · `/faturas/new` · `/faturas/:id/imprimir` | Listagem, geração e preview/PDF de fatura. |
| `/garantidora/parcelas` | Listagem com totais por empresa; gerar fatura a partir de parcelas. |

Status da fatura: aberta · paga · vencida · cancelada.

### Endpoints consumidos

`/api/garantidora/apolices[...]` · `faturas[...]`·`:id/itens` · `parcelas[...]`·`:id/gerar-fatura`·`:id/imprimir-fatura` · `lookup`.

### Permissões

Seguradora/garantidora e admin (1, 2, 9); imobiliária vê suas faturas (escopo `cliente_empresa_id`).

---

## 14. Prospects e Cotações (WBS)

**Objetivo no produto.** Integração com a **WBS O2**: recebe prospects/cotações de seguros e gerencia integradores, seguradoras e produtos por empresa.

**Spec:** Parte 15 · **Fase Linear:** F14 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §14.

### Telas / rotas

| Rota | O que faz |
|------|-----------|
| `/prospects/cotacoes` | Listagem: nome, documento, produto, seguradora, status (badge), valor, data. Filtros (integrador, produto, status, qualificação, data). Botão "Exportar planilha SPA" (XLSX). |
| `/prospects/empresas-produtos` | Tabela cruzada empresa × integrador × produto, com toggle on/off por célula. |
| `/prospects/integradores` | CRUD de integradores WBS (token, URL, status). |

### Endpoints consumidos

`GET /api/prospects/cotacoes[/:id]` · `cotacoes/dados-empresa` · `cotacoes/gera-planilha-spa` · `empresas-produtos[...]` (+`ativar-produto`/`-integrador`/`-sub`) · lookups (integradores/seguradoras/produtos).

### Permissões

Integrador (5) e admin (1, 2). Cotações são read-only no painel; ativação de produto por admin.

---

## 15. Marketplace de Vistorias

**Objetivo no produto.** Rede de prestadores: imobiliárias solicitam vistorias a empresas vistoriadoras (`fornece_vistorias=true`), com proposta de valor, aceite, execução e avaliação.

**Spec:** Parte 16 · **Fase Linear:** F14 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §15.

### Telas / rotas

| Rota | O que faz |
|------|-----------|
| `/marketplace` | Vitrine de prestadores (cards: logo, nome, regiões, especialidades, avaliação); filtros estado/cidade/especialidade; "Solicitar". |
| `/marketplace/rede` | Listagem filtrada por `fornece_vistorias=true` (espelho de `/empresas`). |
| `/marketplace/solicitar/:empresaId` | Form: dados do imóvel, data desejada, valor proposto; "Enviar solicitação". |

Status: solicitada · aceita · em_execucao · concluida · cancelada.

### Endpoints consumidos

`GET /api/marketplace` · `marketplace/rede` · `POST /api/marketplace/solicitar` · `GET /api/marketplace/buscar?cnpj=`.

### Permissões

Imobiliárias (6, 7) solicitam; vistoriadoras aceitam/executam; admin (1, 2) gere a rede.

---

## 16. Editor de Contratos

**Objetivo no produto.** Templates de documentos (HTML com merge tags) editados em **TipTap** e renderizados **server-side em PDF** (mPDF), vinculáveis a entidades do sistema.

**Spec:** Parte 17 · **Fase Linear:** F15 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §16.

### Telas / rotas

| Rota | Tipo | O que faz |
|------|------|-----------|
| `/contratos/templates` | Server (lista) | DataTable: nome, tipo, descrição, versão, status, ações. Filtros nome/tipo/status. |
| `/contratos/templates/new` | Client (editor) | 3 colunas: esquerda = tags disponíveis (clica para inserir); centro = **editor rich text TipTap**; direita = propriedades (nome/tipo/descrição). "Salvar" + "Preview". |
| `/contratos/templates/:id/preview` | Server + ilha | Seletor de entidade (locação/imóvel + id) → renderiza template com tags substituídas; "Download PDF". |
| Modal de Seleção | Dialog | Acionado de outras telas (Locação, CRM): lista templates + busca + preview inline. |

### Fluxos e estados

- **Render server-side:** `POST /api/contratos/gerar-pdf` com `{ template_id, entity_type, entity_id }` → backend monta HTML + dados da entidade → PDF. O preview usa `:id/preview` (merge). O TipTap roda no client; o HTML final é processado no servidor (a regra de merge é do backend).
- *loading* no preview/geração; *error* por `code`.

### Endpoints consumidos

`/api/contratos/templates[...]` · `modal-search`·`:id/modal-grid`·`:id/modal-view`·`:id/tags` · `:id/preview` · `POST /api/contratos/gerar-pdf` · lookups (tipos/lookup).

### Permissões

Admin (1, 2) e operacional senior (3, 9) editam templates; demais consomem via modal de seleção.

---

## 17. Comunicação

**Objetivo no produto.** SMS, WhatsApp, encurtador de URL e validação/normalização de celular. Em grande parte **interno**, acionado por outros módulos via modais; expõe históricos.

**Spec:** Parte 18 · **Fase Linear:** F16 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §17.

### Telas / rotas

| Rota | O que faz |
|------|-----------|
| `/comunicacao/sms/historico` | Tabela: data, escopo, número, mensagem (preview), status. Filtros escopo/status/data. |
| `/comunicacao/whatsapp/historico` | Similar ao SMS (gateway_message_id, status). |
| Modais embutidos | "Enviar SMS/WhatsApp" acionados de Seguro Fiança, Vistoria, Ficha de análise, Visita. |

Status (Sms/Whatsapp): pendente · enviado · entregue · falhou.

### Endpoints consumidos

`POST /api/comunicacao/sms/preparar|enviar|gravar|processar` · `whatsapp/disparar` · `url/encurtar-bitly|encurtar-google` · `celular/validar|tratar` · históricos.

### Permissões

Acionado conforme o módulo de origem (escopo do recurso); históricos por operacional/admin.

---

## 18. Suporte e CMS

**Objetivo no produto.** Central de ajuda com tickets (thread de mensagens) e CMS de páginas/artigos (FAQ, base de conhecimento).

**Spec:** Parte 19 · **Fase Linear:** F16 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §18.

### Telas / rotas

| Rota | O que faz |
|------|-----------|
| `/suporte` | Central de ajuda: categorias (cards), FAQ accordion, "Abrir ticket" (modal). |
| `/suporte/tickets` | Meus tickets: id, assunto, categoria, status (badge), última atualização. |
| `/suporte/tickets/:id` | Thread de mensagens (chat UI) + input + anexos. |
| Elemento "Atendimento" (flutuante) | Botão fixo (canto inferior direito) que abre chat in-app e envia contexto (página, usuário). |
| `/cms/paginas` · `/cms/paginas/new` · `/cms/categorias` | CMS: listagem, editor TipTap, gestão de categorias hierárquicas. |

Status suporte: aberto · em_atendimento · aguardando_cliente · resolvido · fechado (prioridade baixa/media/alta/urgente). CMS: rascunho · publicado.

### Endpoints consumidos

`/api/suporte/tickets[...]` · `:id/mensagens` · `:id/status` · `/api/cms/categorias[...]` · `paginas[...]` · `paginas/:slug` (público).

### Permissões

Tickets abertos por qualquer usuário autenticado (escopo da empresa); atendimento e CMS por operacional/admin (1, 2, 3, 9).

---

## 19. Configurações Globais e Lookups

**Objetivo no produto.** CRUD das tabelas de domínio (lookups), gestão de módulos/controllers/actions e da **matriz de grupos × permissões**, além de metadados de leitura.

**Spec:** Parte 20 · **Fase Linear:** F17 · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` §19.

### Telas / rotas

| Rota | O que faz |
|------|-----------|
| `/config/dominios` | Hub de CRUD de lookups (status, classificações, tipos, itens de análise, modelos de parecer, bancos, categorias, tipos de imóvel, cidade/bairro…), todos no padrão `GET/POST/PUT/DELETE /api/config/{dominio}` (+`/lookup`). |
| `/config/grupos` | Listagem de grupos: id, nome, descrição, qtd usuários, ações. |
| `/config/grupos/:id/permissoes` | **Matriz de permissões**: linhas = `ModuleControllerAction`, colunas = níveis; checkbox por célula; salvar em massa. |
| `/config/permissions` | CRUD de permissões. |
| `/config/modules` (+ controllers/actions) | Hierarquia de menu/ações (Module → ModuleController → ModuleControllerAction). |

### Fluxos e estados

- Lookups alimentam selects de toda a app (consumidos via `useXxxLookup`/TanStack Query). Metadados (`/api/config/metadados/{chave}`) retornam `[{ id, descricao }]`.
- A matriz de permissões é a fonte que `hasAccess` consulta — alterações exigem revalidar sessão.

### Permissões

Exclusivo de SuperAdmin (1) (e Admin 2 para subconjuntos). Strings de módulo/ação/role via enums em `@/shared`.

---

## 20. Relatórios

**Objetivo no produto.** Central de relatórios por domínio (financeiro e operacionais), com filtros, preview e exportação (PDF/XLSX). Gerados server-side.

**Spec:** Partes 10 (Financeiro) e 22 (contrato) · **Fase Linear:** F18.

### Telas / rotas

- `/financeiro/relatorios` — central com 10+ relatórios pré-definidos; cada um: filtros + "Gerar" + preview + download (PDF/XLSX).
- Relatórios embarcados em outros módulos via botão **Exportar** nas listagens (Empresas, Usuários, Análises, Financeiro, Prospects…).

### Endpoints consumidos

`GET /api/financeiro/relatorios/analise-pagamentos|plano-contas|fechamento-mensal|repasse-mes-produtor` · `POST /api/financeiro/relatorios/imprimir` (PDF) · export das listagens.

### Permissões

Operacional/admin conforme o domínio do relatório; financeiro restrito (1, 2, 3, 9).

---

## 21. Portal Público do Proponente

**Objetivo no produto.** Superfícies **sem login** acessadas por **link com token** (uso único + expiração), no route group `(public)/`. Permitem ao proponente (inquilino/fiador) preencher proposta de fiança, completar a análise, validar autenticidade de documentos e enviar anexos. Detalhe completo em [`./17-paridade-portal-publico.md`](./17-paridade-portal-publico.md).

**Spec:** referências espalhadas (Análise §5.8, Vistoria §7.6, Seguro §8) · **Fase Linear:** F20 (HUB-319–323) · **Domínio (backend):** `backend/docs/09-modulos-dominio.md` "Portal do Proponente e Áreas Públicas".

### Telas / rotas (sem login, em `(public)/`)

| Rota | O que faz |
|------|-----------|
| `/proposta-fianca/:token` | Proponente preenche/dá sequência à proposta de Seguro Fiança (dados da pessoa, envio, PDF). |
| `/analise/:token` | Pretendente completa a ficha de análise (dados, itens, concluir); senha = `codigo_formulario_web`. |
| `/vistorias/demo/:token` | Modo demo de vistoria (aceitar termos, preencher cômodos). |
| `/validar/analise/:codigo` · `/validar/vistoria/:codigo` | Validação pública de autenticidade do documento (token no rodapé do PDF). |
| Upload/download de documentos | Anexos do proponente via token. |

### Fluxos e estados

- O **tenant e o recurso vêm do token** (nunca de input do usuário); white-label aplicado por host; respostas genéricas (anti-enumeração); reenvio idempotente; expiração por TTL.
- *loading* otimizado (LCP), *empty/expired* com mensagem clara ("Link expirado"), *error* genérico.

### Endpoints consumidos

`GET|POST /api/public/proposta-fianca/:token` · `GET|POST /api/public/analise/:token` · `GET /api/public/validar/analise|vistoria/:codigo` · `GET /api/public/enderecos/*` · `POST|GET|DELETE /api/public/arquivos/:token` · `POST /api/public/auth/forgot-password|reset-password`.

### Permissões

Sem grupo (público por token). Guard `@PublicToken` no backend; CORS restrito aos domínios white-label.

---

## Ver também

- [`backend/docs/09-modulos-dominio.md`](../../backend/docs/09-modulos-dominio.md) — domínio equivalente (entidades, enums, endpoints) por contexto.
- [`./01-arquitetura.md`](./01-arquitetura.md) — camadas, route groups, Server vs Client.
- [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md) — 10 grupos, `PermissionGate`, `hasAccess`.
- [`./08-consumo-api-dados.md`](./08-consumo-api-dados.md) — envelope, erros por `code`, TanStack Query, `revalidatePath`.
- [`./10-formularios-validacao.md`](./10-formularios-validacao.md) · [`./11-tabelas-listagens.md`](./11-tabelas-listagens.md) — formulários (RHF+Zod, autosave) e DataTable.
- [`./16-glossario.md`](./16-glossario.md) — termos de domínio e de frontend.
- [`./17-paridade-portal-publico.md`](./17-paridade-portal-publico.md) — telas do portal público (F20).
- Spec de telas/contrato: `C:\projetos\wecorp\frontend.md` (Partes 2–20).
</content>
</invoke>
