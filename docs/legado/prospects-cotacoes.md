# Legado (telas) — Prospects/Cotações

> Telas administrativas de listagem/filtro de prospects gerados via integração (plataforma O2 x integradores/imobiliárias). Duas listagens paginadas com filtros (accordion), coluna de "situação" derivada por joins e exportação opcional para Excel (.xls). Não há tela de cadastro/edição própria: as ações de detalhe abrem o "painel" da imobiliária/cliente em nova aba. A view `view.ctp` está vazia (stub).

## Cobertura
- Views .ctp:
  - `app/Plugin/Prospectscotacoes/View/Prospectscotacoes/index.ctp` (listagem principal)
  - `app/Plugin/Prospectscotacoes/View/Prospectscotacoes/view.ctp` (STUB vazio — só comentário de template; nada renderizado)
  - `app/Plugin/Prospectscotacoes/View/Seguroscotacoes/index.ctp` (listagem "Prospects Cotações Seguros")
- Controllers (lidos só para entender comportamento da tela):
  - `app/Plugin/Prospectscotacoes/Controller/ProspectscotacoesController.php` (`index`, `gera_planilhaSPA`, `gera_planilha_spa`, `dadosEmpresa`, `beforeFilter`)
  - `app/Plugin/Prospectscotacoes/Controller/SeguroscotacoesController.php` (`index`, `beforeFilter`)
- Model: `app/Plugin/Prospectscotacoes/Model/Prospectscotacoes.php` (tabela `wbs_prospects_cotacoes`)
- Apoio: `app/Controller/AppController.php::VerificaPermissao` (controle de acesso por grupo).

## Telas / fluxos

### Tela 1 — `Prospectscotacoes/index.ctp` ("Prospects Cotações")
Breadcrumb: Painel > Prospects Cotações. Layout: bloco "block-flat", accordion de filtros (colapsado por padrão — `class="panel-collapse collapsed"`), tabela paginada (5 por página), e bloco "Funções Extras" só para grupo 1.

Formulário de filtros (helper `Search->create('Prospectscotacoes')`), campos:
- `empresa_parceira` — SELECT (`select2 lista_imobiliarias required`, com `empty`) — opções = `$listaImobiliarias`. **Visível só para grupos 1,2,3,4,9** (admin/back-office); demais grupos não veem.
- `pessoa_nome` — TEXT, "Nome prospect", `autocomplete=off`. Filtro: `LIKE %valor%`.
- `pessoa_documento` — TEXT, "Documento (CPF/CNPJ)". Filtro: igualdade exata (`=`). Sem máscara aplicada no input.
- `data_de` / `data_ate` — TEXT com `class="form-control date"` (datepicker JS por classe `.date`), "Data de"/"Data até".
- `Prospectscotacoes.situacao_prospect` — SELECT "Situação", opções `$filtroSituacaoProspects`, com `empty`.
- `Prospectscotacoes.produto` — SELECT "Produto", opções `$filtroProdutos` (lista de `wbs_produtos_seguros` id=>descricao), com `empty`.
- `Prospectscotacoes.situacao_qualificacao` — SELECT "Qualificação", opções `$filtroSituacaoQualificacaoProspects`, **sem empty, default `'1'` (Somente Qualificados)**.
- `Prospectscotacoes.exportar_excel` — CHECKBOX "Gerar Excel", value `'sim'`.
- Botão `Search->end('OK')` (submete o form de busca, método POST/PUT).

Tabela (colunas, com `Paginator->sort`):
- `id`
- Empresa (20% largura): ícone `glyphicon-tasks` que abre em nova aba o painel adm da imobiliária (`/painel/painel/indexadm/{integrador_id}/{empresa_id}/{token_empresa}`), seguido do nome da empresa em negrito.
- Integrador (`Integradores.nome`).
- Nome prospect (`pessoa_nome`).
- Documento — `Dataformat::formatarCpf(pessoa_documento)` (`nowrap`).
- Produto — se `Prospectsprodutos.produto_seguradora_id` vazio: mostra ícone vermelho `glyphicon-remove-circle` com title "Prospect ainda não ofertado a nenhum produto!!!"; senão mostra `Produtosseguros.descricao`.
- Data Cadastro — `Dataformat::datetimeBr(created_at)`.
- Ações: ícone `glyphicon-folder-close` que abre em nova aba o painel do cliente baseado no produto SPA (`/painel/painel/index/{integrador_id}/{empresa_id}/{token_empresa}/{pessoa_codigo_ref}/1/`). Há links comentados (imprimir_fatura, edita_item_fatura, postLink delete) — **NÃO existem essas ações; não replicar.**

Paginação: `Paginator->counter` em PT-BR + `pagination pagination-sm` (prev "← Anterior" / numbers / next "Próximo →"), só renderizada se `pageCount > 1`.

Bloco "Funções Extras" (**somente grupo 1**): dois botões que abrem popups externos do ZipCode (consulta CPF e sincronização) em `o2wbs.net/apiws/public/zipcode/...` com token de integrador hardcoded (`5602115f992be3f6dab30b21303977a7`). Externo/legado — avaliar se migra.

### Tela 2 — `Seguroscotacoes/index.ctp` ("Prospects Cotações Seguros")
Breadcrumb: Painel > Prospects Cotações Seguros. Estrutura igual (accordion + tabela). Accordion aqui usa `panel-collapse collapse` (não `collapsed`).

Filtros:
- `pessoa_parceira` — TEXT "Parceira", **visível só para grupos 1,2,3,4,9**. (Bug: o controller filtra por `$pessoa_parceira` que nunca é setado a partir desse campo — filtro inoperante, ver Gotchas.)
- `pessoa_nome` — TEXT, filtro `LIKE`.
- `pessoa_documento` — TEXT, filtro `=` exato.
- `data_de` / `data_ate` — TEXT `.date`.
- `exportar_excel` — CHECKBOX "Gerar Excel" (sem value definido; nesta tela o controller não trata exportação — checkbox inerte).
- Botão `OK`.

Tabela (colunas, dados crus do `Prospectscotacoes`, recursive=2):
- `id`
- `imobiliaria_token` (em negrito) — coluna rotulada "empresa_parceira_id" no sort.
- `pessoa_nome`
- `pessoa_documento` (sem formatação de CPF aqui)
- `imobiliaria_produto_token` (coluna "produto")
- `pessoa_data_nasc` — `Dataformat::dateBr`
- `zc_data_nascimento` ("Dt nascimento ZCode")
- `zc_data_nascimento` de novo (coluna "Dt integracao" — **bug: repete o mesmo campo**)
- `zipcode_pessoa_id` passado por `Dataformat::dateBr` (coluna "Zipcode" — **bug: aplica formatação de data num id**)
- `created` — `Dataformat::dateBr`
- Ações: links `imprimir_fatura` e `edita_item_fatura` (ações inexistentes no controller — **quebram se clicados; não replicar**). Delete comentado.

## Pontos de entrada (controller::ação que renderiza)
- `ProspectscotacoesController::index` → `View/Prospectscotacoes/index.ctp`. Variáveis: `prospects` (paginado), `listaImobiliarias`, `filtroProdutos`, `filtroSituacaoProspects`, `filtroSituacaoQualificacaoProspects`. URL típica: `/prospectscotacoes/prospectscotacoes`.
- `SeguroscotacoesController::index` → `View/Seguroscotacoes/index.ctp`. Variável: `prospects`. URL: `/prospectscotacoes/seguroscotacoes`.
- `ProspectscotacoesController::gera_planilhaSPA` — chamada inline a partir de `index` quando `exportar_excel=='sim'`: monta `.xls` (Excel5/PHPExcel) e dá `exit()`. (Há também `gera_planilha_spa` quase idêntica, aparentemente não roteada pela UI atual.)
- `view.ctp` existe mas é stub vazio — não há ação `view` funcional renderizando conteúdo.

## Regras de negócio relevantes à UI

### Filtro "Situação do prospect" (`$filtroSituacaoProspects`)
Opções e efeito no SQL (joins com `wbs_prospects_produtos`, `wbs_prospects_pagamentos`):
- `1` Todos — sem condição extra.
- `2` Não ofertados — `Prospectsprodutos.prospect_cotacao_id IS NULL`.
- `3` Ofertados sem pagamentos confirmados — `Prospectspagamentos.produto_seguradora_id IS NULL AND Prospectsprodutos.produto_seguradora_id IS NOT NULL`.
- `4` Ofertados com pagamento confirmado — ambos `produto_seguradora_id IS NOT NULL`.

### Filtro "Qualificação" (`$filtroSituacaoQualificacaoProspects`, default 1)
Depende do produto selecionado ter `flag_verifica_vida = true` (consulta em `wbs_produtos_seguros`). Usa join com `wbs_zipcode_pessoas` (alias `Zipcode`):
- `1` Somente Qualificados (e produto exige vida) — `Zipcode.zc_obito=0 AND zc_idade>18 AND zc_idade<65`.
- `2` Não qualificados — `OR(zc_obito=1, zc_idade<=18, zc_idade>=65)`.
- `3` Indiferente — sem condição (no-op).
Observação de paridade: a "qualificação" só aplica corte de idade/óbito quando o produto exige verificação de vida.

### Filtro "Produto"
`Prospectsprodutos.id = {valor}` (filtra pelo id da oferta `wbs_prospects_produtos`, não pelo id do produto-seguro selecionado no dropdown — ver Gotchas).

### Listagem (joins do `index` principal)
`Prospectscotacoes` (LEFT) join: `empresas` (por `token_empresa = imobiliaria_token`), `wbs_zipcode_pessoas`, `wbs_empresas_produtos`, `wbs_integradores` (por `integrador_id`), `wbs_prospects_produtos`, `wbs_produtos_seguradoras`, `wbs_produtos_seguros`, `wbs_prospects_pagamentos`. Ordenação `id DESC`, limite 5. `recursive=-1`.

### Persistência de filtros
Condições de busca são gravadas em sessão (`Session->write('conditions_post', $conditions)`) no POST/PUT e relidas no GET. (No GET há condicional confusa: só relê se `!empty($conditions)`, que está vazio nesse ramo — efetivamente a persistência via GET quase nunca dispara; ver Gotchas.)

### Datas
- Datepicker via classe CSS `.date` (JS do layout). Datas exibidas com `Dataformat::dateBr` / `datetimeBr`; conversão para SQL via `Dataformat::dateToSql`.
- Documento exibido com `Dataformat::formatarCpf` na tela principal (não na de seguros).

## Máquina de estados / status (refletida na UI)
Não há campo de status persistido editável pela UI. O "estado" do prospect é DERIVADO em tempo de consulta pela presença de registros relacionados:
- Não ofertado: sem `wbs_prospects_produtos` (mostra ícone vermelho).
- Ofertado: tem `produto_seguradora_id` em prospects_produtos.
- Pago/confirmado: tem `produto_seguradora_id` em `wbs_prospects_pagamentos`.
Qualificação (qualificado/não) é derivada de dados do ZipCode (óbito + faixa etária 18–65) e do flag do produto.

## Multi-tenant / white-label
- Acesso controlado por `AppController::VerificaPermissao` no `beforeFilter` (consulta `permissiongroups`/`modulecontrolleractions` por `grupo_id`); sem permissão renderiza `/Dashboard/proibido`.
- Visibilidade de campos e dados por `AuthComponent::user('grupo_id')`:
  - Grupos **1,2,3,4,9** (back-office/admin): veem o filtro de empresa/parceira; `listaImobiliarias = Empresa->getParceiras()` (todas as parceiras).
  - Demais grupos: `listaImobiliarias` restrita à própria empresa via `Empresa->getEmpresaFormaPagto()` (apenas a empresa do usuário).
  - Grupo **2** (admin da parceira): adiciona condição multi-tenant — só prospects de empresas onde a parceira logada é `empresa_resp_seguros` OU `empresa_resp_analises` OU `empresa_resp_vistorias` (`AuthComponent::user('empresa_id')`).
  - Grupo **1** apenas: vê o bloco "Funções Extras" (ZipCode).
- Empresa identificada por `token_empresa`; navegação ao painel usa `integrador_id`, `empresa_id`, `token_empresa` e `pessoa_codigo_ref`.

## Gotchas / decisões kept-bug
- **`view.ctp` vazio**: stub de template do NetBeans, sem ação `view` funcional. Não replicar.
- **Ações inexistentes nos links**: `imprimir_fatura`, `edita_item_fatura`, `delete` (Seguroscotacoes ativos; Prospectscotacoes comentados) NÃO existem nos controllers. Os links ativos em `Seguroscotacoes/index.ctp` quebram. NÃO replicar.
- **Coluna "Dt integracao" duplica `zc_data_nascimento`** e **coluna "Zipcode" formata `zipcode_pessoa_id` como data** (`Dataformat::dateBr` num id). Bugs de exibição. NÃO replicar.
- **Filtro de data inoperante (ambas telas)**: no `index` principal a guard checa `$_REQUEST['data']['data_de']` (chave fora do namespace) e lê `created_at` (campo errado) em vez de `data_de`/`data_ate`; além disso as variáveis `$analise_data_de/$analise_data_ate` são calculadas mas NUNCA entram em `$conditions`. Resultado: filtro de período não filtra nada. NÃO replicar — na migração, implementar filtro de período real por `created_at` (de/até).
- **Filtro "Parceira" da tela Seguros inoperante**: usa `$pessoa_parceira` que nunca recebe valor do campo `pessoa_parceira` postado; o `if(!empty($pessoa_parceira))` é sempre falso. NÃO replicar.
- **Filtro "Produto" usa `Prospectsprodutos.id`** (id da oferta) e não o `produto_seguro_id`, embora o dropdown liste `wbs_produtos_seguros`. Provável bug de semântica; validar regra correta na migração.
- **Persistência de filtros via sessão frágil**: ramo GET só relê `conditions_post` quando `!empty($conditions)` (sempre vazio nesse ponto) — a re-paginação preservando filtro raramente funciona. Reimplementar com filtros na querystring na migração.
- **Exportação Excel acoplada à listagem**: ao marcar "Gerar Excel" o `index` chama `gera_planilhaSPA()` reaproveitando o resultado paginado (limite 5 → exporta no máx. 5 linhas) e usa **valores hardcoded** para Endereço ("Av das Américas, 3000..."), Valor Prêmio ("34,40"), LMI Aluguel ("4000,00") e Vigência ("01/01/2017"). É um protótipo. NÃO replicar como está; na migração definir export real com dados verdadeiros e sem limite de 5.
- **Limite fixo de 5 registros/página** em ambas listagens (paginação muito curta) — provavelmente ajustar.
- **Funções Extras (ZipCode)**: token de integrador hardcoded e domínio externo `o2wbs.net`. Avaliar descontinuação/segredo na migração.
- Helper `Search` (CakePHP Search plugin) gera os campos de filtro; máscaras de CPF/CNPJ NÃO são aplicadas nos inputs (só na exibição da coluna).

## Destino (issues Linear)
- Projeto frontend (wecorp-frontend), fase F15 — tela de listagem de Prospects/Cotações: filtros (empresa/parceira condicional por grupo, nome, documento, período, situação, qualificação, produto), tabela paginada com coluna "situação ofertado/pago" derivada, navegação ao painel da imobiliária/cliente em nova aba.
- Issue separada: exportação Excel real (substituir protótipo hardcoded).
- Issue: segunda listagem "Cotações Seguros" — avaliar se ainda necessária (muitos bugs/colunas redundantes); possivelmente unificar com a principal.
- Backend/services: endpoints de listagem com os joins/derivações de status e regras de qualificação (flag_verifica_vida + ZipCode idade/óbito) e escopo multi-tenant por grupo (1/2/3/4/9 vs. empresa própria; grupo 2 por empresa_resp_*).
