# Legado (telas) — Assinatura eletrônica (Plugin Sign)

> Módulo de gestão de "envelopes" de assinatura eletrônica integrando 3 plataformas externas (Docusign, ClickSign, Autentique). O fluxo principal é: listar envelopes (`index`) → criar (`add`) → gerenciar documentos+signatários (`editprocessar`) → confirmar envio para a plataforma (`confirma_envio` / link externo webservice). Muito do processamento real (gerar/enviar/baixar) acontece em webservices PHP externos abertos em nova aba, não nas telas CakePHP.

## Cobertura

Views .ctp (em `app/Plugin/Sign/View/Sign/`):
- `index.ctp` — listagem + filtros + ações por envelope
- `add.ctp` — criação de novo envelope
- `editprocessar.ctp` — tela principal de gestão (docs + signatários + dados)
- `edit.ctp` — variante de gestão (parcialmente quebrada; ver Gotchas)
- `carrega_signdocs.ctp` — fragmento AJAX: tabela de documentos do envelope
- `carrega_signers.ctp` — fragmento AJAX: tabela de signatários do envelope
- `retornasigners.ctp` — fragmento (`requestAction`) com status dos signatários na listagem
- `proponentes.ctp` — janela modal de busca/seleção de imóvel/proponente
- `confirma_envio.ctp` — tela "vazia" de confirmação (só link "voltar")
- `downloads.ctp`, `ver.ctp` — views CMS copiadas, NÃO pertencem ao fluxo (ver Gotchas)

Controller: `app/Plugin/Sign/Controller/SignController.php` (índices de linha citados).
Model: `Sign.php` (status), `SignSigner.php` (getSigners).
JS: `webroot/js/sign.js` (AJAX docs/signatários). Helper: `Lib/Dataformat.php::formataStatusSign`.

## Telas / fluxos

### index.ctp — "Gerenciar Envelopes" (listagem + filtros)
Formulário de filtro (POST para a própria `index`), campos:
- `dados_gerais` (texto) — busca por `Sign.descricao LIKE %...%`.
- `data_de` / `data_ate` (texto, classe `date datepicker`, máscara/calendário) — filtra `Sign.created` (>= 00:00:00 / <= 23:59:59), convertidos por `Dataformat::dateToSql`.
- `status` (select `statusEnvelope`) — filtra `Sign.status_envelope`.
- `plataforma_assinatura` (select `listaPlataformas`) — filtra `Sign.plataforma_assinatura`.
- `empresa_id` (input autocomplete `#endereco_1`, name `data[Sign][empresa_id]`) — **só aparece para grupos 1,2,3,4,9** (admin/plataforma). Filtra `Sign.empresa_id`.
- Botão "Pesquisar" (`data[submit]`); link reload; botão "Novo envelope" → `/sign/<controller>/add/`.

Tabela: colunas Id, [Empresa — só quando `$mostra_colunas==1`], Finalidade, Ref contrato, Dados Gerais (título + endereço imóvel), Status (label de `statusEnvelope`), Criado (`Dataformat::dateBr`). Paginação 25/pág, ordem `id DESC`.

Coluna **Ações** — botões condicionais por `status_envelope` + `plataforma_assinatura`:
- Sempre: "Editar" → action `editprocessar`.
- `status==1` (Rascunho) + `docusign`: "Confirmar envio" → abre nova aba `<url_wbs>/docusignv3/api_sign.php?op=newsign&idenv=<id>&tokenus=<token>` (confirm() antes).
- `status==1` + `clicksign`|`autentique`: "Confirmar envio" → navega para `confirma_envio/<id>/tokenus=<token>` (mesma janela).
- `status==2` (Processando) + `docusign`: "Gerenciar envelope" → `api_sign.php?op=updatelinkgestor&...`.
- Linha expansível "Signatários" (collapse) que faz `requestAction('/sign/sign/retornaSigners/<id>')` (renderiza `retornasigners.ctp`).
- Dentro do collapse, só docusign: "Atualizar links/enviar e-mail" (`op=updateallemail`) e "Atualizar status dos signatários" (`op=getstatussigners`).
- `status==4` (Concluído): docusign → "Baixar documentos assinados e certificado unificados" (`op=downloaddoc`); clicksign → "Baixar documento assinado" (`clicksign/api_sign.php?op=downloaddoc`). **autentique não tem botão de download.**

`$url_wbs` é montado por heurística de host no topo do arquivo (localhost → `/wecorpwbs`; https/http → `/webservices`; se URL contém "hml" usa HTTP_HOST). Multi-tenant por subdomínio.

### add.ctp — "Novo envelope"
Form `Sign` (parsley, novalidate, autocomplete off), POST para `add`. Campos visíveis:
- `empresa_id` (select2 `lista_imobiliarias required`) — option "Selecione" só para grupos 1 e 2; demais grupos sem opção vazia.
- `plataforma_assinatura` (select2 required, empty) — opções dependem de grupo/empresa (ver Regras).
- Bloco colapsável "Relacionar a um imóvel" (`#dados_imovel`):
  - `garantia_locacao` (texto), `referencia_contrato` (texto) "Cód/Ref contrato".
  - `imovel_id` (texto readonly + lupa `janelaImoveis('Sign','index',...)`); `imovel_endereco` (textarea readonly).
- Bloco "Responsável para receber cópia (somente Docusign)": `aprova_nome`, `aprova_email`, `aprova_celular` (máscara `data-mask="mobile"` → `(xx) xxxxx-xxxx`).
- "Dados complementares": `titulo_assinatura` (required, maxlength 65, "Título do e-mail"), `finalidade_envelope` (required, maxlength 25), `instrucoes_assinatura` (textarea required, rows 10, pré-preenchido com `$texto_padrao_docusign_instrucoes` — corpo do e-mail aos signatários).
- "Termos de uso": textarea readonly com element `termos_uso_sign`; checkbox `data[Sign][aceitou_termos]` (class required, value 1) — **obrigatório aceitar para prosseguir**.
- Submit "Processar" (`data[submit]`).
- Campos `tipo_assinatura` (select `tipoAssinatura` com `onchange=verificaTipo()`) e blocos `#1_vistoria`/`#2_contrato_locacao`/`#3_contrato_adm`/`#4_minuta_sinal` existem mas ficam em `display:none` (fluxo de seleção de origem desativado na prática).

Ao salvar: cria `Sign` com `status_envelope=1`, `documentomodelo_id=0`, `envelope_id=0`, `user_id`/`empresa_id` do Auth; opcionalmente grava signatários a partir de `Locatario`/`Locador`/`Fiador`/`Testemunha` (arrays do POST). Redireciona para `editprocessar/<id>`.

### editprocessar.ctp — tela principal de gestão (passo 2)
Renderizada após criar/editar. Não é um único form: combina blocos AJAX + um form final.
- **Documentos** (`#lista_docs`, carregado por `carregaSignDocs()` → `carrega_signdocs/<id>`):
  - Botão "Incluir documento" abre `#divDoc` (form `FormDocSign`, hidden `id`=sign_id).
  - "Localizar/cadastrar arquivo (vistoria ou título de capitalização)" → `janelaBuscaRegistro('empresas','pesquisarelacao','sign_vistoria','pesquisa',...)`.
  - "Localizar arquivos anexos" → `arquivosAnexos('sign', id, 0, empresa_id, 0, '')`.
  - "Processar/Confirmar" → JS `processaDocumentoSign('FormDocSign',...,'novo')` → POST `docsign_cadastra`.
- **Signatários** (`#lista_signers`, `carregaSigners()` → `carrega_signers/<id>`):
  - "Incluir signatário" abre `#divSigner` (form `FormSigner`, hidden `id`): campos `sig_nome` (maxlength 80), `sig_email` (80), `sig_documento` ("formate CPF ou CNPJ", 80 — **sem máscara automática**), `sig_celular` (placeholder `(21) 99999-9999`, sem máscara).
  - "Confirmar" → `processaSigner('FormSigner',...,'novo')` → POST `docsign_cadastrasigner`.
- Form final `Sign` (hidden `sign_id`): bloco colapsável imóvel (igual add); `empresa_id`, `plataforma_assinatura` (select2 required); Responsável (`aprova_nome/email/celular`); `titulo_assinatura` (required), `finalidade_envelope` (required), `instrucoes_assinatura` (textarea rows 10); termos de uso + checkbox `aceitou_termos`. Submit "Processar e avançar" → POST `editprocessar` → salva e volta a `editprocessar/<id>`.

### edit.ctp — variante de gestão (legado / parcialmente quebrada)
Quase idêntica a editprocessar, MAS:
- Select da plataforma usa name **`plataforma_assiantura`** (typo) → não persiste o campo correto.
- Campo `descricao` (textarea) em vez de bloco imóvel.
- Layout de blocos diferente. Provavelmente versão anterior; `editprocessar` é a usada no fluxo real (index e add redirecionam para `editprocessar`).

### carrega_signdocs.ctp (fragmento AJAX)
Tabela: Arquivo (título + `model_id`), Data (`datetimeBr`), Ação excluir. Link de download varia por `model`:
- `model != 'vistoria'` e `!= 'capseguros'`: `/arquivos/download_file/<model_id>`.
- `model == 'capseguros'`: link direto `_files/<url_arquivo_local>/<nome_arquivo_local>` (nova aba).
- `model == 'vistoria'`: sem link de download (só nome).
Excluir → confirm() → `excluiDocumentoSign(id,'div_mensagem','delregistro')` → POST `docsign_excluidoc/<id>`. Vazio mostra "Nenhum arquivo/documento anexado".

### carrega_signers.ctp (fragmento AJAX)
Tabela (texto em vermelho): Nome, Documento, E-mail, Celular, Data (`datetimeBr`), Excluir. Excluir → confirm() → `excluiSigner(id,'div_mensagem_signer','delregistro')` → POST `docsign_excluisigner/<id>`. Lê `SignSigners` (alias plural).

### retornasigners.ctp (collapse na listagem)
Tabela: Tipo (`tipo_sign` em badge), Nome, Email, Documento, Celular, Status, Assinado em. Status renderizado por `Dataformat::formataStatusSign(status_sign)` (badges coloridos); se vazio mostra "N/d". `data_sign` via `dateBr`.

### proponentes.ctp — modal de busca de imóvel
Form de busca (`id`, `id_ref`, `codigo_ref`, `endereco`) com AJAX GET para `/imovel/imovels/index/<origem>/0/<ajax>/<janela>` (botão `#pesquisarImovel`). Tabela de imóveis; ação selecionar chama `selecionarImovel(...)` + `closeModal()`, editar/ver via `janelaImoveis`. Botão "Novo Proponente" → `janelaProponentes(origem,'novo',...)`. Reutiliza paginação Ajax do CakePHP Js helper.

### confirma_envio.ctp
Tela quase vazia: título "Sign - Assinatura Eletrônica" + link "Voltar a listagem". O efeito real é no controller (muda status para 2). Usada pelo fluxo ClickSign/Autentique.

## Pontos de entrada (controller::ação que renderiza)

- `SignController::index()` (L24) → `index.ctp`. POST = filtro. Seta `statusEnvelope`, `listaPlataformas` (docusign/clicksign/autentique), `token_usuario`, `mostra_colunas`.
- `SignController::add()` (L329) → `add.ctp`. POST cria envelope + signatários, redireciona `editprocessar`.
- `SignController::editprocessar($id)` (L610) → `editprocessar.ctp`. GET carrega `Sign.find('first')`; POST salva.
- `SignController::edit($id)` (L536) → `edit.ctp`. Salva e redireciona para `editprocessar` (não fica em edit).
- `SignController::confirma_envio($id)` (L708) → `confirma_envio.ctp`. **Em GET** muda `status_envelope=2` e salva (sem confirmação extra no servidor).
- `SignController::retornaSigners($sign_id)` (L143) → `retornasigners.ctp` (via requestAction).
- `SignController::carrega_signdocs($sign_id)` (L755) → `carrega_signdocs.ctp` (layout null, AJAX).
- `SignController::carrega_signers($sign_id)` (L786) → `carrega_signers.ctp` (layout null, AJAX).
- `SignController::pesquisa(...)` (L156) → modal busca (origem proponentes).
- AJAX JSON (notRender): `docsign_cadastra` (L812, grava SignDocs + signatários por tipo + tabela `sign_docs_signers`), `docsign_cadastrasigner` (L999), `docsign_excluidoc/<id>` (L1032), `docsign_excluisigner/<id>` (L1048).
- `downloads()` (L737) e `proponentes` view → ver Gotchas.

## Regras de negócio relevantes à UI

- **Aceite de termos obrigatório**: checkbox `aceitou_termos` é `required`; bloqueia o submit pelo Parsley. Em edição, se já aceito vem `checked disabled`.
- **Plataformas disponíveis por grupo/empresa** (add):
  - Grupos 1 ou 2, OU empresa com `ativa_clicksign==1`: Docusign + ClickSign + Autentique.
  - Demais: **somente Docusign**.
  - Em `editprocessar`: grupos 1/2 → 3 plataformas; outros → só a plataforma já gravada (`ucwords`).
- **Lista de imobiliárias por grupo**:
  - Grupos 1,2,3,4,9 → `Empresa->getParceiras()` (todas as parceiras).
  - Grupos 6,7,8 → só a própria empresa do usuário.
  - Demais → própria empresa + `empresa_master`.
- **Validação de propriedade (grupo 2)** em `editprocessar`: confere se a empresa do envelope é cliente da parceira (`empresa_resp_seguros/analises/vistorias`); senão redireciona com flash de "sem permissão".
- **Permissão de módulo**: `index()` chama `VerificaPermissaoModulo('0')` + `VerificaPermissao()`; sem permissão renderiza `/Dashboard/proibido`. Nível 3 vê só os próprios (`Sign.user_id`); nível 2 vê só da própria empresa (`Sign.empresa_id`).
- **Campos obrigatórios UI**: `titulo_assinatura`, `finalidade_envelope`, `instrucoes_assinatura`, `empresa_id`, `plataforma_assinatura` (classe `required`).
- **Limites**: título 65 chars, finalidade 25 chars (no add).
- O processamento pesado (gerar PDF, enviar à plataforma, baixar assinados/certificado, atualizar status) é delegado a webservices externos (`docusignv3/api_sign.php`, `clicksign/api_sign.php`) abertos com `tokenus=<token_usuario>`. As telas só montam os links.

## Máquina de estados / status (refletida na UI)

`Sign::statusEnvelope()` — `status_envelope`:
- 0 N/d · 1 Em Rascunho · 2 Processando · 3 Em andamento · 4 Concluído · 5 Cancelado.
Transições visíveis na UI:
- Criação → 1 (Rascunho).
- "Confirmar envio" → docusign abre webservice (status muda lá); clicksign/autentique → `confirma_envio` seta 2 (Processando) direto no save.
- 2 + docusign → botão "Gerenciar envelope".
- 4 (Concluído) → botões de download (docusign: docs+certificado; clicksign: doc; autentique: nenhum).

`Sign::statusSigner()` — `status_sign` (model): 0 N/d · 1 Não iniciado · 2 Enviado e Aguardando · 3 Visualizado · 4 Assinado e aceito · 5 Recusado · 6 Expirado.
`Dataformat::formataStatusSign()` (badges, divergente do model): 1 Rascunho · 2 Aguardando · 3 Visualizado · 4 Aceito e assinado · 5 Rejeitado (6 Expirado não tratado → badge vazio). Novos signatários gravados com `status_sign=1`.

## Multi-tenant / white-label

- Escopo de dados por `empresa_id`/`user_id` conforme grupo (ver Regras). Coluna "Empresa" na listagem só aparece para perfis admin (`$mostra_colunas==1`).
- `$url_wbs` (webservices externos) resolvido por host/subdomínio em `index.ctp` (localhost vs produção vs "hml"). Sem config central — heurística inline.
- Habilitação de plataformas ClickSign/Autentique controlada por `Empresa.ativa_clicksign` e por grupo — efetivamente feature-flag por tenant.

## Gotchas / decisões kept-bug

- **`downloads.ctp` e `ver.ctp` NÃO são do domínio Sign**: são views CMS copiadas (breadcrumb "/cms/cms/downloads", usam `$cm['Cm']`). Não replicar como telas de assinatura.
- **`edit.ctp` quebrada**: select da plataforma com name `plataforma_assiantura` (typo) — não salva a plataforma. O fluxo real usa `editprocessar`. NÃO replicar o typo.
- **`docsign_cadastra` (Fiador) com bug**: monta array em `$Locador` mas chama `$this->SignSigner->save($Fiador)` (`$Fiador` indefinido) — fiador via documento NÃO é gravado corretamente. Não replicar.
- **`confirma_envio` muda status em GET** (sem POST/CSRF), e o link da listagem usa parâmetro posicional estranho (`/confirma_envio/<id>/tokenus=<token>`) — o token não é usado no controller. Migrar para POST/idempotente.
- **`SignSigners` vs `SignSigner`**: o controller usa ora model singular (`SignSigner`, em add/cadastrasigner) ora plural (`SignSigners`, em carrega_signers/excluisigner). A view `carrega_signers.ctp` lê `SignSigners`, mas `retornasigners.ctp` lê `SignSigner`. Inconsistência de alias a normalizar.
- **Documento de signatário sem máscara**: campo "Documento (formate CPF ou CNPJ)" exige formatação manual pelo usuário; celular idem. Na migração, aplicar máscara/validação de CPF/CNPJ e telefone.
- **`instrucoes_assinatura` padrão** vem com texto fixo Docusign (MP 2.200-2/2001) e menção "anexos" mesmo para ClickSign/Autentique.
- Blocos de "tipo de assinatura" (vistoria/locação/adm/minuta) existem no HTML mas estão ocultos; o seletor de origem do documento não está ativo — documentos são adicionados manualmente via `editprocessar`.
- Bloco "Responsável para receber cópia" é rotulado "(somente Docusign)" no add mas gravado para todas as plataformas.
- Heurística de `$url_wbs` por string "hml"/"localhost" é frágil; substituir por configuração de ambiente na migração.

## Destino (issues Linear)

Fase F12 — "Assinatura eletrônica (telas)" do projeto wecorp-frontend. Telas a recriar (NestJS+Next.js): listagem de envelopes c/ filtros e ações por status/plataforma; wizard criar envelope (dados + termos); gestão de documentos+signatários (componentes AJAX → chamadas API); integração com provedores (Docusign/ClickSign/Autentique) via backend (não webservices PHP inline). Mapear estados de envelope (0–5) e de signatário (0–6) como enums. Issues backend correlatas (wecorp-services/backend) para a integração real das plataformas e download de documentos assinados.
