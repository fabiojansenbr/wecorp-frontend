# Legado (telas) — Marketplace

> Módulo "Market Place" (DB module id 14) com um único parceiro ativo: **Rede Vistorias**. São 3 telas estáticas (landing institucional + landing do parceiro + tela de solicitação que embute o SDK JS da Rede Vistorias). Os controllers são stubs vazios; toda a lógica de negócio vive no SDK externo do parceiro e na montagem do menu (AppController).

## Cobertura
Views .ctp:
- `app/Plugin/Marketplace/View/Marketplace/index.ctp` — landing do Marketplace
- `app/Plugin/Marketplace/View/Redevistorias/index.ctp` — landing do parceiro Rede Vistorias
- `app/Plugin/Marketplace/View/Redevistorias/solicitar.ctp` — tela de solicitação (SDK embarcado)

Controllers:
- `app/Plugin/Marketplace/Controller/MarketplaceController.php` (`index`, `solicitar` — vazios)
- `app/Plugin/Marketplace/Controller/RedevistoriasController.php` (`index` vazio, `solicitar` carrega a Empresa)

Colaboradores:
- `app/Lib/Dataformat.php::formatarCnpj` (formatação do CNPJ injetado no SDK)
- `app/Controller/AppController.php` (~L593) — item de menu do módulo 14 (Marketplace)
- `app/Config/bootstrap.php:124` — `CakePlugin::load('Marketplace')`
- `app/Plugin/Marketplace/webroot/js/ui.js` — JS do plugin (resíduo de template)

NÃO há views/CSS próprios além do logo `webroot/imagens/logos_parceiros/logo-rv-2020.png`.

## Telas / fluxos

### 1) Marketplace/index.ctp — "Market Place" (landing do hub)
- **Conteúdo:** estático. Logo `logo-rv-2020.png` (width 190), título "Conheça a Rede Vistorias, parceira de nosso Marketplace" e texto institucional fixo.
- **Campos:** nenhum (página informativa).
- **Navegação:**
  - Breadcrumb: Painel (`/dashboard`) > Market Place (`/markeplace` — **typo, ver gotchas**).
  - Botão primário "Solicitar nova vistoria" → `/marketplace/redevistorias/solicitar/` (link montado por concatenação de string, não Router padrão).
- **Comportamento por grupo:** o acesso à tela é controlado pelo menu (módulo 14) montado em AppController; a view em si não diferencia grupo.

### 2) Redevistorias/index.ctp — landing do parceiro
- Conteúdo praticamente idêntico ao item 1 (mesmo logo/texto), mas é a landing específica do controller Redevistorias.
- Botão "Solicitar nova vistoria" → `/marketplace/<controller>/solicitar/`, onde `<controller>` = `$this->params['controller']` (resolve para `redevistorias`).
- Breadcrumb: Painel > Market Place (`/markeplace`) > Rede de Vistorias (`/marketplace/redevistorias/`).
- Sem campos/validações.

### 3) Redevistorias/solicitar.ctp — solicitação de vistoria (SDK embarcado)
- **Sem formulário CakePHP.** Renderiza apenas `<div id="sdk-rv" style="height:900px;"></div>` e injeta o **SDK JavaScript da Rede Vistorias** que monta toda a UI dentro dessa div.
- **Script de boot do SDK:**
  - `rv('partner', 'O2')` — código de parceiro **fixo** "O2" (Plataforma O2 / white-label).
  - `rv('document', <CNPJ da empresa logada>)` — CNPJ formatado via `Dataformat::formatarCnpj($DadosEmpresa['Empresa']['documento'])`.
  - `rv('webhook', '')` e `rv('orderType', '')` — vazios.
  - `rv('on','order.created', function(code, price){ /* ... */ })` — callback **vazio** (não persiste/registra a ordem no legado).
  - `rv('init','sdk-rv')` — monta o widget na div.
- **Origem do CNPJ:** `RedevistoriasController::solicitar()` lê `AuthComponent::user('empresa_id')`, faz `Empresa->find('first')` por id e passa `$DadosEmpresa` para a view → multi-tenant por empresa logada.
- Breadcrumb: Painel > Market Place (`/marketplace`) > Rede de Vistorias (`/markeplace/redevistorias/` — typo).

## Pontos de entrada (controller::ação que renderiza)
- `MarketplaceController::index()` → `Marketplace/index.ctp` (stub vazio).
- `MarketplaceController::solicitar()` → existe mas vazio; não há view `Marketplace/solicitar.ctp` (os links apontam para o controller Redevistorias).
- `RedevistoriasController::index()` → `Redevistorias/index.ctp` (stub vazio).
- `RedevistoriasController::solicitar()` → `Redevistorias/solicitar.ctp`; única ação com lógica: carrega Empresa pelo `empresa_id` da sessão e seta `$DadosEmpresa`.
- URLs (rota default do CakePlugin): `/marketplace`, `/marketplace/redevistorias`, `/marketplace/redevistorias/solicitar`.

## Regras de negócio relevantes à UI
- O conteúdo institucional é 100% estático/hardcoded na view (não vem de CMS/DB).
- A solicitação de vistoria é **inteiramente delegada ao SDK externo** da Rede Vistorias; o legado só fornece `partner` e `document` (CNPJ). Não há captura de campos, validação, máscara ou persistência de pedido do lado do legado.
- `order.created` é capturado mas ignorado (callback vazio) → não há registro local da vistoria solicitada nem vínculo com tabelas internas.

## Máquina de estados / status (refletida na UI)
- Nenhuma máquina de estados local. Status/ciclo de vida da vistoria são geridos exclusivamente na plataforma da Rede Vistorias (fora do legado). A UI legada não lista, filtra nem exibe status de pedidos.

## Multi-tenant / white-label
- **Tenant:** identificado por `empresa_id` da sessão (`AuthComponent::user('empresa_id')`); o CNPJ dessa empresa é injetado no SDK como `document`.
- **White-label:** `partner = 'O2'` fixo (Plataforma O2). Marca/branding da tela é da Rede Vistorias (logo + textos fixos), não do tenant.
- **Visibilidade do módulo:** controlada pelo menu dinâmico (AppController, módulo id 14, ícone `fa fa-plug`) que depende das permissões de módulo/controller por grupo gravadas no banco (tabelas Module/ModulosController). A view não reimplementa controle por grupo.

## Gotchas / decisões kept-bug
- **SDK em SANDBOX, não produção (NÃO replicar):** em `solicitar.ctp` a URL de produção `https://integration.redevistorias.com.br/js/app.js` está **comentada** e a ativa é `https://integration.sandbox.redevistorias.com.br/js/app.js`. Na migração deve apontar para produção (e idealmente vir de config/env por ambiente).
- **Typo de URL `/markeplace`** (faltando o "r") em vários breadcrumbs (Marketplace/index, Redevistorias/index, e parte do solicitar). Funciona como link cego pois a rota correta é `/marketplace`; corrigir na migração.
- **Links montados por concatenação de string** (`Router::url('/marketplace/...').'/solicitar/'`) em vez de array de rota — frágil; padronizar.
- **`order.created` ignorado:** o pedido criado não é gravado no legado — avaliar se a nova plataforma deve persistir/auditar a ordem (webhook está vazio também).
- **ui.js é resíduo de template `_blank`:** inicializa `select2` em `#empresa_parceira_id` (campo inexistente nessas telas) apontando para `Pesquisas/buscarempresa`. Não deve ser portado.
- **Ordem de scripts:** `ui.js` é incluído ANTES de `jquery.js` (jQuery inline depois) — ordem invertida; não replicar.
- **MarketplaceController::solicitar()** existe sem view correspondente (links reais usam o controller Redevistorias) — código morto.
- Logo único `logo-rv-2020.png`; estrutura sugere suporte a múltiplos parceiros mas só Rede Vistorias está implementada.

## Destino (issues Linear)
- Fase F16 — Marketplace (telas). Projeto `wecorp-frontend`.
- Sugerido: tela "Marketplace / Parceiros" (hub + landing Rede Vistorias) e integração da solicitação via SDK Rede Vistorias (com partner/document por tenant, URL de produção parametrizável, e persistência/auditoria de `order.created`).
