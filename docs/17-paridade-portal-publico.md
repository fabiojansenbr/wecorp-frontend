# 17 — Paridade e Portal Público do Proponente

> Como garantir que o frontend **faz o que o legado fazia** (sem cópia tela-a-tela) e como o **portal público do proponente** (F20) se encaixa nesse contrato.
> Companion no backend: [`backend/docs/17-paridade-legado.md`](../../backend/docs/17-paridade-legado.md) e a matriz viva [`backend/docs/parity-matrix.md`](../../backend/docs/parity-matrix.md).

---

## 1. Nível de paridade adotado: **equivalência funcional**

O objetivo é **mesmas funcionalidades e mesmas regras de negócio** do legado, com **liberdade total para modernizar a apresentação**:

| Pode mudar | Deve permanecer equivalente |
|---|---|
| Layout, UX, fluxo de navegação, número de telas/steps | Funcionalidades disponíveis ao usuário (o que dá para fazer) |
| Stack, componentes, design system, tipografia/cores | Regras e validações de domínio (refletidas na borda via Zod) |
| Agrupar/dividir telas, transformar página em wizard | Permissões dos **10 grupos** (o que cada perfil vê/faz) |
| Corrigir bugs e inconsistências de UX do legado | Estados e transições possíveis (máquinas de estado refletidas) |
| Trocar máscara/validador/biblioteca de UI | Dados que entram e saem de cada operação (contrato do backend) |

**Não é golden-master de tela:** não exigimos a mesma `.ctp`, o mesmo HTML, nem a mesma disposição de campos pixel-a-pixel. Exigimos o **mesmo efeito de negócio** e a **mesma capacidade funcional** para cada grupo.

> A **regra de negócio crítica vive no backend** (cálculo de prêmio, IOF, comissão, máquina de estado). O frontend garante paridade de **funcionalidade, permissão e fluxo**; consome o contrato e não reimplementa regra crítica. Paridade ≠ replicar defeitos: bug legado claro é corrigido e a decisão registrada na issue.

## 2. Como a paridade é verificada (frontend)

Para a maioria das telas, paridade é verificada pelos **critérios de aceite da issue** + um critério explícito de paridade com a tela legada correspondente:

> **Paridade funcional (frontend):** a tela oferece as mesmas funcionalidades, respeita as mesmas permissões dos 10 grupos e reflete os mesmos estados/transições da(s) tela(s) legada(s) correspondente(s) (ver matriz). Divergências intencionais de UX documentadas na issue.

Esse critério é **Definition of Done universal** ([`../CLAUDE.md`](../CLAUDE.md) §10, último item). A referência legada de cada tela está na coluna "Legado" da [`backend/docs/parity-matrix.md`](../../backend/docs/parity-matrix.md) (a matriz é única para os dois repos — mapeia cada controller/plugin legado → issue de backend e de frontend).

### Critérios de aceite recorrentes (todas as telas)

- Funcionalidades equivalentes à tela legada disponíveis ao(s) grupo(s) corretos (`PermissionGate` + `hasAccess`).
- Estados **loading / empty / error** cobertos; feedback via `sonner`.
- Validação PT-BR (RHF + Zod) espelhando as regras do backend; erro tratado por `code`/`status`.
- Mobile-first e WCAG 2.1 AA.
- Tenant/white-label respeitado (branding por host).

## 3. Onde mora a verificação de alto risco

O frontend **não** refaz os testes de caracterização de cálculo/PDF/estado — esses são do backend (HUB-330, ver [`backend/docs/17-paridade-legado.md`](../../backend/docs/17-paridade-legado.md) §5). No frontend, o equivalente "de alto risco" é:

| Área sensível no frontend | O que verificar | Como |
|---|---|---|
| **`hasAccess` / RBAC dos 10 grupos** | cada grupo vê/faz exatamente o que via no legado | testes de todos os branches ([13](./13-testes.md) §4) |
| **Máquinas de estado refletidas** (análise, vistoria/lock, fiança, sign) | só as transições permitidas aparecem como ação na UI | testes de componente + e2e |
| **Fluxos críticos** | login, análise cadastral, contratação de seguro, vistoria funcionam ponta a ponta | e2e Playwright ([13](./13-testes.md) §6) |
| **Validações de borda** | mesmas regras do backend, mensagens PT-BR | testes de schema Zod ([13](./13-testes.md) §3) |

## 4. Escopo expandido: o portal público

O backlog inicial veio do `frontend.md`, que cobria **apenas o painel admin**. A auditoria de paridade identificou que o legado também expunha **superfícies públicas** ao proponente (inquilino/fiador) — sem login. Decisão: **incluir agora**, como milestone **F20 — Portal Público do Proponente** (frontend), pareado com o **backend F24** (HUB-311..318).

Características das telas públicas:

- **Sem login** — acesso por **link com token** (uso único / expiração).
- **White-label** — branding aplicado por host (mesma engine de tema do painel; ver [06](./06-multitenancy-whitelabel.md)).
- **Mobile-first** — o proponente costuma acessar pelo celular.
- Vivem em `src/app/(public)/` (route group fora de `(modules)`, sem `proxy.ts` de sessão autenticada).

## 5. Telas do portal público (F20) e issues

| Issue | Tela pública | O que entrega | Backend consumido |
|-------|--------------|---------------|-------------------|
| **HUB-319** | **Shell público white-label** | Layout/base das páginas públicas + **autocadastro** do proponente + aceite de **termos/EULA**; branding por host; estados de token. | Infra de token público (**HUB-311**); reset de senha (**HUB-313**) |
| **HUB-320** | **Proposta de Seguro Fiança** (multi-step) | Formulário público **multi-step** de proposta de fiança preenchido pelo proponente. | Proposta pública de fiança (**HUB-314**); proponente da garantidora (**HUB-315**); endereços/CEP (**HUB-312**) |
| **HUB-321** | **Análise Cadastral pública** | Formulário público de **preenchimento da análise pelo pretendente** (equivale ao `codigo_formulario_web` do legado). | Análise pública (**HUB-316**); endereços/CEP (**HUB-312**) |
| **HUB-322** | **Validação de autenticidade** | Páginas públicas de **validação** de documentos (análise/vistoria) — confere autenticidade por código/token. | Validação pública (**HUB-317**) |
| **HUB-323** | **Upload público de documentos** | **Componente reutilizável** de upload **token-gated** de documentos, usado pelas demais telas públicas. | Upload/download token-gated (**HUB-318**) |

> O mapa público ↔ issues está consolidado em [`backend/docs/parity-matrix.md`](../../backend/docs/parity-matrix.md) §16. Endereços/CEP (HUB-312) não é tela própria — é **consumido** pelos formulários (autocomplete de endereço). Reset de senha (HUB-313) é fluxo do shell/auth, não tela isolada.

## 6. Dependência do backend F24

Cada tela pública consome um endpoint público do **backend F24** (HUB-311..318). O frontend **não** implementa a lógica de token, escopo ou validação — apenas consome e cuida da UX, validação de borda e estados:

| Backend (F24) | Função pública | Telas frontend (F20) |
|---|---|---|
| **HUB-311** | Infra de token público (uso único/expiração/escopo) | base de todas (HUB-319..323) |
| **HUB-312** | API de endereços (CEP/UF/cidades/bairros) | HUB-320, HUB-321 (autocomplete) |
| **HUB-313** | Reset de senha por token (Users API) | HUB-319 (fluxo de auth/recuperação) |
| **HUB-314** | Proposta pública de Seguro Fiança | HUB-320 |
| **HUB-315** | Portal público do proponente da Garantidora | HUB-320 (base) / HUB-319 (shell) |
| **HUB-316** | Preenchimento público de Análise Cadastral | HUB-321 |
| **HUB-317** | Validação pública de autenticidade (análise/vistoria) | HUB-322 |
| **HUB-318** | Upload/download token-gated de documentos | HUB-323 |

Ver detalhamento das fases pareadas em [`./15-roadmap-fases.md`](./15-roadmap-fases.md) (F20) e [`backend/docs/15-roadmap-fases.md`](../../backend/docs/15-roadmap-fases.md) (F24).

## 7. Regras das telas públicas (MUST)

- **Acesso por link com token** — a página só carrega com um token válido; token de **uso único** e/ou **com expiração**. Sem token válido → estado de erro, nunca o formulário.
- **Estados de token explícitos:** **válido** (renderiza), **inválido** (mensagem genérica, sem revelar existência de dado), **expirado** (oferece reenvio/contato), **já utilizado** (uso único consumido).
- **Branding por host** — a marca é detectada pelo domínio (mesmo mecanismo white-label do painel); cores via variáveis CSS, nunca hex inline.
- **Mobile-first** — layout pensado primeiro para o celular do proponente.
- **Sem dados sensíveis expostos** — a tela pública só recebe do backend o estritamente necessário ao preenchimento; nada de listas, IDs internos, dados de outros proponentes ou de outro tenant. O backend filtra por escopo do token; o frontend não solicita nem exibe além disso.
- **Sem sessão autenticada** — `(public)` fica fora do `proxy.ts` de auth; não há cookie de sessão do painel, apenas o token do link.
- **Upload token-gated** (HUB-323) — o componente de upload só aceita arquivos com token válido; tipos/limites validados na borda e confirmados pelo backend.

## 8. Sign-off de paridade (frontend)

Um módulo do frontend está "em paridade" quando:

- [ ] As funcionalidades das telas legadas correspondentes (coluna "Legado" da matriz) estão disponíveis aos grupos corretos.
- [ ] Critérios de aceite das issues atendidos (inclui o critério de paridade funcional, §2).
- [ ] `hasAccess` cobre todos os branches; permissões dos 10 grupos verificadas.
- [ ] Máquinas de estado refletidas (só transições válidas viram ação na UI).
- [ ] Fluxos críticos com e2e verde; sem violações `axe`.
- [ ] Divergências intencionais de UX (telas reorganizadas, bugs corrigidos, libs trocadas) documentadas na issue.
- [ ] (F20) Telas públicas respeitam as regras §7 e consomem os endpoints F24.

## Ver também

- [`backend/docs/17-paridade-legado.md`](../../backend/docs/17-paridade-legado.md) — metodologia de paridade (companion).
- [`backend/docs/parity-matrix.md`](../../backend/docs/parity-matrix.md) — matriz viva legado→issue (§16 áreas públicas).
- [`./15-roadmap-fases.md`](./15-roadmap-fases.md) — F20 (portal público) e dependências do backend F24.
- [`./07-auth-sessao-permissoes.md`](./07-auth-sessao-permissoes.md) — `hasAccess`, RBAC dos 10 grupos.
- [`./06-multitenancy-whitelabel.md`](./06-multitenancy-whitelabel.md) — branding por host.
- [`./13-testes.md`](./13-testes.md) — verificação (schemas, `hasAccess`, e2e, axe).
</content>
</invoke>
