# Legado (telas) — CRM (Esteira Digital)

> Plugin `Crms` = "Esteira Digital" do AlugueSeguro: dashboard + sub-módulos Tarefas, Leads, Visitas, Propostas, Proprietários, Contratos. Telas são `.ctp` Bootstrap 3 + CakePHP FormHelper, com listagens carregadas via AJAX (`index` estático + `index_paginator` parcial). Forte acoplamento a `grupo_id` (1–10) para escopo de empresa e colunas exibidas.

## Cobertura
Views `.ctp`:
- `Crms/index.ctp` (dashboard — cards), `Crms/add.ctp` (NA — é form de Vistoria, ver Gotchas)
- `Crmsleads/{index,index_paginator,add,edit}.ctp`
- `Crmspropostas/{index,index_paginator,add,edit,listagarantias,modalgarantia}.ctp`
- `Crmsvisitas/{index,index_paginator,add,edit}.ctp`
- `Crmstarefas/{index,index_paginator}.ctp`
- `Elements/menu_crms.ctp`
- **Follow-ups (cross-cutting)**: `app/View/Followups/{index,lista,novo,editar,ver}.ctp` + `FollowupsController`.
Controllers (só para entender comportamento de tela):
- `CrmsleadsController`, `CrmspropostasController`, `CrmsvisitasController`, `CrmstarefasController` (ações `index`, `index_paginator`, `add`, `edit`, `lead_por_email`).
- `FollowupsController` — `lista`, `novo`, `editar`, `ver`, `delete` (todas com assinatura `($model,$fk,$empresaparceira_id,$ajax,$seguradora_id)`).

NÃO coberto a fundo: `Crmsproprietarios/*`, `Crmscontratos/*`, `Vistorias/*` (sub-telas adjacentes; mesmos padrões).

### Follow-ups — modal de atendimento polimórfico (`Followups/*`)
Recurso **transversal** (não é só CRM): registra histórico de atendimento atrelado a qualquer entidade via par polimórfico `model` + `foreignkey`. Abertos em modal por `janelaFollowups(modulo, fk, empresaparceira_id, ajax, acao, seguradora_id)` (JS em `webroot/js/modules/general/ui.js`, linha ~576) → `openModalFromUrl('/followups/{acao}/{modulo}/{fk}/{empresaparceira_id}/{ajax}/{seguradora_id}', 'Follow-Ups', 100% x 450px)`. Usado em Seguro Fiança (`Segurofiancas/{index_paginator,addproposta}.ctp`), Incêndio, Capitalização, Condomínio, Análise etc.
- `lista.ctp` (ação `lista`): histórico de follow-ups da entidade (somente leitura, dentro do modal).
- `novo.ctp` / `editar.ctp` (form `Followup`): `empresaparceira_id` (select2 required, readonly), `model` (Tipo follow-up, readonly = entidade), `foreignkey` (Código, readonly = id), `seguradora_id` (select required), `status_id` "Situação" (select `$status` required), `texto` "Observações" (textarea TinyMCE `#editor` required), checkbox `enviar_para_email` "Enviar para e-mails da Imobiliária" + `emails` (cópia para e-mails adicionais).
- `ver.ctp`: visualização somente-leitura de um follow-up.
- **NOTA**: `app/View/Wbscotacao/*` (`index,lista,novo,editar,ver`) é uma **cópia quase idêntica** ("Follow-ups" / WbsCotacao) com `WbscotacaoController` — tratar como variante do mesmo padrão; confirmar qual está roteada no menu antes de migrar.

## Telas / fluxos

### Dashboard — `Crms/index.ctp` (rota `/crms/crms`)
- Apenas 4 cards informativos (Clientes/Leads, Visitas, Análises, Contratos) com **números HARDCODED** (`2/10`, `5/20`, `2/6`, `4/12`). NÃO é dado real — apenas mockup visual. **Não replicar os números fixos**; substituir por métricas reais.
- Coluna esquerda renderiza `element('menu_crms')`.

### Menu lateral — `Elements/menu_crms.ctp`
- Links: Dashboard, Tarefas, Leads, Leads/Visitas, Leads/Propostas, Leads/Proprietários, Contrato Digital (`/sign/sign`, target `sign`), Vistorias (`/vistorias/vistorias`), Imóveis (abre `janelaImoveis('Crm','index',...)`).
- Itens com `alert('Em breve!')`: Relatórios, Análise Cadastral, Garantias Locatícias, Escala Atendimentos → **placeholders não implementados, não replicar como features**.

### Leads — listagem `Crmsleads/index.ctp` + `index_paginator.ctp`
- Padrão de listagem AJAX: `index.ctp` tem um accordion "Opções de Filtros" e uma `<div id="paginator">` vazia; `leads.js`/`ui.js` dispara busca (botão `#btnPesquisar` classe `.pesquisar`) que injeta `index_paginator.ctp` via Paginator AJAX (`update:'#paginator'`, indicador `#busy-indicator`).
- Filtros: `status_id` (select `status_lead`), `empresa_parceira_id` (autocomplete em `#endereco_1`, só para grupos 1,2,3,4,5,9), `corretor_id`/`user` (select dependente carregado por AJAX de `users/usuarios_por_empresa/{empresa}/7` ao mudar `#endereco_1`), `cadastro_de`/`cadastro_ate` (datepicker dd/mm/yy), `pessoa_nome`/`pessoa_email`/`pessoa_celular` (LIKE).
- Tabela (`index_paginator`): id, nome, email, celular, cadastro (Dataformat::dateBR), status (`$status_lead[status_id]`), ação Editar. Paginação 10/página, ordem `Crmslead.id DESC`.

### Lead — cadastro `Crmsleads/add.ctp` (e `edit.ctp`)
- Form `Crmslead`. **Bug de UI: título exibe "Cadastrar nova proposta"** (copiado de Proposta). Botão "Cancelar e Voltar" → `crms/crmsleads/index`.
- Seção Dados do cliente: `pessoa_email` (blur dispara busca `crms/Crmsleads/lead_por_email/{email}` que auto-preenche tipo/nome/celular/documento se existir), `pessoa_tipo` (select PF/PJ), `pessoa_nome`, `pessoa_celular` (`data-mask=mobile`), `pessoa_documento` (máscara dinâmica via JS: PF=`999.999.999-99`, PJ=`99.999.999/9999-99`, label muda CPF/CNPJ/Documento).
- Seção Dados do imóvel: campo `Imovel.imovel_id` readonly + lupa que abre `janelaImoveis('Imovel','index','0','1','1')` (popup de imóveis); todos os campos de endereço (cep, endereco, numero, complemento, uf, cidade_id, bairro_id, cod_referencia, imovel_id_referencia) são **readonly** — preenchidos pela seleção do imóvel no popup. UF→cidade→bairro encadeados (`buscaCidades`/`buscaBairros`).
- Submit único "Processar".

### Visitas — listagem `Crmsvisitas/index.ctp` + `index_paginator.ctp`
- Mesmo padrão AJAX. Tabela: Cód, Empresa (só se `$mostra_colunas==1`), Solicitação (data+horário), Nome (com tel/WhatsApp `wa.me` + email mailto), Tipo visita, Imóvel (endereço, bairro, áreas m²), Consultor, Status (label colorido), Ações.
- Status visita (numérico, label CSS): 1=success, 2=primary, 3=danger, 4=danger. Texto via `$listStatus` (model `MetaStatusVisitas`).
- Ações: Editar; botão "Follow-ups" → `janelaFollowups('crmsvisita', id, empresa_id, '1','lista','')` (popup de followups). Botão de agenda existe em bloco condicionado a status 0/1 mas está **vazio** (`<?php } ?>` sem conteúdo) — **feature de agenda na listagem nunca foi implementada, não replicar**.

### Visita — cadastro `Crmsvisitas/add.ctp` (e `edit.ctp`)
- Form `Crmsvisita`. Título "Cadastrar nova visita".
- Dados da visita: `empresa_id` (select; ao mudar carrega consultores via `users/usuarios_por_empresa/{empresa}/7`), `corretor_id` (dependente), `data` (datepicker), `horario` (`data-mask=hora`), `tipo_visita` (Presencial/Virtual), `flag_confirmada_atendimento` (Não/Sim), `obs` (textarea).
- Dados do cliente: usa namespace `CrmLead.*` (email blur → `lead_por_email`, máscara doc dinâmica via `#CrmLeadPessoaTipo`).
- Dados do imóvel: idêntico ao Lead (lupa `janelaImoveis('Crmsvisita',...)`, campos readonly).

### Propostas — listagem `Crmspropostas/index.ctp` + `index_paginator.ctp`
- Filtros: `status_id` (`$status_proposta`), `empresa_parceira_id` (grupos 1,2,3,4,5,9), `corretor_id` (dependente), `codigo`, `cadastro_de`/`cadastro_ate`, `pessoa_nome`, `pessoa_email`, `Imovel.codigo_ref_imovel`.
- Tabela: Cód, Empresa, Data (created), Cliente (+tel/WhatsApp/email), Valor Proposta (`Dataformat::currencyMoeda`), Imóvel (aluguel + endereço), Consultor (+contatos), Status (`$status_proposta[status]`, label sempre `label-success` — **cor não reflete status real**), Ações: Editar, Follow-ups (`janelaFollowups('crmsproposta',...)`), Anexos (`arquivosAnexos('Crmsproposta', id, '0', empresa_id)`).

### Proposta — cadastro `Crmspropostas/add.ctp` (e `edit.ctp`)
- Form `Crmsproposta`. Tela mais rica do módulo.
- Dados gerais: `empresa_id` (required, ao mudar carrega consultores), `corretor_id` (dependente; ao mudar `#user` busca dados via `users/dados_usuarios/{id}` → preenche cpf/email/celular readonly).
- Dados do cliente (namespace `Crmslead.*`): email blur → `lead_por_email` (também seta `Crmslead.pessoa_id` = `lead_id`); tipo PF/PJ com máscara dinâmica; nome, celular(mobile), documento, `renda_informada` (MascaraMoeda), nacionalidade, sexo, data_nasc, estado_civil, `pessoa_pep` (Sim/Não), vínculo empregatício, ocupação.
- Dados do imóvel: idêntico (lupa `janelaImoveis('Imovel',...)`, readonly).
- Dados da proposta: tipo_locacao, inicio/fim contrato (date), meses_contrato_locacao (select), valores monetários `valor_imovel`/`valor_proposto`/`valor_condominio`/`valor_iptu_mensal`/`valor_luz`/`valor_agua` (todos `MascaraMoeda`), `garantia_id` (ao mudar `verificaGarantia(...)` mostra/esconde bloco `#dados_fiador` ou `#dados_capitalizacao`), `obs`, `status`.
- Bloco grande de campos `aceita_*` (caução/título cap/fiador/fiança/locatário solidário/renda mínima) está **comentado** — não está em uso.
- Submit "Processar".

### Garantias da proposta — `modalgarantia.ctp` + `listagarantias.ctp`
- `modalgarantia.ctp`: form `Segurofianca` dentro de popup; campos tipo_fianca_id, valores (aluguel/condomínio/IPTU/luz/água/gás), cobertura preferencial, status_id (editável só para grupos 1,2,3,4,9; senão depende de `$statusFicha`). Submit AJAX (`#gravar`) → POST `crms/crmspropostas/modalgarantia`, valida cobertura preferencial obrigatória, `alert('Gravado com sucesso!')`.
- `listagarantias.ctp`: linhas de garantias/fianças com status_id 1–7 (Rascunho/Enviada/Em análise/Com aprovação/Cancelado/Em emissão/Emitido) e link p/ `contratoseguros/segurofiancas/addproposta/{garantia_id}` (target _blank). **Status duplicado e divergente** dos status de proposta — ver Máquina de estados.

### Tarefas — `Crmstarefas/index.ctp` + `index_paginator.ctp`
- Filtros: `codigo` (number), `status_id` (`$listStatus = 0=Agendadas,1=Vencidas,2=Realizadas,3=Canceladas`), `solicitacao_dataini`/`solicitacao_datafim`, `empresa_parceira_id` (grupos 1,2,3,4,5,9). Mesmo padrão AJAX.

## Pontos de entrada (controller::ação que renderiza)
- `CrmsController::index` → `Crms/index.ctp` (dashboard mock).
- `Crmsleads/Crmspropostas/Crmsvisitas/CrmstarefasController::index` → `index.ctp` (filtros + container vazio).
- `...::index_paginator` → `index_paginator.ctp` com `layout='modal'`; só pagina/filtra quando `GET` traz `$_REQUEST['data']` (senão devolve vazio).
- `...::add` / `::edit($id)` → `add.ctp` / `edit.ctp`.
- `CrmsleadsController::lead_por_email($valor)` → endpoint AJAX (JSON ou `'false'`) consumido pelo blur de email em Lead/Visita/Proposta.

## Regras de negócio relevantes à UI
- Listagens são **carregadas sob demanda**: o `.ctp` index não traz dados; a busca AJAX renderiza `index_paginator.ctp` em `#paginator`. Paginação limite 10, ordem decrescente por id.
- Email do cliente é a chave de deduplicação: `lead_por_email` autopreenche e, em Proposta/Visita, captura `lead_id` num hidden para reaproveitar o Lead existente (em `Crmspropostas::add`, se `pessoa_id` vazio cria Lead novo, senão atualiza).
- Imóvel só entra por popup `janelaImoveis(...)`; campos de endereço readonly. Endereço encadeado UF→Cidade→Bairro (`buscaCidades`/`buscaBairros` em `modules/endereco.js`).
- Máscaras: documento dinâmica por tipo de pessoa (PF/PJ); celular `data-mask=mobile`; hora `data-mask=hora`; monetário via `MascaraMoeda(this,'.',',',event)`; datepicker pt-BR `dd/mm/yy`.
- Consultor depende da empresa: select recarregado por AJAX `users/usuarios_por_empresa/{empresa}/7` (o `7` = grupo de consultores).

## Máquina de estados / status (refletida na UI)
- **Lead**: `status_lead` via `Libmetadados->get('status_lead')` (Metadata por empresa) — valores configuráveis, não hardcoded. Em `add` o controller força `status=1` ao criar/atualizar Lead.
- **Visita**: numérico 1..4 (cores success/primary/danger/danger); textos de `MetaStatusVisitas`.
- **Proposta**: em `add.ctp` o select recebe `$status_proposta = array(1 => 'Em rascunho')` (apenas 1 opção no cadastro); na listagem `$status_proposta` vem do model `StatusProposta` (lista completa). Label de status na tabela é sempre verde (cosmético, não condicional).
- **Garantia/Fiança (listagarantias)**: 1 Rascunho, 2 Enviada, 3 Em análise seguradoras, 4 Com aprovação, 5 Cancelado, 6 Em emissão, 7 Emitido. É o ciclo do produto Fiança (Contratoseguros), exibido dentro da Proposta.

## Multi-tenant / white-label
- Escopo de dados é por **`grupo_id`** e **`empresa_id`** do usuário logado (`AuthComponent::user`). Em `index_paginator` (ex.: Leads) há `VerificaPermissao()`:
  - nível 2 → restringe a `Empresa.id` (grupos 6/7/8/10) ou `Empresa.id + empresa_master` (demais).
  - grupo 2 → vê empresas onde é responsável por seguros/análises/vistorias.
- Filtro/coluna "Imobiliária/Empresa" só aparece para grupos **1,2,3,4,5,9** (admin/master/operação). Grupos 6,7,8,10 (imobiliária/consultor) não escolhem empresa.
- Coluna "Empresa" na listagem de Visitas depende de `$mostra_colunas==1`.
- Edição de status na garantia liberada só para grupos 1,2,3,4,9.

## Gotchas / decisões kept-bug
- **`Crms/add.ctp` NÃO é tela de CRM**: é o formulário completo de **Vistoria** (participantes locação, preços, faturamento, agendamento de vistoriador). Provavelmente arquivo herdado/colado. **Não migrar como tela de CRM** — pertence ao módulo Vistorias.
- **Título errado** em `Crmsleads/add.ctp`: "Cadastrar nova proposta" (deveria ser "novo Lead"). Bug cosmético — corrigir na migração.
- Dashboard `Crms/index.ctp`: contadores **hardcoded** (mock). Não replicar valores; implementar agregação real.
- Botão de agenda na listagem de visitas: bloco condicional **vazio** (sem markup) → não há FullCalendar/agenda real no plugin Crms.
- **Não há FullCalendar no plugin Crms** (grep `fullCalendar`/`eventClick`/`events:` = 0 ocorrências). A "agenda Fullcalendar" mencionada no pedido não existe nestas views; o que há é agendamento simples por data+hora (datepicker + máscara hora) em Visitas e no form de Vistoria. Confirmar se a agenda visual existe em outro módulo antes de prometer paridade.
- Inconsistência de namespace do Lead entre telas: `Crmslead.*` (Lead/Proposta) vs `CrmLead.*` (Visita) vs model `CrmLead`. Os IDs JS (`#CrmsleadPessoa*`, `#CrmspropostaPessoa*`, `#CrmLeadPessoa*`) divergem — atenção ao reescrever validações/handlers.
- HTML mal-formado recorrente: `<h3 ...>...</h2>` (abre h3 fecha h2) e `<botton>` em vez de `<button>` nas tabelas dinâmicas. Não replicar.
- `index_paginator` usa `layout='modal'` e responde vazio sem `$_REQUEST['data']` — a primeira carga da listagem fica em branco até o usuário pesquisar.
- Persistência por `saveField` campo-a-campo em `Crmspropostas::add` (sem transação/validação de model unificada) — risco de gravação parcial. **Não replicar; usar save transacional.**

## Destino (issues Linear)
- Fase **F11 — CRM (telas)** no projeto **wecorp-frontend**. Telas a recriar em Next.js: Dashboard CRM (com métricas reais), Leads (lista+form), Visitas (lista+form), Propostas (lista+form+garantias), Tarefas (lista). Endpoints de apoio (lead_por_email, usuarios_por_empresa, dados_usuarios, janelaImoveis/popup imóveis, followups, anexos) dependem do backend (wecorp-backend) e devem ser linkados como milestones correlatos.
