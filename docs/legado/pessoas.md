# Legado (telas) — Pessoas (F05)

> As telas de `app/View/Pessoas/*.ctp` são scaffolds gerados por `cake bake` (CRUD genérico): listam/exibem/editam campos crus da entidade `Pessoa` sem agrupamento visual, máscaras ou validação client-side. As "seções" de domínio (identificação/contato/endereço/qualificação/renda/cônjuge/referências) NÃO existem como blocos de UI aqui — são agrupamentos conceituais dos campos da tabela `pessoas` + modelos relacionados (`PessoaQualificacao`, `ReferenciasContato`, `ConjugePessoa`). A captura real desses dados acontece em outro fluxo (FichaAnaliseCadastral) — ver Gaps.

## Cobertura
- Views: `app/View/Pessoas/index.ctp`, `add.ctp`, `edit.ctp`, `view.ctp`
- Controller: `app/Controller/PessoasController.php` (index, view, add, edit, delete, retornaRelacionamento)
- Apoio (somente para entender campos/associações): `app/Model/Pessoa.php`

## Telas / fluxos

### index.ctp — Listagem (`PessoasController::index`)
- Painel "Ações" (col-md-3): link "Nova Pessoa" (`action=add`).
- Tabela (col-md-9, `table table-striped`, paginável 50/pág) com colunas ordenáveis via `Paginator->sort`:
  - `id`, `nome` (concatena `nome` + `nome_empresa`), `empresa_id` (exibe `Empresa.nome`), Status (texto via `$statusPessoaCadastro[$pessoa['Pessoa']['pessoa_status']]`), `tipo_pessoa_id` (exibe `Pessoa.tipo_pessoa`), `documento`, `created` (formatado `Dataformat::datetimeBr`).
  - Coluna ações: ícones view (`glyphicon-info-sign`), edit (`glyphicon-edit`), delete (`Form->postLink` com confirm "Deseja mesmo remover o Registro ID # %s?").
- Paginação custom com labels PT ("Anterior"/"Próximo", contador "Página {:page} de {:pages}...").
- GOTCHA: a coluna Status lê `pessoa_status` mas o form (add/edit) grava `pessoa_status_cadastro_id`. Há divergência de nomes de campo entre telas (ver Gotchas).

### add.ctp / edit.ctp — Formulário (`PessoasController::add` / `::edit`)
- Layout idêntico: painel "Ações" (Listar; edit adiciona "Excluir" via postLink com confirm) + form Bootstrap (`Form->create('Pessoa', role=form)`).
- Campos: todos renderizados como `Form->input(...)` SEM tipo/máscara explícita — o tipo (text/select/date) é inferido do schema do banco pelo CakePHP. `class=form-control`, `placeholder` = label humanizado.
  - `nome`, `empresa_id`, `user_id`, `pessoa_status_cadastro_id`, `tipo_pessoa_id`, `documento`, `data_nascimento`, `origem_cadastros_id`, `pessoa_qualificacao_id`, `conjuge_pessoa_id`, `razao_social`, `email_secundario`, `identidade`, `identidade_orgao_emissor`, `identidade_data_emissao`, `inscricao_municipal`, `inscricao_estadual`, `website`, `pessoa_parecer_final`.
  - `edit.ctp` inclui ainda `id` (input visível — scaffold).
- Selects (`*_id`) populados pelo controller via `find('list')`: `empresas`, `users`, `pessoaStatusCadastros`, `tipoPessoas`, `pessoaStatusCadastro1s`, `origemCadastros`, `pessoaQualificacaos`, `conjugePessoas`. (Variáveis `pessoaStatusCadastro1s` e `conjugePessoas` são setadas mas não usadas nos `.ctp`.)
- Botão submit: "Processar" (`name=data[submit]`, `btn btn-default`). Em edit há um botão "Aplicar" (`data[apply]`) COMENTADO.
- SEM validação JS, SEM máscara (CPF/CNPJ/telefone/data), SEM agrupamento por seção, SEM campos de endereço.

### view.ctp — Detalhe (`PessoasController::view`)
- Painel "Ações": Editar / Excluir(confirm) / Listar / Nova.
- Tabela vertical "th/td": cada linha só é exibida `if(!empty($pessoa['Pessoa'][campo]))`. Campos: id, nome, empresa (link p/ empresas/view), user (link), created, modified, pessoa_status_cadastro_id (link), tipo_pessoa_id (link), documento, data_nascimento, origem_cadastros_id (link), pessoa_qualificacao_id (link), conjuge_pessoa_id, razao_social, email_secundario, identidade, identidade_orgao_emissor, identidade_data_emissao, inscricao_municipal, inscricao_estadual, website, pessoa_parecer_final.
- Bloco "Related Pessoa Qualificacaos" (hasMany na view, embora o Model declare belongsTo): tabela com colunas que revelam os campos do grupo **qualificação/renda**: `naturalidade`, `nacionalidade`, `estado_civil_id`, `sexo`, `filiacao_pai`, `filiacao_mae`, `rendas_extras`, `outras_filiais`, `obs_outras_filiais`, `paga_aluguel`, `favorecido_aluguel`, `favorecido_telefones`, `nome_empresa_trabalha`, `telefone_empresa_trabalha`, `reda`, `pessoa_id`. Ações CRUD apontam para controller `pessoa_qualificacaos`.
- Bloco "Related Referencias Contatos" (grupo **referências/contato**): colunas `nome`, `telefones`, `created`, `modified`, `pessoa_id`, `banco_id`, `agencia`, `obs`, `identidade`, `identidade_orgao_emissor`, `identidade_data_emissao`. CRUD aponta p/ `referencias_contatos`.
- Labels em inglês ("Created", "Modified", "Website", "Related ...") — scaffold não traduzido.

## Pontos de entrada (controller::ação que renderiza)
- `PessoasController::index()` → `Pessoas/index.ctp`
- `PessoasController::view($id)` → `Pessoas/view.ctp` (404 `NotFoundException` se `!Pessoa->exists($id)`)
- `PessoasController::add()` → `Pessoas/add.ctp` (POST salva; sucesso redireciona p/ index)
- `PessoasController::edit($id)` → `Pessoas/edit.ctp` (POST/PUT salva; sucesso redireciona p/ index)
- `PessoasController::delete($id)` → sem view; só POST/DELETE; redireciona p/ index
- `PessoasController::retornaRelacionamento($categoria,$acao)` → AJAX/JSON, sem render (usado por outras telas; retorna lista de Relacionamentos, filtro `locatario_solidario=1` quando `$categoria=='solidario'`).

## Regras de negócio relevantes à UI
- Acesso: `beforeFilter()` chama `VerificaPermissao()` (herdado de AppController). Se `false`: flash de erro + renderiza `/Dashboard/proibido`. `view/add/edit/delete` repetem a checagem e redirecionam p/ index quando negado.
- Escopo de listagem por nível de permissão (`index()`):
  - nível 2: filtra `Pessoa.empresa_resp_id = empresa do usuário logado`.
  - nível 3: filtra `FichaAnaliseCadastral.empresa_parceira_id = empresa` AND `FichaAnaliseCadastral.user_id = usuário` (join implícito — ver gotcha).
  - `grupo_id == 2` (admin empresa parceira): adiciona OR em `Empresa.empresa_resp_seguros / empresa_resp_analises / empresa_resp_vistorias = empresa`.
- Validação do Model `Pessoa` (server-side): apenas `numeric` em `pessoa_status`, `origem_cadastros_id`, `pessoa_qualificacao_id`, `relacionamentos_id`. `nome`/`documento` NÃO são obrigatórios. Validação de `tipo_pessoa` está comentada (desativada).
- Associações relevantes (`Pessoa.php`): belongsTo `Empresa` (FK **`empresa_resp_id`**, não `empresa_id`), `User` (`user_id`), `OrigemCadastros`, `Relacionamentos`; hasMany `ReferenciasContato` (`pessoa_id`), `PessoaExtra` (className `PessoaExtras`).
- `displayField = 'nome'`.

## Máquina de estados / status (refletida na UI)
- Status exibido na listagem vem de `PessoaStatusCadastro::getList()` indexado por `Pessoa.pessoa_status` (inteiro). Os rótulos vêm da tabela `pessoa_status_cadastros`.
- O form grava em `pessoa_status_cadastro_id` (FK) — não há transição de estado controlada na tela; é apenas seleção livre via select. Sem botões de ação de workflow nas telas de Pessoas (o workflow de análise vive em FichaAnaliseCadastral).

## Multi-tenant / white-label
- Multi-tenant por empresa: filtros de `index()` por `empresa_resp_id` / `empresa_parceira_id` conforme nível/grupo do usuário logado (`AuthComponent::user('empresa_id'|'id'|'grupo_id')`).
- Nenhuma customização visual white-label nas telas de Pessoas (usam layout default do dashboard, breadcrumb fixo "Painel / Pessoas").

## Gotchas / decisões kept-bug
- **Código morto em `edit()`**: o bloco `if (isset($this->request->data['apply']))` está APÓS `return $this->redirect(...)` — nunca executa. O botão "Aplicar" também está comentado na view. NÃO replicar.
- **Divergência de nomes de campo**: `index.ctp` lê `pessoa_status`/`tipo_pessoa` enquanto add/edit/view usam `pessoa_status_cadastro_id`/`tipo_pessoa_id`. Indica colunas legadas duplicadas/renomeadas no schema. Confirmar nomes reais na migração.
- **FK Empresa**: form usa `empresa_id`, mas a associação real é `empresa_resp_id`. O `Empresa.empresa_id` mostrado pode não bater com a FK efetiva.
- **`index()` nível 3 / grupo 2** referenciam `FichaAnaliseCadastral.*` e `Empresa.*` em `conditions` sem join/contain explícito declarado em `paginate` — depende de `recursive=0` + associações; risco de erro SQL se a associação não existir. Verificar em runtime.
- **Sem máscara/validação client-side** em CPF/CNPJ, telefone, datas, CEP — campos texto puro. NÃO replicar a ausência: o novo front deve aplicar máscaras e validação por tipo de pessoa (PF/PJ).
- **Scaffold não traduzido**: títulos "Add Pessoa"/"Edit Pessoa", labels "Created/Modified/Website", "Related ...". Substituir por UI agrupada PT-BR.
- **input `id` editável** em edit.ctp (scaffold) — remover no novo front.
- view.ctp itera `$pessoa['PessoaQualificacao']` como hasMany, mas o Model não declara essa associação (declara só belongsTo via `pessoa_qualificacao_id`); bloco pode nunca renderizar. Confirmar relação real PessoaQualificacao↔Pessoa.

## Destino (issues Linear)
- Frontend (wecorp-frontend): tela de cadastro de Pessoa com seções agrupadas — Identificação (nome/razao_social, tipo PF/PJ, documento CPF/CNPJ com máscara, identidade+órgão+data, inscrições municipal/estadual, data_nascimento), Contato (email_secundario, website, telefones), Endereço (a definir — não existe no legado Pessoas; ver Gaps), Qualificação (naturalidade, nacionalidade, estado_civil, sexo, filiação pai/mãe, nome/telefone empresa trabalha), Renda (rendas_extras, paga_aluguel, favorecido_aluguel/telefones, reda), Cônjuge (conjuge_pessoa_id → vincular outra Pessoa), Referências (ReferenciasContato: nome, telefones, banco/agência, identidade, obs).
- Listagem com filtros por empresa/status conforme permissão; máscaras e validação por tipo de pessoa.
</content>
</invoke>
