# Legado (telas) — Suporte e CMS

> Dois plugins independentes administrativos. **Suporte** = sistema de chamados/tickets (bugs, melhorias, novas implementações) com observações/interações e máquina de status, mais um conjunto de "ferramentas de higienização" de dados (SQL cru — NÃO replicar). **CMS** = gestão de conteúdo (artigos/downloads/aprendizado) com editor CKEditor e categorias. Ambos cheios de código morto, hardcodes de outro projeto ("sistemajtavares"/"embracei") e bugs de copy-paste.

## Cobertura
Views/controllers efetivamente cobertos:
- **Suporte**: `Controller/SuporteController.php`; views `index.ctp`, `index_paginator.ctp`, `add.ctp`, `edit.ctp`, `editobs.ctp`, `view.ctp`, `principal.ctp`, `mostraobs.ctp`, `homogenizar.ctp`, `modulo_higienizar.ctp`, `metadados.ctp`, `buscacampos.ctp`, `buscacampos2.ctp` (gerado no controller), `buscavalores.ctp`. Models `Suporte.php`, `SuporteObs.php`.
- **CMS**: `Controller/CmsController.php`, `Controller/CmsCategoriasController.php`; views `Cms/index.ctp`, `add.ctp`, `edit.ctp`, `ver.ctp`, `downloads.ctp`, `aprendizado.ctp`; `CmsCategorias/index.ctp`, `add.ctp`, `edit.ctp`. Models `Cm.php`, `CmsCategoria.php`.
- NÃO indexados (regra): `_SuporteController*`, `_add.ctp`, `_index_paginator_*` (backups datados), `Vendor/`, behaviors do CMS (`Behavior/*`).

## Telas / fluxos

### SUPORTE

**`index.ctp`** (Suporte::index) — tela de listagem de tickets com barra de filtro.
- Filtros (inputs, todos opcionais): `cod` (Nº do chamado, texto livre), `status` (select), `responsavel` (select populado de `$analistas` = `['1'=>'Suporte','2'=>'Usuario']`), `nivel` (select Baixo/Médio/Alto). Existem ainda filtros `flag_versao`, `flag_usuario`, `module_id`, `modificacao` aceitos no controller mas SEM campo nessa view.
- Botões: Pesquisar (`.pesquisar`, JS em `Suporte/js/ui.js`), Limpar (link p/ `suporte/suporte/index`), "Abrir chamado" (link p/ `add`).
- Select de status do filtro usa rótulos: 1 Aberto, 2 Em andamento, 3 Fechado em homologação, 4 Fechado em produção, 5 Cancelado — **diferente** do mapa real exibido na listagem (ver Máquina de estados).
- A lista em si NÃO é renderizada aqui; o `<div id="alvopaginator">` é preenchido por AJAX que chama `index_paginator` (paginação Ajax via PaginatorHelper + JsHelper).

**`index_paginator.ctp`** (Suporte::index_paginator) — fragmento AJAX da lista.
- Cada linha (sem tabela; grid Bootstrap) mostra: Cod (`Suporte.id`), Módulo (`Module.nome`) + Funcionalidade (`Modulecontrolleraction.descricao`), Aberto por / Responsável (`User.userName`), Descrição (`descricao_formatada`, quebrada com `wordwrap(...,60)`), Criação/Modificação (`Dataformat::dateBr`), Status (label colorido), Prioridade (label colorido), Ações (editar/excluir).
- Status mapeados aqui (0..10,13,99) — ver seção de estados. Prioridade: 1 Baixo (info), 2 Médio (primary), 3 Alto (danger), demais "Sem prioridade".
- Ações: editar (`edit/<id>`), excluir (`postLink delete/<id>` com confirm "Tem certeza que deseja apagar?").
- Paginação: 10/página, ordenada por `Suporte.modified DESC`.

**`add.ctp`** (Suporte::add) — "Cadastrar novo ticket".
- Form `Suporte` multipart. Campos: `sistema` (select `$sistemas`=['1'=>'Plataforma']), `module_id` (select de módulos, `onChange` → `pesquisarFuncionalidades()`), `view` (select Funcionalidade, populado via AJAX `buscarfuncoesmenu`), `nivel_prioridade` (Baixo/Médio/Alto), `descricao` (textarea + TinyMCE pt_BR), file `upload`.
- **Bug**: o file input chama-se `upload` mas o controller lê `$this->request->data['Suporte']['foto_chamado']` → upload da foto nunca funciona pela tela atual.
- JS `pesquisarFuncionalidades(id)` faz `getJSON` em `/suporte/suporte/buscarfuncoesmenu?id=` e repopula `#SuporteView`.
- No POST o controller força `flag_usuario=2`, `status=1`, `abertopor=Auth.id`, gera `descricao_formatada` (remove `<p>`), e move upload só se extensão ∈ jpg/jpeg/gif/png para `WWW_ROOT/fotos/`.

**`edit.ctp`** (Suporte::edit) — "Editar ticket".
- Datepicker pt_BR (`.data`, formato dd/mm/yy) + TinyMCE.
- Campos: `id` (input), `module_id` (select **disabled**), `status` (select dinâmico por estado+master — ver estados), `responsavel` (select de usuários), `nivel_prioridade` (**disabled**), `abertopor` (texto **disabled**, mostra username), `descricao` (textarea **disabled**).
- Abaixo: tabela "Lista de observações/interações" carregada por `requestAction(mostraobs/<codigo>)`. Colunas: Observação, Data, Data início, Data término, Responsável, Aberto por, Status (1 Aberto / 2 Em análise / 9 Cancelado), ação editar (`editobs/<obsId>/<suporteId>`).
- Bloco "Nova interação/observação" (toggle por `#btn_nova_interacao`): `obs` (textarea), e SE `ismaster=='S'`: `data_inicio`/`data_conclusao` (datepicker, required), `responsavel` (select required); `status_id` (select, default 2); `suportes_id` (hidden). Campos de data/responsável só aparecem para master.
- POST com `SuporteObs` grava observação (converte datas via `Convert::dataDB`, seta `user_id`, `descricao_formatada`) e redireciona p/ `edit/<id>`. POST com `Suporte` grava o ticket.

**`editobs.ctp`** (Suporte::editobs) — edição de uma observação existente. Mesmos campos da nova interação, todos preenchidos; mostra Criação/Modificação. Botão "Voltar ao ticket". Usa `$status_suporte` (mapa antigo 1/0/2/3/4) — inconsistente com `edit`.

**`view.ctp`** (Suporte::view, layout `modal`) — somente leitura; exibe `id, controller, view, campo, descricao, status (numérico cru), created`. Referencia colunas legadas (`controller`,`campo`) e ações em inglês ("Edit/Delete/List/New"). Tela praticamente abandonada.

**`principal.ctp`** (Suporte::principal) — 3 abas (Bugs/Tarefas, UC casos de uso, RN regras de negócio) com `<iframe>` apontando para `/sistemajtavares/Suportes...` (URLs de OUTRO sistema, quebradas). NÃO replicar.

**Ferramentas de dados (NÃO replicar — ver Gotchas):**
- `homogenizar.ctp` (Suporte::homogenizar): selects de Tabela de Origem/Referência montados de `SHOW TABLES` com chave hardcoded `Tables_in_embracei_db_jtavares_v2`; AJAX para `buscacampos/buscacamposatualiza/buscacampos2/buscavalores`; submit roda `UPDATE` cru montado por concatenação.
- `modulo_higienizar.ctp` (Suporte::modulo_higienizar): higieniza `imovel_comodos_old.cod_comodo` contra `meta_comodos`; busca AJAX `busca_tabela_antiga`.
- `metadados.ctp` (Suporte::metadados): hub estático de links para `/sistemajtavares/meta*` (Atendimento, Comodos, Profissão, etc.). Apenas âncoras; sem dados dinâmicos.
- `buscacampos.ctp`/`buscavalores.ctp`/`buscacampos2` (gerado inline): geram `<select>`/checkboxes de colunas e valores via `information_schema.columns` e `SELECT ... GROUP BY`.

### CMS

**`Cms/index.ctp`** (CmsController::index) — listagem (limit 5000, `id DESC`). Tabela: id, categoria (resolvida via `$categorias[categoria_id]`), titulo, resumo, Data (`created` via `Dataformat::dateBr`), ação editar. View/excluir comentados. Ação "Novo Cms" → `add`.
- Requer permissão (`VerificaPermissao()`); sem permissão renderiza `/Dashboard/proibido`.

**`Cms/add.ctp` / `Cms/edit.ctp`** (CmsController::add/edit) — form `Cm`:
- `categoria_id` (select `$categorias`), `titulo` (texto, maxlength 145), `resumo` (textarea 4 linhas), `descricao`/Conteúdo (textarea, **CKEditor 4** no id `CmDescricao`, config `o2config.js`, plugins sourcearea/embed, file browser via CKFinder), `status` (select 1 Ativo / 0 Inativo). Submit "Processar" (`data[submit]`).
- Vários campos comentados no HTML: `ordem`, `codigo_video_embed`, `link_video`, `link_video_tipo`, `tags`.
- **Bug edit (controller)**: em `edit()`, o `find('list')` usa `fields => CmsCategoria.descricao` (no add usa `CmsCategoria.nome`) — rótulos de categoria divergem entre add e edit. Ambos requerem permissão.

**`Cms/ver.ctp`** (CmsController::ver) — exibição pública/leitura de um conteúdo: titulo, resumo (itálico), `descricao` renderizada como HTML cru (sem `h()`), botão Voltar (`history.go(-1)`).

**`Cms/downloads.ctp`** (CmsController::downloads) — lista cards de conteúdos com `categoria_id=1` (hardcoded); cada card linka p/ `ver/<id>`. Texto fixo "downloads de fichas de cadastro...".

**`Cms/aprendizado.ctp`** (CmsController::aprendizado) — idem com `categoria_id=2` (hardcoded); texto "materiais de aprendizado".

**`CmsCategorias/index|add|edit.ctp`** (CmsCategoriasController) — CRUD simples de categorias. Campos: `modulo_id`, `nome`, `descricao`, `ordem` (todos `Form->input` genéricos, sem máscara). index lista id/modulo_id/nome/descricao/ordem/created + ver/editar/excluir (limit 5). edit tem link Excluir (`postLink delete`).

## Pontos de entrada (controller::ação que renderiza)
- `SuporteController::index` → `index.ctp` (filtros; lista vem por AJAX)
- `SuporteController::index_paginator` → `index_paginator.ctp` (fragmento AJAX)
- `SuporteController::add` → `add.ctp`
- `SuporteController::edit($id)` → `edit.ctp`
- `SuporteController::editobs($id,$idsuporte)` → `editobs.ctp`
- `SuporteController::view($id)` → `view.ctp` (layout modal)
- `SuporteController::principal` → `principal.ctp`
- `SuporteController::mostraobs($suporte_id)` → retorna array (consumido via requestAction em edit.ctp); `mostraobs.ctp` parece código de outra tela (referencia `ClienteFollowup`)
- `SuporteController::homogenizar|modulo_higienizar|metadados|buscacampos|buscacamposatualiza|buscavalores` → ferramentas de dados
- `SuporteController::buscarfuncoesmenu` → JSON (funcionalidades por módulo, INNER join modulecontrollers/modulecontrolleractions, `flag_menu=1`)
- `CmsController::index|add|edit($id)|ver($id)|downloads|aprendizado`; `view($id)` sem .ctp dedicado relevante; `delete` desativado
- `CmsCategoriasController::index|add|edit($id)|view($id)|delete($id)`

## Regras de negócio relevantes à UI
- **Abertura de chamado (add)**: sempre `status=1` (Aberto), `flag_usuario=2`, `abertopor=usuário logado`, `nivel_prioridade` escolhido na tela. `descricao_formatada` = descrição sem tags `<p>`.
- **Foto do chamado**: aceita só jpg/jpeg/gif/png, salva em `WWW_ROOT/fotos/` (mas o campo da tela está com nome errado — ver bug).
- **Observações (SuporteObs)**: cada interação grava `user_id` (logado), `descricao_formatada`, datas convertidas de dd/mm/yyyy para DB. Campos de data/responsável são exclusivos do master.
- **Funcionalidade dependente de Módulo**: o select Funcionalidade (`view`) é populado dinamicamente ao escolher o Módulo, via `buscarfuncoesmenu` (só ações com `flag_menu=1`).
- **CMS categorias fixas**: `downloads`→categoria 1, `aprendizado`→categoria 2 (hardcoded no controller).
- **CMS `ver`**: renderiza HTML do conteúdo sem escapar (esperado, é conteúdo rico do CKEditor).

## Máquina de estados / status (refletida na UI)
**Há TRÊS mapeamentos de status conflitantes** — replicar com cuidado, unificando:

1. **Mapa "real" exibido na lista** (`index_paginator.ctp`): 0=Aberto em desenvolvimento, 1=Aberto, 2=Em análise, 3=Analisado/aguardando desenvolvimento, 4=Em desenvolvimento, 5=Homologação usuário, 6=Homologação deferida, 7=Homologação indeferida, 8=Em produção, 9=Concluído, 10=Cancelado, 13=Concluído, 99=Indeferido.
2. **Mapa do filtro/add** (`index.ctp` filtro e `add()`): 1=Aberto, 2=Em andamento, 3=Fechado em homologação, 4=Fechado em produção, 5=Cancelado.
3. **Mapa antigo** (`editobs`): 1=Aberto, 0=Aberto em desenvolvimento, 2=Fechado em homologação, 3=Fechado em produção, 4=Cancelado.

**Transições no `edit()` (dependem do status atual e de `ismaster`)** — intenção do fluxo:
- 1 Aberto → master: {2 Em análise, 9 Indeferir}; não-master: {9 Indeferir}
- 2 Em análise → master: {3 Aguardando desenv., 4 Em desenvolvimento, 9 Indeferir}; não-master: {10 Cancelar}
- 3 Aguardando desenv. → master: {4 Em desenvolvimento, 10 Cancelado}; não-master: {10 Cancelar}
- 4 Em desenvolvimento → master: {5 Homologação Usuário, 10 Cancelar}; não-master: {10 Cancelar}
- 5 Homologação usuário → master: {8 Em produção, 10 Cancelar}; não-master: {14 Homologação recusada, 13 Concluído, 10 Cancelar}
- 8 Em produção → master: {9 Concluído}; não-master: {10 Cancelar}
- 9 Concluído → master: {9 Concluído}; não-master: {10 Cancelar}
- **Cuidado**: o PHP do `edit()` usa `elseif (cond) if(master){}else{}` sem chaves no `elseif` (dangling-else) → o agrupamento real diverge da intenção acima em vários ramos. Na migração, implementar a TABELA acima (intenção) e NÃO copiar o aninhamento bugado.

## Multi-tenant / white-label
- **Nenhum isolamento por tenant** nas telas de Suporte/CMS. Não há filtro por imobiliária/empresa nas queries de listagem (CMS lista tudo; Suporte filtra só por campos do formulário).
- Hardcodes que quebram multi-tenant e devem ser parametrizados/eliminados:
  - Nome de banco fixo `embracei_db_jtavares_v2` em `homogenizar.ctp`.
  - URLs fixas `/sistemajtavares/...` em `principal.ctp`, `metadados.ctp`, `homogenizar.ctp`, `modulo_higienizar.ctp`, `editobs.ctp`/`view` (TinyMCE em `/sistemajtavares/tinymce/...`).
  - E-mails de destino fixos (`fabiotorres0202@gmail.com`, `suporte@gridweb.com.br`) no envio de chamado.
- Permissão: CMS usa `VerificaPermissao()` (AppController) → tela `/Dashboard/proibido`. Suporte só aplica em `principal`. `ismaster` (campo `User.master == 'S'`) controla visibilidade de campos e transições de status.

## Gotchas / decisões kept-bug
- **NÃO replicar** todo o módulo de "higienização" (`homogenizar`, `modulo_higienizar`, `buscacampos*`, `buscavalores`, `busca_tabela_antiga`, `metadados`): são ferramentas de manutenção que executam SQL cru concatenado (`UPDATE`/`SHOW TABLES`/`information_schema`) — injeção de SQL trivial e dependente do schema legado. `popula_permissao()` faz `saveField` em loop de forma incorreta (sobrescreve sempre o mesmo registro).
- **Envio de e-mail quebrado** em `add()`/`edit()`: usa SMTP com destinatário hardcoded e `viewVars` referenciando variáveis indefinidas de um sistema escolar (`$descunidade,$usuario,$matricula,$nome,$anoletivo,$turma,$curso,$serie,$turno,$dadosfinanceiro`) → gera notices e mensagens de flash erradas ("Matrícula Alterada com Sucesso!"). NÃO replicar; modelar notificação real do ticket.
- **Código morto após `return`**: em `CmsController::edit` e `CmsCategoriasController::edit` há blocos `if(apply)` após `return $this->redirect(...)` que nunca executam (botão "Aplicar" está comentado nas views).
- **`Cms::delete` desativado**: começa com `echo 'Acesso proibido!'; die();`.
- **Campo de upload com nome divergente** em Suporte `add.ctp` (`upload` vs `foto_chamado`) → upload nunca é persistido pela tela atual.
- **`mostraobs.ctp`** contém marcação de outra tela ("teste", referências a `ClienteFollowup`) — não corresponde ao consumo real (em `edit.ctp` os dados vêm do retorno de `requestAction`, não desse template).
- **Status inconsistentes** entre as três telas (ver Máquina de estados) — unificar na migração.
- **`validacpf()`** usa `ereg_replace` e `$cpf{$c}` (sintaxe removida no PHP 7+) — função legada que provavelmente nem roda; não está ligada às telas de ticket.
- `view.ctp` (Suporte) e os campos `controller`/`campo` pertencem a um schema antigo; tela praticamente abandonada.

## Destino (issues Linear)
- Fase **F19 — Suporte e CMS (telas)**. Sugestão de quebra para os 3 projetos:
  - **wecorp-backend**: API de tickets (CRUD `suportes` + `suportes_obs`), máquina de status unificada com regra master/não-master, endpoint de funcionalidades por módulo (substituir `buscarfuncoesmenu`), upload de anexo do chamado, notificação por e-mail real. CRUD de CMS (`cms` + `cms_categorias`) com status Ativo/Inativo e categorias.
  - **wecorp-frontend**: telas de lista/filtro de tickets, abrir chamado, editar ticket + timeline de interações, edição de observação; telas de CMS (lista, editor rich-text substituindo CKEditor, ver conteúdo, downloads, aprendizado, CRUD categorias).
  - **wecorp-services**: editor de conteúdo rico (sanitização do HTML do CMS) e serviço de e-mail de notificação de chamado.
  - **Explicitamente fora de escopo / não migrar**: ferramentas de higienização de dados, `metadados` hub de links `/sistemajtavares/`, `principal` com iframes, `validacpf` legado.
