# Legado (telas) — Login, Sessão e Permissões (white-label)

> Tela única de login (`Users/login.ctp`) renderizada dentro de layouts white-label escolhidos por domínio (`HTTP_HOST`). Apenas `username` + `password` (sem captcha ativo e sem "lembrar-me"). Pós-login grava ~15 chaves de sessão (tokens, empresa responsável, módulos fornecidos/contratados), valida status da empresa e EULA (só grupo 2), e redireciona por grupo (6/8 vão a painéis de imobiliária). Permissões são por grupo via tabelas RBAC (`permissiongroups`).

## Cobertura
- Views: `app/View/Users/login.ctp`, `app/View/Users/termos_uso_eula.ctp`
- Layouts: `app/View/Layouts/login_layout.ctp`, `app/View/Layouts/login_layout_buskaza.ctp` (e variantes `login_layout_completto.ctp`, `login_layout_localhost.ctp` — mesma estrutura, branding diferente)
- Element: `app/View/Elements/termos_uso_eula.ctp` (conteúdo do termo; não lido em detalhe)
- Controller: `UsersController::login()`, `::logout()`, `::termos_uso_eula()`, `::beforeFilter()`, `::captcha()`
- App-wide: `AppController::beforeFilter()` (config Auth, gate de EULA, redirect por grupo), `::beforeRender()` (variáveis de sessão para a view), `::VerificaPermissao()`, `::VerificaPermissaoMenu()`

## Telas / fluxos

### `Users/login.ctp` (tela de login)
- **Campos** (form `User` via `FormHelper`, POST para `/users/login`):
  - `username` — texto, `class="form-control lowercase"`, placeholder "Login". Label "Login".
  - `password` — password, `class="form-control lowercase"`, placeholder "Senha". Label "Senha".
  - Botão submit "Continuar" (`btn btn-primary w-100`).
- **Sem campo captcha** e **sem checkbox "lembrar-me"** na tela (ver Gotchas).
- **Máscaras/validações de cliente**: nenhuma validação JS própria; a classe `lowercase` é apenas estilística/CSS (não há JS que force lowercase nesta view). Validação real é server-side (Auth + scope `User.status = 1`).
- **Branding dinâmico no topo da própria view**: a `.ctp` repete a detecção de `HTTP_HOST` (mesma lógica do controller, mas com pequenas divergências de conteúdo) para definir `$logo_plataforma`, `$logo_tamanho`, `$titulo_solucoes`, `$frase_solucoes`, `$texto_solicite_apresentacao`, `$suporte_atendimento`. Marcas reconhecidas: `localhost`, `wecorp`, `alugueseguro`, `hubbpro`, `hubstate`, `buskaza`, `upgradegarantia`, `esteiratech`.
- **Imagem de fundo** fixa: `imagens/work.png`.
- **Flash messages** exibidas via `$this->Session->flash()` (erro de autenticação, empresa inativa, etc.).
- **Scripts**: carrega jQuery (Google CDN + `js/jquery.js` no layout), `main.js`, `popper`, `owl.carousel`. Há um handler `.creload` (recarrega imagem de captcha) — **resíduo morto**, pois não existe `<img class="imgreload">` na tela.

### `Users/termos_uso_eula.ctp` (aceite de EULA)
- Layout `login_layout`. Exibido **somente após login** quando o usuário é grupo 2 e ainda não aceitou (`flag_aceita_eula != 1`).
- **Campos**: `<textarea readonly>` com o texto do termo (element `termos_uso_eula`) + checkbox `data[User][aceitou_termos]` (value=1, `class="required"`). Botão "Continuar".
- **Validação**: server-side em `termos_uso_eula()` — se `aceitou_termos` vazio ou `!= 1`, flash de erro e re-exibe. Se aceito, grava `UserTermos.data_aceitou_eula` (now) e `flag_aceita_eula=1`, e **redireciona para `/users/login`** pedindo novo login (não entra direto).

## Pontos de entrada (controller::ação que renderiza)
- `UsersController::login()` → renderiza `Users/login.ctp` em layout white-label. Liberada em `beforeFilter` via `Auth->allow('login', ...)`.
- `UsersController::termos_uso_eula()` → renderiza `Users/termos_uso_eula.ctp` (layout `login_layout`). Também liberada no allow.
- `UsersController::logout()` → `redirect($this->Auth->logout())` (volta à `loginAction` padrão do Cake = `/users/login`).
- `UsersController::captcha()` → gera imagem captcha (component `Captcha.Captcha`), layout `ajax`. **Não usado pela tela atual** (captcha desativado).
- Login por API (não-tela): `login_api()` (JWT), `login_token()` — fora do escopo de UI, mas compartilham `Auth->login()` e scope `User.status=1`.

## Regras de negócio relevantes à UI
- **Autenticação** (`AppController::beforeFilter`): `Auth->authenticate = ['Form' => fields username/password, userModel User, scope User.status=1], ['BzUtils.JwtToken' ...]`. Logo, **usuário com `status != 1` não loga** (cai no flash genérico de "Erro na autenticação").
- **Pós-login bem-sucedido** (`login()` após `Auth->login()`):
  1. Carrega `Empresa` do usuário (status, tokens, flags master/associada/parceira, fornece_*, empresa_resp_*).
  2. Gera `token_usuario` via `getToken(10)` e grava várias chaves de sessão (ver "Multi-tenant").
  3. `RecuperaEmpresaResponsavel()`: busca empresas responsáveis por seguros/análises/vistorias (LEFT JOINs) → grava `EmpRespSeguros/Analises/Vistorias`.
  4. Carrega integrador padrão da imobiliária (`Painel.WbsEmpresasIntegradores` → `WbsIntegradores`) → `Auth.User.codigo_integrador`.
  5. `UPDATE users SET num_acessos=num_acessos+1, ultimo_acesso=now, token_usuario=...` (query SQL crua).
  6. **Se `status_empresa != 1`** → flash "Sua empresa não está ativa..." e `redirect(Auth->logout())`.
  7. **EULA**: se `flag_aceita_eula != 1` E `grupo_id == 2` → redirect `/users/termos_uso_eula`. Caso contrário → redirect `/dashboard`.
- **Falha de login** → flash "Erro na autenticação! Verifique seu login e senha..." e redirect `/users/login`.
- **Gate de EULA app-wide** (`AppController::beforeFilter`): qualquer navegação de usuário grupo 2 sem EULA aceita (fora de users/termos_uso_eula) força `redirect('/users/logout')`.
- **Redirect por grupo pós-dashboard** (`AppController::beforeFilter`): se controller acessado é `dashboard`:
  - grupo **6** (Master imobiliária parceira) → `/painel/painel/indexadm/{integrador}/{empresa}/{token_emp}`.
  - grupo **8** (Assistente imobiliária) → `/painel/painel/indexadmsimula/...`.
  - (grupo 7 = corretores; sem redirect aqui.)
- **Permissões (RBAC)** — não na tela de login mas governam acesso pós-login:
  - `VerificaPermissao()`: query em `permissiongroups`/`modulecontrolleractions`/`modulecontrollers`/`modules` casando `group_id`+`controller`+`action`. Vazio ⇒ retorna `false` (chamadores tipicamente flasham "sem permissão" e redirecionam a `Dashboard/proibido`). Senão retorna `{permissao_id, nivel}`.
  - `VerificaPermissaoMenu()`: monta o menu lateral por grupo + módulos contratados da empresa (`empresas_modulos`), filtrando `flag_menu=1`.

## Máquina de estados / status (refletida na UI)
- **`User.status`**: só `1` autentica (scope do Auth). Outros valores ⇒ login falha silenciosamente (mensagem genérica).
- **`Empresa.status_empresa`**: `1` = ativa. Qualquer outro ⇒ login é desfeito (logout) com flash de empresa inativa. (`motivo_bloqueio` existe no modelo mas **não é exibido** na tela.)
- **`flag_aceita_eula`**: `1` = aceitou. Para grupo 2 sem aceite ⇒ fluxo de EULA obrigatório antes do dashboard.
- **Grupos (`grupo_id`)** observados no código:
  - 1 = admin plataforma; 2 = administrador de parceira fornecedora (sujeito a EULA); 3 = analista; 4,5,9 = grupos "fixos" com `mostra_colunas=1`;
  - 6 = master da imobiliária parceira; 7 = corretor; 8 = assistente da imobiliária (6/7/8 ⇒ `mostra_colunas=2` e integrador default 1).

## Multi-tenant / white-label
- **Seleção de marca por domínio** (`HTTP_HOST`) determina **o layout** em `login()`:
  - `localhost`/`xxxbuskaza`, `wecorp`, `alugueseguro`, `hubbpro`, `hubstate`, `upgradegarantia`, `esteiratech` → `login_layout`.
  - `buskaza` → `login_layout_buskaza`.
  - `completto` → `login_layout_completto`.
  - default (else) → `login_layout`.
- Cada layout define banner/logo/cores próprios (ex.: buskaza usa `imagens/banner_buskaza.png` + `buskaza_rosa.svg`).
- **Sessão grava o host**: `Auth.User.host = $host`.
- **Chaves de sessão de tenancy gravadas no login** (`Auth.User.*`): `token_emp`, `token_usuario`, `empresa_master`, `flag_empresa_associada`, `flag_empresa_parceira`, `empresa_resp_seguros|analises|vistorias`, `fornece_seguros|analises|vistorias`, `EmpRespSeguros|Analises|Vistorias` (dados completos da empresa responsável: nome, email, url_logotipo, telefone, uf, dominio_corporativo), `codigo_integrador`, `host`.
- Branding pós-login (logo da empresa responsável etc.) usa `url_logotipo`/`dominio_corporativo` dessas empresas responsáveis — base para o white-label das telas internas.

## Gotchas / decisões kept-bug
- **NÃO replicar — Captcha**: todo o bloco de captcha em `login()` está **comentado**; `captcha()`/component existem mas a tela não tem campo nem imagem. O handler `.creload` na view é código morto. Na migração, login NÃO valida captcha.
- **NÃO replicar — "Lembrar-me"**: **não existe** checkbox nem CookieComponent/RememberMe. A classe `lowercase` nos inputs é meramente cosmética.
- **NÃO replicar — duplicação de branding**: a lógica de `HTTP_HOST` é repetida em `login.ctp` E em `UsersController::login()`, com textos ligeiramente divergentes (ex.: `texto_solicite_apresentacao`/`suporte_atendimento`). A view define variáveis de texto que **não chegam a ser usadas** nos pontos onde o controller já as define para o layout — fonte de inconsistência. Migrar para configuração de tenant única.
- **Detecção de host frágil**: `preg_match('/wecorp\b/', $host)` casa por substring de domínio; ordem dos `elseif` importa (ex.: `localhost` e `xxxbuskaza` no mesmo ramo). Não confiar nessa heurística — usar mapeamento explícito domínio→tenant.
- **EULA exige novo login**: após aceitar termos, o usuário é mandado de volta a `/users/login` (não entra direto). Comportamento real a decidir se mantém.
- **SQL cru / risco**: `VerificaPermissao()`/`VerificaPermissaoMenu()` e o `UPDATE` de contadores usam concatenação de strings com valores de sessão/request — **SQL injection latente**. NÃO replicar; usar query parametrizada/ORM.
- **Mensagem de erro genérica**: usuário inativo, senha errada e usuário inexistente produzem a mesma flash. (Bom para segurança; manter.)
- **`Empresa.motivo_bloqueio`** é carregado mas nunca mostrado na UI de login.

## Destino (issues Linear)
- Fase **F01/F03** — projetos `wecorp-backend` (Auth/JWT, RBAC, sessão de tenant) e `wecorp-frontend` (tela de login white-label, EULA).
- Sugerido mapear para: tela de login multi-tenant (frontend), serviço de autenticação + scope status (backend), modelagem de permissões por grupo/módulo (backend), fluxo de aceite de EULA (frontend+backend). Issues específicas: ver milestones F01/F03 em `reference_linear.md` (não consultado aqui).
