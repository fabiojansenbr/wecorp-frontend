# Base de conhecimento do legado — Frontend weCorp

> **O que é:** dossiês que destilam **como as telas/fluxos do sistema legado (CakePHP 2.x, views `.ctp` em
> `C:\projetos\wecorp\app`) funcionavam** — para reconstruir a UI nova com **equivalência funcional**.
>
> **Como usar:** abra **só o dossiê do módulo** da sua issue. Dúvida profunda → skill **`/legacy-lookup "<pergunta>"`**.
> Mapeamento controller/view legados na [`../../../backend/docs/parity-matrix.md`](../../../backend/docs/parity-matrix.md).
>
> **Anti-context-rot:** referência **on-demand**; um dossiê por tarefa; ~≤500 linhas.
>
> **Fonte (read-only):** `C:\projetos\wecorp\app` (views `.ctp` + controllers). Excluir: `Vendor/`, `lib/Cake/`, `*~`, `*-bkp.*`, `*.zip`.

| Módulo de tela | Fase | Dossiê | Status |
|---|---|---|---|
| Login, sessão e permissões (white-label) | F01/F03 | `auth-login-whitelabel.md` | ✅ |
| Empresas | F04 | `empresas.md` | ✅ |
| Usuários e autocadastro | F03 | `usuarios.md` | ✅ |
| Pessoas | F05 | `pessoas.md` | ✅ |
| Análise cadastral (telas + tela do operador) | F06 | `analise-cadastral.md` | ✅ |
| Vistorias | F07 | `vistorias.md` | ✅ |
| Imóveis | F08 | `imoveis.md` | ✅ |
| Seguros (fiança/incêndio/capitalização) | F09 | `seguros.md` | ✅ |
| Financeiro | F10 | `financeiro.md` | ✅ |
| CRM | F11 | `crm.md` | ✅ |
| Assinatura eletrônica | F12 | `assinatura-eletronica.md` | ✅ |
| Sinistros | F13 | `sinistros.md` | ✅ |
| Garantidora | F14 | `garantidora.md` | ✅ |
| Prospects/cotações | F15 | `prospects-cotacoes.md` | ✅ |
| Marketplace | F16 | `marketplace.md` | ✅ |
| Editor de contratos | F17 | `editor-contratos.md` | ✅ |
| Suporte e CMS | F19 | `suporte-cms.md` | ✅ |
| Dashboard e painel do cliente | F22 | `dashboard-painel.md` | ✅ |
| Configurações | F20 | `configuracoes.md` | ✅ |
| Portal público do proponente (proposta, análise, validação, autocadastro/termos) | F20 | `portal-publico.md` | ✅ |
