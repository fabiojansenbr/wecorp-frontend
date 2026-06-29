# Legado (telas) — Usuários e autocadastro

> Telas de gestão de usuários do legado CakePHP (AlugueSeguro). Dois fluxos paralelos: o "completo" página inteira (`index`/`add`/`edit`) usado por Admin da plataforma/parceiras, e o "modal AJAX" (`lista`/`novo`) embutido em outras telas (cadastro de empresa). O comportamento dos formulários (campos visíveis, grupos selecionáveis, empresas selecionáveis) muda fortemente conforme o `grupo_id` do usuário logado.

## Cobertura
- Views: `app/View/Users/index.ctp`, `app/View/Users/add.ctp` (usada por `add` E `edit`), `app/View/Users/lista.ctp`, `app/View/Users/novo.ctp`, `app/View/Users/view.ctp`, `app/View/Users/usuarios_por_empresa.ctp`, `app/View/Users/autocadastro.ctp` (órfã — ver Gotchas).
- Controller: `app/Controller/UsersController.php` métodos `index`, `add`, `edit`, `lista`, `novo`, `usuarios_por_empresa`, `beforeFilter`, `hashPassword`.
- Colaboradores: `AppController::VerificaPermissao`, `AppController::getToken`, `app/Metadata/Libmetadados.php` (`statusCadastro`, `tipoCreci`).
- NÃO coberto (fora do escopo telas-usuários): `login`, `login_api`, `login_token`, `termos_uso*`, `captcha`, `dashboard`, `block`, `dados_usuarios*`, `RecuperaEmpresaResponsavel`.

## Telas / fluxos

### `index.ctp` — Listagem completa (página inteira)
- Renderizada por `UsersController::index`. Layout padrão (com breadcrumb Painel > Usuários).
- Sidebar "Ações": links Novo Usuário (`/users/add`) e Listar Usuários (`/users/index`).
- Painel de filtros (CakeDC Search Plugin, `$this->Search`): Código (`usuario_id`), Nome (`usuario_nome`), Login (`usuario_login`), E-mail (`usuario_email`), Grupo (`usuario_grupo_id`, select de `$grupos`), Status (`usuario_status`, select de `$statusCadastro`).
- Filtro extra **Imobiliária/Empresa** (input texto autocomplete `#filtro_empresa_id`, name `data[User][usuario_empresa_id]`) só aparece para `grupo_id` ∈ {1,2,3,4,9}.
- Tabela: ID, Nome, Empresa (`$user['Empresa']['nome']`), Grupo (`$grupos[grupo_id]`), Login, Status (`$statusCadastro[status]`), Data (`Dataformat::datetimeBr(created)`), ação Editar (link p/ `edit`).
- Paginação 50/página, ordem `User.id DESC`.

### `add.ctp` — Cadastro/Edição completa (página inteira)
- Renderizada por `UsersController::add` E `UsersController::edit` (edit faz `$this->render('add')` no final). É o formulário mais rico.
- Campos: `empresa_id` (select2, required), `grupo_id` (select required `$grupos`), `name` (required), `apelido` (required), `email`, `celular_corporativo` (required, `data-mask="mobile"`), `telefones`, `funcao`/Cargo (required), `tipo_pessoa` (PF/PJ, required), `documento`/CPF (required, `maxlength=11`, "só números"), `creci` (`maxlength=12`), `tipo_creci` (Estagiário/Definitivo), `validade_creci` (datepicker, `div #creci_data_validade` começa `display:none`), `experiencia_corretor`, `tempo_experiencia_corretor`, `area_atuacao`, `realizou_curso` (Sim/Não), `username`/Login (required), `password_new` (type password), `password_app_vistoria` (senha do App de Vistoria), `status` (required, `$statusCadastro`), 4 campos de notificação por e-mail CSV (`notifica_emails_seguros`, `_analise`, `_vistorias`, `_sign`), `email_boas_vindas` (Sim/Não — dispara e-mail de ativação).
- `empresa_id`: para grupos {1,2,3,4} é `required select2`; demais grupos recebe a versão sem `required` na classe (mas ainda com atributo `required`). Lista de empresas (`$empresas`) é filtrada por grupo no controller.
- Botão "Documentos/arquivos anexados" (`arquivosAnexos('Users', $id, ...)`) só aparece quando `$this->params['action']=='edit'`.
- Submit: `data[submit]` "Processar". Aviso: "Para não alterar a senha, deixe em branco."
- JS de máscaras/select2 em `js/modules/users/ui.js`.

### `novo.ctp` — Cadastro rápido em MODAL (AJAX)
- Renderizada por `UsersController::novo` quando aberta via `janelaUsuarios(model, fk, empresaparceira_id, '1', 'novo', '0')`.
- Form com `action='novo/{model}/{fk}/{empresaparceira_id}/1'`, id `User`, submetido via `jQuery.ajaxForm()` com barra de progresso.
- Campos (reduzido): `empresa_id` (select2; bloco fica `display:none` exceto p/ grupos {1,2,3,4}), `grupo_id` (`$groups`), `name` (label "Nome (*)"), `email`, `celular_corporativo` (`data-mask="mobile"`), `telefones`, `funcao` ("Cargo/Função (*)"), `status` (`$statusCadastro`).
- **Validação client-side** (JS no próprio .ctp, handler `#ProcessaCadastro`): bloqueia se `name` vazio, se `funcao` vazia, se `grupo_id` vazio. Se `username` (`#UserUsername`) vazio, copia o nome para o login automaticamente.
- Sucesso: backend retorna `true`/`1`; alert "Cadastro de usuário realizado com sucesso!" e reabre a listagem modal `janelaUsuarios(..., 'lista')`. Caso contrário alert genérico de erro.
- Nota de rodapé: para definir login/senha use Administração > Usuários (o modal não cadastra senha aqui — campo password não existe no form, apesar de `novo()` tratar `password_new` se enviado).

### `lista.ctp` — Listagem em MODAL (AJAX)
- Renderizada por `UsersController::lista`. Botão "Novo Usuário" chama `janelaUsuarios(model, fk, empresaparceira_id, '1', 'novo', '0')`.
- Mesmos filtros de `index.ctp` (painel começa `display:none`). Filtro Imobiliária/Empresa idem grupos {1,2,3,4,9}.
- Tabela: ID, Nome, Empresa, Grupo, Login, Status. Coluna de ações **escondida** (`<span style="display:none">` com links edit/ok) — na prática a listagem modal não permite editar inline.
- Paginação 15/página, ordem `User.id DESC`.

### `view.ctp` — Detalhe (template scaffold/legado)
- View de scaffold básica do bake (dl/dt/dd). Referencia `$user['User']['group_id']` (campo **group_id**, não `grupo_id`) e ações Edit/Delete/List/New. Aparentemente não usada no fluxo real (não há action `view` no controller). Tratar como morta.

### `usuarios_por_empresa.ctp` + `UsersController::usuarios_por_empresa`
- Endpoint auxiliar (`layout='modal'`) que retorna `find('list')` de usuários por `empresa_id` (e opcionalmente `grupo_id`). Usado para popular selects dependentes em outras telas. View tem 3 linhas (só `compact('users')`).

## Pontos de entrada (controller::ação que renderiza)
- `UsersController::index` → `index.ctp` (listagem página inteira).
- `UsersController::add` → `add.ctp` (novo, página inteira).
- `UsersController::edit($id)` → `add.ctp` via `$this->render('add')` (edição reaproveita o form de add).
- `UsersController::lista($model,$fk,$empresaparceira_id,$ajax)` → `lista.ctp` (modal AJAX; quando `$ajax=='1'` usa `ajaxRequest()->jsonResponse()->notRender()` + `autoRender=false`).
- `UsersController::novo(...)` → `novo.ctp` (modal AJAX cadastro).
- `UsersController::usuarios_por_empresa($empresa_id,$grupo_id)` → `usuarios_por_empresa.ctp`.

## Regras de negócio relevantes à UI

### Validação grupo × empresa (server-side, `add`/`edit`)
1. **Anti-escalonamento Admin de Imobiliária**: se logado é `grupo_id==6` e tenta criar/editar usuário em grupo fora de {6,7,8,10} → `die('Você não pode adicionar/alterar um usuário em um grupo de Administrador!!!')` (em `add`) / `die(...)` (em `edit`). Resposta crua, sem flash.
2. **Anti-escalonamento SuperAdmin**: se logado `grupo_id <> 1` e tenta setar `grupo_id==1` → flash de erro "...você está tentando burlar o sistema!!! Você poderá ser banido!" + redirect `index`. (TODO no código: enviar e-mail de notificação — não implementado.)
3. **Admin só de empresa fornecedora**: se a empresa-alvo não fornece Seguros nem Análises nem Vistorias (e em `edit` também não é `empresa_integradora`) e o `grupo_id` alvo é 2 ou 1 → flash "Você não pode transformar um usuário em Administrador de uma empresa/cadastro que não é um fornecedor..." + redirect `index`.
4. **Unicidade login/e-mail**: `add` rejeita se já existe `username` ou `email` (flash de erro, NÃO salva). Em `edit` a checagem é `username` AND `username <> username` — condição contraditória que sempre retorna vazio → **a verificação de duplicidade nunca dispara no edit** (kept-bug, ver Gotchas).
5. **Senhas**: `password_new` → `hashPassword` (SimplePasswordHasher) gravado em `password`. `password_app_vistoria` → `cryptPassword` gravado em `password_app`. Em branco = mantém senha atual. `view`/`add` removem `password` do array carregado no edit (`unset`).
6. **Token**: `add` gera `token` e `token_usuario` via `getToken(10)`.
7. **E-mail de boas-vindas**: se `email_boas_vindas==1`, dispara `EmailController::emailNotification('user','usuario_bemvindo', ...)` após salvar (em add e edit).
8. **validade_creci**: convertida `Dataformat::dateToSql` ao salvar e `Dataformat::dateBr` ao exibir.

### Grupos selecionáveis no form, por grupo logado
O conjunto de grupos (`$grupos`/`$groups`) e empresas (`$empresas`) oferecidos depende do `grupo_id` logado:
- **index / add / edit**:
  - `1` (SuperAdmin): todos os grupos; todas as empresas.
  - `2`: grupos {2,3,4,5,6,7,8,10}; empresas vinculadas (resp_seguros/analises/vistorias/master).
  - `3/4/9`: grupos {3,4,5,6,7,8,10} (em index: {2,3,4,5,6,7,8,10}); empresas vinculadas.
  - `6/8`: grupos {6,7,8,10}; empresas vinculadas.
  - default: grupos {5,6,7,8,10}.
- **novo (modal)**: `1`→todos; `6`→{6,7,8}; `5`→{5}; default→{6,7,8}.
- **lista (modal)**: `1`→todos; `2`→{6,7,8,10}; `5`→{5}; default→{6,7,8,10}.
- Mapa de grupos: tabela `grupos` (id 1..10). Nomes não estão no dump (`aluguese_hml.sql` só tem estrutura). Inferência pelo uso: 1=SuperAdmin plataforma, 2=Admin parceira/fornecedor, 3/4/9=perfis administrativos plataforma, 5=corretor/autônomo, 6=Admin Imobiliária(cliente), 7/8/10=sub-usuários da imobiliária. **Confirmar nomes na migração** (gap).

### Visibilidade de campos por grupo (UI)
- Bloco `empresa_id` visível só p/ grupos {1,2,3,4} em `novo.ctp`/`add.ctp`; demais ficam `display:none` (mas ainda enviado).
- `novo.ctp` esconde empresa para não-admin (`else` com `display:none`).
- `novo.ctp` (edit-style em `autocadastro.ctp`) condiciona empresa_id+grupo_id a `grupo_id <> 6`.
- Filtro Imobiliária/Empresa nas listagens só p/ {1,2,3,4,9}.

## Máquina de estados / status (refletida na UI)
- `status` do usuário vem de `Libmetadados::statusCadastro`:
  - `1` Ativo, `2` Inativo, `3` Bloqueado, `4` Pendente, `5` Alugado.
- Exibido como coluna Status nas listagens e como select no form (add/edit/novo). `lista.ctp`/`index.ctp` traduzem via `$statusCadastro[$user['User']['status']]`.
- `block($status,$id)` (action existente, fora das telas) altera status — provavelmente liga ao botão "ok" oculto na `lista.ctp`.
- `tipoCreci`: `1` Estagiário, `2` Definitivo. `tipo_pessoa`: PF/PJ. `realizou_curso`/`email_boas_vindas`: Sim(1)/Não(0).

## Multi-tenant / white-label
- Escopo por empresa via `VerificaPermissao()` que retorna `nivel` (1/2/3):
  - `nivel==2`: filtra `User.empresa_id == empresa logada` (em index também `Empresa.empresa_master == empresa`).
  - `nivel==3`: filtra por empresa + `FichaAnaliseCadastral.user_id == id logado` (usuário só vê o que está ligado a ele).
  - grupo `2` em index também restringe a empresas onde é responsável por seguros/análises/vistorias e exclui usuários grupo 1.
- `permissao` vem de query em `permissiongroups`/`modulecontrolleractions` por `group_id`+controller+action. Se vazio → sem permissão (index/add/edit redirecionam para `Dashboard/proibido`).
- Empresas selecionáveis no form sempre filtradas por vínculo (resp_seguros/resp_analises/resp_vistorias/empresa_master) exceto SuperAdmin.
- `empresaparceira_id` (param do modal) restringe a lista de empresas no `novo` a uma única empresa.

## Gotchas / decisões kept-bug
- **NÃO replicar**: `die('...')` cru nas validações anti-escalonamento (`add`/`edit`/`novo`) — vaza HTML/mensagem sem layout. Migrar para erro 403/validação estruturada.
- **Bug — duplicidade no edit nunca verifica**: condição `array('username' => $x, 'username <> ' => $x)` (e idem email) é auto-contraditória → sempre vazio. No legado o edit não bloqueia login/e-mail duplicado. **Decidir na migração** se replica ou corrige (recomendado corrigir).
- **`view.ctp` usa `group_id`** (não `grupo_id`) — campo inexistente no schema atual; view morta de scaffold.
- **`autocadastro.ctp` é órfã**: existe a view mas NENHUMA action a renderiza (grep "autocadastro" só acha `Empresas/add.ctp`). Conteúdo é um form "Gerenciar Usuário" (empresa_id/grupo_id/name/email/username/password_new/status, condicionado a `grupo_id<>6`, action `index`). O "autocadastro" real de proponente está em `Empresas/add.ctp` / actions `addproponente`/`clienteproponente` (liberadas no `beforeFilter`) — fora desta tela. **Gap: confirmar onde fica o autocadastro citado no escopo.**
- **`novo.ctp`** trata `password_new` no controller mas o form não tem campo de senha → senha nunca definida por aqui (intencional; rodapé orienta usar Administração).
- **`add.ctp` empresa_id**: passa `required` no atributo mas remove `required` da classe para não-admin — inconsistência de obrigatoriedade visual vs HTML5.
- `index`/`lista` gravam filtros em sessão (`conditions_post`) mas a paginação usa `url` apontando para `controller=posts/action=search` (provável copy-paste) — paginação com filtros pode comportar-se de forma estranha; validar na migração.
- Listas usam `$this->User->recursive = 1` para trazer `Empresa.nome` — atenção a N+1 ao migrar.

## Destino (issues Linear)
- Projeto **wecorp-frontend** (Next.js): telas Usuários (lista com filtros + paginação, form criar/editar com campos condicionais por papel, modal de cadastro rápido). Fase F03.
- Backend correlato (**wecorp-backend**, NestJS): endpoints de listagem com escopo multi-tenant por papel, criação/edição com validações grupo×empresa (regras 1–8), unicidade login/e-mail (corrigir bug do edit), e-mail boas-vindas.
- Verificar milestones de "Usuários/Autocadastro" (HUB-116..HUB-382) — mapear esta tela à milestone correspondente antes de criar issues.
