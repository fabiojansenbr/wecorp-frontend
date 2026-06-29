# Legado (telas) — Seguros (Fiança / Incêndio / Capitalização / SPA)

> Plugin CakePHP `Contratoseguros` com 4 produtos de seguro locatício: Fiança/Garantias (fluxo mais rico, multi-proponente + composição de renda + cotação multi-seguradora), Incêndio, Capitalização (CAP/Icatu) e SPA (Proteção Aluguel). Telas server-rendered (.ctp + Bootstrap 3 + jQuery), fortemente acopladas a `AuthComponent::user('grupo_id')` para visibilidade de campos e a um motor de status por produto. Muito código morto/comentado nas views — não replicar.

## Cobertura
Views `.ctp` (todas em `app/Plugin/Contratoseguros/View/`):
- **Segurofiancas/**: `index.ctp`, `index_paginator.ctp`, `precadastro.ctp` (cadastro principal), `addproposta.ctp`, `addproponente.ctp`, `buscarpessoa.ctp` (form proponente via AJAX), `parecer.ctp` (emissão/contratação), `retorna_cotacoes.ctp`, `retornastatus.ctp`, `status.ctp`, `visualiza_parcelas.ctp`, `editarparcelaseguro.ctp`, `pdf_propostafianca.ctp`, `pdf_fianca_garantia.ctp`, `acesso_invalido.ctp`. (Variantes mortas: `precadastro_1..4`, `precadastro_fer`, `precadastrox` — ver Gotchas.)
- **Seguroincendio/**: `index.ctp`, `index_paginator.ctp`, `add.ctp`.
- **Seguroincendioapi/** (incêndio via API multi-seguradora): `index.ctp`, `index_paginator.ctp`, `novo.ctp`, `add.ctp`, `edit.ctp`, `enviarcotacao.ctp`, `efetivarcontratacao.ctp`, `exportardocumentos.ctp`, `retornacoberturas.ctp`, `novo_recupera_seguradoras.ctp`. (Mortas: `add_1.ctp`, `add_2.ctp`.)
- **Capseguros/**: `index.ctp`, `add.ctp`, `liberartransmissao.ctp`, `retornadados_sign.ctp`, `index_paginator.ctp`.
- **Spa/**: `index.ctp`, `add.ctp`.
- **Condomínio (4 produtos):** `Segurocondominio/{indexcondo,addcondo}.ctp` (Seguro Condomínio), `Condoincendioconteudo/{index,add}.ctp` (Incêndio Conteúdo), `Condovidafuncionario/{index,add}.ctp` (Vida/Funeral Funcionário), `Fichainspecaopredial/{index,add}.ctp` (Incêndio Compreensivo Condomínio / Inspeção predial). (Mortas: `Fichainspecaopredial/{add-,add-old,index-old}.ctp`.)

Controllers (lidos só p/ comportamento da tela):
- `Controller/SegurofiancasController.php` (5134 linhas — god file), `SeguroincendioController.php` (795), `SeguroincendioapiController.php` (~2.460), `CapsegurosController.php` (1687), `SpaController.php` (470), `SegurocondominioController.php`, `CondoincendioconteudoController.php`, `CondovidafuncionarioController.php`, `FichainspecaopredialController.php`.
- Colaboradores: `app/Controller/AppController.php::VerificaPermissao` / `::VerificaPermissaoModulo`; `app/Metadata/Libmetadados.php` (mapas de status).

## Telas / fluxos

### Fiança — listagem (`Segurofiancas/index.ctp` + `index_paginator.ctp`)
- Botão dropdown **"Nova Cotação/Análise"** → `precadastro/pf` ou `precadastro/pj`.
- Filtro (accordion): `fianca_id` (código), `pessoa_nome`, `pessoa_documento` (CPF/CNPJ), `ref_loc`, `num_apolice`, `imovel_end`, `fianca_data_de`/`fianca_data_ate` (datepicker), `status_segurofianca` (select de `StatusSegurofianca`).
- **Por grupo**: `grupo_id==1` vê filtro extra **"Situação proponentes"** (`followups_fianca`); grupos 1,2,3,4,9 veem campo autocomplete **"Imobiliária/Empresa"** (`empresaparceira_id`). Demais grupos não.
- Botão "Pesquisar" (`#btnPesquisar`) dispara AJAX paginado em `#paginator` (`segurofianca.js`); "Limpar filtro" = `location.reload()`.
- `index_paginator.ctp` monta a tabela: Cod, (empresa se `mostra_colunas==1`), Imóvel+aluguel, Data created + vigência (calcula "Expira em X meses" via `DateTime::diff`), Status (cor por `StatusSegurofianca.id`: 2/3/12=default, 7=success, 6=warning, 4=primary, 5=danger). Linha chama `statusPropostaFianca(...)` → AJAX `retornacotacoes`.

### Fiança — pré-cadastro/cotação (`precadastro.ctp`, ação `precadastro($tipopessoa)`)
Form `Segurofianca` com `parsley-validate`/`novalidate`, `onsubmit="return verificaForm(this)"`.
- **verificaForm (JS, único ativo)**: bloqueia se aluguel `<= 250,00` ("informe o valor do aluguel! Superior a 250,00"). Todo o restante (% comprometimento de renda 30/20/15%) está **comentado/desativado**.
- **Imobiliária** (`empresaparceira_id`, select2 `required`): grupos 1 e 2 têm opção vazia "Selecione"; demais não (auto-preenchida). Campo "Usuário logado" readonly + hidden `user_id`/`empresa_id`.
- **Tipos forçados por hidden** (selects originais comentados): `tipo_fianca_id=1`, `tipo_pessoa_contratante=strtoupper(tipopessoa)`, `tipo_cobertura_preferencial_id=2`, `tipo_cotacao_fianca_id=2` (Resumido). Há também um select visível de plano de coberturas (`listaCoberturas`, default 2) — duplicado pelo hidden.
- **Dados da locação**: `vlraluguel` (required), `vlrcondominio` (required), `vlriptu`, `vlrluz`, `vlragua`, `vlrgas` — todos máscara `MascaraMoeda(this,".",",",event)`, value inicial "0,00", `onblur=somadespesa()`. `meses_vigencia` (select `numParcelas`, required).
- **Dados do imóvel**: `tipolocacao_id` (Finalidade/uso, required, `onchange` mostra "pintura_nova" e blocos), `tipo_imovel_id` (select required), `pintura_nova` (Sim/Não, exibido condicional). Endereço por CEP: `imovel_cep` (máscara cep, `onblur=searchAddressByCep(...)`), `imovel_endereco` (readonly), `imovel_numero` (required), `imovel_complemento`, `imovel_bairro_id`/`imovel_cidade_id`/`imovel_uf` (selects encadeados `buscaBairros`/`buscaCidades`), `imovel_codigo_ref`.
- **Proponente PF (resumido)**: `doclocatario` (CPF, `data-mask=cpf`, required) — `blur` chama `getDocumento()` que busca dados e habilita `nomelocatario`; `emaillocatario` (required, type email), `celularlocatario` (`data-mask=mobile`), `nomemaelocatario`, `sexo` (M/F/N), `nascimentolocatario` (date), `estadocivillocatario` (1-5), `nacionalidade` (default "Brasileiro(a)"), `nome_social` (opcional). Hidden `statuslocatario=3`, `status_segurofianca=3`.
- **Bloco composição de renda (cotação completa — atualmente oculto, `display:none`)**: tabela dinâmica via JS `addrenda()`/`addpessoa()`/`somadespesa()`/`calperc()`; checkbox "Incluir Cônjuge para compor renda" abre painel com nome/CPF/tipo união/vínculo/renda. Cards "Custo Locação", "Total de Rendas", "Comprom. da Renda (%)".
- **Proponente PJ**: `nomeloca` (razão social), `docloca` (CNPJ `data-mask=cnpj`). Aviso: "análise poderá não ocorrer caso CNPJ < 2 anos".
- **Enviar link web ao cliente** (só `action=='precadastro'`): `email_enviar_link`, `celular_enviar_link` + confirmação.
- Submit "Analisar / Cotar"; "Desistir" com `confirm()` + `history.go(-1)`.
- **POST (ação)**: valida (`validata`) só se `pf` E cotação completa (`tipo_cotacao_fianca_id==1`); converte moedas via `Dataformat::currencyToDecimal`; gera `token` (`getToken(10)`); salva `Segurofianca`; cadastra `Pessoa` (relacionamentos_id=4 Pretendente Locatário, pessoa_status 3 p/ PF resumido / 1 p/ PJ); dispara e-mails (`emailNotification('seguro_fianca',...)`); redireciona p/ `index`. Tudo em transação com rollback.

### Fiança — proposta/proponentes (`addproposta.ctp`, `addproponente.ctp`, `buscarpessoa.ctp`)
- `addproposta` (ação `addproposta($id)`): cabeçalho com botão **"Formulário web para o Cliente"** (mostra token, link encurtado, WhatsApp `web.whatsapp.com/send?phone=55...`, checkbox `ativa_form_web`, datas de envio/conclusão). Botões Follow-ups (`janelaFollowups`). Aviso amarelo se `status_id==1` (Rascunho). Botão "Anexos da locação" só se `tipo_cotacao==2`. Edição completa dos dados da cotação.
- `addproponente.ctp`: usa element `gerenciar` (abas Proposta/Proponente). Lista lateral de proponentes (ícone vermelho = `flag_principal==1`), "Novo Proponente". Clique → JS `buscardados(pessoaid,fiancaid)` → AJAX `buscarpessoa/{pessoaid}/{fiancaid}` injeta `buscarpessoa.ctp` em `#pessoa`. Com `token` (acesso externo do cliente) some o element gerenciar e exibe nome da imobiliária + instruções.
- `buscarpessoa.ctp` (1224 linhas): formulário completo do proponente (dados pessoais, endereço, rendas, referências, documentos) renderizado via AJAX, validado por `verificaFormPessoa()`.

### Fiança — cotações & emissão (`retorna_cotacoes.ctp`, `parecer.ctp`)
- `retorna_cotacoes.ctp` (AJAX, ação `retornacotacoes`): grid de seguradoras (logo, status colorido via `cor_status`/`status_fianca_cotacoes` do metadata `fianca_analise_cotacoes`). Mostra links PDF (Apólice/Aprovação/Cotação) quando `seguradora_aprovada_id==seguradora_id`. Botão **"Contratar"** → `parecer/{fianca_id}/{seguradora_id}` só se `fianca_status==4` (Aprovado) e status cotação ∈ {4,10}. Pottencial (`seguradora_id==16`) + status 4 mostra link biometria. Mensagem "Nenhuma análise encontrada..." se vazio.
- `parecer.ctp` (ação `parecer($id,$seguradoraid)`): tela de emissão/contratação. Form bloqueia processamento (`disabled`) se `status_id!=4`. Edição completa só grupos 1/2 (parsley); demais form simples. Botões: abrir proposta, visualizar proponentes, gerar Certificado (só `seguradoraid==20`), visualizar parcelas (modal `visualiza_parcelas`). POST valida `termino_vigencia <= termino_contrato_locacao`; converte moedas/datas; trata valores por seguradora (16 Pottencial, 10 Tokio, 12 Porto parseiam plano concatenado por `-`).
- `visualiza_parcelas.ctp` / `editarparcelaseguro.ctp`: CRUD de parcelas do seguro (modal). `incluiparcelaseguro`/`removerparcelaseguro` no controller.
- `pdf_propostafianca.ctp` / `pdf_fianca_garantia.ctp`: geração de PDF via component Mpdf.

### Incêndio (`Seguroincendio/add.ctp`, `index.ctp`)
- Modelo `WbsProspectsProduto` + `WbsProspectsCotacoe`. `produto_id=2`.
- `add.ctp`: Imobiliária (select2; grupos 1/2 "Selecione"), Tipo pessoa (radio PF=1/PJ=2 com JS `trocarMascara` troca máscara CPF↔CNPJ e telefone), documento, nome, celular. Endereço por CEP. `tipo_ocupacao_id` (`onchange=verificaTipoOcupacao` mostra "Atividade Comercial"). 
- Simulação: `valor_aluguel` (moeda) + botão Calcular → calcula **LMI padrão** (`lmi_aluguel` readonly, `lmi_incendio` editável → `calculaValores()`+`calculaPercentualAgravo()`). Bloco **"LMI proposto"** (`lmi_incendio_sugerido` + obs). `percentual_agravo`, `pagto_parcelas` (`onchange=calculaParcelas`), `valor_premio_parcela`, `forma_cobranca`. Status (`status_cotacao_id`, `disabled` se `disableStatus`).
- **verificaForm (JS)**: LMI sugerido não pode ser < LMI padrão; teto R$ 2 mi (não-residencial ocupação 1/2) e R$ 5 mi (residencial 3/4); ocupação não-residencial exige atividade comercial.

### Capitalização — CAP/Icatu (`Capseguros/add.ctp`, `index.ctp`, `liberartransmissao.ctp`)
- `add.ctp`: **gate por white-label** — se `AuthComponent::user('empresa_resp_seguros')==0` exibe "módulo Icatu indisponível" e bloqueia. Mensagens de bloqueio se `status_proposta==3` (em transmissão) ou `==4` (transmitida) e `opcao!='CL'`.
- `verificaForm (JS)`: valida identidade do representante/locador PF (nº, órgão, UF, data); contratante PF (identidade completa) ou PJ (razão social, nome fantasia, CNPJ); **valor unitário do título ≥ R$ 1.000,00 e ≥ valor do aluguel**.
- `liberartransmissao.ctp` / `retornadados_sign.ctp`: liberação de transmissão e retorno de assinatura digital (sign). Ações `liberartransmissao`, `retornadados_sign`, `montarDadosCap`.

### SPA — Proteção Aluguel (`Spa/add.ctp`, `index.ctp`)
- Modelo `WbsProspectsProduto`/`WbsProspectsCotacoe`. `add.ctp`: Imobiliária, Tipo contratante (radio PF/PJ + `trocarMascara`), documento (máscara cpf), nome, `pessoa_data_nasc` (datepicker), celular. Fluxo de cotação simplificado (sem multi-seguradora rica da fiança).
- `index.ctp`: filtro por nome + datas, botão "Nova Cotação" → `add/`.

### Incêndio via API (`Seguroincendioapi/*`) — produto incêndio multi-seguradora por integração
Distinto do Incêndio "clássico" (`Seguroincendio/`): aqui a cotação/contratação trafega por API de seguradoras. Modelo `WbsProspectsProduto`/`WbsProspectsCotacoe` (`produto_id` incêndio).
- `index.ctp`: dashboard de vencimentos no topo (cards "VENCE EM 35/07 DIAS", "A vencer em 05/15/30 dias") que disparam `processaForm('btnPesquisar','WbsProspectsProdutoAvencer',<dias>)` e por status (`WbsProspectsProdutoSolicitacaoStatus`). Filtros (form `WbsProspectsProduto`, parsley/novalidate): `id` (Código, number), `solicitacao_dataini`/`solicitacao_datafim` (datepicker), `data_inicio_vigencia_de`/`_ate`, `solicitacao_status` (select `$statusCotacao`), `avencer` (number 1..360), `pesquisar_em`+`conteudo_pesquisar` (busca campo+valor), `empresa_id` (select2 `lista_imobiliarias`, só internos). Botão `#btnPesquisar.pesquisar` (AJAX → `index_paginator`). Linha chama `mostraCoberturas(itemid,retorno,div,empresaparceira_id,token)` → `seguroincendioapi/retornaCoberturas/...` (expande coberturas/cotações da seguradora).
- `novo.ctp`/`add.ctp` (`novo($empresaid)` → `add($empresaid,$seguradora_id)`): wizard de cotação. Imobiliária (`empresaparceira_id` select2 required), Tipo Pessoa (radio PF/PJ `trocarMascara`), `pessoa_documento`/`pessoa_nome`/`pessoa_data_nasc`/`pessoa_sexo`/`pessoa_identidade`(+origem/data)/`pessoa_celular`(mobile)/`pessoa_email`/`pessoa_profissao_id`(select2)/`pessoa_pep`/`pessoa_estrangeiro`/`pessoa_renda_id`(faixa)/`pessoa_renda`(MascaraMoeda). Bloco **coberturas** (`item_cobertura[<codigo>][item_valorcobertura]`) com regra "prédio + conteúdo = 100,00". Beneficiário (`beneficiario_is_segurado` → `checaDadosSeguradoBeneficiario()`). LMI: `lmi_incendio`/`lmi_incendio_sugerido` com mesmos tetos do incêndio clássico (5 mi residencial ocup. 3/4; 2 mi não-residencial ocup. 1/2; sugerido ≥ padrão).
- `enviarcotacao.ctp` (`enviarcotacao($empresaid)`) → submete cotação às seguradoras via API; `efetivarcontratacao.ctp` (`efetivarcontratacao($empresaid)`) → emissão/contratação; `exportardocumentos.ctp` (`exportardocumentos($tipo_doc)`) → download de apólice/condições; `retornacoberturas.ctp` (AJAX `retornaCoberturas`) fragmento das coberturas. `confirmarcotacao($id)` confirma a cotação retornada. Status na UI = `seguroincendiocotacaoApi` (1 Rascunho..14 Contratado — ver Máquina de estados).

### Seguros de Condomínio (4 produtos `Contratoseguros`)
Família voltada a **condomínios** (contratante PJ por padrão), todos usando `WbsProspectsProduto`/`WbsProspectsCotacoe` (exceto a ficha de inspeção, modelo próprio). Form padrão `parsley-validate`/`novalidate`, Imobiliária (`empresaparceira_id` select2 required), endereço por CEP (`imovel_cep` → `searchAddressByCep`, `imovel_endereco` readonly, número/complemento, UF→`buscaCidades`→`buscaBairros` encadeados), `imovel_codigo_ref`.
- **Seguro Condomínio** — `Segurocondominio/{indexcondo,addcondo}.ctp` (ações `indexcondo`, `addcondo`). `addcondo`: Tipo do Contratante radio (PF comentado; **PJ fixo `checked`**), `pessoa_documento` (cnpj required), `pessoa_nome` "Nome do Condomínio", `pessoa_telefone`. Dados do prédio: `tipo_ocupacao_id` "Tipo condomínio", `ano_construcao`, `num_blocos`, `num_andares_bloco`, `num_elevadores_bloco`, `area_construida` (MascaraMoeda required), `estrutura` (select `$tipoEstrutura`), `num_vagas`. **OBS**: `indexcondo.ctp` tem `$empresa_parceira_id = 2;` hardcoded ("Prohome, depois pegar pela Session") — não replicar.
- **Incêndio Conteúdo** — `Condoincendioconteudo/{index,add}.ctp` (`<h3>Seguro Incêndio Conteúdo`). Form quase idêntico ao condomínio, PJ fixo, documento cnpj, nome/telefone, endereço por CEP.
- **Vida/Funeral Funcionário** — `Condovidafuncionario/{index,add}.ctp` (`<h3>Vida Funeral Funcionário Condomínio`). Mesmo esqueleto PJ; cotação de seguro de vida/funeral para funcionários do condomínio.
- **Incêndio Compreensivo Condomínio / Inspeção Predial** — `Fichainspecaopredial/{index,add}.ctp` (`<h3>Incêndio Compreensivo Condomínio`, subtítulo "Módulo para cotação de Seguros de Condomínio / Inspeção predial"). Form `Fichainspecaopredial` (modelo próprio): Imobiliária, **Tipo Condomínio**/Informe o tipo/**Tipo Gestor**, CNPJ, Razão Social, telefones comercial/celular, e-mail, endereço por CEP. É a contraparte interna do `Formulariosweb/segurocondo.ctp` público (ver dossiê `portal-publico.md`).

## Pontos de entrada (controller::ação que renderiza)
- `SegurofiancasController::index` / `::index_paginator` → listagem fiança.
- `SegurofiancasController::precadastro($tipopessoa)` → renderiza **`precadastro.ctp`** (default; não chama render para variantes).
- `::addproposta($id)`, `::addproponente($id,$tipoCotacao)`, `::buscarpessoa($pessoaid,$fiancaid)` (AJAX), `::clienteproponente(...)` (acesso externo c/ token).
- `::parecer($id,$seguradoraid)` → emissão; `::parecer_porto(...)`.
- `::retornacotacoes(...)` → `retorna_cotacoes.ctp` (AJAX); `::statusfianca(...)` → `retornastatus.ctp`; `::status` → `status.ctp`.
- `::visualiza_parcelas($id)`, `::editarparcelaseguro($id)`, `::incluiparcelaseguro`, `::removerparcelaseguro`.
- `::pdf_propostafianca($id,$pessoaid)`, `::pdf_fianca_garantia($id,$pessoaid)`.
- `SeguroincendioController::index`/`index_paginator`/`add($empresaid)`/`edit($id,$tipo)`/`confirmarcotacao($id)`/`efetivarcotacao($id)`.
- `CapsegurosController::index`/`add($empresaid)`/`edit($id,$opcao)`/`novo($pessoaid,$fkid,$origem)`/`liberartransmissao($id)`/`retornadados_sign(...)`.
- `SpaController::index`/`add($empresaid)`/`edit($id)`.

## Regras de negócio relevantes à UI
- Aluguel mínimo p/ fiança: **> R$ 250,00** (bloqueio no submit).
- CAP: título unitário **≥ R$ 1.000,00 e ≥ aluguel**.
- Incêndio: tetos de LMI sugerido (2 mi não-residencial / 5 mi residencial); sugerido ≥ padrão.
- Composição de renda (% comprometimento 30%/20%/15%) e modo "Completo" estão **desativados/comentados** — hoje só cotação "Resumido" (`tipo_cotacao_fianca_id=2`).
- Conversão de moeda BR sempre via `Dataformat::currencyToDecimal`; máscara de exibição `MascaraMoeda`.
- Token de 10 chars por cotação (`getToken(10)`), usado no link da Área do Cliente (`ativa_form_web`, expiração `token_expirado`).
- Disparo de e-mail/SMS na criação (`emailNotification`, `ComunicacaoController::encurtaURL` — chamada parcialmente quebrada, ver Gotchas).
- Botão "Contratar" só aparece com fiança Aprovada (`fianca_status==4`) e status cotação 4/10.

## Máquina de estados / status (refletida na UI)
Definidos em `app/Metadata/Libmetadados.php`:
- **`StatusSegurofianca`** (tabela, usado no filtro/listagem): cores na UI por id — 2/3/12=default, 4=primary, 5=danger, 6=warning, 7=success.
- **`followups_fianca`** (situação proponentes): 1 Em análise, 2 Pendência documental, 3 Aprovado, 4 Recusado score, 5 Recusado insuficiência, 6 Recusado restrição, 7 Recusado pendente, 8 Recusado, 9 Emissão pendente, 10 Emitido.
- **`fianca_analise`**: 0 Espera,1 Aprovado,2 Recusado,5 Biometria,6 Pré-aprovado,7 Pendências,8 Análise,10 Erro,99 Erro.
- **`fianca_analise_cotacoes`** (status por seguradora no grid): 0 Pendente,1 Concluído,3 Recusado,4 Biometria pendente,7 Proposta emitida,8 Proposta recusada,9 Proposta em emissão,10 Aprovado,94 Biometria recusada,99 Erro.
- **`seguroincendiocotacaoApi`**: 1 Rascunho,2 Para/Em cotação,3 Cotado,4 Para/Em contratação,5 Emitido,6 Cancelado,10 Confirmado,13 Falha,14 Contratado.
- **`cap`**: 1 Em rascunho,2 Em análise,3 Em transmissão,4 Transmitida,5 Proposta com pendências,6 Pagamento confirmado,7 Cancelado/excluído.
- **`spa`**: 1 Em rascunho,2 Documentação em análise,3 Em análise seguradoras.
- Comentário no metadata mapeia para o app Laravel novo (STATUS_RASCUNHO=1...STATUS_EXPIRADO=11; cotação x seguradora STATUS_FILA_PENDENTE=0...) — útil para paridade na migração.

## Multi-tenant / white-label
- Visibilidade/comportamento dirigidos por **`AuthComponent::user('grupo_id')`**:
  - Grupos **1, 2** = internos admin/operacional (veem opção "Selecione" na imobiliária, filtros extras, edição em `parecer`, parsley).
  - Grupos **1,2,3,4,9** = internos (veem filtro de imobiliária; lista de parceiras via `Empresa->getParceiras()`; até 120 parcelas).
  - Grupos **5,6,7,8** (e 10) = usuários de imobiliária — `VerificaPermissao` retorna `nivel==2` ⇒ queries filtradas por `Segurofianca.empresaparceira_id = empresa_id` do usuário; imobiliária pré-selecionada; até 60 parcelas.
- `Empresa.flag_fianca_empresarial==1` troca opções de tipo de fiança (taxa fixa/empresarial vs seguro fiança/capitalização/fiador/proteção) e default.
- CAP gated por `empresa_resp_seguros` (corretora responsável); seguradoras filtradas por `flag_fianca` + flag pf/pj em `WbsSeguradora`/`WbsProdutosSeguradoras`.
- Acesso externo do cliente (sem login) via `token` em `addproponente`/`clienteproponente`/`buscarpessoa` — Auth desabilitado nessas ações (ver `beforeFilter` comentado).
- Incêndio/SPA vinculam empresa por `imobiliaria_token`/`empresaparceira_id`.

## Gotchas / decisões kept-bug
- **NÃO replicar variantes mortas** de `precadastro`: `precadastro_1.ctp`..`_4`, `precadastro_fer.ctp`, `precadastrox.ctp` — a ação só renderiza `precadastro.ctp`. Idem `add-bkp.ctp`, `index-BKP.ctp`.
- **`beforeFilter` do SegurofiancasController está 100% comentado** (`parent::beforeFilter()` apenas) — Auth/permite-tudo desativado ali; a proteção real vem de `VerificaPermissaoModulo`/`VerificaPermissao` nas ações.
- Muitos campos da `precadastro.ctp` estão `display:none` ou em comentários HTML (cotação completa, corretor/filial, seguradora preferencial) — herança de versões anteriores. A tela efetiva hoje é **PF/PJ resumida**.
- `precadastro` POST: bloco de cadastro de rendas/`PessoaExtra` está dentro de comentário `/* ... */` — não grava rendas extras hoje, apesar do JS de composição existir.
- **Bug de e-mail de link**: em `precadastro`, `$url = $url_curta->encurtaURL(...)` está **comentado** mas logo abaixo usa `$url['url_curta']` ⇒ índice indefinido (envia link vazio). Não replicar — corrigir na migração.
- Campo duplicado: `tipo_cobertura_preferencial_id` aparece como select visível **e** como hidden `value=2` no mesmo form (o último vence).
- `verificaForm` da fiança compara `Number(vlr_aluguel) === '0.00'` (number vs string) — comparação frágil; manter só a regra "> 250,00".
- `index_paginator.ctp`: bloco `cor_status` tem ramos duplicados (id 6 e 7 repetidos) — efeito inócuo.
- `retorna_cotacoes.ctp` usa `print_r($rs['planos_analise'])` direto no HTML (debug vazado em produção). Não replicar.
- Validação cliente em `parsley` + `novalidate=TRUE` (HTML5 desligado) — validação real é JS/parsley, não nativa.
- IDs de seguradora hardcoded nas telas: 16 Pottencial, 10 Tokio, 12 Porto, 20 (Certificado garantia). Tratar via configuração na migração.

## Destino (issues Linear)
- Projeto **wecorp-frontend** (Next.js). Domínio "Seguros" — fase F09.
- Sugestão de épicos/telas a mapear: (1) Listagem+filtros de Fiança; (2) Wizard de Cotação Fiança PF/PJ (resumida; reabrir composição de renda como feature nova); (3) Proposta + Proponentes + Área do Cliente (token/link); (4) Grid de cotações multi-seguradora + Emissão/Parecer + Parcelas; (5) Incêndio (cotação+LMI); (6) Capitalização (gate white-label + transmissão/sign); (7) SPA. Mapear status para enums do app Laravel (comentário em `Libmetadados.php`). Buscar milestones de F09 em wecorp-frontend (HUB-116..HUB-382).
