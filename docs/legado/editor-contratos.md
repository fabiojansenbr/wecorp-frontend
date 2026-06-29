# Legado (telas) — Editor de contratos (telas)

> Plugin CakePHP `Editorcontratos` que gerencia "Modelos de Documentos" (templates de contrato em HTML rico via TinyMCE). Telas: listagem (grid), criar/editar modelo com editor rich-text + inserção de "Tags" (variáveis), e modais auxiliares (busca de modelo, busca de tag, visualização/geração do contrato a partir de uma proposta). As tags são substituídas por dados reais da proposta/contrato de locação no momento da geração.

## Cobertura
- Views `.ctp`:
  - `app/Plugin/Editorcontratos/View/Modelodocumentos/index.ctp` (listagem em grid)
  - `app/Plugin/Editorcontratos/View/Modelodocumentos/add.ctp` (criar/editar modelo — reutilizada por `edit`)
  - `app/Plugin/Editorcontratos/View/Modelodocumentos/modal_search.ctp` (modal de busca de modelos via datagrid)
  - `app/Plugin/Editorcontratos/View/Modelodocumentos/modal_tag.ctp` (modal de seleção de tag)
  - `app/Plugin/Editorcontratos/View/Modelodocumentos/modal_view.ctp` (visualizar/gerar contrato preenchido)
- Controller: `app/Plugin/Editorcontratos/Controller/ModelodocumentosController.php`
- Apoio (para entender comportamento da tela):
  - `app/Plugin/Editorcontratos/webroot/js/ui.js` (init do TinyMCE + plugin custom "editorcontratos" + `selectTag`)
  - `app/webroot/js/modules/editorcontratos/datagrid.js` (DataTable da listagem)
  - `app/Plugin/Editorcontratos/Lib/NavigationEditorcontratos.php` (abas do módulo)
  - Models: `Modelodocumento.php`, `Tag.php`, `Tipocontrato.php`, `DadosContrato.php`
  - `EditorcontratosAppController.php` (vazio — herda `AppController`)

## Telas / fluxos

### 1. Listagem — `index.ctp` (`/editorcontratos/modelodocumentos/index`)
- Cabeçalho com título hardcoded **"Controle de Serviços"** e breadcrumb "Painel / Financeiro / Serviços" — rótulos errados/copy-paste de outro módulo (o `moduleTitle` correto definido no controller é "Controle de Modelos de Contratos"). **Não replicar** os rótulos errados; usar "Modelos de Contratos".
- Tabela `#ModeloGrid` com colunas: **Id.** (8%), **Título** (60%), **Criado em** (20%), e coluna de ações (12%, sem cabeçalho).
- Grid server-side via DataTables (`datagrid.js`): `sAjaxSource = editorcontratos/modelodocumentos/index` (a mesma action responde JSON quando `is('ajax')`). Sem filtro, sem paginação client, sem info (`bFilter:false, bPaginate:false, bInfo:false`), coluna de ações não ordenável (target 3).
- Ação por linha (model `Modelodocumento::createGridLinks`): botão azul (lápis `fa-pencil`) linkando `/editorcontratos/modelodocumentos/edit/{id}`. Não há ação de excluir.

### 2. Criar / Editar modelo — `add.ctp` (`/editorcontratos/modelodocumentos/add` e `/edit/{id}` que renderiza `add`)
- Form CakePHP `Modelodocumento` dentro de painel "Modelo de Documento". Campos:
  - **Nome** (`titulo`) — input texto `form-control`, largura col-md-8. Sem máscara, sem validação client.
  - **Tipo de Contrato** (`tipocontrato_id`) — `select` populado por `Tipocontrato::find('list')` (id => descricao), `empty='Selecione'`, col-md-4.
  - **Conteúdo** (`conteudo`) — `textarea` com classe `rich_editor` (rows=20), transformado em editor TinyMCE.
- Botão único no footer: **"Gravar Documento"** (submit, `btn-primary`).
- Sem validação de formulário no model (`$validate` não declarado) — salva mesmo com campos vazios. Flash de sucesso/erro renderizado no topo (`Session->flash()`).
- **Editor TinyMCE** (`ui.js`): `tinymce.init` sobre `.rich_editor`, idioma `pt_BR`, font sizes `8pt..36pt`. Plugins: advlist, autolink, lists, link, image, charmap, print, preview, anchor, searchreplace, visualblocks, code, fullscreen, insertdatetime, media, table, contextmenu, paste e o custom **editorcontratos**. Toolbar: undo/redo, styleselect, bold/italic, alinhamentos, listas/indent, **editorcontratos** (botão "Inserir Tag"), fontselect, fontsizeselect.
- **Botão "Inserir Tag"** (plugin custom): abre janela TinyMCE com um botão "Buscar Tag" e um textbox "Tag". Buscar Tag chama `openModalFromUrl('editorcontratos/modelodocumentos/modal_tag/' + <valor de #ModelodocumentoTipocontratoId>, ...)` (modal 70% largura). Ou seja, as tags listadas dependem do **Tipo de Contrato selecionado** no form. Ao submeter a janela, o texto do textbox é inserido no editor via `editor.insertContent`.
- O `addMenuItem('editorcontratos')` ("Example plugin" → abre tinymce.com) é código de exemplo morto — **não replicar**.

### 3. Modal de tags — `modal_tag.ctp` (`/editorcontratos/modelodocumentos/modal_tag/{tipo_id}`)
- Tabela com colunas **Tag**, **Descrição**, **Ações**. Para cada `Tag` (filtrada por `tipocontrato_id = {tipo_id}`; se `tipo_id` nulo, lista vazia): mostra `Tag.tag` (o placeholder, ex. uma string substituível) e `Tag.nome`.
- Botão "Selecionar" chama `selectTag('<tag>')` (em `ui.js`): preenche `#TagInput` com o valor da tag e fecha a modal. O usuário então confirma na janela do TinyMCE para inserir o placeholder no conteúdo.

### 4. Modal de busca de modelos — `modal_search.ctp` (`/editorcontratos/modelodocumentos/modal_search`)
- Painel "Modelos de Contrato" com Datagrid (helper `Datagrid`) `#ModelosModalGrid`, ajax source `/editorcontratos/modelodocumentos/modal_grid/`, paginado. Colunas: ID, Titulo, Data de criação, Ações.
- Ação por linha (`Modelodocumento::createModalGridLinks`): botão verde (checkmark) com classe `modalSelectItem` e `data-id`/`data-titulo` — usado por um seletor de item de modal genérico (padrão do framework legado) para escolher um modelo. **Gap:** o gatilho que abre esta modal não foi localizado nas views/JS deste plugin (provavelmente acionado a partir do módulo Locacoes/Proposta).

### 5. Modal de visualização/geração — `modal_view.ctp` (`/editorcontratos/modelodocumentos/modal_view/{doc_id}/{proposta_id}`)
- Form `Modelodocumento` no painel "Visualização". Campos:
  - **Titulo** (`titulo`) — input texto editável.
  - **conteudo** — textarea `rich_editor` (TinyMCE) já preenchido com o modelo **com as tags substituídas** pelos dados reais da proposta (ver controller/model abaixo).
  - `proposta_id` — hidden (carrega valor da rota).
- Botão **"Gerar Contrato"** (submit). O fluxo de persistência do contrato gerado ocorre fora deste plugin (não há action de POST para gerar no controller deste plugin — a submissão vai para o módulo de Locações/Contratos). **Gap:** destino do submit não confirmado neste plugin.

## Pontos de entrada (controller::ação que renderiza)
- `ModelodocumentosController::index()` — renderiza `index.ctp`; em `is('ajax')` retorna `Modelodocumento->grid()` (JSON, sem render). Seta `moduleTabs` (NavigationEditorcontratos::getPrimary).
- `ModelodocumentosController::add()` — renderiza `add.ctp`; em POST chama `Modelodocumento->save($this->request->data)` com flash `box_success`/`box_error`. Passa `tipos` (lista de tipos de contrato) e `moduleTabs`.
- `ModelodocumentosController::edit($id)` — em PUT seta `Modelodocumento->id=$id` e salva (flash `alert` success/error); carrega registro via `findById($id)`; passa `tipos`+`moduleTabs`; `$this->render('add')` (reusa a mesma view).
- `ModelodocumentosController::modal_tag($tipo_id = NULL)` — `ajaxRequest()`; carrega `Tag->find('all', conditions tipocontrato_id=$tipo_id)`; passa `tags`.
- `ModelodocumentosController::modal_search()` — só `ajaxRequest()` (renderiza a view com o datagrid).
- `ModelodocumentosController::modal_grid()` — só em `is('ajax')`; carrega behavior Datagrid com `actionColCallback=createModalGridLinks` e retorna `grid()` em JSON.
- `ModelodocumentosController::modal_view($doc_id,$proposta_id)` — `ajaxRequest()`; busca modelo, chama `Modelodocumento->updateDocument($contrato,$proposta_id)` (substitui tags) e injeta em `request->data`; passa `proposta_id`.

## Regras de negócio relevantes à UI
- **Tags = variáveis de template.** Tabela `documentos_tags` (model `Tag`) com campos: `tag` (placeholder/texto a inserir), `nome` (rótulo), `tipo` (`db` ou outro), `model`, `campo`, `tipocontrato_id`. As tags exibidas no modal dependem do **Tipo de Contrato** selecionado.
- **Substituição (`Modelodocumento::changeTags`)** percorre **todas** as tags (`Tag->find('all')`, sem filtro por tipo — pega geral) e:
  - Se `tipo == 'db'`: substitui pelo valor `$data[model][campo]` da proposta/contrato (se existir).
  - Caso contrário: chama dinamicamente `call_user_func(model.'::'.campo, proposta_id)` — ex.: métodos estáticos de `DadosContrato` (`getClientesContrato`, `getProprietariosContrato`, `getImovelContrato`).
  - `str_replace(tag, conteudo, conteudoModelo)` — substituição literal de string.
- **`updateDocument`** monta os dados a partir de `Proposta` (plugin Locacoes) + `ContratosLocacao` (findByPropostaId) e aplica `formatData` antes de substituir.
- **`formatData`** aplica formatação BR para exibição no contrato gerado:
  - `valor_aluguel` → `R$ x.xxx,xx (extenso) mensais`.
  - `dia_vencimento_aluguel` → `N (extenso)`.
  - `data_inicio_contrato` / `data_fim_contrato` → data por extenso.
  - `dias_tolerancia` → `Nº (ordinal extenso)`.
  - `percentual_juros_dia` → `N ( extenso por cento )`.
  - `valor_multa` → se `tipo_multa==1` percentual `N ( extenso por cento )`; senão monetário `R$ ... ( extenso por cento )` (**bug de rótulo:** valor monetário também escreve "por cento" — não replicar).
- `DadosContrato::getClientesContrato` usa participação `propostas_tipoparticipacao_id = 4` (locatários/clientes); `getProprietariosContrato` usa `= 3` (proprietários). Monta string com nome, nacionalidade, profissão, CPF, identidade, endereço/bairro, separados por " / ".

## Máquina de estados / status (refletida na UI)
- Não há máquina de estados/status para o modelo de documento (entidade simples: id, titulo, tipocontrato_id, conteudo, created). O "estado" relevante é apenas: rascunho de template (CRUD) vs. contrato gerado (preenchido em `modal_view`, persistido no módulo de Locações).

## Multi-tenant / white-label
- **NÃO há escopo multi-tenant neste módulo.** Em `Modelodocumento::beforeSave`, as linhas que setariam `user_id` e `empresa_id` estão **comentadas** (`//$this->data[...]['empresa_id'] = AuthComponent::user('empresa_id')`). Logo, modelos de documento são **globais**, não filtrados por empresa/imobiliária. Idem `index`/`edit`/grid: nenhuma condition por empresa.
- **Risco de paridade:** na migração, decidir se os modelos passam a ser por tenant (recomendado) — o legado vaza modelos entre todas as empresas. Documentar como decisão de produto.
- Nenhum comportamento por "grupo" de usuário identificado nas telas (sem ramificação por perfil/role).

## Gotchas / decisões kept-bug
- `index.ctp`: título "Controle de Serviços" e breadcrumb "Financeiro/Serviços" são copy-paste errado — **não replicar**, usar "Modelos de Contratos".
- `formatData`: valor de multa monetário escreve "por cento" indevidamente — corrigir na migração.
- `changeTags` ignora `tipocontrato_id` ao substituir (usa todas as tags), embora a modal de inserção filtre por tipo. Inconsistência: tags de outros tipos também serão substituídas se o placeholder existir no texto.
- `Modelodocumento::add` salva sem validação (sem `$validate`); aceita título/conteúdo vazios. Adicionar validação (titulo e tipo obrigatórios) na migração.
- Não há exclusão de modelos pela UI (só criar/editar).
- `ui.js` tem `addMenuItem` "Example plugin" apontando para tinymce.com — código morto, descartar.
- TinyMCE versão 4.x (skin lightgray) embutida no plugin (`webroot/js/tinymce`). Na migração frontend (Next.js) substituir por editor rich-text moderno (ex. TipTap) preservando: inserção de variáveis/tags por tipo de contrato, fontselect/fontsize, listas, alinhamento, tabela, code view.
- Substituição de tags é `str_replace` literal — sensível ao texto exato do placeholder; placeholders devem ser únicos/inequívocos.
- `EditorcontratosAppController` é vazio: autenticação/layout vêm do `AppController` global (não inspecionado aqui).

## Destino (issues Linear)
- Fase F17 — "Editor de contratos (telas)". Projeto **wecorp-frontend** (telas) com apoio de **wecorp-backend** (CRUD de modelos, tabela `modelos_documentos`, `documentos_tags`, `tipocontratos`) e **wecorp-services** (engine de substituição de tags + formatação BR por extenso/moeda). Mapear para os milestones de "Editor de contratos / Modelos de documentos" (HUB-116..HUB-382 — confirmar issue exata na fase F17).
