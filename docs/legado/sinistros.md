# Legado (telas) — Sinistros

> Módulo de abertura, listagem, visualização e edição de sinistros de seguros (Fiança, CAP/Título, Incêndio, Proteção Aluguel, Incêndio Condomínio). Plugin CakePHP `Sinistros`. Fluxo principal vive em `gerencia` (filtros + grid + form de abertura via modal/AJAX). `index` está praticamente morta (tabela escondida). Detalhe e edição abrem em modal sobre o grid.

## Cobertura
- Views `.ctp`:
  - `app/Plugin/Sinistros/View/Sinistros/index.ctp` (tela legada/quase morta — grid oculto)
  - `app/Plugin/Sinistros/View/Sinistros/gerencia.ctp` (tela principal: filtros, grid, form de abertura)
  - `app/Plugin/Sinistros/View/Sinistros/ver_sinistro.ctp` (modal somente-leitura)
  - `app/Plugin/Sinistros/View/Sinistros/editar_sinistro.ctp` (modal de edição)
- JS de UI: `app/Plugin/Sinistros/webroot/js/ui.js` (show/hide por produto, modais, select2, DataTable AJAX)
- Controller: `app/Plugin/Sinistros/Controller/SinistrosController.php` (lido p/ entender comportamento de tela)
- Model: `app/Plugin/Sinistros/Model/Sinistro.php` (alias real `Sinistros`, tabela `sinistros`)
- Helpers compartilhados: `AppController::VerificaPermissao` / `VerificaPermissaoModulo` / set global `$mostra_colunas`

## Telas / fluxos

### 1) `gerencia.ctp` — tela principal (`SinistrosController::gerencia($empresa_parceira_id)`)
Estrutura: cabeçalho com breadcrumb → form de **filtros** (`SinistrosFiltro`) → botão "Abrir sinistro" → form de **abertura** (`Sinistros`, oculto, expandido por JS) → **grid** paginado de sinistros.

**Form de filtros (`SinistrosFiltro`)** — submit recarrega a página (POST normal, sem AJAX):
- `seguradora_id` — select2 (lista de seguradoras, `empty=Selecione`).
- `nome_segurado` — texto (busca `LIKE %...%`).
- `documento_segurado` — texto (busca **igualdade exata**, sem LIKE — ver gotcha).
- `numero_sinistro` — texto (`LIKE %...%`).
- `numero_apolice` — texto (`LIKE %...%`).
- `filtro_data` — select do tipo de data a filtrar: `fim_vigencia` / `confirma_abertura` / `cadastro`.
- `data_ini` / `data_fim` — texto `.date .datepicker` (dd/mm/yyyy → `Dataformat::dateToSql`). Só aplicam se **ambos** preenchidos.
- `garantidos` — texto "Garantido (Nome ou CPF)"; o controller tenta formatar CPF se for só dígitos e busca `LIKE %...%`.
- `solicitacao_status` — select de status (1..5).
- `empresa_parceira_id` — select2 de imobiliária, **renderizado só se `$mostra_colunas==1`** (grupos internos).

**Grid de sinistros** (paginação CakePHP, 25/pág, ordem `Sinistros.id DESC`). Colunas:
- `id`
- Empresa + "Usuário: {User.name}" — **só se `$mostra_colunas==1`**.
- Pessoa: `nome_segurado` + "Doc.: {documento_segurado}", senão "n/d".
- Imóvel: endereço/número/complemento + bairro/cidade; se houver `nome_condominio`, mostra Condomínio + Unidade.
- Produto: `Produtosseguros.descricao` + "(Seguradora.nome - codigo_interno)".
- Status: mapeado por `$status_sinistros`.
- Ações: **Ver** (`janelaSinistro(empresaId, id)`), **Editar** (`janelaSinistroEdit(...)`, **só se `$mostra_colunas==1`**), **Anexos** (`arquivosAnexos('Sinistros',id,'0',empresaId)`), **Follow-ups** (`janelaFollowups('sinistros',id,empresaId,'1','lista','')`).

**Form de abertura (`Sinistros`)** — `id=FormSinistros`, action `grava_sinistro/{empresa_parceira_id}`, submetido via `ajaxForm` no botão `#processaSinistro`. Campos:
- `empresa_parceira_id` — hidden (vem do parâmetro/usuário).
- `produto_id` — select obrigatório; `onchange=verificaProduto(this.value)` controla quais blocos aparecem.
- Bloco **Fiança** (`#div_campos_fianca`): `fianca_entregou_chaves` (Sim/Não, `onchange=verificaEntregouChaves("fianca",...)`); textos instrucionais condicionais (`#fianca_entregou_chaves` / `#fianca_nao_entregou_chaves`).
- Bloco **Incêndio** (`#div_campos_incendio`): `data_evento` (date datepicker), `horario_evento` (`.time`), `tipo_evento` (select: 1=Incêndio parcial, 2=Incêndio total).
- Bloco **SPA / Proteção Aluguel** (`#div_campos_spa`): `motivo_seguro_aluguel` (1=Invalidez, 2=Invalidez permanente, 3=Morte, 4=Perda de renda), `nome_segurado`, `cpf_segurado` (classe `.cpf`).
- Bloco **CAP/Título** (`#div_campos_cap`): `cap_entregou_chaves` (Sim/Não), e se "Sim" → `para_pagto_debitos` (Sim/Não) com textos instrucionais (`#para_pagamento_debitos` / `#nao_para_pagamento_debitos`).
- Bloco **Dados do imóvel** (`#div_dados_imovel`): `cep` (maxlength 11, classe `.cep`, botão busca via `searchAddressByCep`), `endereco` (readonly, required), `endereco_numero` (required), `endereco_complemento`, `uf` (select siglas, `onChange=buscaCidades`), `cidade_id` (`onChange=buscaBairros`), `bairro_id`, `imovel_codigo_ref`, `tipo_imovel_id` (select required).
- Bloco **Dados do condomínio** (`#div_dados_condominio`): `nome_condominio`, `condominio_unidade` (`data-mask=telefone`).
- Bloco comum: `tipo_pessoa_segurado` (PF/PJ), `nome_segurado`, `documento_segurado` (placeholder "Somente números"), `flag_apolice_renovada` (checkbox → habilita `num_apolice_anterior` via `habilitarnumapolice()`), `numero_apolice`, `numero_sinistro`, `fim_vigencia` (date datepicker), `data_confirmacao_abertura` (date datepicker), `comentarios` (textarea required), `garantidos` (textarea, Nome e CPF).
- Botão `#processaSinistro`: valida no cliente `produto` e `comentarios` não vazios (alert), copia `empresa_id` p/ `SinistrosEmpresaParceiraId` se vazio, e envia via `ajaxForm` com barra de progresso. Sucesso (`data===true`) → alert + `window.location.reload()`.

**Atenção:** há um **segundo `nome_segurado`/`documento_segurado`** no bloco comum além dos do bloco SPA → IDs duplicados no DOM (gotcha de migração). Há também um bloco de radios `data[FichaAnaliseCadastral][forma_pagto]` (`$listprodutos`) totalmente oculto (`display:none`), resíduo copiado de outro módulo — não replicar.

### 2) `index.ctp` — tela legada (`SinistrosController::index()`)
Form `Sinistros` com filtro de `empresa_parceira_id` (select2) + botão "Gerenciar" (`#gerencia_empresa` → redireciona p/ `sinistros/sinistros/gerencia/{empresa_id}`). Os filtros extras (`#filtros_sinistro`: status, data_de, data_ate), o submit e a **tabela de resultados estão com `display:none`**. Na prática esta tela só serve para escolher a empresa e ir para `gerencia`. O link de Ação aponta para `action=edit` (rota inexistente — botão morto). **Não replicar como tela funcional**; tratar como redirecionador para `gerencia`.

### 3) `ver_sinistro.ctp` — modal somente-leitura (`SinistrosController::ver_sinistro($empresa_parceira_id, $id)`)
Aberto por `janelaSinistro()` em modal (`openModalFromUrl`, 60%x450). Todos os campos `readonly`. Mostra produto, comentários, blocos condicionais por `produto_seguro_id` (ver máquina de produtos abaixo), tipo pessoa/nome/documento, nº apólice, fim de vigência, nº sinistro, confirmação de abertura, comentários (repetido), garantidos, criado em, responsável. Form `create('Sinistro', ...)` com action `altera_repasse/...` **que não existe** e botão de processamento (`#processaRepasseAltera` / `#ProcessaSeguroRepasse`) **que não casa com nenhum elemento** → bloco JS morto; a tela é efetivamente só visualização. Não replicar a parte de "salvar".

### 4) `editar_sinistro.ctp` — modal de edição (`SinistrosController::editar_sinistro($empresa_parceira_id, $id)`)
Aberto por `janelaSinistroEdit()`. Form `Sinistros`, `id=ProcessaSinistro`, action `editar_sinistro/{empresa}/{id}`, submit AJAX via `#BtnProcessaSinistro`. Campos editáveis:
- `produto_id` — disabled (somente exibição).
- `created`, `user_id` (responsável) — disabled.
- `comentarios`, `garantidos` — textarea editáveis.
- Blocos condicionais por `produto_seguro_id`: Fiança (`fianca_entregou_chaves`), Incêndio (`data_evento`, `horario_evento`, `tipo_evento`), CAP (`cap_entregou_chaves`, `para_pagto_debitos`), SPA (`motivo_seguro_aluguel`).
- `status_sinistro` (select de status — mapeado p/ `status_id` no controller), `num_apolice_anterior`, `tipo_pessoa_segurado`, `nome_segurado`, `documento_segurado`, `numero_apolice`, `numero_sinistro`, `fim_vigencia` (date), `data_confirmacao_abertura` (date).
- Sucesso AJAX (`data===true`) → alert + reabre o próprio modal de edição via `janelaSinistroEdit(empresa,id)` (não recarrega a página inteira).

## Pontos de entrada (controller::ação que renderiza)
- `SinistrosController::index()` → `index.ctp` (tela legada/redirecionador).
- `SinistrosController::gerencia($empresa_parceira_id=null)` → `gerencia.ctp` (tela principal; também processa POST de filtros).
- `SinistrosController::ver_sinistro($empresa_parceira_id, $id)` → `ver_sinistro.ctp` (modal, `$this->render('ver_sinistro')`).
- `SinistrosController::editar_sinistro($empresa_parceira_id, $id)` → GET renderiza `editar_sinistro.ctp`; POST grava e devolve `json_encode(true/false)`.
- `SinistrosController::grava_sinistro($empresa_parceira_id=null)` → endpoint AJAX (sem view); grava abertura e dispara e-mail.
- `SinistrosController::listasinistros($empresa_parceira_id)` → endpoint JSON p/ DataTable (`carregaSinistros()` no ui.js); **não é usado** pelo grid atual do `gerencia.ctp` (que usa paginação server-side CakePHP); só seria usado se `carregaSinistros()` fosse chamado (está comentado). Tratar como legado opcional.

## Regras de negócio relevantes à UI

**Mapa de produto → blocos visíveis** (`ui.js::verificaProduto`, e `if`s nas views de detalhe/edição). Atenção: o JS de abertura e os `if` das views de detalhe **divergem** nos IDs de produto:
- `verificaProduto` (abertura): Fiança = 5,6,7,8,11; Incêndio = 4; CAP/Título = 2,3; Proteção Aluguel (SPA) = 1; Incêndio condomínio = 10.
- `ver_sinistro.ctp` / `editar_sinistro.ctp`: Fiança = 5,6,7; Incêndio = 4; CAP = 2,3; SPA = 1. (Não cobrem 8/10/11.)
- **Divergência relevante**: na abertura `produto_id` é o id de `WbsProdutosSeguradora` (produto×seguradora), mas o JS compara contra ids 1..11 que parecem ser de `wbs_produtos_seguros` (tipo de produto). Comportamento de exibição depende de coincidência de ids no ambiente legado — verificar mapeamento real na migração; não replicar os números hardcoded sem validar.

**Status do sinistro** (`$status_sinistros`, definido repetidamente no controller):
- 1 = Pendente de análise
- 2 = Pendente de documentação
- 3 = Indenização em andamento
- 4 = Finalizado com indenização
- 5 = Finalizado com Negociação

**Outros selects fixos:**
- `tipo_evento`: 1 = Incêndio parcial, 2 = Incêndio total.
- `motivo_spa`: 1 = Invalidez, 2 = Invalidez permanente, 3 = Morte, 4 = Perda de renda.
- `flag` (Sim/Não): 1 = Sim, 0 = Não.
- `tipoPessoa`: PF / PJ.

**Datas:** entradas dd/mm/yyyy convertidas por `Dataformat::dateToSql`; exibição por `Dataformat::dateBr` / `datetimeBr`. Em `index/gerencia`, `created` recebe sufixo `00:00:00`/`23:59:59` conforme tipo de filtro.

**Abertura (`grava_sinistro`):** gera `token` (`getToken(8)`), grava `status_id=1`, resolve `imovel_bairro`/`imovel_cidade` por nome a partir de `bairro_id`/`cidade_id` (via `Endereco`), e ao salvar dispara e-mail `EmailController::emailNotification('sinistro','novo',$dados)`. Tudo em transação (begin/commit/rollback) — porém o `commit` é chamado mesmo se o save falhou (o `pr('Erro...')` não interrompe o fluxo) → kept-bug, não replicar.

**Máscaras / validações de tela:**
- `documento_segurado`: força só números no cliente (`#SinistrosDocumentoSegurado.ForceNumericOnly()`), placeholder "Somente números".
- `cpf_segurado`: classe `.cpf` (máscara de CPF).
- `cep`: classe `.cep`, maxlength 11; busca endereço por CEP preenche endereço/cidade/bairro/uf.
- `condominio_unidade`: `data-mask=telefone` (máscara herdada incorreta — não replicar a máscara de telefone para unidade).
- `datepicker` nas datas; `.time` no horário.
- Form usa Parsley (`parsley-validate`, `novalidate`), mas a validação real é mínima (apenas `produto` e `comentarios` no `#processaSinistro`).

## Máquina de estados / status (refletida na UI)
- Abertura sempre nasce em `status_id=1` (Pendente de análise).
- Transições feitas manualmente em `editar_sinistro` via select `status_sinistro` → gravado em `status_id` (qualquer 1↔5, sem regra de transição imposta).
- `ver_sinistro` é read-only (não altera status).
- Não há workflow automático; status é puramente um campo livre de classificação.

## Multi-tenant / white-label
- **Escopo por empresa parceira (imobiliária)** via `empresa_parceira_id` em quase todas as rotas.
- Visibilidade por grupo (`AuthComponent::user('grupo_id')`):
  - Grupos **1,2,3,4,9** = internos/master → veem todas as imobiliárias (`Empresa->getParceiras()`), `empresa_parceira_id` esvaziado (vê tudo).
  - Demais grupos → travados na própria empresa (`getEmpresaFormaPagto()`), `empresa_parceira_id = empresa_id` do usuário.
- `$mostra_colunas` (setado globalmente em `AppController`, linhas ~340-344): `1` para grupos 1,2,3,4,5,9 → mostra coluna Empresa/Usuário, filtro de imobiliária e botão **Editar**; `2` para demais → oculta essas colunas/ações. (Note: grupo 5 é `mostra_colunas=1` aqui, mas **não** está na lista de "vê todas as parceiras" do controller — leve inconsistência.)
- `VerificaPermissao()['nivel']`: se `nivel==2`, força filtro `Sinistros.empresa_parceira_id = empresa_id` do usuário.
- Grupo **2** (caso especial): filtro adicional OR por `Empresas.empresa_resp_seguros / empresa_resp_analises / empresa_resp_vistorias = empresa_id` (empresa responsável por seguros/análises/vistorias).
- `VerificaPermissaoModulo('0')` chamado no início de toda ação (gate de acesso ao módulo).
- Seleção de empresa no select2 usa endpoint global `Pesquisas/buscarempresa` (ui.js).

## Gotchas / decisões kept-bug
- **`index.ctp` é tela morta**: tabela/filtros/submit todos `display:none`; link de ação aponta para `action=edit` inexistente. Serve só para escolher empresa e ir a `gerencia`. Não migrar como tela funcional.
- **`ver_sinistro.ctp`**: form aponta para action `altera_repasse/...` inexistente e o JS (`#processaRepasseAltera`/`#ProcessaSeguroRepasse`) não casa com nenhum elemento → bloco de salvar é morto; é só visualização. Não replicar.
- **`tipo_evento` em ver/editar** usa `'value'=>$Sinistro['produto_seguro_id']` em vez de `tipo_evento` → bug: pré-seleciona o select pelo id do produto, não pelo tipo de evento. Não replicar.
- **Filtro `documento_segurado` em gerencia** usa igualdade exata (`'Sinistros.documento_segurado ' => $valor`), enquanto `nome_segurado`/`numero_sinistro`/`numero_apolice` usam LIKE. Provável inconsistência; em migração padronizar (decidir LIKE vs exato).
- **Filtro `garantidos`**: `str_replace(array('.','-','/'), $data, ...)` passa o **array `$data` inteiro como replacement** — código frágil/errado; só funciona por acaso. Reescrever (objetivo: normalizar CPF e buscar LIKE).
- **`nome_segurado`/`documento_segurado` duplicados** no form de abertura (bloco SPA + bloco comum) → IDs repetidos no DOM. Consolidar em campo único.
- **`grava_sinistro` comita mesmo com falha no save** (não dá rollback no `else`). Corrigir na migração (validar antes de commit).
- **Bloco de radios `FichaAnaliseCadastral][forma_pagto]`** em `gerencia.ctp` (oculto) é resíduo copiado de outro módulo. Ignorar.
- **`index.ctp` filtro `nome_segurado`** monta LIKE com aspas literais (`"'%...%'"`) — provavelmente quebra a query; mais um motivo para não migrar `index`.
- `listasinistros` (DataTable JSON) e `carregaSinistros()` estão presentes mas **desativados** (chamada comentada); o grid real é server-side paginado. Migrar apenas a paginação server-side; o DataTable AJAX é opcional/legado.
- Hardcode de ids de produto (1,2,3,4,5,6,7,8,10,11) divergente entre JS de abertura e views de detalhe — validar mapeamento real de `wbs_produtos_seguradoras` × `wbs_produtos_seguros` antes de portar a lógica de exibição.
- Modais via `openModalFromUrl` e ações `arquivosAnexos`/`janelaFollowups` são integrações globais (Anexos, Follow-ups) externas a este módulo.

## Destino (issues Linear)
- Projeto **wecorp-frontend** (telas). Domínio Sinistros, fase F13.
- Sugerir milestone/issues:
  - Tela "Gerência de Sinistros" (filtros + grid + abertura por modal/drawer), com visibilidade por grupo (`mostra_colunas`) e escopo multi-tenant por `empresa_parceira_id`.
  - Form de abertura com blocos condicionais por produto (Fiança/CAP/Incêndio/SPA/Condomínio) e dados de imóvel/condomínio (CEP autofill, máscaras CPF/CEP).
  - Modal de visualização (read-only) e modal de edição (status + campos por produto).
  - Backend correlato (wecorp-backend/services): endpoints de listagem paginada/filtros, grava_sinistro (token, status inicial, notificação e-mail), editar status, anexos e follow-ups.
  - Descartar/objetar a tela `index` legada e os blocos mortos (`altera_repasse`, radios `forma_pagto`).
