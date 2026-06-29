# Legado (telas) — Imóveis (F08)

> Cadastro/gestão de imóveis do plugin `Imovel` (CakePHP 2). Todas as telas abrem em **modal** (`layout='modal'`) via `janelaImoveis(origem, acao, id, ajax, janela)` e são acionadas a partir de outros módulos (CRM, Vistoria, Segurofianca, Sign, etc.) por `origem`. O cadastro é gravado por AJAX (`jQuery.ajaxForm`) com barra de progresso; a listagem é filtrada e paginada via AJAX.

## Cobertura
Views (`app/Plugin/Imovel/View/Imovels/`):
- `index.ctp` — tela de filtro + container do paginador
- `index_paginator.ctp` — tabela de resultados (parcial AJAX)
- `novo.ctp` — formulário de cadastro/edição (usado por `novo` E `edit`)
- `ver.ctp` — visualização somente-leitura + características
- `fotos.ctp` — galeria de fotos do imóvel
- `anexos.ctp` — lista de documentos anexos
- `invalido.ctp` — mensagem de imóvel inexistente/acesso inválido

Controller: `app/Plugin/Imovel/Controller/ImovelsController.php` (ações `index`, `index_paginator`, `novo`, `edit`, `ver`, `fotos`, `anexos`, `downloadFotos`, `downloadAnexos`).
JS: `app/webroot/js/modules/imoveis/ui.js` (navegação modal, seleção, troca de consultor por empresa).

> Não há views .ctp para `ImoveisTiposController` / `ProprietariosController` — fora do escopo de telas.

## Telas / fluxos

### `index.ctp` — Gestão de Imóveis (filtro)
- Botão "Novo Imóvel": `closeModal(); janelaImoveis(origem,'novo', id,'1','1')`.
- Form GET `#imovel` com hidden: `origem`, `ajax`, `janela`, `bloco_select`.
- Filtros (texto): `id`, `id_ref`, `codigo_ref`, `endereco`, `numero`, `bairro`. Selects: `empresa_id` (select2, classe `lista_imobiliarias required`), `corretor_id` (consultor; recarregado via `carregaUsuarioImovel(empresaId)` → `users/usuarios_por_empresa/<id>`, alvo `#alvo_usuarios`), `status_cadastro` (select2), `motivo_cadastro` (bloco `.motivo`, oculto por padrão com `hide`).
- Comportamento JS: ao mudar `.status_cadastro`, se valor `== 6` mostra `.motivo`, senão oculta (ui.js linhas 39-52). **Atenção: ui.js só trata o status 6; o JS inline de `novo.ctp` trata 5 e 6 — ver Gotchas.**
- Botão "Pesquisar" (`#pesquisarImovel`): serializa o form e chama por GET `/imovel/imovels/index_paginator/<origem>/0/<ajax>/<janela>/<bloco_select>`, injetando o HTML em `#paginator_imovel`. Indicador "Aguarde..." em `#busca_imovel`.
- A lista NÃO carrega automaticamente: `#paginator_imovel` inicia com "Faça a pesquisa acima para filtrar os imóveis".
- Scripts incluídos: select2, `modules/endereco.js`, jquery-ui, `modules/imoveis/ui.js`; se `inclui_js==1`, injeta jQuery 3.5.1 do CDN (ver Gotchas — lógica de flag quebrada).

### `index_paginator.ctp` — tabela de resultados
- Colunas: Id, Id Ref, Cod Ref, Tipo do imóvel + Imobiliária, Endereço (`endereco, numero complemento`), Bairro/Cidade + CEP, Usuário (`User.name`), Cadastro (`Dataformat::dateBr(created,true)`), Status (`$statusCadastro[status_cadastro]`), Ações.
- Tipo do imóvel: `$imovelTipos[tipo_imovel_id]` ou "N/d" se vazio.
- Ações por linha (todas chamam `closeModal()` antes):
  - **Selecionar** (`glyphicon-ok`) — só aparece quando `bloco_select=='sim'` OU `origem!='0'`; chama `selecionarImovel(origem,'add',id,tipo,id_ref,cod_ref,endereco,numero,complemento,bairro_id,cidade_id,bairro,cidade,uf,cep,localizacao_aproximada,nome_condominio)`. Strings escapadas com `addslashes`.
  - Editar → `janelaImoveis(origem,'edit',id,'1','1')`
  - Visualizar → `janelaImoveis(origem,'ver',id,'1','1')`
  - Follow-ups → `janelaFollowups('imovel',id,empresa_id,'1','lista','')`
  - Fotos → `janelaImoveis(origem,'fotos',id,'1','1')`
  - Anexos → `janelaImoveis(origem,'anexos',id,'1','1')`
- Paginação: `limit=5` (controller), `Paginator->counter/prev/numbers/next`, ordem `Imovel.id DESC`.

### `novo.ctp` — Cadastro/Alteração (compartilhada por `novo` e `edit`)
- `$action` decidido por `$this->params['action']` (`edit` vs `novo`). Form action: `/<action>/<origem>/<id>/0/1`, `id='Imovel'`, `onsubmit='return verificaForm(this)'`, `autocomplete=off`.
- Quando há `$id`, exibe badge "Id: N" e botões Fotos/Documentos no topo.
- **Validação JS `verificaForm` (inline):**
  - Se `empresa_novo_id` vazio OU `cep` vazio → alert "Informe a empresa! Informe tipo, cep e endereço do imóvel!" e retorna false.
  - Se `tipo_imovel_id` é `'1'` ou `'3'` E `complemento` vazio → alert "Para imóvel do tipo informado, informe o complemento! Ex: 201" e false.
  - **Bug kept:** lê `endereco_numero`/`endereco_complemento` de `#ImovelNumero`/`#ImovelComplemento` mas só usa `complemento`; e o `else` final não submete (`form.submit()` comentado) — quem submete é o `ajaxForm`.
- Seções/campos:
  - **Imobiliária/Empresa:** `empresa_novo_id` (select2 `lista_imobiliarias required`; `empty='Selecione'` só para grupo 1/2), `corretor_novo_id` (consultor; recarregado via `carregaUsuarioImovelNovo()` em `change` e botão lupa → `users/usuarios_por_empresa/<id>`, alvo `#alvo_usuarios_novo`).
  - **Tipologia e Endereço:** `tipo_imovel_id` (select required), `cep` (maxlength 11, `onblur`/botão dispara `searchAddressByCep(...,'endereco_imovel_cep','rf_end_res','rf_bairro_res','rf_cidade_res','rf_uf_res')`), `endereco` (readonly required), `numero` (required), `complemento`, `uf` (select required, `onChange=buscaCidades('rf_cidade_res',value)`), `cidade_id` (`onChange=buscaBairros('rf_bairro_res',value)`), `bairro_id`, `codigo_ref_imovel`, `id_ref_imovel`, `administrado` (select `flagVF` = S/N, required), `localizacao_aproximada`, `nome_condominio`.
  - **Proprietário:** hidden `proprietario_id`; `proprietario_nome`, `proprietario_tipo_pessoa` (PF/PJ), `proprietario_documento`, `proprietario_email`, `proprietario_celular` (`data-mask=mobile`).
  - **Informações financeiras:** checkbox `status_locacao` ("Disponível para aluguel") + `valor_imovel_aluguel`, `valor_iptu_mensal`, `valor_condominio` (todos com máscara `onkeypress=MascaraMoeda(this,'.',',',event)`); checkbox `status_venda` + `valor_imovel_venda` (máscara moeda). (As `span#disponivel_locacao`/`#disponivel_venda` existem mas não há JS de show/hide ligado a elas nesta view.)
  - **Ficha técnica:** `numero_salas`, `numero_quartos`, `numero_suites`, `garagens`, `numero_banheiros` (number, min 0), `mobiliado` (select `flagSN` 1/0), `descricao_imovel`, `obs_extras` (textarea), `area_util`, `sol` (posicaoSol), `posicao_imovel` (posicaoImovel), `conservacao`, `nome_condominio` (**duplicado** — aparece novamente aqui além da seção endereço; ver Gotchas).
  - **Informações Administrativas:** `inscricao_iptu`, `local_chaves` (locaisChave), `ocupante_atual` (ocupantes), `marcar_visitas` (textarea).
  - **Status:** `status_cadastro` (select required, `default=3`), com dois blocos de motivo ocultos: `.motivo` (`motivo_cadastro`/`motivoAlugado`, alvo `#alvo_motivos`) e `.motivoInativo` (`motivoInativo`, alvo `#alvo_motivos_inativo`).
- **JS de status (inline, novo.ctp 518-535):** ao mudar `.status_cadastro`: `==6` mostra `.motivo`; `==5` oculta `.motivo` e mostra `.motivoInativo`; senão oculta ambos.
- **Submit:** botão `#ProcessaCadastroImovel` (`data[submit]`, troca texto para "Aguarde..."). Usa `jQuery('#Imovel').ajaxForm(...)` com `#progressbar` (uploadProgress %). `success`: se resposta `=== true` → alert "Operação realizada com sucesso!" e volta para `janelaImoveis(origem,'index','0','0','1')`; senão alert de erro. Barra reseta após 2s.

### `ver.ctp` — Visualização (somente leitura, mesma estrutura de campos)
- Alertas "Modo de visualização!" no topo e rodapé; botão "Editar imóvel" → `janelaImoveis(origem,'edit',id,'1','1')`.
- Form com `novalidate=TRUE`. Campos iguais ao `novo.ctp` (empresa, tipologia/endereço, ficha técnica, financeiro) **mas sem readonly/disable real** — os inputs ainda são editáveis; a "leitura" é apenas visual (ver Gotchas).
- Seção exclusiva: três colunas de características renderizadas iterando `$ImovelCaracteristicas` cruzado com `$CaracteristicasImoveis` / `$CaracteristicasEdificios` / `$CaracteristicasRedondezas` (lista com ícone check verde). Não há `<form submit>` funcional para gravar.

### `fotos.ctp` — Galeria
- Cabeçalho com Id/endereço do imóvel e botões Listar/Editar/Fotos/Documentos.
- Se `$listaFotosImovel != null`: botão "Baixar fotos" → `/imovel/imovels/downloadFotos/imoveis/<id>/1/1` (target _blank); grid `.gallery` com `<img>` linkando para `JS_ROOT_PATH + foto_url + foto_nome`, legenda = `foto_legenda`.
- **Não há upload de fotos nesta tela** (apenas listagem/download).

### `anexos.ctp` — Documentos
- Mesmo cabeçalho. Se `$listaAnexosImovel != null`: botão "Baixar arquivos" → `/imovel/imovels/downloadAnexos/imoveis/<id>/1/1`; lista `<ul><li><a href=JS_ROOT_PATH_files<url><nome>>`. Senão "Não há arquivos anexados a este imóvel."
- **Não há upload de anexos nesta tela.**

### `invalido.ctp`
- Alerta "Imóvel inexistente ou acesso inválido!!!". Renderizado por `edit`/`ver` quando o `find('first')` retorna vazio (fora do escopo do tenant).

## Pontos de entrada (controller::ação que renderiza)
- `ImovelsController::index($origem,$id,$ajax,$janela)` → `index.ctp` (layout modal se `janela==1`). Seta `imovelTipos`, `listaImobiliarias`, `statusCadastro`, `motivoAlugado`, `consultor=[]`.
- `ImovelsController::index_paginator(...,$bloco_select)` → `index_paginator.ctp`. Sempre layout modal. Aplica filtros GET (`_GET['data']['Imovel'][...]`) e paginação (limit 5).
- `ImovelsController::novo(...)` → `render('novo')`. POST grava via AJAX e retorna `json_encode(true|false)`.
- `ImovelsController::edit(...)` → `render('novo')` (reusa view) ou `render('invalido')` se não achar. POST grava e retorna json bool.
- `ImovelsController::ver(...)` → `render('ver')` ou `invalido`.
- `ImovelsController::fotos(...)` → `render('fotos')`; `anexos(...)` → `render('anexos')`.
- `ImovelsController::downloadFotos/downloadAnexos` → `autoRender=false` (gera download/zip).

## Regras de negócio relevantes à UI
- **Consultor depende da empresa:** trocar `empresa_*_id` recarrega o select de consultores via `users/usuarios_por_empresa/<empresa_id>`. Em `index` alvo `#alvo_usuarios`; em `novo`/`edit` alvo `#alvo_usuarios_novo`.
- **Empresa→Imovel no POST:** controller copia `empresa_novo_id` → `empresa_id` (`novo` linha 459, `edit` linha 685).
- **Proprietário inline:** os campos `proprietario_*` criam/atualizam um registro `Proprietario` (model separado). Em `novo`, cria se `proprietario_nome` preenchido. Em `edit`, cria se `proprietario_id` vazio, senão atualiza. `token` gerado por `getToken(10)`; `empresa_id` herda do imóvel.
- **Endereço:** controller resolve nomes de bairro/cidade a partir de `bairro_id`/`cidade_id` (`Endereco->getBairroNome/getCidadeNome`) e grava `bairro`/`cidade` textuais. Bairro/cidade na edição podem vir de `getAddressFromCorreiosByCep`.
- **Valores monetários:** UI usa máscara `MascaraMoeda`; controller converte com `Dataformat::currencyToDecimal` ao salvar e `Dataformat::currencyMoeda` ao exibir (`aluguel`, `venda`, `iptu_mensal`, `condominio`).
- **Filtro de tipos por origem:** se `origem=='Vistoria'`, `tiposImoveis` é filtrado por `ImovelTipos.flag_vistoria=1`.
- **`user_responsavel_id`** = usuário logado (auditoria de quem cadastrou); `user_id` (consultor) só setado se informado.

## Máquina de estados / status (refletida na UI)
- `status_cadastro` vem de `Libmetadados->get('status_imovel')`; default no cadastro = **3**.
- Comportamento de motivo por status (JS):
  - **6** → exibe bloco `.motivo` (motivos de "alugado", `motivo_alugado`).
  - **5** → exibe `.motivoInativo` (`motivo_inativo`). (Somente no JS de `novo.ctp`; o `index.ctp`/`ui.js` só tratam o 6.)
  - demais → nenhum motivo.
- `status_locacao` / `status_venda` são flags booleanas (checkbox) de disponibilidade.
- Filtro da listagem também permite `motivo_cadastro`.

## Multi-tenant / white-label
- Escopo por `grupo_id` + `empresa_id` do usuário logado (`AuthComponent::user`):
  - **grupos 1,2,3,4,9** (admin/parceiras): veem todas as imobiliárias (`Empresa->getParceiras()`); para 2/3/4/9 ainda há filtro adicional por `empresa_resp_seguros/analises/vistorias`.
  - **grupo 5** e demais: restritos à própria empresa e às `empresa_master` (franqueadora). Cliente final (`flag_empresa_associada==0`) exclui empresas associadas (`empresa_associada<>1`).
  - `permissao['nivel']==2` força `Imovel.empresa_id = empresa do usuário`.
- Em `index.ctp`/`ver.ctp` o `empty` do select de empresa é "Selecione" só para grupo 1/2; para os demais o select fica restrito.
- `edit`/`ver` aplicam o mesmo escopo no `find`; fora do escopo → `invalido.ctp`.
- `VerificaPermissao()` em `index`; **`index_paginator` força `$permissao=true` (NÃO checa permissão)** — ver Gotchas.

## Gotchas / decisões kept-bug
- **`index_paginator` sem checagem de permissão:** `VerificaPermissao()` comentado e `$permissao=true`. O endpoint AJAX só confia no escopo SQL por `grupo_id/empresa_id`. NÃO replicar o bypass; manter o escopo de tenant no backend.
- **`if($origem!='Crm' or $origem!='' or $origem!='0')` (index.ctp:15):** condição sempre verdadeira (OR de desigualdades) → `bloco_select` e `inclui_js` sempre `'0'`. Resultado prático: jQuery do CDN nunca é injetado e `bloco_select` sempre 0. Bug lógico — na migração, definir intenção real (provavelmente AND).
- **`ver.ctp` não é realmente somente-leitura:** inputs não têm `readonly/disabled`; o form até tem `action` de `ver`, mas não há ação POST `ver` que grave. É "read-only" só por convenção visual. Na migração, tornar a tela de visualização efetivamente não-editável.
- **`nome_condominio` duplicado** em `novo.ctp` (seção endereço linha 186 e ficha técnica linha 395) — dois inputs com o mesmo name; o último vence no submit. Consolidar em um único campo.
- **`verificaForm`:** valida só empresa/cep e complemento p/ tipos 1 e 3; mensagem de alerta fala em "tipo, cep e endereço" mas não valida tipo/endereço. Texto enganoso, manter validação coerente na migração.
- **Duas definições de `carregaUsuarioImovel`** em `ui.js` (linhas 23 e 77) com alvos diferentes (`#alvo_usuarios_index` vs `#alvo_usuarios`); a 2ª sobrescreve a 1ª. Idem `$('.status_cadastro').change` aparece em ui.js e inline em `index.ctp`.
- **`fotos`/`anexos` são read-only na UI** (só listam/baixam). O upload acontece em outro fluxo (model `ImovelFotos`/`ImovelAnexos`); não há tela de upload aqui.
- **Paginação fixa em 5 registros** — provavelmente baixo demais para produção; reavaliar na migração.
- **`empty` no select `bairro_id`/`cidade_id`** difere entre `novo` (com `empty=>false`) e `ver` (sem) — pequena divergência de comportamento.
- Campos do form usam IDs gerados pelo Cake (`ImovelEmpresaNovoId`, `ImovelTipoImovelId`, `ImovelCep`, etc.) que o JS referencia diretamente; preservar o mapeamento name→id ao migrar.

## Destino (issues Linear)
- Projeto **wecorp-frontend**, fase **F08 — Imóveis (telas)**. Telas a migrar (Next.js): lista+filtro (index/paginator), formulário cadastro/edição (novo), visualização (ver), galeria de fotos, lista de anexos, estado inválido. Backend correspondente (NestJS): CRUD imóvel + proprietário inline + escopo multi-tenant + endpoints de filtro/paginação e download de fotos/anexos. (IDs específicos de issues não confirmados nesta análise — mapear no board F08.)
