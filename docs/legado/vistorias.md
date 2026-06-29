# Legado (telas) — Vistorias

> Módulo de gestão de vistorias de imóveis (CakePHP 2, plugin `Vistorias`). As telas cobrem listagem/filtro, agendamento, mudança de status (histórico), revisão da vistoria, revisão por cômodo (com fotos/itens), reordenação de fotos, cômodos e aditivos (follow-ups). Forte acoplamento ao App mobile externo (`DOMINIO_WS`) que envia fotos/textos; várias telas são read-only espelhando o que o vistoriador preencheu no app.

## Cobertura
Views (`app/Plugin/Vistorias/View/`):
- `Vistorias/index.ctp` + `Vistorias/index_paginator.ctp` (listagem AJAX paginada)
- `Vistorias/addagenda.ctp`, `Vistorias/addhistorico.ctp` (status), `Vistorias/alterarobs.ctp`
- `Vistorias/revisao.ctp`, `Vistorias/revisaocomodo.ctp`, `Vistorias/ordenarfotos.ctp`, `Vistorias/novocomodo.ctp`, `Vistorias/revisaocomodo.ctp`
- `VistoriaAditivos/lista.ctp`, `VistoriaAditivos/novo.ctp`, `VistoriaAditivos/aditivo.ctp`
- `StatusVistorias/index.ctp`, `StatusVistorias/add.ctp`

Controllers (só para entender comportamento da tela):
- `VistoriasController.php` (god file ~4.458 linhas): `index`, `index_paginator`, `aceitaTermos`, `addagenda`, `addhistorico`, `revisao`, `revisaocomodo`, `ordenarfotos`, `novocomodo`, `novoitemcomodo`, `alterarobs`, `girarFoto`, `imprel`
- `VistoriasFotosController.php`: `upload`, `uploadaditivo`
- `VistoriaaditivosController.php`, `StatusVistoriasController.php`
- `AppController::VerificaPermissao()` (definição de `nivel`)

## Telas / fluxos

### 1. Listagem — `Vistorias/index.ctp` → `index_paginator.ctp`
Tela em duas partes: `index.ctp` renderiza o cabeçalho + painel de filtros (accordion "Opções de Filtros") e um `<div id="paginator">` vazio; o grid real é carregado via AJAX por `index_paginator.ctp` (JS em `/Vistorias/js/vistoria.js`, botão `#btnPesquisar` classe `.pesquisar`).

Campos de filtro (form `Vistoria`, `parsley-validate`, `novalidate`, `autocomplete=off`):
- `codigo` — number (`min=1`)
- `status_id` — select de `StatusVistoria` (`empty=Todas`)
- `solicitacao_dataini` / `solicitacao_datafim` — text `.date .datepicker` (filtra por `solicitacao_data`)
- `endereco_imovel` — texto (LIKE no logradouro; dica: pesquisar "Presidente Vargas", não "Av Presidente Vargas")
- `endereco_imovel_numero` — texto
- `empresa_parceira_id` — input autocomplete (select2-like, `#endereco_1`), **só para grupos 1,2,3,4,5,9** (internos)

Botões de topo: "Nova Vistoria" (→ `/vistorias/vistorias/add/`), "Relatórios" (→ `imprel`, exibido se `grupo_id<>''`), "Calendario" (→ `/event/scheduler`, só grupos 1,2,3,4,9), e link para o app Android Play Store.

Grid (`index_paginator.ctp`) — colunas: Cod, **empresa** (só se `$mostra_colunas==1`), Data Solicitação e Confirmação, Tipo, Imóvel (endereço+bairro+áreas m²), Situação Imóvel, Modalidade entrega (`descricaopreco`), Vistoriador, Status, Ações. Paginação CakePHP (10/página, ordem `Vistoria.id DESC`), AJAX em `#paginator` com indicador `#busy-indicator`.

Cor do status (badge `label-*`): 1=warning, 2=primary, 3=success, 4=danger, 5=default, 6/7/8=default.

Ações por linha (dependem de grupo + status — ver seção máquina de estados):
- Editar (sempre) → `edit`
- Anexos (`anexovistoria(id, empresa_parceira_id)`)
- Agendar (`addagenda`) — só status 1 ou 2
- Atualizar Status (`addhistorico`)
- Revisão (`revisao`, target _blank) e Follow-up/Aditivos (`janelaAditivos(...)`) — só status 4,5,6,7
- Laudos: `laudo_pdf/{id}/1` (completo), `/2` (resumido), `/3` (navegador) — só status 4,5,6,7

### 2. Agendamento — `Vistorias/addagenda.ctp`
Form `Vistoria`. (controller `addagenda`, linha 1150). Tela de agenda data/hora/vistoriador. Não detalhada aqui além do papel: define data/hora e vistoriador, muda status para agendado.

### 3. Atualizar Status / Histórico — `Vistorias/addhistorico.ctp`
Form `Historicostatusvistoria`. Cabeçalho com dados read-only da vistoria (empresa, solicitante, solicitação data/hora, endereço completo, pré-agenda se houver, data/hora vistoria + select `vistoriador_id`). 
- Caixa "Histórico/observações" — exibe `obsvistoria_historico` (HTML; `<hr />` é trocado por linha divisória), scroll vertical.
- `status_vistoria_id` — select required, `onchange='trocarstatus(this.value)'`; opções limitadas pela regra de negócio (ver máquina de estados). Hidden `status_vistoria_old`.
- `obs_status` — textarea.
- Bloco `#div_pontuacao` (Valor Vistoriador + Pontuação) — **exibido apenas quando status == 6** (JS `trocarstatus`); `vlr_vistoriador` com máscara `MascaraMoeda(this,".",",",event)`, default `'20,00'`; `pontuacao` select 1..10 (bug: rótulos `'6'=>7,'7'=>7` — duplica 7, pula 6).

### 4. Revisão da Vistoria — `Vistorias/revisao.ctp` (controller `revisao`, target _blank)
Form `Vistoria`. Read-only: empresa, solicitante/data/hora, imóvel completo (CEP/endereço/nº/compl/UF/bairro/cidade/cód ref/id ref), agenda. Campos editáveis:
- `vistoriador_id` (Alterar Vistoriador) e `revisor_id` — selects de `usuarios` (grupos 10,1,2,3,4,6,7,8,5).
- `status_vistoria_id_atual` — select desabilitado (status 4,5,6).
- Observação da Vistoria — tabela Vistoriador vs Revisor; botão "Atualizar" abre `alterarObs(idtexto, id)` (modal → `alterarobs.ctp`).
- "Cômodos Vistoriados" — lista sortable (`#lista_comodos`, jQuery UI sortable, ícone glyphicon-move) com hidden `ordem_comodo_vistoriado[]` e `id_itemvistoriarealizada[]`; ao gravar, a ordem é persistida (`ordem_comodo` sequencial). Por cômodo: botão "Revisão" (`revisaocomodo(vistoria,comodo,numcomodo)`) e "Excluir" (`deletarcomodo(...)`). Botão "Novo Cômodo" → `novocomodo/{id}`.
- `Vistoria.flag_foto_data` (Sim/Não) — exibe data/hora nas fotos do laudo.
- `data_vistoria_laudo` — `.date .datepicker required`.
- `flag_exibe_campos_assinatura` (Sim/Não, default Sim).
- `finalizar`/`fim_vistoria` (Não/Sim, `onchange='mostrardiv'` mostra `#divfim`). Se status já == 7 mostra aviso "já Finalizada/Entregue" e bloqueia retorno (só "Sim").
- Bloco final (`#divfim`, grupos 1,2,3,4,9 ou `flag_vistoria_autonoma==0`): `vlr_vistoriador` (máscara moeda, default '20,00'), `pontuacao` (1..10, mesmo bug do 7 duplicado). Campos `obs_vistoriador` existem mas `display:none` (duplicados).
- Alerta vermelho "Atenção, vistoria sem fotos!" se `qtd_fotos==0`.
- Botão "Processar". Ao submeter com `fim_vistoria==1` → status vira **7 (Entregue)**.
- Editor TinyMCE (plugin table) nas textareas.

### 5. Revisão por Cômodo (fotos + itens) — `Vistorias/revisaocomodo.ctp` (carregada em janela/modal)
Tela dividida em duas colunas: **fotos** (esq) e **itens vistoriados** (dir). Envia headers no-cache. URL base das fotos = `DOMINIO_WS` (constante; serve as fotos enviadas pelo app mobile). `params['pass'][0]=vistoria, [1]=comodo, [2]=numcomodo`.
- Form de upload (`VistoriasFotos`, action `upload/{vistoria}/{comodo}`, `type=file`, `#FormFotosVistoria`): input `fotos[]` multiple; checkboxes `flag_laudo` (checked) e `ignora_redimensionamento`; barra de progresso via `jquery.form` ajaxForm; padrão 800px largura, recomenda Chrome. Sucesso → recarrega `revisaocomodo(...)`.
- Lista de fotos (`#lista_fotos`, jQuery UI sortable): cada foto com ícone laudo Sim/Não (`mudaFotoLaudo(fotoId, 0/1, div)`), girar -90/+90 (`rotateImage` → AJAX `vistorias/vistorias/girarFoto`), excluir (`excluirFotoLaudo(...)`), lightbox, overlay data/hora (`Photo.created` ou `tiradaEm`). Hidden por foto: id/ordem/nome (`data[Fotosvistoria][id][...]`). Botões "Anexar/Enviar novas fotos" e "Reordenar fotos" (→ `ordenarfotos`).
- Coluna itens: observação do cômodo (vistoriador vs revisor) e por item: `Itensvistoriado.nome`, texto original do vistoriador + textarea `data[Itensvistoriasrealizada][id][texto_revisado]`; excluir item (`excluirItemVistoriado(...)`). Botão "Adicionar item ao cômodo" abre form `#FormItemComodo` (action `novoitemcomodo/{vistoria}/{comodo}`, select `item_id` de `itensComodos`).
- Botão "Gravar revisão dos itens". TinyMCE com toolbar reduzida.

### 6. Reordenar Fotos — `Vistorias/ordenarfotos.ctp`
Versão "só fotos" da `revisaocomodo` (miniaturas 220x260), `#lista_fotos` sortable; mesmos controles laudo Sim/Não, excluir, lightbox, upload. Botão "Gravar ordenação" persiste `ordem` das fotos. Mesmo bloco de upload e JS.

### 7. Novo Cômodo — `Vistorias/novocomodo.ctp`
Form `Vistoria` com dados read-only do imóvel/agenda + select `comodo_id` (`listComodos`, `onchange='exibiritem(this.value)'`) que carrega itens via AJAX numa tabela `#tabitem`. Botão "Gravar a Revisão do Cômodo". (Idêntica à estrutura de `revisaocomodo.ctp` topo.)

### 8. Alterar Observação — `Vistorias/alterarobs.ctp` (modal)
Tabela Vistoriador (read-only `$textovistoriador`) vs Revisor (textarea `data[Itensvistoriasrealizada][idobs][texto_revisado]`). Botão "Gravar a Alteração". TinyMCE.

### 9. Aditivos (Follow-ups) — `VistoriaAditivos/*` (em modal/janela `janelaAditivos`)
- `lista.ctp`: grid de aditivos da vistoria. Colunas id/Imobiliária (só se `$mostra_colunas==1`), Observações (`substr(strip_tags(texto),0,99).' ...'`), Data (`data_aditivo`+`hora_aditivo`). Ações: editar (`janelaAditivos(id,1,'aditivo','aditivo')`) e PDF (`vistoriaaditivos/aditivo_pdf/{vistoria}/{aditivo}/1`, _blank). Botão "Novo aditivo" (`janelaAditivos(vistoriaid,1,'novo','novo')`).
- `novo.ctp`: form `Vistoriaaditivos` (action `novo/{fk}/1/novo/novo`, `autocomplete=off`). `empresaparceira_id` select required readonly; cabeçalho imóvel/agenda read-only; `texto` textarea required (CKEditor `#editor` com `o2config.js`); `data_aditivo` (`.date`) e `hora_aditivo` (`.time`). Submit "Gravar e avançar" → ajaxForm com progress; sucesso volta para `lista`. Mensagem: "Após gravar... você poderá enviar as fotos".
- `aditivo.ctp`: edição do aditivo + envio de fotos do aditivo (form `VistoriasFotos` action `upload/{aditivo}/{...}` com hidden `flag_aditivo=1`, `aditivo_id`, `vistoria_id`). Mesmos controles de foto/upload/lightbox da `revisaocomodo`.
- `lista_aditivos.ctp` (em `Vistorias/`): variante que renderiza "Follow-up" (`janelaFollowups(model, fk, empresaparceira, 1, 'novo'/'editar'/'ver', seguradora)`); editar só grupos 1,2,3. Usa model `Followup`.

### 10. Status (cadastro) — `StatusVistorias/index.ctp` + `add.ctp`
CRUD administrativo simples de `StatusVistoria` (id, descricao). Index: lista paginada com editar/`postLink` delete (confirm "Deseja mesmo remover o Registro ID # %s?"). Add: form de cadastro.

## Pontos de entrada (controller::ação que renderiza)
- `VistoriasController::index()` → `index.ctp` (filtros vazios; lista status 1,2,3,4 no GET inicial)
- `VistoriasController::index_paginator()` → `index_paginator.ctp` (layout `modal`; aplica filtros de `$_REQUEST['data']`)
- `VistoriasController::addagenda($id)` → `addagenda.ctp`
- `VistoriasController::addhistorico($id)` → `addhistorico.ctp`
- `VistoriasController::revisao($id)` → `revisao.ctp`
- `VistoriasController::revisaocomodo($vistoriaid,$comodoid,$numcomodo)` → `revisaocomodo.ctp`
- `VistoriasController::ordenarfotos($vistoriaid,$comodoid,$numcomodo)` → `ordenarfotos.ctp`
- `VistoriasController::novocomodo($vistoriaid)` → `novocomodo.ctp`
- `VistoriasController::novoitemcomodo($vistoriaid,$comodoid)` → JSON (adiciona item, retorna true)
- `VistoriasController::alterarobs($idobs,$id)` → `alterarobs.ctp`
- `VistoriasController::aceitaTermos($empresa_id)` → AJAX (grava aceite, retorna true)
- `VistoriasController::girarFoto()` → AJAX (rotaciona imagem, retorna nome)
- `VistoriasFotosController::upload($vistoria_id,$comodo_id)` / `uploadaditivo($aditivo_id)` → JSON true
- `VistoriaaditivosController` → `lista/novo/aditivo/aditivo_pdf`

## Regras de negócio relevantes à UI
- **Aceite de termos**: `index.ctp` só mostra filtros/lista se `aceitou_termos_vistoria==1`; caso 0, exibe textarea com contrato + botão `#BtnAceitoTermosUsoVistoria` (confirm "Concordar e continuar?", POST `aceitaTermos/{empresa_id}`, reload). Vem de `checaAceiteVistoria()` — só aplica a `nivel==2` (empresa-cliente externa).
- **Versão Demo**: `checaDemo()` (só `nivel==2`) exibe banner: máx. 3 vistorias, data de expiração; fase 1 (success) ou fase 2 expirada (danger). Limita cadastro.
- **Fotos vêm do App mobile**: a base das fotos é `DOMINIO_WS` (web service externo). Telas de revisão são predominantemente read-only espelhando textos/fotos enviados pelo vistoriador via app (`o2vistoria.gridweb.com.br`). `tiradaEm` indica origem app; `created` indica upload web.
- **flag_laudo** por foto controla se a foto entra no laudo PDF (toggle Sim/Não em tempo real).
- **Faturamento (`status_fatura`)**: em `addhistorico`, vira 1 quando transição old→new ∈ {2→7, 3→7, 2→8, 3→8, 4→8, 4→7} — momento em que a vistoria fica apta a ser cobrada.
- **Histórico de status**: cada mudança de status/obs anexa linha `<b>data - usuário</b> - novo status: X` em `obsvistoria_historico` (separador `<hr />`). Só grava se mudou status OU há `obs_status`.
- **E-mails automáticos** (`addhistorico`): old≠5 & new==2 → "confirma_processamento"; old≠6 & new==7 → "confirma_entrega" (via `EmailController::emailNotification`).
- **Finalizar na revisão** (`revisao`): `fim_vistoria==1` força `status_vistoria_id=7`; também reordena cômodos e converte `vlr_vistoriador` por `currencyToDecimal`.

## Máquina de estados / status (refletida na UI)
StatusVistoria (IDs observados): 1=Em aberto, 2=Confirmada e agendada, 3=Em andamento, 4=Enviada, 5=(default/—), 6=(usado p/ pontuação; "concluída p/ pagamento"), 7=Entregue, 8=Cancelada.

Transições permitidas no select de `addhistorico.ctp` (limitadas por status atual):
- 1 → {1, 8}
- 2 → {2, 8}
- 3 → {3, 8}
- 4 → {4, 8}
- 7 → {7, 8}
- 8 → {8}
(observação: tela permite essencialmente "manter" ou "cancelar"; demais transições vêm de outros fluxos como `revisao`/app.)

Ações na listagem por status:
- Agendar: status 1,2.
- Atualizar status: status 1–8 (grupos internos/autônoma).
- Revisão + Aditivos: status 4,5,6,7 (grupos 1,2,3,4,9,10 ou autônoma).
- Laudos PDF: status 4,5,6,7.

`revisao` só lista status {4,5,6} no select; `pontuacao` aparece em status 6.

## Multi-tenant / white-label
- **`VerificaPermissao()['nivel']`** define escopo: `nivel==2` = usuário de empresa-cliente externa → filtra por `empresa_parceira_id == AuthComponent::user('empresa_id')` (e bloqueia acesso direto a registros de outra empresa em `revisao`/`addhistorico` com flash de permissão + redirect index).
- **`$mostra_colunas`** controla exibição da coluna "Imobiliária/Empresa" (apenas usuários internos/admin veem).
- **Grupos** (`AuthComponent::user('grupo_id')`):
  - 1,2,3,4,9 (e 5,10 em alguns pontos) = internos/admin → veem todas, filtro de empresa, calendário, etc.
  - 2 = admin de empresa parceira → escopo por `empresa_resp_seguros/analises/vistorias`.
  - 5,6,7,8,9,10 = imobiliárias → escopo `User.empresa_id`; grupos 6,7,8,10 não enxergam `empresa_master` (só própria empresa).
  - `flag_vistoria_autonoma` (na Empresa) = imobiliária faz vistoria própria → libera ações de revisão/finalização e zera preço (`descricaopreco='Autonoma'`).
- **`empresa_master`**: na busca, grupos não-restritos incluem vistorias de empresas filhas (`Empresa.empresa_master == empresa_id`).

## Gotchas / decisões kept-bug
- **`pontuacao` select**: valores `'6'=>7,'7'=>7` — o número 6 é pulado e 7 aparece duas vezes (em `revisao.ctp`, `addhistorico.ctp`). **NÃO replicar** — corrigir para 1..10 sequencial no novo sistema.
- **`vlr_vistoriador` default `'20,00'`** hard-coded quando não há valor — confirmar se deve continuar.
- **Dois IDs `#progressbar` duplicados** no mesmo DOM (lista + spinner) em revisaocomodo/ordenarfotos — HTML inválido; não replicar.
- **Campos `obs_vistoriador` duplicados** com `display:none` em `revisao.ctp` (linhas 367–378) — morto; remover.
- **URL base de fotos**: `DOMINIO_WS` é constante global (web service externo legado). Migrar para storage/CDN próprio; tratar fotos antigas com `base_url`/`url`/`nome` concatenados e `tiradaEm` vs `created`.
- **`index.ctp` carrega scripts inline** (`vistoria.js`, `ui.js`, TinyMCE) e depende de `ROOT_PATH`/`VERSION` globais.
- **`revisaocomodo`/`ordenarfotos` divergem na URL base**: ordenarfotos usa `Router::url($url_base...,true)` e `'/'` para fotos sem `tiradaEm`; revisaocomodo usa string vazia. Comportamento inconsistente para fotos legadas sem `tiradaEm`.
- **`finalizar` em status 7** não bloqueia de fato no backend, apenas avisa visualmente ("abra um chamado para retornar").
- **`exibiritem`/`novocomodo`**: tabela `#tabitem` é populada por AJAX (não há fallback sem JS).
- Editores misturados: revisão usa **TinyMCE**, aditivos usam **CKEditor** — padronizar no novo front.

## Destino (issues Linear)
- Fase **F07 — Vistorias (telas)** do projeto **wecorp-frontend** (Next.js). Telas a recriar: Listagem+filtros, Agendamento, Atualizar Status/Histórico, Revisão da Vistoria, Revisão por Cômodo (fotos+itens), Reordenar Fotos, Novo Cômodo, Alterar Observação, Aditivos (lista/novo/editar+fotos/PDF), CRUD Status. Backend correspondente (upload de fotos, máquina de estados, faturamento, e-mails) em wecorp-backend/services.
