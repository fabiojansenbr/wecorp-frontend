# Legado (telas) — Análise Cadastral (telas + tela do operador)

> Módulo de Análise Cadastral de pretendentes (locatário/fiador, PF/PJ) para locação. Telas: gestão/listagem com filtros e cards expansíveis de proponentes, criação/edição da ficha (mesma view `add.ctp` para `add`+`edit`; variante simplificada `add_user_imobiliaria.ctp` para imobiliárias), tela do operador para execução de parecer item-a-item (modal `edita_ficha_analise.ctp`) e geração de PDF (Completa/Expressa/Final) via mPDF.
> Comportamento fortemente condicionado por `grupo_id` do usuário (WeCorp/operadores vs. imobiliária) e por `tipoficha_analises_id=1` (Cadastral) — análises RH/Sindicância (`=2`) são tratadas por telas paralelas (`ficha_rh*`) fora do escopo principal.

## Cobertura
- Views: `add.ctp`, `add_user_imobiliaria.ctp`, `index.ctp`, `index_paginator.ctp`, `edita_ficha_analise.ctp`, `view.ctp`, `addcr.ctp` (versão alternativa antiga do form de criação).
- Controller: `FichaAnaliseCadastralsController.php` — ações `index`, `index_paginator`, `view`, `add`, `edit`, `edita_ficha_analise`, `executa_analise`, `alteraStatusGrupoAnalise`, `mostraPessoasExtras`, `checaDocumento`, `pdf_analise_pessoa`, `carrega_socios`/`novo_socio`/`altera_socio` (sócios PJ na tela do operador).
- JS: `app/webroot/js/modules/fichacadastral.js` (`janelaAnalise`, `verificaPagamentoEmpresa`, `carregaSocios`, busca AJAX da index, `addFormField`).
- NÃO cobertos a fundo (fora do escopo / paralelos): `ficha_rh*`, `pdf_ficha_rh.ctp`, `pdf_htmlcelisa.ctp`, `pdf_analise_pessoa_1.ctp`, integrações Procob/Serasa (apenas o ponto de consumo na UI).

## Telas / fluxos

### 1. `index.ctp` — Gestão de Análises (shell + filtros)
- Botão "NOVA ANÁLISE" (dropdown): links diretos para `add/locatario/pf`, `add/locatario/pj`, `add/fiador/pf`, `add/fiador/pj` (define tipoCadastro + tipoPessoa pela URL).
- Bloco acordeão "Filtros e pesquisas": campos `analise_id` (Código), `pessoa_nome` (Nome/Razão), `pessoa_documento` (CPF/CNPJ), `status_analise` (select StatusFicha, empty="Todas"), `analise_data_de`/`analise_data_ate` (datepicker), `empresa_parceira_id`.
- Combo "Imobiliária/Empresa" só aparece para `grupo_id` 1 ou 2; para grupos 3/4/5 vira hidden = `empresa_id` do usuário.
- Botão "Pesquisar" (`#btnPesquisar`) NÃO submete o form: serializa `#FichaAnaliseCadastralIndexForm` e faz GET AJAX para `ficha_analise_cadastrals/index_paginator/`, injeta resultado em `#paginator`. A listagem em si é o `index_paginator.ctp`.
- Máscaras/validações: datas via classe `datepicker`; sem validação client-side obrigatória nos filtros.

### 2. `index_paginator.ctp` — Listagem (tabela + cards de proponentes)
- Tabela: Cod (id), [Empresa/usuário — só se `$mostra_colunas==1`], Imóvel (endereço), Tipo garantia, Status, Data, Ações. Ordenável via `Paginator->sort`. Paginação 5/página (`limit=>5`).
- Coluna Status: se `status_fichas_id==1` (Rascunho), mostra ícone "Enviar Grupo de Análise para Execução" → link `alteraStatusGrupoAnalise/{id}`.
- Botão "Proponentes" expande linha de detalhe (collapse) com:
  - Token da análise; dropdowns "Novo cadastro PF/PJ" gerando links `edit/{ficha}/novo/{locatario|fiador}/{pf|pj}/0/0`.
  - Para cada `FichaAnaliseCadastralPessoa`: separa por `relacionamento_id` (4=Locatário, 5=Fiador). Para PF mostra nome; para PJ mostra `nome_empresa`.
  - Ações por proponente: "Abrir" (link `edit/{ficha}/principal/{locatario|fiador}/{pf|pj}/{pessoa}/0`), "Anexos" (`arquivosAnexos(...)`), executar análise (`janelaAnalise(...)` — abre modal do operador), "Contratar CAP" (link cross-módulo capitalização), e botões de PDF.
  - Sub-proponentes (solidários/sócios) carregados via `requestAction mostraPessoasExtras` (relacionamento_id=0, `pessoaprincipalid`), com dropdown "Locatário solidário PF" / "Fiador solidário PF" / "Sócio PF" conforme tipo do principal.
  - Aviso de duplicidade: `requestAction checaDocumento` por CPF; se >0 ocorrências exibe badge vermelho "este CPF foi encontrado em (N) análises" (só retorna >0 para grupo 1/2 — ver Gotchas).
- Comportamento por grupo (botões executar análise + PDF Completa/Expressa): exibidos para `grupo_id` 1,2,3,4,5,9. Para os demais (imobiliária) só o botão "Imprimir" PDF tipo `0` (Final).

### 3. `add.ctp` — Criação E edição da ficha (form principal)
- Usada tanto por `add` quanto por `edit` (controlado por `$this->params['action']`). É o coração do cadastro.
- Cabeçalho/identificação:
  - `empresa_parceira_id` (select2, classe `lista_imobiliarias required`); `onchange=verificaPagamentoEmpresa(this.value, origem)`. Empty="Selecione" só p/ grupos 1-5.
  - Em `edit`: campos read-only `id` (Id Análise), `sigla` (`sigla_tipo_analise`), Usuário que cadastrou (`nomeUsuarioAnalise`/hidden `user_id`,`empresa_id`); em `add`: usuário logado.
  - `forma_pagto` (radios a partir de `listaformaPagto`): em `edit` ficam desabilitados exceto o `tipoPagamento` atual (`checked`).
- Dados do Imóvel (opcional):
  - `imovel_cep` (classe `cep`, maxlength 12) com `onblur=searchAddressByCep(...)` preenchendo endereço/bairro/cidade/uf; `imovel_endereco` (max 60); `imovel_bairro`/`imovel_cidade`/`imovel_uf` selects encadeados (`buscaBairros`, `buscaCidades`); `imovel_tipo`; `imovel_codigo` (Ref); `valor_aluguel`/`valor_iptu`/`valor_condominio_mensal` (máscara `MascaraMoeda(this,".",",",event)`); `imovel_locador_documento` (classe `documento`, max 14); `imovel_locador`.
  - `tipo_garantia_id` (required); `modalidade_uso_imovel` aparece só quando tipoPessoa = `pj`.
- Pretendente: inclui o element `dados_pessoa_principal` (campos da Pessoa). Em `addcr.ctp` (versão antiga) há também elements de cônjuge, sócios, referências bancárias/comerciais/pessoais e patrimônios — em `add.ctp` cônjuge/sócios foram desativados (comentários "Pedido para Desativar | Euzébio 27/06/2019").
- Link web para o cliente:
  - Em `add`: bloco "Enviar link web para o cliente?" com `email_enviar_link` (data-mask email), `celular_enviar_link` + confirmação (data-mask mobile).
  - Em `edit`: botão "Formulário web para o Cliente" mostra token, link curto (`link_area_cliente`), botão WhatsApp (`web.whatsapp.com/send?phone=55...`), checkbox `ativa_form_web`, data de conclusão do cliente, e aviso vermelho se `token_expirado != '0'`.
- Termos de uso: textarea readonly (element `termos_uso`) + checkbox `aceitou_termos` (required). Em `edit`, se já aceito vem `checked disabled` e mostra data de aceite (`data_aceite_termos`).
- `status_fichas_id` (select obrigatório). No `add` os status são limitados a: `''`=Selecione, `1`=Rascunho, `6`=Processar Análise.
- `flag_permite_cobrar_analise` ("Gerar cobrança para o cliente"): só renderiza se `Empresa.flag_permite_cobrar_analise` setado ou em `add`; valores via `flagAutorizaPagto` (0=Não,1=Sim,2=Pago na unidade); guarda valor original em hidden `flag_permite_cobrar_analise_orig`.
- Botão "Processar" (`#processar_analise`): em `add` está `disabled` (habilitado só em `edit` no markup — ver Gotchas); `onsubmit=verificaForm()` valida que PF tenha CPF + nome.
- JS inline: ao sair (`blur`) de `imovel_locador_documento` e `PessoaDocumento`, chama `getDocumento()` para autopreencher (nome/nome_mãe/nascimento/sexo para 11–13 dígitos = CPF; razão_social/fantasia para 14 = CNPJ). Documento <11 dígitos: alert "Verifique o documento".

### 4. `add_user_imobiliaria.ctp` — variante para imobiliária (grupo 6/7/8)
- `add()` chama `$this->render('add_user_imobiliaria')` quando `grupo_id` ∈ {6,7,8}.
- Formulário simplificado: `empresa_parceira_id`, `user_id`, `empresa_id` ficam hidden = empresa/usuário logado; `imovel_endereco` hardcoded `'n/d'`; `Pessoa.relacionamentos_id` hidden derivado de tipoCadastro (locatario=4, fiador=5); `Pessoa.tipo_pessoa` hidden = `strtoupper(tipoPessoaNova)`.
- Mostra "Você está solicitando uma análise de perfil {tipoCadastro} ({PF|PJ})"; foco em Dados do Cliente. Submit usa mesmo `verificaForm` + spinner "Aguarde, processando...".

### 5. `edita_ficha_analise.ctp` — TELA DO OPERADOR (modal de execução do parecer)
- Aberta via `janelaAnalise(ficha_analise_pessoa_id, ficha_id, pessoa_id, pessoa_respid)` → `openModalFromUrl('.../edita_ficha_analise/...')` (modal 60% x 450px).
- Cabeçalho: Pessoa (nome/nome_empresa), CPF/documento + tipo_pessoa, "Tipo consulta O2" (`TipoAnalises.descricao`), aviso `checaDocumento` (badge se CPF em N análises), "Tipo Consulta" (`TabelaTipoConsulta.nome` + `codigo_integrador`).
- Bloco Sócios (só PJ): mostrar/esconder sócios da PJ via `carregaSocios()` (AJAX `carrega_socios`), form "Novo Sócio" (nome, documento, identidade/OE/dt emissão, naturalidade, cargo, percentual) → `ProcessaAlteracaoSocio('FormSocioNovo',...,'novo')`.
- Form principal (`#PessoaAnalise`) → POST AJAX `executa_analise/{fap}/{ficha}/{pessoa}/{resp}`. Hiddens: `ficha_analise_pessoa_id`, `ficha_analiseid`, `ficha_pessoaid`, `pessoa_respid`, `tipoficha_analise`, `PessoaNome`, `PessoaDoc`, `Imovel`, `UsuarioCodigo`.
- Tabela "Itens da Análise x Pessoa" (loop `$ItensAnalise`): por item exibe
  - Retorno do Webservice: bloco scrollável com `requestAction abreSerasa` (síntese/quantidade/endereço/telefones) + parse de `retorno_webservice` (JSON) — multi-coluna.
  - Textarea "Retorno externo (manual)" (`retorno_consulta_externa`, classe `rich_editor`/TinyMCE).
  - Select `modelos_parecer_id` (classe `obs_consta`) + textarea "Obs p/ PDF" (`parecer_obs`).
  - Select `status_analises_id` (classe `status_item_analise`).
  - Ação por item: lixeira `confirmaParecerItem(id)`.
  - Dropdowns de massa nos cabeçalhos: "Marcar todos" como Nada Consta(1)/Consta(2) e Pendente(1)/Em andamento(2)/Concluída(3) via `selecionaTodosSelect`.
- Parecer da pessoa (só se `tipoficha_analise==1` para os campos de risco): `parecer_analisepessoa_statusid` (Status da Análise da Pessoa), `obs_geral_analise` (não vai p/ PDF), `classificacao_risco_id`, `percentual_classificacao_risco` + `obs_classificacao_risco` (só PJ), `obs_ref_bancarias_comerciais`, `obs_analise_documentos`, `obs_bens_garantia`, `obs_analise_contatos` (+ tabela de contatos/referências), `data_entrega` (datetimepicker) com checkbox "Atualizar data da entrega", `analise_parecer` (Parecer final).
- Avisos repetidos "itens em vermelho serão impressos no PDF".
- Submit via `jquery.form` ajaxForm com progressbar; sucesso → reabre `janelaAnalise(...)`; alerta "Análise executada com sucesso!" ou erro.

### 6. `view.ctp` — scaffold CRUD legado
- View bootstrap/scaffold padrão CakePHP (campos crus: id, empresa, user, tipo_garantia, valores, status, parecer, endereço imóvel) + tabela "Related Patrimonios". Ações: Editar/Excluir/Listar/Novo. Aparenta não ser usada no fluxo real (a UI real usa index/cards). Tratar como legado morto — NÃO replicar.

## Pontos de entrada (controller::ação que renderiza)
- `index()` → `index.ctp` (shell + filtros; popula `statusFicha`, `relac`, `listaImobiliarias`). Layout `modal` quando há busca GET.
- `index_paginator()` → `index_paginator.ctp` (chamado por AJAX da busca; `recursive=2`).
- `add($tipoCadastro,$tipoPessoa)` → `add.ctp` (default) ou `add_user_imobiliaria.ctp` (grupo 6/7/8). POST chama `cadastraFicha()`.
- `edit($id,$tipo,$tipocadastro,$tipopessoa,$idpessoa,$idpessoaprincipal)` → `$this->render('add')` (reusa add.ctp). Se `$tipo=='novo'` chama `cadastraFicha`; senão salva Ficha + Pessoa + referências/sócios.
- `edita_ficha_analise($fap,$ficha,$pessoa,$resp)` → `edita_ficha_analise.ctp` (modal do operador).
- `executa_analise(...)` → JSON (`true`/`false`); salva itens + parecer da pessoa em transação.
- `alteraStatusGrupoAnalise($id)` → redirect index; seta `status_fichas_id=2`.
- `pdf_analise_pessoa($analise_id,$ficha_id,$pessoa_id,$tipo_pdf)` → render `pdf_analise_pessoa.ctp` via mPDF (layout=null, output 'D' download).
- `view($id)` → `view.ctp` (scaffold).
- Auxiliares AJAX/`requestAction`: `mostraPessoasExtras`, `checaDocumento`, `abreSerasa`/`abreProcob`, `carrega_socios`, `novo_socio`/`altera_socio`/`nova_empresa_socio`.

## Regras de negócio relevantes à UI
- URL semântica de criação/edição: `tipoCadastro` ∈ {locatario, fiador}; `tipoPessoa` ∈ {pf, pj}; em edit `tipo` ∈ {principal, solidario, socio, novo} e `tipocadastro`/`tipopessoa`/`idpessoa`/`idpessoaprincipal`.
- `tipoficha_analises_id=1` = Análise Cadastral; toda a listagem filtra por isso (RH/Sindicância `=2` ficam de fora).
- Campos de risco (classificação, score, % risco, observações temáticas) só aparecem/são salvos quando `tipoficha_analise==1`.
- Valores monetários: máscara JS `MascaraMoeda`; controller converte com `Dataformat::currencyToDecimal` ao salvar e `currencyMoeda` ao exibir.
- Datas: máscara/picker no front; `Dataformat::dateToSql` / `datetimeBr` na conversão.
- Autopreenchimento por documento (`getDocumento`) e por CEP (`searchAddressByCep`).
- PDF: `tipo_pdf` 1=Completa, 2=Expressa, 0=Final do cliente (deriva sigla de `FichasAnalisesGruposAnalise.tipo_analises_id`: '2'→Completa, '1'→Expressa). Filtra itens por `ItensAnalise.flag_completa`/`flag_expressa` e exclui `ItensAnalise.id=105`. Footer com token de autenticidade.
- `verificaPagamentoEmpresa`: consulta `empresas/empresa_dados/{id}/fatura`; se faturado → marca/destrava `forma_pagto_5` (faturado) e desabilita as demais formas; senão desmarca/desabilita e avisa que cobrança será via PIX. Em `add` também consulta `empresas/tabela_analise` p/ exibir aviso de tabela de preço personalizada.

## Máquina de estados / status (refletida na UI)
- `StatusFicha` (status do GRUPO de análise / `status_fichas_id`): valores conhecidos pela UI — `1`=Rascunho, `2`=Enviada para Execução (set por `alteraStatusGrupoAnalise`, grava `data_enviada_execucao`), `6`=Processar Análise (opção no add). Demais labels vêm de `StatusFicha->getList()`.
- Transição Rascunho→Execução: botão na coluna Status da listagem (só quando `status_fichas_id==1`).
- `status_analises_id` (status por ITEM na tela do operador): `1`=Pendente, `2`=Em andamento, `3`=Concluída.
- `modelos_parecer_id` em massa: `1`=Nada Consta, `2`=Consta (semântica dos atalhos "Marcar todos").
- `parecer_analisepessoa_statusid` (status da análise DA PESSOA): `4`=Concluída → dispara notificação por e-mail à assessoria (bloco de e-mail está comentado em `executa_analise` — ver Gotchas).

## Multi-tenant / white-label
- Acesso por `grupo_id`:
  - 1,2 = WeCorp (admin/operador): veem combo de imobiliárias completo, aviso de duplicidade de CPF (`checaDocumento` só retorna >0 p/ 1 e 2), podem cobrar análise.
  - 3,4,5,9 = operadores internos: executam análise + imprimem PDF Completa/Expressa.
  - 6,7,8 = usuários de imobiliária (admin parceira/corretor/assistente): tela de criação simplificada (`add_user_imobiliaria`), empresa forçada à própria, só PDF Final (tipo 0); ao enviar p/ execução dispara e-mail à assessoria (`emailNotification('assessoria','nova')`).
- `VerificaPermissao()` níveis: `2`=restringe às análises da própria empresa (e `empresa_master`/empresas resp. por seguros/análises/vistorias); `3`=restringe ao próprio usuário + empresa.
- White-label no PDF: `flag_exibe_logo_analise` + `url_logotipo_empresa` da empresa parceira controlam exibição do logotipo no relatório.
- Forma de pagamento "faturado" depende de flag da empresa (`pagamento_faturado`); `forma_pagto_todos` vs `forma_pagto` conforme grupo/empresa.

## Gotchas / decisões kept-bug
- `add.ctp`: botão "Processar" tem `disabled` quando `action != 'edit'` — ou seja, em `add` o submit principal fica desabilitado no markup; o fluxo real de submit/validação é via JS (`#processar_analise` no script inline / `verificaForm`). NÃO replicar literalmente o `disabled`; o equivalente funcional é: habilitar submit após validação client-side.
- `index()` tem lógica de paginação parcialmente morta: monta `$options`/paginate dentro de `if(!empty($_REQUEST['data']))`, mas a listagem real é servida por `index_paginator()` via AJAX. A `index.ctp` em si só renderiza o shell — a tabela só aparece após pesquisa. NÃO replicar a duplicação; unificar listagem.
- `checaDocumento()` retorna contagem só para `grupo_id` 1 ou 2; para os demais sempre `false`. O badge "CPF encontrado em N análises" nunca aparece para imobiliárias mesmo havendo duplicidade — comportamento atual, decidir se mantém.
- `executa_analise()`: quando `parecer_analisepessoa_statusid==4` monta dados de e-mail mas a chamada `$emailControl->emailNotification(...)` está COMENTADA — notificação de "análise concluída" NÃO é enviada hoje. Decisão de produto: reativar ou não.
- Uso de `each()` em `edita_ficha_analise.ctp` (parse de retorno webservice) — função removida no PHP 8; é dívida técnica. NÃO replicar; usar `foreach`.
- `view.ctp` é scaffold CRUD não usado no fluxo (Patrimônios etc.) — tratar como legado morto.
- Cônjuge e sócios inline foram desativados em `add.ctp` (comentados "Euzébio 27/06/2019"); ainda existem em `addcr.ctp`. O cadastro de solidários/sócios passou a ser feito como pessoas relacionadas separadas (links "Novo cadastro" na listagem). NÃO replicar os blocos comentados.
- `requestAction` dentro das views (checaDocumento, mostraPessoasExtras, abreSerasa) gera N+1 de requisições por linha/proponente — antipadrão de performance; na migração, resolver server-side em lote.
- Strings SQL em `pdf_analise_pessoa` usam crases em condições de JOIN com valores literais (ex.: `` `ItensAnalise.id` = `105` ``) — frágil/peculiar do MySQL legado; reescrever com binds.

## Destino (issues Linear)
- Projeto **wecorp-frontend** (Next.js). Domínio "Análise cadastral" (fase F06). Telas a recriar:
  - Gestão de Análises (lista + filtros + cards de proponentes) — equivale `index`/`index_paginator`.
  - Form de criação/edição da ficha (PF/PJ, locatário/fiador) com variante simplificada p/ imobiliária — equivale `add`/`add_user_imobiliaria`.
  - Tela do operador (execução de parecer item-a-item + parecer da pessoa) — equivale `edita_ficha_analise`.
  - Geração/visualização de PDF (Completa/Expressa/Final, white-label) — equivale `pdf_analise_pessoa` (consome serviço backend).
- Vincular ao milestone de Análise Cadastral; back-end correlato em **wecorp-backend** (CRUD ficha/pessoa/itens, status) e **wecorp-services** (integrações Procob/Serasa, geração de PDF, envio de link web/SMS/e-mail).
