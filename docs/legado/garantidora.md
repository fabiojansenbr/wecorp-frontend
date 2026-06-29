# Legado (telas) — Garantidora (F14)

> Plugin `Garantidora` = módulo de **faturamento "Upgrade Garantidora"**: agrega parcelas de seguros-fiança (`seguros_parcelas`) por imobiliária/empresa parceira, gera fatura consolidada e extrato/PDF. As telas vivas são **Parcelas** (relatório + gerar fatura + extrato PDF) e **Faturas** (lista + editar/baixar). O `GarantidoraController` (3033 linhas) é um **clone morto** do fluxo de Seguro Fiança (precadastro/addproposta/addproponente) — não possui views próprias no plugin e a navegação aponta para `contratoseguros/segurofiancas/*`. O "proponente público" real é servido pelo controller `Formulariosweb` (app principal), não por este plugin.

## Cobertura
Views .ctp cobertas (plugin `Garantidora/View`):
- **Parcelas/**: `index.ctp`, `gerar_fatura.ctp`, `imprimir_fatura.ctp` (PDF) — vivas. `view.ctp`, `add.ctp`, `edit.ctp`, `edita_item_fatura.ctp` — scaffold morto.
- **Faturas/**: `index.ctp`, `edita_item_fatura.ctp` — vivas. `view.ctp`, `add.ctp`, `edit.ctp`, `imprimir_fatura.ctp` — scaffold morto.
- **Elements/**: `gerenciar.ctp` — abas (Cotação / Proponente) do clone seguro-fiança; aponta para `contratoseguros/segurofiancas`.

Controllers cobertos (só p/ entender a tela):
- `Garantidora/Controller/ParcelasController.php` (502 linhas): `index`, `gerar_fatura`, `imprimir_fatura`.
- `Garantidora/Controller/FaturasController.php` (216 linhas): `index`, `edita_item_fatura`.
- `Garantidora/Controller/GarantidoraController.php` (clone seguro-fiança; só mapeado, ver Gotchas).
- `AppController::geraNossoNumero` (app/Controller/AppController.php:763) — gerador de "nosso número".
- `Metadata/Libmetadados.php` — mapas de status/forma de pagamento usados nos selects.

## Telas / fluxos

### 1. Parcelas/index.ctp — "Gerenciar Parcelas x Faturas" (entrada principal)
Tela de relatório: lista, **agrupada por empresa parceira**, as parcelas em aberto a faturar no período. Filtros num accordion ("Opções de filtros e pesquisas", aberto por padrão, `in`):
- **Data Inicial** (`from`) e **Data final** (`to`) — texto; o JS `ui.js` aplica jQuery UI datepicker: `from` formato `dd/mm/yy`, `to` formato `mm/yy`. O botão OK (`#btnSearchParcela`) começa **desabilitado** e só habilita quando ambas as datas estão preenchidas (`btnDisabled()` em `Garantidora/webroot/js/ui.js`).
- **Tombamento** (`tombamento`) — select fixo no .ctp: `2=Todos, 0=Não, 1=Sim`, default `2`. (Tombamento = garantias migradas de outra carteira.)
- **Status Parcelas** (`status_parcela`) — select `statusFaturaParcelas` (Libmetadados `status_fatura_parcelas`), default `1` (Aberta).
- **Imobiliária/Empresa** — input texto `data[User][usuario_empresa_id]` (autocomplete de empresa), **só exibido p/ grupos 1,2,3,4,9** (admin/backoffice). Para outros grupos não aparece (escopo travado pela própria empresa).
- Form usa helper `Search` (não `Form`) — `$this->Search->create()` / `->end()`.

Comportamento: a listagem só é renderizada **após POST** (no GET inicial a tabela fica vazia; `index()` só popula `$parcelas` dentro do `if is post/put`). Colunas: Empresa (nome + ID), Dia fechamento (`Empresa.dia_vencimento_fatura`), Garantias (`totalGarantiasEmpresa`), Parcelas (`totalParcelasEmpresa`), Total (`vlr_parcela_total_empresa`, `R$ number_format(...,2,',','.')`), Período (mostra **`date('m/Y')` fixo** — sempre o mês atual, não o filtro). Ações por linha:
- Ícone **barcode** → `gerar_fatura/{empresa_id}/{data_de}/{data_ate}/{tombamento}` (target `_blank`).
- Ícone **print** → `imprimir_fatura/{empresa_id}/{status_parcela}/{tombamento}/{data_de}/{data_ate}` ("Gerar Extrato", `_blank`).

Janela de vencimento (controller `ParcelasController::index`): parcelas com `dtvencimento >= data_de` E `<= CONCAT(data_ate, '-', Empresa.dia_vencimento_fatura)` — `data_ate` é convertido para `YYYY-MM` (`substr`), o dia vem do cadastro de cada empresa.

### 2. Parcelas/gerar_fatura.ctp — confirmar/gerar a fatura consolidada
Aberta em nova aba a partir do barcode da index. Cabeçalho: empresa/cliente, CPF/CNPJ (`Dataformat::formatarDocumento`). Tabela lista cada parcela do período (Locatário+documento, Imóvel endereço/bairro/cidade/uf/cep, Parcela `num/total`, Vencimento `Dataformat::dateBr`, Valor `Dataformat::currencyMoeda`). Cada linha grava `data[Parcelas][parcela_id][]` (hidden) para vincular as parcelas à fatura. Rodapé: Subtotal (`array_sum($valor)`), Valor descontos, **Total a pagar** (`Parcelas[0][0]['vlr_parcela_total']`).

Form de geração (`Form->create('Parcelas', action '/gerar_fatura/')`, `novalidate`, parsley):
- `valor` (pré-preenchido com soma, máscara `MascaraMoeda` no keypress, required).
- `valor_desconto` (máscara moeda, `onblur=calculaValorFatura()`, required).
- `valor_total` (máscara moeda, required; pré = vlr_parcela_total).
- `data_vencimento` (datepicker, required; pré = `{dia_vencimento_fatura}/{mês atual}/{ano atual}`).
- `forma_pagamento` (select `formaPagto` = `forma_pagto_todos`, required).
- `status_fatura` (select `statusFatura` = `status_fatura`, required).
- `obs_fatura` (textarea).
- hiddens: `empresaparceira_id`, `user_id` (usuário logado), `empresa_id` (empresa do usuário logado).
- Submit "Gerar fatura" (`data[submit]`).

Ao salvar (`ParcelasController::gerar_fatura` POST): cria registro em `fatura`, faz loop em `parcela_id[]` setando `Parcela.fatura_id` (vincula), força `tipo_servico='upgradegarantia'`, gera `nosso_numero` via `geraNossoNumero(empresa_id, fatura_id, banco_id=8)`, flash com o ID da fatura e `redirect($this->referer())`. JS carregado: `modules/financeiro.js` (cálculo do total/desconto).

### 3. Parcelas/imprimir_fatura.ctp — extrato em PDF (mPDF)
HTML standalone (`$this->layout=null`) renderizado pelo componente `Mpdf`. Logo fixo `logo-upgrade-garantia-locaticia.png` (ignora `Empresa.url_logotipo`, ver Gotchas). Tabela por parcela: REF (`referencia_contrato_locacao`), Locatário+doc, Imóvel, Parcela `num/total`, Período (`Dataformat::mesAno`), Valor. Subtotal, desconto, **Total a pagar**. Se `Fatura.status_fatura==3` exibe faixa vermelha **"ATENÇÃO, FATURA PAGA"**. Bloco "Dados para pagamento" tem dados bancários comentados (PIX/Inter ocultos via `display:none`). Mensagem fixa: encargos de 2% + juros 1% ao mês após o vencimento. Rodapé mPDF: "Powered By Plataforma HubState". Filename: `fatura_Upgrade_ID-{dmYHi}_{empresa_id}.pdf`.

### 4. Faturas/index.ctp — "Gerenciar Faturas"
Lista de faturas já geradas. `FaturasController::index` força `Fatura.tipo_servico='upgradegarantia'` (só faturas do módulo) e `limit=5`/página. Filtros (helper `Search`): Código (`id`), Data Inicial/Final (datepicker), Status (`status_fatura`), Imobiliária/Empresa (texto, só grupos 1,2,3,4,9). **Atenção**: o controller só aplica o filtro `status_fatura` no POST; os demais campos do form não são tratados (ver Gotchas).
Colunas: id, Empresa (nome + `empresaparceira_id`), Valor (`currencyMoeda`), Desconto, Juros, Vencimento, Status (`statusFatura[...]`), Data baixa, Hora baixa, Nosso número, Criação. Ações:
- print → `../parcelas/imprimir_fatura/{empresaparceira_id}/{status_fatura}` (extrato).
- barcode → `imprimir_boleto/{id}` (**ação inexistente no controller** — link morto).
- edit → `edita_item_fatura/{id}`.
- delete (`postLink`) — **comentado/oculto** no markup.

### 5. Faturas/edita_item_fatura.ctp — editar/baixar fatura
Aviso fixo: "Caso a Fatura seja 'paga' ou 'cancelada' também fará o mesmo com os itens financeiros relacionados!". Mostra Código da fatura e Empresa/Cliente (read-only). Campos: `valor` (máscara moeda, required), `valor_desconto` (máscara, `onblur=verificaValorFatura()`), `valor_total` (readonly, máscara), `nosso_numero`, `data_vencimento` (datepicker), `forma_pagamento` (select required), `status_fatura` (select required), `obs_fatura`. Se `status_fatura==3` exibe bloco "Fatura marcada como paga" com data/hora baixa. Botão **"Processar"** (`data[submit]`). Link **"Excluir fatura"** (`postLink` → `delete`) só para grupos 1 e 2 (a ação `delete` também não existe no controller → link morto).

Lógica de baixa (`FaturasController::edita_item_fatura` POST): recalcula `valor_total = valor - valor_desconto`, grava `data_baixa=date('Y-m-d')` e `hora_baixa=date('H:i')` **sempre** (em qualquer save). Se `status_fatura==3` (Paga) → `Parcela.updateAll(status=2)` para `fatura_id`. Se `status_fatura==4` (Cancelada) → `Parcela.updateAll(status=3)`. JS: `modules/financeiro.js`.

### 6. Elements/gerenciar.ctp — abas do clone Seguro Fiança (provavelmente não usado)
Header "Seguro Fiança e Garantias", breadcrumb p/ `contratoseguros/.../index`. Duas abas: (1) "Cotação Seguro Fiança" → `contratoseguros/segurofiancas/addproposta/{id}`; (2) "Proponente(s)" → `contratoseguros/segurofiancas/addproponente/...`. Aba 2 escondida por uma condição com bug (`grupo!=5 || !=6 || ...` sempre verdadeiro — ver Gotchas). Campos read-only: imobiliária parceira, usuário logado; hiddens user_id/empresa_id.

## Pontos de entrada (controller::ação que renderiza)
- `Garantidora.Parcelas::index` → `Parcelas/index.ctp` (relatório, só lista no POST).
- `Garantidora.Parcelas::gerar_fatura($empresa_id,$data_de,$data_ate,$tomb)` → `gerar_fatura.ctp` (e grava fatura no POST).
- `Garantidora.Parcelas::imprimir_fatura($empresa_id,$status_id,$tomb,$data_de,$data_ate)` → `imprimir_fatura.ctp` (PDF mPDF).
- `Garantidora.Faturas::index` → `Faturas/index.ctp`.
- `Garantidora.Faturas::edita_item_fatura($id)` → `edita_item_fatura.ctp` (edição/baixa).
- **Sem ação correspondente** (views órfãs / scaffold): Faturas `add/edit/view/imprimir_fatura`, Parcelas `add/edit/view/edita_item_fatura`; links `imprimir_boleto`, `delete`.
- "Proponente público" → **fora deste plugin**: `Formulariosweb::proponentefianca` / `addproponente` (app/View/Formulariosweb/proponentefianca.ctp, addproponente.ctp), acessado por token. `GarantidoraController::clienteproponente($id,$tipo,$acessoexterno,$token)` é o clone que valida token + `acessoexterno=='s'` e renderiza `addproponente`/`acesso_invalido` (views inexistentes no plugin → morto).

## Regras de negócio relevantes à UI
- **Período de faturamento por empresa**: o "dia de fechamento" não é global — vem de `Empresa.dia_vencimento_fatura`; a query monta o limite superior como `CONCAT(data_ate, '-', dia_vencimento_fatura)`. A UI exibe esse dia na coluna "Dia fechamento".
- **Vínculo parcela→fatura**: gerar_fatura preenche `seguros_parcelas.fatura_id` para as parcelas selecionadas; só entram parcelas com `status=1` (Aberta) no período e do `empresaparceira_id`.
- **Cascata de status fatura→parcelas** (edita_item_fatura): Paga(3) marca parcelas como `status=2`; Cancelada(4) marca `status=3`. Não há reversão automática se mudar de volta.
- **Escopo multi-tenant**: filtro "Imobiliária/Empresa" e `VerificaPermissao()['nivel']==2` restringem por `empresa_id` do usuário (grupos não-admin só veem a própria carteira).
- **Nosso número**: `{ano2dígitos}{fatura_id zero-pad 4}{'01'}` (geraNossoNumero) — banco fixo `8`, sufixo boleto fixo `'01'`.
- **Máscaras/formatos**: moeda via `MascaraMoeda(this,'.',',',event)` no front e `Dataformat::currencyToDecimal` (entrada) / `Dataformat::currencyMoeda` (saída) no back; datas `Dataformat::dateToSql` ↔ `Dataformat::dateBr`; período `Dataformat::mesAno`.

## Máquina de estados / status (refletida na UI)
- **status_fatura** (`Libmetadados::status_fatura`): `1=Não faturada, 2=Pendente, 3=Paga, 4=Cancelada`. (`5=Pendente de Pagamento` comentado.)
- **status_fatura_parcelas** (parcela): `1=Aberta, 2=Faturada, 3=Paga, 4=Cancelada`. (`5=A pagar` comentado.)
- **status_pagamento**: `1=Aberto, 2=Pago, 3=Cancelado`.
- **tombamento** (filtro UI): `2=Todos, 0=Não, 1=Sim`.
- **forma_pagto_todos** (select forma pagamento): `1=Boleto bancário, 2=Depósito, 3=Transferência, 5=Faturado`.
- Transições refletidas na tela: fatura Paga(3) → parcelas Faturada(2); fatura Cancelada(4) → parcelas Paga(3) (mapeamento aparentemente invertido — ver Gotchas).

## Multi-tenant / white-label
- Marca fixa "Upgrade Garantidora" / domínio `upgradegarantidora.com.br` hard-coded no extrato PDF e títulos das telas — **não** é white-label por tenant. O PDF ignora `Empresa.url_logotipo` e sempre usa `logo-upgrade-garantia-locaticia.png` (linha comentada que usaria o logo do tenant).
- Isolamento por empresa: via `VerificaPermissao` (nível 2) e filtro de empresa só para grupos admin/backoffice (1,2,3,4,9). `FaturasController::index` ainda restringe `Fatura.tipo_servico='upgradegarantia'`.

## Gotchas / decisões kept-bug
- **GarantidoraController é clone morto**: replica `precadastro/addproposta/addproponente/parecer/clienteproponente/buscarpessoa/pdf_propostafianca` do Contratoseguros, mas **não há views** correspondentes no plugin e a navegação (`gerenciar.ctp`) aponta para `contratoseguros/segurofiancas/*`. **NÃO replicar** — o fluxo de proponente/fiança vivo é o do plugin Contratoseguros + `Formulariosweb` (público). Migrar apenas Parcelas+Faturas.
- **Views scaffold órfãs**: Faturas `add/edit/view/imprimir_fatura` e Parcelas `add/edit/view/edita_item_fatura` são bake leftovers (cabeçalho `<!--?php debug($schema)?-->`, breadcrumb "Manutenção", `faturas index/Financeiro`) sem ação no controller. **Não migrar.**
- **Links mortos** na Faturas/index: `imprimir_boleto` e `delete` (e o "Excluir fatura" do edita_item) chamam ações inexistentes → erro 404 se clicados. Boleto nunca foi implementado (forma "Boleto" marcada como "ainda não disponível" no Libmetadados).
- **Coluna "Período" sempre mês atual**: index mostra `date('m/Y')` fixo, ignorando o intervalo filtrado.
- **`data_baixa`/`hora_baixa` gravados em todo save** de `edita_item_fatura`, mesmo quando a fatura não foi marcada como paga. Comportamento real do legado.
- **Cascata de status invertida (suspeita)**: fatura Paga(3) seta parcela=2 (Faturada, não Paga=3) e fatura Cancelada(4) seta parcela=3 (Paga). Validar regra correta na migração; **não copiar cegamente**.
- **Faturas/index só filtra status**: Código/Datas/Empresa existem no form mas o controller não os usa (só `status_fatura` no POST). Datepicker de Data no índice é cosmético.
- **Aba condicional sempre verdadeira** (`gerenciar.ctp` linha 75): `($grupo!=5 || $grupo!=6 || ...)` é tautologia → aba Proponente nunca esconde. Bug irrelevante (element provavelmente não usado).
- **SQL bruto extenso** em `ParcelasController::index`/`gerar_fatura`/`imprimir_fatura` com subqueries de soma/contagem e concatenação de strings (risco de injeção via `$empresa_id`/`$tomb` interpolados). Reescrever com query builder/ORM e parâmetros na migração — **não portar o SQL string-concat**.
- **`Auth` desabilitado/comentado**: `GarantidoraController::beforeFilter` tem todo o controle de Auth comentado; `ParcelasController::beforeFilter` tem `VerificaPermissao` comentado (sem checagem de permissão). Garantir autorização adequada no destino.

## Destino (issues Linear)
- Projeto **wecorp-frontend** (Next.js), fase F14 "Garantidora (telas)". Telas a recriar: (1) Relatório de Parcelas a faturar (filtros período/empresa/tombamento/status, agrupado por empresa); (2) Gerar fatura consolidada (seleção de parcelas + desconto + forma pagto + status); (3) Lista de Faturas (filtros + ações); (4) Editar/baixar Fatura (cascata de status para parcelas); (5) Extrato PDF (gerar no backend wecorp-services). Backend correspondente: wecorp-backend (fatura, seguros_parcelas, nosso_numero) / wecorp-services (geração de PDF/boleto). Descartar GarantidoraController clone e views scaffold.
