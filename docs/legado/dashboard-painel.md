# Legado (telas) — Dashboard e painel do cliente (F22)

> Tela inicial (`/dashboard`) renderizada por `DashboardController::index`, com 3 variantes de layout decididas por `grupo_id` do usuário logado. Inclui cards de produtos (Fiança, Incêndio, Capitalização, Análises), bloco de "Totais de Repasses" (AJAX) e bloco de "Gráficos" (Highcharts via AJAX). Telas auxiliares: `proibido`, `modulo_inativo`. Redirecionamentos de grupos parceiros (6/8) saem do dashboard para o plugin `Painel`.

## Cobertura
- `app/Controller/DashboardController.php` — `index`, `totalizarperiodorepasse`, `graficos`, `getData` (privado), `modulo_inativo`, `solicita_apolice`, `exportar`.
- `app/View/Dashboard/index.ctp` — tela principal (3 variantes por grupo).
- `app/View/Dashboard/proibido.ctp` — acesso negado.
- `app/View/Dashboard/modulo_inativo.ctp` — módulo não habilitado.
- `app/View/Users/dashboard.ctp` — stub (apenas `echo "Dashboard";`), NÃO replicar.
- `app/View/Elements/graficos.ctp` — bootstrap dos gráficos Highcharts Dashboards.
- `app/webroot/js/requestDataCharts.js` — `GraficoService` (consome `dashboard/graficos`).
- `app/View/Layouts/default_painel_cliente.ctp` — layout cliente (NÃO usado pelo dashboard; ver nota).
- `app/Plugin/Painel/Controller/PainelController.php` — `indexadm`, `indexadmsimula`, `index`, `dashboard_empresa_modulo`, `solicita_apolice` (alvo dos redirects de grupo 6/8).
- `app/Plugin/Painel/View/Painel/` — `indexadm.ctp`, `indexadmsimula.ctp`, `index.ctp`, `painel.ctp`, `proibido.ctp` (telas reais do painel da imobiliária/parceira). Elements `Painel.atendimento_imobiliarias`, `Painel.modais_seguros`. (Mortas: `indexadm-Old.ctp`, `PainelController-Fer.php`.)
- `app/Controller/AppController.php` — `beforeFilter`/`beforeRender` (redirecionamento por grupo, `mostra_colunas`), `VerificaPermissaoModulo` (origem do `modulo_inativo`), `proibido`.
- Ignorados por regra: `app/View/Dashboard/index_bkp.ctp` (backup; variante antiga de grupos parceiros, conteúdo igual ao bloco grupo 6-10 do `index.ctp`).

## Telas / fluxos

### `Dashboard/index.ctp` — variante por `grupo_id` (3 ramos `if/elseif/else`)
Toda a tela é envolta num `Form->create('dashboard', action=index, target=_new)` (id `dashboardIndexForm`) usado só como container para o submit do export de repasses.

**Ramo A — grupos 6,7,8,9,10 (Empresas Parceiras / Integradores):**
- Cabeçalho: "Bem vindo(a), `$nomeUsuario`" + "Parceira: `$Empresa['Empresa']['nome']`" + breadcrumb "Plataforma Digital".
- Card "Painel Dashboard": texto estático ("Acesse o menu do sistema..."). Botão "Gerenciar Análises" (link `/ficha_analise_cadastrals/`) está `display:none`.
- Card "Ajuda": accordion Bootstrap; 2 primeiros painéis (`Como solicitar uma análise?`, `Documentos obrigatórios`) com `display:none`; visível só "Suporte ao Cliente" (telefone/e-mail O2/o2assessoriacadastral). Sem campos de entrada.
- OBS: na prática grupos 6 e 8 NÃO chegam a renderizar esta view — são redirecionados antes (ver Pontos de entrada). Logo o ramo A só atinge 7, 9, 10 efetivamente.

**Ramo B — grupo 5 (Imobiliária / parceiro AlugueSeguro):**
- Cabeçalho com logotipo da empresa: `<img src="$Empresa['Empresa']['url_logotipo']">` (white-label, ver seção própria). Breadcrumb "Plataforma AlugueSeguro.com".
- Card "Painel Dashboard": link estático para doc de APIs (`https://hml.o2wbs.net/api/documentation`) + texto "Em breve conteúdo dedicado ao integrador/parceiro" (hardcoded HML).
- Card "Em destaque": accordion com 1 painel "Suporte" (e-mail suporte@gridweb.com.br).

**Ramo C — demais grupos (1,2,3,4 etc — usuários internos/admin):** dashboard "geral".
- Linha de 4 cards de produto (`col-md-3`), cada um link para o módulo:
  - "Painel Seguro Fiança e Garantias" → `/contratoseguros/segurofiancas/`. Mostra "Aprovadas" = `$Analises[0]['AnalisesAndamento']` e "Emitidas" (vazio). BUG: o número exibido em "Aprovadas" vem de `$Analises` (contagem de fichas de análise), não de fianças.
  - "Painel Seguro Incêndio" → `/contratoseguros/seguroincendioapi/`. "Orçados"/"Emitidos" exibem apenas um ícone (`glyphicon-dashboard`), sem valor real.
  - "Painel Capitalização" → `/contratoseguros/capseguros/`. "Gerados" = `$Vistorias[0]['VistoriasEnviadas']`, "Emitidos" = `$Vistorias[0]['VistoriasProcessamento']`. BUG: rótulos de Capitalização preenchidos com contadores de Vistorias.
  - "Painel Análises" → `/ficha_analise_cadastrals/index`. "Em andamento"/"Finalizadas" também usam `$Vistorias[0][...]` (mesmo bug de fonte de dados).
  - NÃO replicar a fonte cruzada de dados desses cards; é claramente legado inconsistente.
- Linha oculta (`display:none`) com 4 imagens de atalho (Análises/Seguros/Vistorias/Assinatura eletrônica) — legado morto.
- Bloco "Totais de Repasses": `select` "Mês do repasse" (`mesrepasse`, options `$mesrepasses` jan-dez, `value=$mesatual`, `empty=Selecione`, `onChange=totalizarperiodo()`). Exibe `$valortotalrepasse` (do mês atual no load) ou "0,00". Quando há valor, vira link que chama `exportar()`.
  - BUG cosmético: título "Valor total do mês aaaa" quando há valor (placeholder esquecido).
- Bloco "Gráficos": `select` "Filtro de mês" (`select_date_for_charts`: mes_atual[selected], mes_anterior, ultimos_3_meses, ultimos_6_meses, ultimos_12_meses) + `$this->element('graficos')`.

### JS embutido no `index.ctp`
- Carrega jQuery 1.9.1 (conflita com jQuery 3.1.1 carregado em `graficos.ctp` — duplicação de versões).
- `exportar()`: lê `#mesrepasse`, seta `action` do form para `financeiro/relatorios/imprelrepassemesprodutordashboard/3/<mes>` e dá submit (abre em `_new`). Nota: lê `#mesrepasse` mas `totalizarperiodo()` lê `#tipoarquivo` (ids divergentes — ver gotchas).
- `totalizarperiodo()`: monta data_inicio/data_fim do mês selecionado e faz GET AJAX para `dashboard/totalizarperiodorepasse` (params param1/param2); on success escreve em `#vlrtotalrepasse`; on error `alert('Período inválido')`.

### `graficos.ctp` (element)
- Importa Highcharts + Highcharts Dashboards + jQuery 3.1.1 + `js/requestDataCharts.js`. `<div id="charts">` e `new GraficoService()` no ready.

### `requestDataCharts.js` — `GraficoService`
- `url = ROOT_PATH + 'dashboard/graficos'`. Liga `change` do `#select_date_for_charts` e dispara `trigger('change')` no init.
- `buscarDados(select)`: POST `{date:select}`, dataType json. Para cada item monta options (`setChart`) e renderiza via `Dashboards.board('charts', ...)` (`setDashboard`).
- `setChart`: type = `chart.type||'bar'`; series mapeia `{y:quantidade, name:categoria}`; xAxis categorias.
- `setDashboard`: layout fixo de 3 linhas x 2 células (6 gráficos esperados, na ordem do controller). Se `response` vier com !=6 itens, `options[n]` indefinido → erro JS.

### Plugin `Painel` — painel da Imobiliária/Parceira (`Painel/View/Painel/*`)
Telas reais para onde **grupos 6 e 8 são redirecionados** (o `Dashboard/index.ctp` não chega a renderizar para eles — ver Pontos de entrada). Layout próprio com header O2, abas e cards de módulo. Não confundir com `Dashboard/index.ctp`.
- **`indexadm.ctp`** (`indexadm($parceiro,$imobiliaria,$imobiliaria_token)`): dashboard "De olho nos números!" com `R$ <?=Dataformat::currencyMoeda($valortotalrepasse)?>` (repasse do mês). Menu superior: "Painel Imobiliária" | "Simulações e Contratações" (`/painel/painel/indexadmsimula/{parceiro}/{imob}/{token}`), "Central de Sinistros" (`/sinistros/sinistros/gerencia/{imobiliaria}`). Abas: Início / Dúvidas-Atendimento (abas Seguros/Assessoria/Vistoria/Faturas estão comentadas). Cards de módulo com botão "Gerenciar": **Seguros** (→ indexadmsimula), **Análise Cadastral** (→ `/ficha_analise_cadastrals/index`), **Vistoria Locatícia** (→ `/vistorias/vistorias`), **Faturas e Repasses**, **Simulações e Contratações**. Lista de produtos disponíveis (`WbsProdutosSeguros`, badge "Conheça" abre modal `popup_url`). Banner "Simule e contrate". Element `atendimento_imobiliarias`.
- **`indexadmsimula.ctp`** (`indexadmsimula(...)`): variante "Simulações e Contratações" (grupo 8) — foco em iniciar cotações/contratações dos produtos.
- **`index.ctp`** / **`painel.ctp`**: variantes do painel com saudação ao prospect (`$dados_prospect['pessoa_nome']`), blocos "Certificados" e "Atendimento".
- **`proibido.ctp`** (plugin): acesso negado dentro do painel.
- `solicita_apolice` (AJAX): solicitação de apólice a partir do painel; `dashboard_empresa_modulo($id,$modulo_id)`: ativa/gerencia módulo da empresa.

### `proibido.ctp`
- "Acesso não permitido!" + flash + "Entre em contato com nosso suporte e atendimento." Sem campos.

### `modulo_inativo.ctp`
- "Acesso ainda não habilitado :(" + breadcrumb "Painel - Página inicial". Texto explicando ativação de módulos; botão "Painel de Simulações e Contratações" → `/dashboard`; renderiza `$this->element('Painel.atendimento_imobiliarias', plugin=Painel)`.

## Pontos de entrada (controller::ação que renderiza)
- `DashboardController::index` → `View/Dashboard/index.ctp` (rota `/dashboard`).
- `DashboardController::modulo_inativo` → `modulo_inativo.ctp` (também alvo de redirect de `AppController::VerificaPermissaoModulo` quando empresa não tem o módulo e `status_empresa != 4`).
- `AppController::proibido` → `Dashboard/proibido.ctp` (acesso negado genérico).
- `DashboardController::totalizarperiodorepasse` (AJAX-only, json) — total de repasses no período; lança Exception se não-AJAX.
- `DashboardController::graficos` (AJAX-only, json) — dados dos 6 gráficos; lança Exception se não-AJAX.
- `DashboardController::solicita_apolice` (AJAX/json) — envia e-mail de solicitação de apólice individual. BUG: destinatário fixo `fernandononato@gmail.com` (hardcoded). NÃO replicar.
- `DashboardController::exportar` — stub morto (`exit('aqui...function...dashboard')`). NÃO replicar.
- **Redirects por grupo (em `AppController::beforeFilter`, ~linha 261):**
  - `controller==dashboard && grupo==6` → `/painel/painel/indexadm/<codigo_parceiro>/<codigo_imobiliaria>/<token_emp>`.
  - `controller==dashboard && grupo==8` → `/painel/painel/indexadmsimula/<codigo_parceiro>/<codigo_imobiliaria>/<token_emp>`.
  - `codigo_parceiro` default = 1 (O2) quando vazio.

## Regras de negócio relevantes à UI
- `index` consulta (queries SQL cruas em `Empresa->query`):
  - `$Vistorias`: contagens de vistorias por status (2=andamento, 4=enviadas, 5=processamento).
  - `$Analises`: contagem de `ficha_analise_cadastral_pessoas` com `parecer_analisepessoa_statusid=2`.
  - `$valortotalrepasse`: soma `repasses_corp.valor_repasse` JOIN `empresas.codigo_corp = produtor_id` filtrando `empresa_id` logado e `data_previsao` no mês atual. ATENÇÃO: SQL concatena `empresa_id`/datas direto (SQL injection latente; usar query parametrizada no novo).
- `graficos`/`getData` montam 6 datasets, sempre no intervalo `startDate..endDate` derivado do filtro:
  1. Fiança por seguradora (`WbsSeguradora`/`Wbscotacao`, type `pie`).
  2. Fiança por status (`Segurofianca` + `status_segurofiancas`).
  3. Capitalização por status (`Capseguro` + `status_capseguros`, `useTable=capseguros`).
  4. Incêndio (`WbsProspectsCotacoe.status_cotacao_id IN (3,5)`; 3=Cotados, 5=Emitido; usa `created_at`).
  5. Vistorias por status (`Vistoria` + `status_vistorias`; agrupa tudo != 'Entregue' em "Cadastrada").
  6. Assinatura eletrônica (`Sign.status_envelope`; !=4 → "Cadastrados", 4 → "Concluídos").
  - Filtros de data por `request.data['date']`: mes_atual(default)/mes_anterior/ultimos_3/6/12_meses.
  - Se um dataset fica vazio, injeta `data[] = []` (evita erro mas pode gerar gráfico vazio).

## Máquina de estados / status (refletida na UI)
- Vistorias: 2=Em andamento/Cadastrada, 4=Enviadas, 5=Processamento; descrição "Entregue" tratada à parte nos gráficos.
- Incêndio (status_cotacao_id): 3=Cotados, 5=Emitido (únicos exibidos).
- Sign (status_envelope): 4=Concluído; demais=Cadastrado.
- Fiança / Capitalização: status vindos das tabelas `status_segurofiancas` / `status_capseguros` (descrição dinâmica).
- Análises: `parecer_analisepessoa_statusid=2` = em andamento (único contado).

## Multi-tenant / white-label
- Grupos (de `AppController` comentários e branches):
  - 1=admin/interno, 2=usuário c/ EULA, 3,4=internos, 5=Imobiliária parceira (AlugueSeguro), 6=Master Imobiliária Parceira, 7=Corretores Parceira, 8=Assistente Parceira, 9,10=variações Empresas Parceiras.
- `mostra_colunas` (beforeRender): grupos 1-5 e 9 → '1'; demais → '2' (usado em outras telas/menus).
- White-label: ramo B (grupo 5) exibe `url_logotipo` da empresa no topo do dashboard. Demais ramos não trocam logo. O nome "Parceira" vem de `$Empresa['Empresa']['nome']`.
- `getDadosEmpresaLogado()` (Model Empresa) alimenta `$Empresa` usado no cabeçalho/logo.
- Layout: o dashboard usa o layout `default` (não o `default_painel_cliente`). `default_painel_cliente.ctp` é o layout da ÁREA DO CLIENTE (controllers `Garantidora`, `Segurofiancas`, `Formulariosweb`, `Clientesegurofianca`), com branding "Plataforma Digital", rodapé de logos de seguradoras, chat TomTicket (id EP22145) e GA UA-133676373-1; tem hardcodes de `ROOT_PATH`/HML comentados. Relevante para paridade do painel do cliente mas alheio às telas de dashboard administrativo.

## Gotchas / decisões kept-bug
- Cards do dashboard geral (ramo C) usam fontes de dados trocadas (Capitalização/Análises mostram contadores de Vistorias; Fiança mostra contagem de Análises). NÃO replicar — refazer com métricas corretas.
- "Valor total do mês aaaa": placeholder textual; corrigir.
- `exportar()` lê `#mesrepasse` mas `totalizarperiodo()` lê `#tipoarquivo`; o `select` tem `name=mesrepasse`/`id=mesrepasse` mas é gerado por `Form->select('tipoarquivo', ...)` (CakePHP gera `id=tipoarquivo` por padrão, sobrescrito? Ambiguidade) — validar id real no DOM. Comportamento de export/totalização frágil.
- Duas versões de jQuery carregadas (1.9.1 no index, 3.1.1 no element graficos) — conflito potencial.
- `graficos` e `totalizarperiodorepasse` exigem AJAX (`$this->request->is('ajax')`), senão lançam Exception não tratada.
- `totalizarperiodorepasse` faz `SELECT sum(...) from repasses_corp where data_previsao between $param1 and $param2` SEM filtrar por empresa (diferente do load inicial que filtra por empresa logada) e com params concatenados (SQL injection). Inconsistência + risco — NÃO replicar.
- `solicita_apolice` envia para e-mail pessoal hardcoded. NÃO replicar.
- `exportar()` do controller é stub morto; o export real é via submit do form para `financeiro/relatorios/imprelrepassemesprodutordashboard`.
- `Users/dashboard.ctp` é stub (`echo "Dashboard"`) — não é a tela real; ignorar.
- `setDashboard` assume exatamente 6 gráficos com layout fixo 3x2; menos itens quebra JS.
- Doc de APIs e textos apontam para ambiente HML (`hml.o2wbs.net`) — não levar URLs HML para produção.

## Destino (issues Linear)
- Projeto: wecorp-frontend (telas). Domínio F22 — Dashboard e Painel do cliente.
- Sugestão de issues:
  - Tela Dashboard (Next.js): cards de produto com métricas CORRETAS por módulo (Fiança/Incêndio/Capitalização/Análises), substituindo as fontes cruzadas do legado.
  - Bloco "Totais de Repasses" com filtro de mês + export (endpoint backend parametrizado e filtrado por empresa).
  - Bloco "Gráficos" (6 gráficos: fiança por seguradora/status, capitalização, incêndio, vistorias, assinatura) com filtro de período; backend novo retornando datasets já agregados.
  - Variantes por perfil/grupo (parceira/imobiliária/interno) e white-label (logotipo da empresa).
  - Telas auxiliares: acesso negado (proibido) e módulo inativo.
  - (Backend) substituir queries SQL cruas/injectáveis e e-mail hardcoded.
