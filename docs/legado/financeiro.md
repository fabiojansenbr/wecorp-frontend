# Legado (telas) — Financeiro

> Módulo financeiro do AlugueSeguro (plugin CakePHP `Financeiro`): Contas a Pagar, Contas a Receber, Baixa de pagamentos, Parcelas, Plano de Contas e Relatórios. UI Bootstrap 3 (tema "block-flat") com selects `select2`, máscaras jQuery e muitas chamadas AJAX para combos dependentes. Multi-tenant por `empresa_id` (holding/centro de custo). Contém muito código morto herdado de um sistema escolar ("academico/frequencias/matriculas") e relatórios desativados.

## Cobertura
Views (`app/Plugin/Financeiro/View/`):
- **Pagrecebes/**: `index.ctp` (Contas a Pagar), `add.ctp`, `edit.ctp`, `index_contasreceber.ctp`, `add_contasreceber.ctp`, `edit_contasreceber.ctp`, `baixarcontaspagar.ctp`, `confirmarbaixarcontaspagar.ctp` (morta), `view.ctp`
- **Parcelas/**: `index.ctp`, `add.ctp`, `edit.ctp`, `view.ctp`
- **Planodecontas/**: `index.ctp`, `add.ctp`, `edit.ctp`, `view.ctp`
- **Relatorios/** (financeiros): `imprelcontasapagar.ctp`, `imprelcontasareceber.ctp`, `imprelcontasrecebidas.ctp`, `imprelplanodecontas.ctp`, `imprelfechamentomensal.ctp`, `imprelrepassemesprodutor*.ctp` (demais `*aluno*`/`*turma*`/`*diario*` são do legado escolar — fora de escopo)

Controllers (lidos só p/ entender o comportamento da tela):
- `Controller/PagrecebesController.php` (1857 ln): `index`, `add`, `edit`, `index_contasreceber`, `add_contasreceber`, `edit_contasreceber`, `delete`, `baixarcontaspagar`, `confirmarbaixacontaspagar`, `getcontasapagar`, `autorizarcontasapagar`, `validaData`, combos `buscar*`
- `Controller/PlanodecontasController.php` (177 ln): `index`, `add`, `edit`, `view`
- `Controller/ParcelasController.php` (131 ln): CRUD scaffold
- `Controller/RelatoriosController.php` (1675 ln): `imprelcontasapagar`, `imprelcontasareceber`, `imprelplanodecontas`, `imprelfechamentomensal`, `imprelrepassemesprodutor`

**Financeiro de análise (app-level, fora do plugin)** — telas de cobrança dos serviços de análise cadastral:
- `app/View/Financeiros/` (`index.ctp`, `novo_item_financeiro.ctp`, `incluir_item_financeiro.ctp`, `edita_item_financeiro.ctp`, `add.ctp`, `edit.ctp`, `view.ctp`) + `app/Controller/FinanceirosController.php` (`carrega_servicos`, `novo_item_financeiro($empresa_parceira_id)`, `incluir_item_financeiro($empresa_parceira_id,$ficha_analise_id,$ficha_analisepessoa_id)`, `edita_item_financeiro($id)`).
- `app/View/Relatorios/` (`index.ctp`, `financeiro.ctp`, `financeiro_lista.ctp`) + `app/Controller/RelatoriosController.php` (app-level, ≠ plugin) — `index`, `financeiro`, `busca_analises`, `relatorio_financeiro`, `relatorio_solicitacoes`, `realizar_pagamento`, `realizar_desconto`.

**Repasses de comissão de seguros (plugin `Seguros`, ≠ plugin `Financeiro`)**:
- `app/Plugin/Seguros/View/Seguros/` (`index.ctp`, `add.ctp`, `liberartransmissao.ctp`, `repasses.ctp`) e `Segurosrepasses/` (`index.ctp`, `add.ctp`, `edit.ctp`, `gerencia.ctp`, `listarepasses` via `modal_repasses.ctp`, `edita_repasse.ctp`, `baixarrepasses.ctp`, `confirmarbaixarepasse.ctp`, `fechamentorepasse.ctp`, `importararquivos.ctp`).
- `app/Plugin/Seguros/Controller/SegurosrepassesController.php` — `index`, `add`, `edit`, `gerencia($empresa)`, `listarepasses`, `grava_repasse`, `edita_repasse`, `altera_repasse`, `carregapercentual($produto,$empresa)`, baixas/fechamento.

## Telas / fluxos

### 1. Contas a Pagar — listagem (`Pagrecebes/index.ctp` → `PagrecebesController::index`)
- **Painel de totalizadores** (topo, tabela): Vencidas, A vencer (7/14/21 dias), Pagas no mês, Receitas do mês, Saldo. Valores formatados pt-BR. Cada bloco é um link `onclick="exportar(N,...)"` que dispara o relatório de impressão — **mas o relatório está desativado (ver Gotchas)**.
- **Filtro** (form POST p/ a própria `index`, `target=_new`): `empresa_id` (Empresas, select2 required), `planodeconta_id` (Centro de custo, select2 required), `dtvencimentoi`/`dtvencimentof` (Período Venc., class `date`), `dtpagamentoi`/`dtpagamentof` (Período Pagto), `clientefornecedor_id` (Fornecedor, select2), `situacao` (Status: `9`=Todas, `1`=A vencer, `2`=Autorizadas, `3`=Realizadas, `4`=Vencida). Botão "pesquisar".
- Campos `hidden` (`rel_empresaid`, `rel_datainicio`, `rel_datafim`, `rel_fornecedor`, `rel_situacao`) preenchidos via JS `exportar()` para repassar o filtro ao relatório.
- **Tabela**: checkbox de seleção (omitido quando `status==3` Realizado), coluna **Empresa** só aparece se `isHolding=="1"`, Centro de custo, Nº Doc., Parcela (`parcela/numparcelas`), Data Vencto, Data Pagto, Valor (R$), Status (texto colorido: Vencida=vermelho, A vencer=verde, Autorizado/Realizado=azul, Cancelado), Fornecedor.
- **Ações por linha**: ver (sempre); editar (só se `status!=3`); excluir via `postLink` com confirm (**só se `status==1` "A vencer"**).
- **Ações em lote**: "Autorizar conta(s)" → `autorizarContasapagar()` coleta checkboxes e navega para `financeiro/pagrecebes/autorizarcontasapagar/<ids>`; "Imprimir conta(s)" → `exportar(9)`. Existe também "Novo Lançamento" → `add`.
- Paginação CakePHP (`Paginator`).

### 2. Lançamento Contas a Pagar (`Pagrecebes/add.ctp` → `add`)
- Form `Pagrecebe` com Parsley desativado (`novalidate`), `autocomplete=off`.
- Campos: `empresa_id` "Empresa Centro Custo" (select2, `OnChange=pesquisaPlanodecontas`); `planodeconta_id` "Centro de Custo da Despesa/Compra" (select2 required, `OnChange=pesquisaCategoriaPlanodecontas`); `categoriaplanodeconta_id` "Categoria" (select2 required); `numdocumento` (text required).
- **Fornecedor**: `stfornecedor` "Fornecedor Existe?" (Sim/Não, `onchange=trocartipo`). Sim → mostra `#fexiste` (`clientefornecedor_id` select2). Não → mostra `#fnaoexiste` com cadastro inline de `Empresa`: `tipo_pessoa` (PJ/PF, `onChange=trocarMascara`), `documento` (máscara cnpj/cpf), `nome` (fantasia), `razao_social`, contatos (`contato_nome/email/telefone/celular`), `telefone_principal/outros`, `email_principal/secundario` (tipo email).
- Competência/valores: `mes` "Competência Mês" (select `$listMeses`), `ano` (text, default `date('Y')`), `dtvencimento` (class `date`), `vlrfatura` "Valor Parcela" (`onkeypress=MascaraMoeda`), `numparcelas` (number, min 1, default `01`), `observacao` (textarea).
- **Máscaras**: documento PJ → `99.999.999/9999-99`, PF → `999.999.999-99`; telefone/celular `data-mask`; moeda via `MascaraMoeda`.
- **Combos dependentes (AJAX GET)**: `pesquisaPlanodecontas` → `financeiro/pagrecebes/buscarplanodecontas`; `pesquisaCategoriaPlanodecontas` → `buscarcategoriaplanodecontas`; autocomplete jQuery-UI em `#buscaConta` → `financeiro/planodecontas/buscarContaContabil`.
- Botão "Processar" (submit).

### 3. Editar Conta a Pagar (`Pagrecebes/edit.ctp` → `edit($id,$idparcela)`)
- Mesmos campos do add (sem cadastro inline de fornecedor); valores pré-carregados via variáveis do controller (`$empresaid`, `$planodecontaid`, `$categoriaid`, `$numerodocumento`, `$clientefornecedorid`, `$mes`, `$ano`, `$valorparcela`). `parcela_id`/`pagrecebe_id` em hidden.
- Ação "Excluir" via `postLink` → `delete/<pagrecebeid>/<parcelaid>`.
- **Bug**: `numparcelas` tem `value=>'01'` fixo (não reflete o valor salvo).

### 4. Contas a Receber — listagem (`Pagrecebes/index_contasreceber.ctp` → `index_contasreceber`)
- Estrutura igual à de pagar. Colunas: Empresa (se holding), Tipo de receita (`Planodeconta.descricao`), Cliente (`Seguradora.nome`), Data de recebimento, Valor. Ações: ver + editar (`edit_contasreceber`). **Sem excluir e sem checkbox** (comentados).

### 5. Lançamento Contas a Receber (`Pagrecebes/add_contasreceber.ctp` → `add_contasreceber`)
- Campos: `empresa_id` "Empresa (origem)" (default = `AuthComponent::user('empresa_id')`, `OnChange=pesquisasubtiposoperacao + pesquisaBancosempresa`); `planodeconta_id` "Tipo de entrada" (receitas, `OnChange=pesquisaClientefornecedor`); `banco_id` "Banco origem" (oculto por padrão); `seguradora_id` "Cliente/Seguradora"; `clientefornecedor_id` "Cliente" (oculto); `bancodestino_id` "Banco destino"; `dtvencimento` "Data recebimento"; `vlrfatura` "Valor" (`MascaraMoeda`).
- **Comportamento por subtipo de operação** (`pesquisaClientefornecedor(id)` faz switch sobre o id do tipo de entrada):
  - `1` Seguro → mostra `#cboseguradora`, esconde banco origem e cliente; carrega seguradoras.
  - `2` Empréstimo / `3` Aporte → mostra `#cboclientefornecedor` (cliente), esconde seguradora.
  - `6` Transferência entre contas → mostra `#bancoorigem`, esconde seguradora e cliente.
  - default → mostra cliente.
- Combos AJAX: `buscarsubtiposoperacao`, `buscarplanodecontasreceitas`, `buscarclientefornecedor/<subtipo>`, `buscarbancosempresa` (popula banco origem e destino).

### 6. Baixa de pagamento (`Pagrecebes/baixarcontaspagar.ctp` → `baixarcontaspagar` / `confirmarbaixacontaspagar`)
- **Filtro**: `empresa_id` (default user), `dtvencimentoi/f` (Período de Vencimento), `fornecedor_text` + hidden `fornecedor` + botão lupa → `buscarContasapagar()`.
- `buscarContasapagar()` faz AJAX GET `financeiro/pagrecebes/getcontasapagar` (que filtra **`Parcela.status=2` Autorizado**) e monta a tabela client-side com checkboxes (Centro de Custo, Fornecedor, Nº Doc, Parcela, Vencimento, Valor).
- "Confirmar baixa(s)" → `efetuarBaixaLote()` → navega `financeiro/pagrecebes/confirmarbaixacontaspagar/<ids>`.
- **`confirmarbaixacontaspagar`**: na confirmação grava em cada `Parcela` os campos `vlrdesconto`, `vlrmulta`, `vlrjuros`, `vlrpago`, `dtpagamento` e seta `Parcela.status=3` (Realizado); seta `Pagrecebe.status=4` no pai. Transacional.

### 7. Parcelas (CRUD scaffold) (`Parcelas/*` → `ParcelasController`)
- `index` lista TODAS as colunas do banco (id, pagrecebe_id, parcela, vlrparcela, dtvencimento, vlrdesconto, vlrmulta, vlrjuros, vlrpago, dtpagamento, created, modified, empresa_id, user_id, movimentobancarios_id) com links para `pagrecebes/view`, `empresas/view`, `users/view`. `edit.ctp`/`add.ctp` são inputs crus (sem máscara/validação). É **tela técnica/admin**, não fluxo de usuário final.

### 8. Plano de Contas (`Planodecontas/*` → `PlanodecontasController`)
- `index`: colunas id, Código contábil, Descrição, Tipo de operação (`tipooperacao=='R'`→Receita, senão Despesa), Empresa, Tipo de conta (`A`→Analítica, `S`→Sintética). Paginado `limit=5`. **Excluir comentado** (não há delete na UI).
- `add`/`edit`: `descricao` (text required, max 50); `tipooperacao` (select do model `Tipooperacao`, `OnChange=pesquisaSubtipooperacao` → `buscarsubtipooperacao`); `subtipooperacao_id`; `empresa_id`; `contaprincipal_id` "Plano de Conta principal (Nível superior)" (select2 required); `tipoconta` (A/S); `codigocontabil` (text required, max 50). `add()` só faz `save()` (validação apenas pelos `required` de HTML).

### 9. Relatórios financeiros (`Relatorios/imprel*` → `RelatoriosController`)
- Views de impressão (PDF/tela). Acionados pelos totalizadores e pelo botão "Imprimir conta(s)" via `exportar(tipo,inicio,fim)` postando para `financeiro/relatorios/imprelcontasapagar/<tipo>/<ini>/<fim>`.
- Mapeamento `tipo` em `imprelcontasapagar`: `1` a vencer, `2` a vencer 7 dias, `3` pagas no mês, `4` vencidas, `5` a vencer 14 dias, `6` a vencer 21 dias, `9` conforme filtro. Monta SQL cru `WHERE par.status = N ...`.
- **`imprelcontasapagar` está DESATIVADO**: primeira linha do método é `exit('imprelcontasapagar...');` — todo o relatório (e os links dos totalizadores) só imprimem essa string. NÃO replicar esse estado.

### 10. Financeiro de análise — itens de cobrança (`app/View/Financeiros/*` → `FinanceirosController`)
- **Não pertence ao plugin Financeiro**: é a cobrança dos serviços de análise cadastral, ligada à ficha/proponente. `beforeFilter` faz gate por permissão (`render('/Dashboard/proibido')`).
- `novo_item_financeiro.ctp` / `incluir_item_financeiro.ctp` / `edita_item_financeiro.ctp` — form do item financeiro (`Form` model `Financeiro`): `valor` (MascaraMoeda required), `data_vencimento` (class `date` required), `forma_pagamento_padrao` (select `$formaPagto` required), `tipo_servicoid` (select `$tipoServico` required, `empty=false`), `status_pagamento` (select `$statusPagamento` required), `fatura_id` (text, `disabled`). `incluir_item_financeiro` recebe `($empresa_parceira_id,$ficha_analise_id,$ficha_analisepessoa_id)` — vincula o item à análise.
- `index.ctp` — listagem (`Paginator->sort('status_pagamento')`, coluna Status Pagamento). Mapas `statusPagamento`/`statusFatura`/`tipoServico`/`formaPagto` vêm do controller. Cross-link com `analise-cadastral.md` (botão "Contratar CAP"/cobrança a partir da ficha).

### 11. Relatórios (app-level) (`app/View/Relatorios/*` → `app/Controller/RelatoriosController`)
- Distinto do `Financeiro/Relatorios`. `index.ctp` (menu de relatórios, breadcrumb Painel/Relatorios) e `financeiro.ctp`/`financeiro_lista.ctp` (Relatório Financeiro de análises).
- Ações: `financeiro()` + `busca_analises()` (filtra análises), `relatorio_financeiro($id)`, `relatorio_solicitacoes($id)`, `realizar_pagamento($id)`, `realizar_desconto($id)` — operações financeiras sobre solicitações de análise.

### 12. Repasses de comissão de seguros (`Plugin/Seguros/View/Segurosrepasses/*` → `SegurosrepassesController`)
- Gestão do repasse de comissão de seguros às empresas parceiras. `index.ctp`: filtro Empresa (autocomplete `#empresa_parceira_id`), `status_id` (select `$status_repasses`), `data_de`/`data_ate` (Período, datepicker); tabela id, Nome Segurado, Nº Apólice, Status, ação Editar; botão "Novo Repasse" → `add`.
- `gerencia.ctp` (ação `gerencia($empresa_parceira_id)`): tela "Lançamentos repasses". Botão "Novo repasse" abre painel `#add_repasse` (form `Segurosrepasse` → `grava_repasse/{empresa}`): `produto_id` (select `$listprodutos`, `onchange=verificaPercentualEmpresaProduto` → `carregapercentual/{produto}/{empresa}`), `qtd` (vidas), `qtd_parcelas` (>4 gera N linhas), `data_recebimento`, `nome_segurado`, `cpf_segurado`, `numero_apolice`. Lista de lançamentos via `listarepasses` (`modal_repasses.ctp`); edição inline `edita_repasse`/`altera_repasse`.
- `baixarrepasses.ctp` / `confirmarbaixarepasse.ctp` (baixa de repasses), `fechamentorepasse.ctp` (fechamento do período), `importararquivos.ctp` (importação de retorno). `Seguros/repasses.ctp` é a listagem-espelho no controller `Seguros`.

## Pontos de entrada (controller::ação que renderiza)
- `Financeiro/Pagrecebes::index` → Contas a Pagar (lista + dashboard)
- `Financeiro/Pagrecebes::add` / `::edit` → lançar/editar conta a pagar
- `Financeiro/Pagrecebes::index_contasreceber` → Contas a Receber
- `Financeiro/Pagrecebes::add_contasreceber` / `::edit_contasreceber` → lançar/editar conta a receber
- `Financeiro/Pagrecebes::baixarcontaspagar` → buscar contas autorizadas p/ baixa
- `Financeiro/Pagrecebes::confirmarbaixacontaspagar` → confirma baixa (status 3)
- `Financeiro/Pagrecebes::autorizarcontasapagar/<ids>` → status 2
- `Financeiro/Pagrecebes::getcontasapagar` (JSON), `::buscar*` (combos JSON)
- `Financeiro/Planodecontas::index|add|edit|view`
- `Financeiro/Parcelas::index|add|edit|view` (admin)
- `Financeiro/Relatorios::imprel*` (impressão)

## Regras de negócio relevantes à UI
- **Validação de lançamento (`validaData`)**: obrigatórios `planodeconta_id`, `numdocumento`, `mes`, `ano`; `dtvencimento` válida; `vlrfatura >= 1`; `numparcelas >= 1`. Mensagens com typos legados ("Cetro de Custo", "Data ... inváda", "Valor o documento"). Validação de receita (`validaDataReceita`) está toda comentada (não valida nada).
- **Geração de parcelas (`add`)**: `vlrfatura` salvo no `Pagrecebe` = valor informado × `numparcelas` (total); cada `Parcela` recebe o valor unitário. Vencimentos são mensais a partir de `dtvencimento`. Tratamento de fim de mês: fevereiro fixado em dia 28 (**ignora ano bissexto**); dia 31 em meses de 30 dias (4/6/9/11) vira 30.
- **Cadastro inline de fornecedor**: se `clientefornecedor_id` vazio no add, cria `Empresa` com `isfornecedor=1` e `empresa_master = empresa_id` do usuário, antes de gravar o financeiro (mesma transação).
- **Status inicial**: `dtvencimento < hoje` → conta nasce `4` (Vencida); senão `1` (A vencer).
- **Atualização lazy de vencidas**: ao abrir `index`, roda `UPDATE parcelas SET status=4 WHERE status=1 AND dtpagamento IS NULL AND dtvencimento <= ontem`.
- **Tipo de operação**: nas queries de contas a pagar usa `Planodeconta.tipooperacao = '2'` (despesa, id numérico do model `Tipooperacao`), enquanto a listagem de plano de contas compara com `'R'` (Receita). Representação inconsistente — atenção na migração.

## Máquina de estados / status (refletida na UI)
`Parcela.status`:
- `1` = **A vencer** (verde) — único estado que permite excluir.
- `2` = **Autorizado** (azul) — único elegível para baixa (`getcontasapagar` filtra `status=2`). Setado por `autorizarcontasapagar`.
- `3` = **Realizado/Pago** (azul) — setado em `confirmarbaixacontaspagar`; some o checkbox e o botão editar na lista.
- `4` = **Vencida** (vermelho) — automático no `index` ou no lançamento com data passada.
- `9` = **Cancelado** (sem cor) — exibido na lista, mas o fluxo de cancelar está comentado na UI.

Fluxo: lançar (1 ou 4) → Autorizar (2) → Baixar/Confirmar (3). No `Pagrecebe` (pai), `confirmarbaixacontaspagar` seta `status=4` (valor reaproveitado, sem semântica de "vencida" no nível pai).

## Multi-tenant / white-label
- Empresa corrente = `AuthComponent::user('empresa_id')`. Usado como default em filtros, `empresa_id` de parcelas e como `empresa_master` de novos fornecedores.
- `Holdingempresa::getListEmpresas()` / `Empresa->find('list')` populam o combo de empresas; fornecedores são filtrados por `empresa_master IN (array_keys($empresas))` (escopo da holding).
- Flag `isHolding` (setada como `1`) controla se a coluna **Empresa** aparece nas listagens — em telas de baixa está hardcoded `true`.
- Sem white-label de marca nessas telas (layout fixo "block-flat"/Bootstrap 3); a segmentação é puramente por empresa/holding.

## Gotchas / decisões kept-bug
- **NÃO replicar**: `imprelcontasapagar` começa com `exit('imprelcontasapagar...')` → "Imprimir conta(s)" e todos os links dos totalizadores do dashboard estão quebrados. Há ainda variável indefinida `$vencto7dihoraas` (typo) no `tipo==2`.
- **NÃO replicar**: `$listMeses` em `add`/`edit`/`confirmarbaixacontaspagar` só vai de **Janeiro a Setembro** (faltam Outubro/Novembro/Dezembro) — competência de 10/11/12 fica inselecionável.
- **NÃO replicar**: flash de sucesso ao autorizar diz "Registro(s) excluído(s) com sucesso!" (mensagem errada).
- **NÃO replicar**: `numparcelas` com `value='01'` fixo no `edit` (perde o valor real).
- **NÃO replicar**: filtros em `index` montam SQL por concatenação de string com dados do request (`$criterio .= ... . $this->request->data[...]`) → **SQL injection**. Migrar com query parametrizada/ORM.
- **NÃO replicar**: fevereiro sempre fixado em 28 na geração de parcelas (ignora bissexto).
- Arquivo `confirmarbaixarcontaspagar.ctp` (com "r" extra) é **view morta** cheia de copy-paste do legado escolar (`buscarAluno`, `academico/frequencias/deletarlote`, `matriculas`); a rota ativa é `confirmarbaixacontaspagar` (sem o segundo "r"). Vários scripts em outras views também carregam funções escolares inúteis e jQuery duplicado/antigo inline.
- `Parcelas/*` é scaffold cru (expõe colunas internas) — provavelmente fora do menu; tratar como ferramenta administrativa, não fluxo de usuário.
- Plano de Contas não permite excluir (postLink comentado) — manter regra ou rever na migração.

## Destino (issues Linear)
- Projeto **wecorp-frontend**, fase **F10 — Financeiro (telas)**. Telas a recriar em Next.js: Contas a Pagar (lista+dashboard), Lançamento/Edição de conta a pagar, Contas a Receber (lista/lançamento por subtipo), Baixa de pagamentos (busca+confirmação), Plano de Contas (lista/cadastro), Relatórios financeiros. Backend equivalente (status, geração de parcelas, baixa) em wecorp-backend. Issues HUB-116..HUB-382 (ver `reference_linear.md`/`project_migration.md` para o mapeamento de milestone exato — não confirmado neste estudo).
