# Legado (telas) — Configurações (telas) / Lookups (F20)

> Conjunto de telas administrativas de "Manutenção" geradas por scaffold (CakePHP bake): ~21 CRUDs de tabelas de domínio/lookup (quase todas com um único campo `descricao`) + 1 modal de troca de Ano Letivo (`Configs`) + 1 editor de junção (TabelaTipoConsultas x Itens de Análise). Padrão visual e de fluxo idêntico entre todas; comportamento real tem bugs relevantes (guarda de permissão não bloqueia a action; código morto no `edit`; page size fixo 5).

## Cobertura

Controllers cobertos (`app/Controller/*.php`):
- `ConfigsController.php` (ação `changeAnoLetivo`)
- `BancosController.php`, `ClassificacaoRiscosController.php`, `TabelaTipoConsultasController.php` (lidos na íntegra)
- Demais lookups (mesmo molde scaffold, confirmados por amostragem de campos): `EnderecoTiposController`, `EstadoCivilsController` (não listado em /Controller mas tem views), `ItenstipoanalisesController`, `MetaTipoReferenciasController`, `PatrimoniosTiposController`, `TabelaTipoConsultasController`, `TipoArquivosController`, `TipoGarantiasController`, `TipoPessoaCadastrosController`, `TipoPessoasController`, `TipoRelacionamentosController`, `TiposImovelsController`, `GruposController`, `PermissiongroupsController` (permissões — só campos amostrados)
- `AppController.php::VerificaPermissao` / `::VerificaPermissaoMenu` (guarda usada por todos)

Views cobertas (`app/View/<Modulo>/{add,edit,index,view}.ctp`):
- Bancos, ClassificacaoRiscos, EnderecoTipos, EstadoCivils, ItensTipoAnalises, MetaTipoReferencias, OrigemCadastros, PatrimoniosTipos, PessoaQualificacaos, PessoaStatusCadastros, Relacionamentos, StatusAnalises, StatusFichas, StatusPessoas, TabelaTipoConsultas (+ `tabelaconsultaitens.ctp`), TipoArquivos, TipoGarantias, TipoPessoaCadastros, TipoPessoas, TipoRelacionamentos, TiposImovels

Lookups adicionais (mesmo molde scaffold "Manutenção" — `index/add/edit/view` + delete via `postLink`; controllers com 5-6 ações cada, campos auto-derivados do schema):
- **`Patrimonios`** (`app/View/Patrimonios/*` + `PatrimoniosController`) — cadastro de patrimônios (≠ `PatrimoniosTipos`). Breadcrumb "Painel / Manutenção".
- **`ItensAnalises`** (`app/View/ItensAnalises/*` + `ItensAnalisesController`) — itens de análise (campos `descricao`); ligado ao domínio Análise Cadastral.
- **`ModelosParecers`** (`app/View/ModelosParecers/*` + `ModelosParecersController`) — modelos de parecer (campos `descricao`, `status`); usados na emissão de parecer da análise.
- **`ReferenciasContatos`** (`app/View/ReferenciasContatos/*` + `ReferenciasContatosController`) — referências/contatos (domínio Pessoas).
- **`Parceiros`** (`app/View/Parceiros/*` + `ParceirosController`) — cadastro de parceiros (relacionado a Empresas; ver também `empresas.md`).
- **`Telefones`** (`app/View/Telefones/*`) e **`Enderecos`** (`app/View/Enderecos/*`) — CRUD scaffold standalone dos sub-recursos de Pessoa/Empresa (na prática editados via elements nas telas-mãe; estas views são técnicas/admin).
- **`LogDetails`** (`app/View/LogDetails/*` + `LogDetailsController`, 5 ações) — detalhes de log/auditoria (admin; só leitura útil).

## Telas / fluxos

Todas as telas compartilham o mesmo esqueleto (Bootstrap 3, layout do painel):
- Cabeçalho `page-head` com `<h2>` do módulo e breadcrumb `Painel / Manutenção`.
- Coluna esquerda `col-md-3` "Ações" (panel) com links contextuais; conteúdo em `col-md-9`.
- Sem máscaras JS, sem validação client-side custom: validação é model-level, exibida inline pelo `FormHelper` do Cake. Labels e tipos de input são auto-derivados do schema pelo `Form->input()`.

`index.ctp` (lista):
- Tabela `table table-striped`; cabeçalhos com `Paginator->sort(campo)` (ordenação por clique).
- Ações por linha: ver (`glyphicon-info-sign` → `view`), editar (`glyphicon-edit` → `edit`), excluir (`Form->postLink` → `delete`, confirm: "Deseja mesmo remover o Registro ID # %s?").
- Link "Novo <X>" no painel de ações → `add`.
- Paginação: `prev('← Anterior')` / `numbers` / `next('Próximo →')`, só aparece se `pageCount > 1`. Contador: "Página {:page} de {:pages}, exibindo {:current} registros do total de {:count}…". **Page size fixo em 5** (todos os controllers: `'limit' => 5`).

`add.ctp` / `edit.ctp` (formulário):
- `Form->create('<Model>', ['role'=>'form'])`; cada campo em `div.form-group` com `class form-control` e `placeholder`.
- Botão `Form->submit('Processar', name=data[submit])`. Botão "Aplicar" existe comentado em todas as views.
- `edit` adiciona input `id` visível (editável!) e link "Excluir" (postLink com confirm) no painel de ações.
- Headings não traduzidos: literalmente "Add Banco" / "Edit Banco" (string `__('Add Banco')`).

`view.ctp` (detalhe):
- Tabela read-only `th/td`; **só renderiza linhas de campos não-vazios** (`if(!empty(...))`).
- Pode incluir blocos "Related …" (ex.: `Bancos/view.ctp` lista `ReferenciasContato` associadas, com links cross-controller para `referencias_contatos`).

Campos por lookup (derivados de `add.ctp`):
| Módulo | Campos |
|---|---|
| Bancos | `sigla`, `nome` |
| ClassificacaoRiscos | `empresa_id` (input livre!), `sigla`, `descricao`, `fonte`, `tipo_pessoa`, `percentual_risco` |
| EnderecoTipos | `descricao` |
| EstadoCivils | `descricao` |
| MetaTipoReferencias | `descricao` |
| OrigemCadastros | `nome` |
| PatrimoniosTipos | `descricao` |
| PessoaStatusCadastros | `descricao` |
| Relacionamentos | `descricao` |
| StatusAnalises | `descricao` |
| StatusFichas | `descricao` |
| StatusPessoas | `descricao` |
| TipoArquivos | `descricao` |
| TipoGarantias | `descricao`, `status` |
| TipoPessoaCadastros | `descricao` |
| TipoPessoas | `sigla`, `descricao` |
| TipoRelacionamentos | `descricao` |
| TiposImovels | `descricao` |
| ItensTipoAnalises | `tipo_analises_id`, `itens_analises_id` (junção/relação) |
| PessoaQualificacaos | NÃO é lookup: campos de pessoa (`naturalidade`, `nacionalidade`, `estado_civil_id`, `sexo`, `filiacao_pai/mae`, `rendas_extras`, `paga_aluguel`, `favorecido_aluguel`, …, `pessoa_id`) — scaffold genérico, fora do escopo de "config" |

### Tela especial — Análises/Consultas x Itens (TabelaTipoConsultas)
- `index.ctp`: colunas custom (`nome`, `nome_interno`, `descricao`, `tipo_pf`, `tipo_pj`, `codigo_integrador`, `webservice_integrador_id`); ações por linha: "Gerenciar itens de análise" (→ `tabelaconsultaitens`) e "Editar". Ordenação `id DESC`. **Link "Nova Análise/Consulta" aponta para `action => '#'` (quebrado — não chama `add`).**
- `add.ctp`/`edit.ctp`: além dos campos acima, `valor_integrador`, `classificacao_consultas_id`, `tipo_consulta`. `edit` recarrega `ClassificacaoConsulta` em `find('list')`.
- `tabelaconsultaitens.ctp`: editor de junção — checkbox por `ItensAnalise` (com "Marca/desmarca todos" via `selecionaTodos(this,'checkbox_item')`), pré-marcados conforme `itensTipoAnalise`. Campos `id`/`nome` da análise em readonly. Submete `data[ItensTipoAnalises][analise_id][]`.

### Modal — Troca de Ano Letivo (Configs::changeAnoLetivo)
- AJAX (`ajaxRequest()`); GET retorna `$list` de anos letivos exceto o atual da sessão (`App.AnoLetivo.id`).
- POST: valida `Anoletivo.anoletivo` não-vazio ("É necessário informar um ano letivo"), chama `Anoletivo::changeAnoLetivo`, responde JSON (`notRender()->jsonResponse()`).
- Resíduo de sistema escolar (Ano Letivo) — ver Gotchas.

## Pontos de entrada (controller::ação que renderiza)

- `<Lookup>Controller::index()` → `View/<Lookup>/index.ctp`
- `::add()` → `add.ctp` (POST grava); `::edit($id)` → `edit.ctp`; `::view($id)` → `view.ctp`; `::delete($id)` (sem view, redireciona)
- `TabelaTipoConsultasController::tabelaconsultaitens($id)` → `tabelaconsultaitens.ctp`
- `ConfigsController::changeAnoLetivo()` → modal AJAX (sem layout)

## Regras de negócio relevantes à UI

- CRUD scaffold idêntico em todos os lookups:
  - `add`: `create()`+`save()` → flash sucesso "(x) Registro criado com sucesso!" e redirect `index`; falha → flash danger.
  - `edit`: 404 se `!exists($id)`; em POST/PUT `save()` → flash "(x) Registro alterado com sucesso!" e redirect `index`.
  - `delete`: `onlyAllow('post','delete')`, exige confirm no postLink; flash sucesso/erro; redirect `index`.
  - `index`: `recursive=0`, `limit=5`.
- FKs (`classificacao_consultas_id`, `estado_civil_id`, etc.) renderizam como `<select>` apenas se o controller passar a lista correspondente; caso a convenção de nome não bata (ex.: var `$classificacaoConsultas` vs campo `classificacao_consultas_id`), caem para input texto.

## Máquina de estados / status (refletida na UI)

- Não há FSM nas telas de config; status são apenas dados de domínio em tabelas-lookup: `StatusAnalises`, `StatusFichas`, `StatusPessoas`, `PessoaStatusCadastros` armazenam apenas `descricao` (a UI não impõe transições). `TipoGarantias` tem campo `status` livre.

## Multi-tenant / white-label

- Lookups são **globais** (sem `empresa_id`), exceto `ClassificacaoRiscos`, que tem `empresa_id` exposto como **input numérico livre** no formulário — sem escopo automático ao tenant logado nem `<select>` de empresas. Risco de vazamento/atribuição incorreta entre tenants.
- A guarda de menu (`AppController::VerificaPermissaoMenu`) usa `empresa_id` da sessão para resolver módulos por empresa (`empresas_modulos`), mas as actions de lookup não filtram por empresa.

## Gotchas / decisões kept-bug

- **Guarda de permissão NÃO bloqueia a action**: `beforeFilter()` chama `VerificaPermissao()` e, se `false`, faz `Session::setFlash` + `$this->render('/Dashboard/proibido')` — porém NÃO faz `return`/`exit`. No fluxo Cake 2 a action ainda executa após o beforeFilter. NÃO replicar: no destino, o guard deve efetivamente bloquear (throw/forbidden/redirect) antes da action.
- `VerificaPermissao` monta SQL por **concatenação de strings** (`group_id`, `controller`, `action`) — SQL injection latente. NÃO replicar; usar query parametrizada/RBAC.
- **Código morto no `edit`**: o bloco `if (isset($this->request->data['apply'])) { redirect(referer()) }` está após um `return $this->redirect(action=>index)` → nunca executa. O botão "Aplicar" está comentado em todas as views. No destino: ou implementar "Salvar e continuar" corretamente, ou remover.
- **Page size fixo = 5** em todos os controllers (UX ruim para tabelas de domínio); sem busca/filtro nas listas.
- `edit.ctp` expõe o `id` como input editável.
- TabelaTipoConsultas: link "Nova" → `'#'` (criação inacessível pela UI, embora `add()` exista). `tabelaconsultaitens()` faz `deleteAll` e re-insere sem transação; em erro chama `die('Erro ao gravar…')` deixando estado parcial. NÃO replicar (usar upsert transacional).
- `Configs::changeAnoLetivo` / model `Anoletivo` e `App.AnoLetivo` são resíduo de sistema escolar legado; provavelmente **não aplicável** ao domínio de seguros do WeCorp. Confirmar antes de migrar; tratar como descartável.
- Strings de heading não traduzidas ("Add Banco"/"Edit Banco").
- `PessoaQualificacaos` está sob este grupo de scaffold mas é cadastro de pessoa, não config — não migrar como lookup.

## Destino (issues Linear)

- Frontend (wecorp-frontend): tela genérica de CRUD de lookups (tabela com busca/paginação configurável, formulário de campo único `descricao` + variantes), reaproveitável para todos os domínios (~18 lookups de `descricao`).
- Telas específicas: ClassificacaoRiscos (campos extras + escopo por empresa correto), TipoGarantias (`status`), TipoPessoas (`sigla`+`descricao`), Bancos (`sigla`+`nome`).
- Editor de junção Análise↔Itens (substituir tabelaconsultaitens por multiselect/transfer transacional).
- Backend (wecorp-backend): endpoints CRUD por lookup com RBAC efetivo e escopo multi-tenant; descartar Ano Letivo.
- (Mapear aos milestones HUB de "Configurações"/F20 conforme reference_linear.)
