# Legado (telas) — Empresas

> Cadastro/pré-cadastro de empresas parceiras (imobiliárias/administradoras) em formulário longo único, dividido em seções por `<h4 class="text-primary">`. Blocos inteiros aparecem/somem conforme `grupo_id` do usuário. Inclui busca de endereço por CEP (AJAX backend), upload de logo, contatos dinâmicos (só em edit), vínculo Empresa×Módulos e webhook Buskaza ao salvar.

## Cobertura
- Views: `app/View/Empresas/index.ctp`, `add.ctp` (usada por add **e** edit), `view.ctp`, `empresa_modulo.ctp`, `contatos_carrega.ctp`, `pesquisarelacao.ctp`, `atendimento.ctp`
- Controller: `app/Controller/EmpresasController.php` (`beforeFilter`, `index`, `view`, `atendimento`, `add`, `edit`, `empresa_modulo`, `empresa_dados`, `contatos_carrega`, `contato_novo`, `contato_exclui`, `tabela_analise`, `pesquisarelacao`)
- JS de apoio: `app/webroot/js/modules/empresa.js`, `app/webroot/js/modules/endereco.js`
- NÃO coberto (ignorado por regra): `add-online-24-06-2019.ctp` (backup datado)
- Relacionado (CRUD scaffold à parte): `app/View/Parceiros/{index,add,edit,view}.ctp` + `ParceirosController` — registro de **Parceiros** vinculado a Empresa (`Parceiro belongsTo Empresa`; colunas `nome`, `razao_social`, `documento`, `email`, `empresa_id`). Molde "Manutenção"; detalhado em `configuracoes.md`.

## Telas / fluxos

### index.ctp — Listagem + filtros
- Acordeão "Filtros e pesquisas" (componente `Search`): `empresa_id` (Código), `empresa_nome` (Nome/Razão social), `empresa_documento` (CPF/CNPJ), `status_empresa` (select de metadados). Botão "OK".
- Tabela paginada (50/pág, `order id DESC`): colunas ID, Nome, Razão social, Documento (formatado via `Dataformat::formatarCnpj`/`formatarCpf` conforme `tipo_pessoa`), Ações.
- Ações por linha: Visualizar (`view`), Editar (`edit`), Anexos (`arquivosAnexos(...)` JS), Empresa×Integradores×Produtos / ×SubProdutos (controller `prospectscotacoes/empresasprodutos`), Empresa×Módulos (`empresa_modulo`), Follow-up (`janelaFollowups(...)`).
- Atalhos laterais: Novo Cadastro/Pré-cadastro (`add`), Listar (`index`).

### add.ctp — Cadastro/Pré-cadastro e Edição (mesma view)
A view é renderizada tanto por `add()` quanto por `edit()` (este faz `$this->render('add')`). Comportamento muda por `$this->params['action']`:
- `action == 'add'`: exibe aviso "Após o cadastro... pode incluir contatos adicionais"; campo `status_empresa_cadastro` em modo consulta para grupos não-admin.
- `action == 'edit'`: exibe campo hidden `empresa_id`, acordeão **"Contatos do cadastro"** (tabela dinâmica + form "Novo contato"), e campo `created` (somente leitura, formatado `Dataformat::dateBr`).
- `$requerido` é calculado (`''` em edit, `'required'` em add) mas **não é usado** nos inputs — kept-bug irrelevante. `$senha_empresa` é sempre forçado a `''` (linha 18).

Form: `Form->create('Empresa', [... 'type'=>'file'])` com `parsley-validate`/`novalidate`. Submit único "Processar" (`name=data[submit]`).

Seções (ordem) e campos principais:
1. **Dados de Cadastro/Pré-Cadastro**: `tipo_pessoa` (select PJ/PF, `required`), `documento` (maxlength 14, classe `documento` p/ máscara JS CPF/CNPJ), `nome` (Nome corretor/fantasia, `required`), `razao_social` (dentro de `#div_dados_pj`), `contato_nome`, `contato_email`, `contato_telefone` (mask `telefone`), `contato_celular` (mask `mobile`).
2. **Bloco restrito a grupos 1,2,3,4,9** (abre em `data_constituicao`): `data_constituicao` (datepicker), `creci_jur`, `creci_pf`, `creci_pf_titular` + `creci_pf_titular_cpf` (em `#div_dados_pf`), `telefone_principal` (mask telefone), `telefone_outros`, `email_principal` (type email, `required`), `email_secundario` (type email), `inscricao_municipal`. Sub-bloco `#div_adm_condominio`: `adm_condo_razao_social`, `adm_condo_cnpj` (mask `cnpj`).
   - **Overview**: `quantas_filiais`, `numero_funcionarios`, `realiza_adm` (select Sim/Não), `carteira_imoveis`, `valor_medio_alugueis` (MascaraMoeda), `media_mensal_locacao`, `qtd_unidades_vacancia`, `qtd_unidades_condominiais`, `qtd_condominios_adm` (números com max/min).
   - **Configurações/parâmetros gerais**: `percentual_iss` (MascaraMoeda; usado no cadastro Tokio/incêndio).
   - **Endereço comercial/principal**: `cep` (maxlength 11, botão lupa → `searchAddressByCep`), `endereco` (readonly), `endereco_numero`, `endereco_complemento`, `bairro_id` (select readonly), `cidade_id` (select, onChange `buscaBairros`), `uf` (select, onChange `buscaCidades`).
   - **Responsável administrativo**: `nome_responsavel` (req), `telefone_responsavel` (req, mask telefone), `telefone_responsavel_celular` (mask mobile), `email_responsavel` (req).
   - **Relacionamento comercial/financeiro**: `nome_resp_financeiro` (req), `telefone_responsavel_financeiro` (req, mask telefone), `email_financeiro` (req).
   - **Relacionamento comercial - Assessoria**: `modalidade_pagamento` (select metadados), `dia_vencimento_fatura` (select 10/15/20), `pagamento_faturado` (checkbox).
   - **Customizações e Notificações**: checkboxes `flag_permite_autocadastros` (+ `dominio_corporativo`), `flag_exibe_logo_analise`, `flag_permite_cobrar_analise`, `flag_vistoria_layout`, `flag_vistoria_autonoma`, `flag_vistoria_demo` (toggle exibe `#data_limite_demo_vistoria` via JS) + `vistoria_demo_data_limite` (datepicker). Textareas `emails_notifica_seguros`, `emails_notifica_followups`, `emails_notifica_sinistros` (CSV de e-mails, com tooltips explicativos).
   - **Financeiro seguros**: `fatura_imob` (select Sim/Não); `forma_pagto_faturas_seguros` e `qtd_parcelamento` existem mas estão `display:none`.
   - **Conta corrente p/ transferência de indenizações** (`containd_*`) e **p/ repasse de comissões** (`contarep_*`): tipo, banco (select), agência, c/c, favorecido, documento (alguns com mask cnpj).
3. **Integrações/Webservices** (visível a todos): `token_empresa` (req; gerado por `getToken(8)` no add, lido do banco no edit), `integrador_id` (select; default = integrador da empresa), `codigo_corp` (infoCORP), `login_docusign`/`senha_docusign` (ambos `disabled`; checkbox `#edita_docusign` habilita via JS), `ativa_clicksign` (checkbox).
4. **Logotipo** (restrito a grupos 1,2,3,4,9): preview `<img>` se `url_logotipo`, input `upload` (type file), checkbox `remover_logo`. Obs: máx 200x60px (apenas texto informativo, não validado).
5. **Sobre a parceria**: `obs_perfil_seguros` (textarea).
6. **Regras/obs da parceria nos serviços** (somente grupos 1 e 2): `condicoes_seguro_fianca`, `condicoes_titulo`, `condicoes_seguro_incendio`, `condicoes_analise` (textareas).
7. Rodapé: `status_empresa` (select metadados, `required`), `motivo_bloqueio` (textarea), `origem_cadastro_id` (select).
8. **Regras para atendimentos** (somente `grupo_id == 1`): `empresa_resp_seguros`, `empresa_resp_analises`, `empresa_resp_vistorias` (selects de empresas associadas, default = empresa do logado); **Relacionamento na Plataforma**: checkboxes `empresa_parceira`, `empresa_associada`, `fornece_seguros`, `fornece_analises`, `fornece_vistorias`, e `codigo_susep`.
   - Para `add` + grupo≠1: campos `empresa_resp_*` são gravados como **hidden** com base nas flags `fornece_*`/`empresa_associada` da empresa logada (lógica em add.ctp linhas 849-875).
9. JS carregados: `modules/empresa.js`, `modules/endereco.js`, `jquery-ui`.

### view.ctp — Visualização (modo consulta)
- Subconjunto reduzido do formulário ("Alterações permitidas somente ao Administrador"). Mesmos campos de endereço/máscaras. Inclui `taxa_administracao`, `condicoes_gerais` (campos que **não existem** no add.ctp). Blocos de relacionamento comercial e condições só para grupos 1/2. Form com `action=>false`. Apesar do título "consulta", os inputs não são `disabled` — é apenas uma view alternativa de leitura.

### empresa_modulo.ctp — Empresa × Módulos
- Lista todos os `modulos` como checkboxes (`data[Empresamodulo][N][modulo_id]`), marcados se `in_array(id, $modulos_empresa)`. Mostra nome da empresa no topo. Submit "Processar".

### contatos_carrega.ctp — Tabela de contatos (AJAX, embutida no edit)
- Renderizada via `carregaEmpresaContatos()` em `#tabela_contatos`. Tabela: Nome, Email, Telefone, Cargo/Função, Obs + ação Excluir (com `confirm()`). Há um form oculto "Alteração de Sócio" por linha (campos documento, identidade, naturalidade, cargo, %) mas o botão de abrir esse form de alteração **não está exposto na tabela** (só o de excluir) — código de edição de sócio presente porém inacessível pela UI.

### pesquisarelacao.ctp — Modal de busca p/ assinatura eletrônica (O2 Sign)
- Layout `modal`. Form de pesquisa: `id`, `tipo_busca` (select: 1=Vistoria locatícia, 2=Título capitalização), `titulo_documento`, `documento`, `endereco`. Busca AJAX GET para `/empresas/pesquisarelacao/<origem>/0/<ajax>/<janela>`, resultado injetado em `#paginator`. Tabela de resultados com colunas dependentes de `tipo_busca`; ação "Selecionar registro" chama `selecionarItemSign(...)` + `closeModal()`.

### atendimento.ctp
- Wrapper que apenas renderiza o element `Painel.atendimento_imobiliarias`. Sem campos próprios.

## Pontos de entrada (controller::ação que renderiza)
- `EmpresasController::index()` → `index.ctp` (filtros via POST salvos em sessão `conditions_post`).
- `EmpresasController::view($id)` → `view.ctp` (via `$this->render('view')`).
- `EmpresasController::add()` → `add.ctp`.
- `EmpresasController::edit($id)` → `add.ctp` (via `$this->render('add')`).
- `EmpresasController::empresa_modulo($id)` → `empresa_modulo.ctp`.
- `EmpresasController::contatos_carrega($empresa_id)` → `contatos_carrega.ctp` (layout null, AJAX).
- `EmpresasController::contato_novo()` / `contato_exclui($id)` → JSON/no-render (chamados por `empresa.js`).
- `EmpresasController::pesquisarelacao(...)` → `pesquisarelacao.ctp` (layout modal).
- `EmpresasController::atendimento($id)` → `atendimento.ctp`.

## Regras de negócio relevantes à UI
- **Máscaras** (data-attrs aplicadas por lib global): `telefone`, `mobile`, `cnpj`. Moeda via `onkeypress=MascaraMoeda(this,".",",",event)`. Datas via classe `datepicker`/`date`.
- **CEP/Endereço**: `searchAddressByCep()` (endereco.js) → GET `/enderecos/api_get_address_by_cep/?cep=`. Sucesso preenche endereço/uf e injeta options em bairro/cidade; falha → `alert` "CEP não existe" e libera campos (remove readonly). `buscaCidades`→`/enderecos/api_get_cities/<uf>`; `buscaBairros`→`/enderecos/api_get_regions/<cidade>`.
- **add()**: converte `valor_medio_alugueis`/`percentual_iss` (currency→decimal), `vistoria_demo_data_limite` (BR→SQL); seta `empresa_master` = empresa do logado, `responsavel_user_id` = user logado; se `razao_social` vazia, copia `nome`; se `status_empresa` vazio → **4 (Pré-cadastro)**; resolve `bairro_nome`/`cidade_nome`; cria vínculo `WbsEmpresasIntegradores`; dispara e-mail `empresa_novocadastro`; **redireciona para `edit`** após salvar (sucesso/erro via flash).
- **edit()**: remove logo (`unlink` do arquivo + limpa `url_logotipo`) se `remover_logo==1`; upload de logo aceita só jpg/jpeg/gif/png movido p/ `imagens/logos_imobiliarias/`; se `empresa_senha_nova` preenchida, gera hash SHA1; salva vínculo integrador; **se `integrador_id==10` (Buskaza Pro)** envia webhook POST p/ `https://staging.buskaza.com.br/api/webhooks/hubstate/<codigo_referencia_cliente>` com dados da empresa; redireciona para `edit` novamente.
- **Contatos** (edit): `contato_novo()` grava em `Empresacontatos` (campos nome, cargo_funcao, email, telefone, obs, user_id, empresa_id); `contato_exclui($id)` faz delete; lista recarregada por `carregaEmpresaContatos()`.
- **empresa_modulo()**: ao salvar, faz `deleteAll` dos módulos da empresa e re-insere os marcados. Mesmo sem nenhum módulo marcado, executa deleteAll (zera vínculos). Flash messages com texto "permissiongroup" (copy/paste de outra tela — kept-bug de copy).

## Máquina de estados / status (refletida na UI)
- `status_empresa`: vem de metadados `Libmetadados::get('status_empresa')`. Valor **4 = Pré-cadastro** é o default quando não informado.
- Para grupos **não** internos (≠1,2,3,4,9) no `add()`: `status_empresa` é restrito a `array('4'=>'Pré-cadastro')` — só conseguem criar pré-cadastros. Esses grupos veem o status apenas em campo `status_empresa_cadastro` **disabled** (modo consulta).
- `motivo_bloqueio` é o campo livre associado a status de bloqueio.

## Multi-tenant / white-label
- **Permissão**: `beforeFilter` chama `VerificaPermissao()`; se falso (e ação ≠ `pesquisarelacao`/`atendimento`) redireciona para `Dashboard/proibido`.
- **Visibilidade por grupo** (`AuthComponent::user('grupo_id')`): 1 = SuperAdmin (vê tudo, incl. "Regras para atendimentos", flags fornece_*, codigo_susep); 2 = Admin empresa parceira; 3/4/9 = perfis internos. Blocos financeiros/config/logo gated em `grupo_id ∈ {1,2,3,4,9}`; condições de parceria só em `{1,2}`.
- **Escopo de dados**: `index`/`view`/`edit` filtram por `Empresa.id`/`Empresa.empresa_master` = empresa do logado quando `permissao['nivel']==2`; grupo 2 também enxerga empresas onde é `empresa_resp_seguros/analises/vistorias`. `index` sempre exclui `isfornecedor=1`.
- **White-label**: flags `flag_exibe_logo_analise`, `flag_vistoria_layout` (identidade visual nos laudos), logo (`url_logotipo`), `dominio_corporativo` + `flag_permite_autocadastros` (auto-cadastro por e-mail corporativo, perfil OPERADOR).
- **Integradores**: select `integrador_id` restrito ao integrador da empresa para grupos não-internos; fallback p/ integrador padrão.

## Gotchas / decisões kept-bug
- `add.ctp` é compartilhada por add e edit; muita lógica condicional por `$this->params['action']`. **Não replicar como tela única** — separar Add/Edit.
- `$requerido` calculado mas nunca aplicado; `$senha_empresa` sempre zerado. Variáveis mortas — não replicar.
- Flash de `empresa_modulo()` diz "permissiongroup" (texto errado herdado).
- `view.ctp` referencia `taxa_administracao`/`condicoes_gerais` que não estão no add — divergência de schema entre telas; confirmar campos reais no backend.
- Edição de sócio em `contatos_carrega.ctp` tem form completo porém sem botão para abri-lo (código órfão).
- Webhook Buskaza aponta para **staging** com token hardcoded e `action` fixo `'activated'` ignorando o `status_empresa` calculado — NÃO replicar; mover p/ config/serviço.
- Upload de logo não valida dimensão (200x60 é só texto); `unlink` sem checar existência do arquivo pode gerar warning.
- `forma_pagto_faturas_seguros` e `qtd_parcelamento` ficam `display:none` (campos legados ocultos).
- `pesquisarelacao` tem grandes blocos de permissão comentados — atualmente sem filtro de escopo real para grupos não-internos (potencial vazamento; revisar na migração).

## Destino (issues Linear)
- Fase **F04 — Empresas (telas)** no projeto **wecorp-frontend**. Telas a recriar: Listagem+filtros, Formulário Add, Formulário Edit (com abas/seções e contatos dinâmicos), Visualização, Empresa×Módulos. Integrações de apoio (CEP, máscaras, upload logo, webhook Buskaza) referenciam wecorp-backend/services.
</content>
</invoke>
