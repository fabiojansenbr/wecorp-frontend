# Legado (telas) — Portal público do proponente (F20)

> Conjunto de telas públicas (sem login) que o proponente/inquilino acessa por link com **token** para preencher a proposta de Seguro Fiança, a Ficha de Análise Cadastral, visualizar assinatura eletrônica (O2 Sign), responder cotação de seguro condomínio/inspeção predial, validar autenticidade de uma análise e (parcialmente) autocadastrar-se como novo proponente. Tudo roda no controller `FormularioswebController` + `ValidaanalisesController`, com layout `default_painel_cliente` e o JS `Contratoseguros/js/segurofianca.js`.

> **Anexos/Upload:** o widget de upload de documentos do proponente apoia-se no subsistema transversal de arquivos do backend — mecanismo, token-gate e storage documentados em [`../../../backend/docs/legado/anexos-upload.md`](../../../backend/docs/legado/anexos-upload.md).

## Cobertura
Controllers:
- `app/Controller/FormularioswebController.php` (god file ~2.975 linhas) — métodos: `beforeFilter`, `index`, `index_analise`, `proponentefianca`, `addproponente`, `buscarpessoa`, `buscarpessoaanalise`, `enviarproposta`, `enviaranalise`, `pdf_propostafianca`, `segurocondo`, `enviarpropsegurocondo`, `index_segurocondo`, `validaFianca`, `authFormularioweb`, `analisecadastral`, `o2sign`.
- `app/Controller/ValidaanalisesController.php` — `beforeFilter`, `index`.
- `app/Controller/ValidavistoriasController.php` — `beforeFilter`, `index` (validação pública de vistoria, irmã da análise).

Views `.ctp` (em `app/View/Formulariosweb/` salvo indicação):
- `proponentefianca.ctp` — shell da proposta de fiança (lista proponentes + dados imóvel).
- `buscarpessoa.ctp` (~73k) — formulário do proponente da fiança (carregado via AJAX).
- `analisecadastral.ctp` (~25k) — shell da ficha de análise cadastral (com "login" por senha).
- `buscarpessoaanalise.ctp` (~79k) — formulário do proponente da análise (AJAX).
- `o2sign.ctp` — visualização das assinaturas eletrônicas.
- `segurocondo.ctp` (~61k) — ficha de inspeção predial / seguro condomínio.
- `addproponente.ctp` — variante interna (sem token) da lista de proponentes.
- `index.ctp`, `index_analise.ctp`, `index_segurocondo.ctp` — telas de "obrigado / pendências" pós-envio.
- `pdf_propostafianca.ctp` (~41k) — PDF (Mpdf) da ficha individual.
- `app/View/Validaanalises/index.ctp` — validação de autenticidade de análise.
- `app/View/Validavistorias/index.ctp` — validação de autenticidade de vistoria.
- Layout: `app/View/Layouts/default_painel_cliente.ctp`.

## Telas / fluxos

### 1. Proposta de Seguro Fiança — `proponentefianca.ctp` + `buscarpessoa.ctp`
URL: `/formulariosweb/proponentefianca/{segurofiancaid}/{tipoCotacao}/{acessoexterno}/{token}` (link enviado por e-mail usa `/{id}/1/s/{token}`).
- Coluna esquerda: lista de proponentes (`$datapessoa`); clique chama JS `buscardados(pessoaid, fiancaid, tipocotacao, acessoexterno, token)` → `GET formulariosweb/buscarpessoa/...` injeta o form em `#pessoa`.
- Proponente principal marcado com ícone vermelho. Link "Novo Proponente" (`buscardados(0,...)`) **só aparece** se `Segurofiancas.ativa_form_web==1 AND tipo_pessoa_contratante=='PF'`.
- Se `ativa_form_web==0`: exibe aviso "proposta em processamento" e **bloqueia edição** (form fica read-only no `buscarpessoa.ctp`, linhas ~156-161).
- Botão "Enviar ficha" (só com `ativa_form_web==1`) → `/formulariosweb/enviarproposta/{id}/1/s/{token}`.
- Acordeão "Dados do imóvel/locação" (somente leitura: CEP, endereço, número, bairro/cidade/UF via select readonly, valores aluguel/condomínio/IPTU formatados com `Dataformat::currencyMoeda`).
- Bloco "Dados Complementares" via element `clientesegurofianca_pf_nao_residencial` aparece só quando `tipoContratante=='PF' AND tipoCotacao=='2'` (não residencial).
- Botão flutuante de WhatsApp fixo (telefone `55021990036571` hardcoded).

Form do proponente (`buscarpessoa.ctp`) — campos principais (todos sob `data[Pessoa][...]` / `data[PessoaExtra][...]`):
- `Pessoa.tipo_pessoa` (select PF/PJ, classe `pretendente_tipo_pessoa` controla exibição de blocos PF vs PJ).
- `Pessoa.documento` — CPF/CNPJ "somente números", classe `documento`, `onclick=verificaDocumento('documento',tipoDoc)`; `tipoDoc` derivado de `tipo_pessoa_contratante`.
- PF: `nome, sexo, nacionalidade, tempo_residente, identidade, tipo_identidade, identidade_orgao_emissor, identidade_data_emissao, data_nascimento, qtd_dependentes, filiacao_mae, telefones, telefone_celular, email, estadocivil_id`.
- Cônjuge (bloco `#div_dados_conjuge`, exibido por JS `verificaEstadoCivil()` quando `estadocivil_id` == 2 ou 4): `conj_nome, conj_sexo, conj_documento, conj_data_nascimento, conj_tipo_documento, conj_identidade, conj_identidade_oe, conj_identidade_dt_emissao`, dados de trabalho do cônjuge (`conj_trabalho_*`), endereço (`conj_enderecocom_*`), `conj_compoe_renda`, `PessoaExtra.conjuge_residira_imovel`, `PessoaExtra.conjuge_pagara_aluguel`.
- Endereço residencial: `endereco_cep, endereco, endereco_numero, endereco_complemento, endereco_bairroid, endereco_cidadeid, endereco_uf, tempo_moradia, tipo_moradia, paga_aluguel, imovel_nome_locador, imovel_tels_locador`.
- Renda/trabalho: flags toggle (checkbox classe `badgebox`) `aposentado, publico, clt, autonomo, empresario` (gravados 'on' → 1/0); `tiporendaprincipal_id`, `nome_empresa_trabalha`, `cargo_funcao`, `renda`, `rendas_extras`, `cnpj_empresa_trabalha` (mask 14 dígitos).
- PJ: `nome_empresa, enquadramento_tributario, empresa_fundacao, empresa_capital_social`, e `PessoaExtra.aluguel_atual/condominio_atual/iptu_atual/luz_atual/gas_atual/agua_atual`.
- Tabelas dinâmicas de referências adicionadas via JS `addFormField(input, table)`: bancárias (`PessoaReferenciaBancaria`, tipo 3), comerciais clientes (5), fornecedores (4), pessoais (1), moradores (6), rendas (`PessoaReferenciaRendas`, 7), sócios (`PessoaSocios`).
- Botão "Anexar documentos/arquivos" → `arquivosAnexos('Segurofianca', segurofiancaid, pessoaid, empresaUsuario)` (plugin AjaxMultiUpload); link "Veja a lista de DOCUMENTOS para COMPROVAÇÃO de renda" abre popup `como_comprovar_rendas`.
- Hidden `Pessoa.processa_envio`; campo `Pessoa.id` (0 = novo).

### 2. Ficha de Análise Cadastral — `analisecadastral.ctp` + `buscarpessoaanalise.ctp`
URL: `/formulariosweb/analisecadastral/{analiseid}/{tipoCotacao}/{acessoexterno}/{token}`.
- **"Login" por senha**: a tela inicia com `.container_formulario` oculta e `.container_login` visível. O proponente digita a senha recebida por e-mail; submit faz AJAX para `authFormularioweb/{analiseid}/1/s/{token}` (compara `Clienteanalisecadastral.codigo_formulario_web`). Em sucesso grava `sessionStorage.autenticado='true'` e mostra o formulário. **A liberação é puramente client-side via sessionStorage** (ver Gotchas).
- Mesma estrutura de lista de proponentes da fiança; JS chama `buscarpessoaanalise/...`.
- Blocos white-label/marketing condicionais por `empresa_parceira_id` (116 ou 181 = Renascença): card "Fiança Empresarial", modal `popup-fianca-corp-renascenca`, logo da imobiliária.
- Bloco "Pagamento da Análise" (PagSeguro/transferência) aparece quando `Empresa.flag_permite_cobrar_analise==1 AND Clienteanalisecadastral.flag_permite_cobrar_analise==1`.
- Botão "Enviar ficha" → `/formulariosweb/enviaranalise/{analiseid}/1/s/{token}`.

### 3. Assinatura eletrônica — `o2sign.ctp`
URL: `/formulariosweb/o2sign/{id}/{tipoCotacao}/{acessoexterno}/{token}`. Lista signatários (`SignSigners`), `buscardados(...)` → `formulariosweb/buscarsign/...`. Somente visualização + modal informativo. Dados do imóvel em acordeão. (Método `buscarsign` referenciado pelo JS mas não está entre os métodos listados do controller — ver Gaps.)

### 4. Seguro condomínio / inspeção predial — `segurocondo.ctp`
URL: `/formulariosweb/segurocondo/{id}/{tipoCotacao}/{acessoexterno}/{token}`. Form salva em `FormularioswebSegurocondo` (`Fichainspecaopredial`): `area_construida` (currency→decimal), itens dinâmicos `itemprotecional[]` (sistema protecional) e `itemcobertura[]` (cobertura + valor), recriados via deleteAll+save. Envio final via `enviarpropsegurocondo`; tela "obrigado" = `index_segurocondo.ctp`.

### 5. Validar autenticidade da análise — `Validaanalises/index.ctp`
URL: `/validaanalises/index/{token}/{documento}`. Form `ValidaAnalise` com 2 campos: `analise_token` (texto, classe `required`) e `analise_documento` (CPF/CNPJ, classe `documento required`). POST busca `FichaAnaliseCadastralPessoa` por `documento` + `FichaAnaliseCadastral.token`; flash de sucesso mostra nome da pessoa + data da ficha, ou erro "Análise não encontrada". Selos de segurança GoDaddy/SiteLock e link externo analiseweb.com.br. Validação Parsley no front (`parsley-validate`, `novalidate`).

### 5b. Validar autenticidade da vistoria — `Validavistorias/index.ctp`
URL: `/validavistorias/index/{token}/{documento}`. Mesmo padrão da validação de análise, porém com **um único campo**: `vistoria_token` (texto, classe `required`, value pré-preenchido com `$token` da URL, `strip_tags` aplicado no controller). Form `ValidaVistoria` (`parsley-validate`/`novalidate`), botão "Verificar". POST confere o token da vistoria e retorna flash de sucesso/erro. Há link (oculto `display:none`) para o site da O2 Assessoria Cadastral.

### 6. Telas de retorno pós-envio
- `index.ctp` (fiança) / `index_analise.ctp` (análise): se `$this->params['pass'][4]` (`$texto_retorno`) preenchido → mostra "Atenção! Resolva as pendências" + lista de campos/documentos faltantes; senão "Obrigado! Aguarde a resposta". Botão "Voltar para a Proposta/Proponentes" reconstrói o link com token. `index_segurocondo.ctp` é só "Obrigado!".

## Pontos de entrada (controller::ação que renderiza)
- `FormularioswebController::proponentefianca` → view `proponentefianca` (ou `acesso_invalido`).
- `FormularioswebController::buscarpessoa` → view `buscarpessoa` (layout `ajax`).
- `FormularioswebController::analisecadastral` → view `analisecadastral` (ou `acesso_invalido`).
- `FormularioswebController::buscarpessoaanalise` → view `buscarpessoaanalise` (layout `ajax`).
- `FormularioswebController::o2sign` → view `o2sign` (ou `acesso_invalido`).
- `FormularioswebController::segurocondo` → view `segurocondo` (ou `/Dashboard/proibido`).
- `FormularioswebController::enviarproposta` → redirect `index/...`; `enviaranalise` → redirect `index_analise/...`; `enviarpropsegurocondo` → `index_segurocondo`.
- `FormularioswebController::authFormularioweb` → resposta JSON (não renderiza view).
- `FormularioswebController::index|index_analise|index_segurocondo` → telas de retorno.
- `ValidaanalisesController::index` → view `Validaanalises/index`.
- `ValidavistoriasController::index` → view `Validavistorias/index`.

## Regras de negócio relevantes à UI
- **Acesso válido**: `proponentefianca`/`analisecadastral`/`o2sign` exigem `!empty($datapessoa) AND !empty($token) AND !empty($acessoexterno) AND $acessoexterno=='s'`; caso contrário renderiza `acesso_invalido`. `segurocondo` exige registro casando `id`+`token`, senão renderiza `/Dashboard/proibido`.
- **tipoCotacao é sobrescrito** pelo `Segurofiancas.tipolocacao_id` (1=Residencial, 2=Não residencial) em `proponentefianca` — o valor da URL é ignorado para definir o tipo.
- **Validação de envio** (`validaFianca`, usada por `enviarproposta`/`enviaranalise`): para PF exige sexo, identidade, tipo_identidade, estadocivil_id, filiacao_mae (mãe), data_nascimento (≠ '' e ≠ '00/00/0000'), telefone fixo OU celular, nacionalidade (só fiança), endereço completo (cep/endereço/número/bairro/cidade/uf, só fiança); se casado/união estável (estadocivil 2 ou 4) exige dados do cônjuge; se `conj_compoe_renda==1` exige TODOS os dados do cônjuge (trabalho+endereço); se `tiporendaprincipal_id` 3 ou 4 (CLT/empresário) exige nome_empresa + cargo. Para PJ exige `aluguel_atual`. Além disso exige **pelo menos 1 arquivo anexado** (`qtdArquivos==0` bloqueia). Mensagens concatenadas vão para `$texto_retorno` exibido em `index`/`index_analise`.
- **Conversões** no save (`buscarpessoa`): `Dataformat::currencyToDecimal` para rendas/salários/valores; `Dataformat::dateToSql` para datas; checkboxes 'on'→'1'/'0'.
- **Máscaras (JS `verificaFormPessoa`)**: datas `99/99/9999` (`PessoaDataNascimento`, `PessoaIdentidadeDataEmissao`, `PessoaConjTrabalhoDtAdmissao`, `PessoaConjDataNascimento`, `PessoaConjIdentidadeDtEmissao`, `PessoaEmpresaFundacao`); CNPJ empresa `99999999999999`; campos de moeda usam `MascaraMoeda(this,'.',',',event)` no `onkeypress`. CEP/endereço via `modules/endereco.js` (autocomplete Correios — controller usa `Endereco::getAddressFromCorreiosByCep`).
- **Toggles JS** (`segurofianca.js`): `verificaEstadoCivil` (cônjuge), `verificaModalidadeLocacao` (PF não residencial / tipos de imóvel Loja/Sala/Andar/Prédio quando locação=2), `verificaDiv` (ônus/sócios), `verificaTipoCotacao` (resumido vs completo), `verificaSeguradora`.

## Máquina de estados / status (refletida na UI)
- `Segurofiancas/Clienteanalisecadastral.status_id`: ao enviar (`enviarproposta`/`enviaranalise`) seta `status_id=2` ("enviada"), `data_enviada=now`, `ativa_form_web=0`.
- `ativa_form_web`: **flag mestre de edição** da tela pública. `1` = proponente pode preencher/editar e ver botões "Novo Proponente"/"Enviar ficha"; `0` = tela em modo leitura, exibe "Em processamento / Aguarde o resultado". Após envio bem-sucedido vira 0 (trava reedição).
- Envio dispara e-mail: fiança → `EmailController::emailNotification('seguro_fianca','cliente_concluiu_fianca')`; análise → `('assessoria','cliente_concluiu_analise')`.

## Multi-tenant / white-label
- **Isolamento por token**: cada proposta/análise é acessada por `id`+`token` na URL. As queries sempre adicionam `token` como condicional (`Segurofiancas.token`, `Clienteanalisecadastral.token`, `FormularioswebSegurocondo.token`). `Session->write('token_id_cliente', $token)`.
- Nome da imobiliária/administradora exibido a partir de `Empresa.nome` (`$datapessoaBusca[0]['Empresa']['nome']`).
- **White-label hardcoded por ID de empresa**: `empresaparceira_id/empresa_parceira_id == 116` (e `== 181` na análise) = Renascença → logos específicos (`logos_imobiliarias/logo-renascenca.png`, `logo_renascenca21re.png`), card e modal "Fiança Empresarial". São condicionais hardcoded, não configuráveis.
- Layout `default_painel_cliente.ctp`: rodapé "Powered By HubState", banner `rodape-logos-seguradoras.png`, copyright dinâmico por ano. Menu de usuário (Home/Meu Perfil/Minha Empresa/Sair) só aparece quando `controller !== 'validaanalises'`.

## Gotchas / decisões kept-bug
- **NÃO replicar — falha de validação token/ID (segurança)**: em `enviarproposta` (l.1638) e `enviaranalise` (l.1731) a checagem é `if ($id != $param && $token != $param)` usando **AND**. Como ambos sempre vêm do mesmo registro, a condição praticamente nunca bloqueia; deveria ser `OR` (e o `find` já filtra por token). No destino, validar token+id de forma robusta (rejeitar se não casar).
- **NÃO replicar — "login" por senha apenas client-side**: o gate da `analisecadastral` é `sessionStorage.getItem('autenticado')=='true'`. Qualquer um pode setar isso no navegador e ver o formulário; `authFormularioweb` retorna 200 sem amarrar a sessão server-side. No destino, autenticar/autorizar no backend (token de acesso por análise).
- **NÃO replicar — Auth totalmente desabilitado**: `beforeFilter` faz `$this->Components->disable('Auth')` + `$this->Auth->allow('*')` e dezenas de `allow(...)` redundantes/legados (incl. controllers `clientesegurofianca`, plugin contratoseguros). É acesso público amplo; no destino expor só endpoints públicos específicos com rate-limit.
- **Hardcodes**: telefone WhatsApp `55021990036571`, e-mail de suporte em comentários (`webmaster@o2seguros.com.br`), IDs de empresa 116/181, IDs de tipos de imóvel (21/22/27/24, 2, 1) embutidos no JS.
- `index.ctp`/`index_analise.ctp` leem parâmetros via `$this->params['pass'][0|3|4]` em vez de variáveis `set()` — frágil à ordem da rota.
- `addproponente.ctp` é variante interna (sem token) ainda apontando para `formulariosweb/buscarpessoa/...` sem token — provável legado.
- Muitos `try/catch` engolem exceções com flash genérico (ex.: `proponentefianca` salva e só dá flash de sucesso; `segurocondo` tem o `try/catch` comentado, então erro de save pode quebrar a transação sem rollback explícito).
- `tipoCotacao` vindo da URL é decorativo na fiança (sobrescrito pelo banco) — não usar como fonte de verdade.
- `strip_tags` aplicado a token/documento na validação de análise (sanitização básica) — manter sanitização equivalente.
- "Novo Proponente" só para PF (`tipo_pessoa_contratante=='PF'`); PJ não pode adicionar proponentes pela tela pública.

## Destino (issues Linear)
Projeto **wecorp-frontend**. Sugestão de mapeamento (confirmar milestone F20 "Portal público do proponente"):
- Tela proposta de fiança (lista proponentes + form do proponente PF/PJ + anexos + envio).
- Tela ficha de análise cadastral (com autenticação server-side por token/senha — corrigir kept-bug).
- Tela validação de autenticidade de análise (token + documento).
- Tela assinatura eletrônica (visualização).
- Tela seguro condomínio/inspeção predial.
- Componentes transversais: máscaras (CPF/CNPJ/data/moeda/CEP), autocomplete CEP, toggles condicionais (estado civil/cônjuge/modalidade), white-label por empresa (substituir hardcode 116/181 por config), validação de envio (paridade com `validaFianca`).
