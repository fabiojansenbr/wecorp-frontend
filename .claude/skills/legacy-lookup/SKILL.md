---
name: legacy-lookup
description: Consulta a base de conhecimento do sistema LEGADO da migração weCorp (CakePHP em C:\projetos\wecorp\app, ou appSeg/Lumen em D:\projetos\appSeg no repo services). Use ao implementar uma feature no sistema NOVO e precisar saber COMO o legado fazia algo — regra de negócio, fórmula, fluxo, máquina de estado, integração, e-mail/PDF, multi-tenant. Também gera/atualiza dossiês de domínio em docs/legado/. Aciona via "/legacy-lookup <pergunta>" ou "/legacy-lookup gerar dossiê <domínio>".
---

# Skill: legacy-lookup

Dar acesso **rápido** ao conhecimento do legado **sem poluir o contexto** do agente implementador: a leitura
pesada acontece num **subagente** e só o essencial volta.

## Fonte legada (configurável — NÃO hardcode)
1. Leia a linha **"Fonte legada:"** na seção *Consulta ao legado* do `CLAUDE.md` **deste repo**.
2. Caminho passado como argumento pelo usuário **prevalece**.
3. Defaults: backend/frontend → `C:\projetos\wecorp\app` (CakePHP 2.x; telas em views `.ctp`);
   services → `D:\projetos\appSeg` (Lumen) + `packages/gridweb/*`.

A fonte é **read-only**: **NUNCA** escreva em `C:\projetos\wecorp` nem em `D:\projetos\appSeg`. Dossiês vão em `docs/legado/` deste repo.

## Exclusões (NUNCA ler/indexar)
`**/Vendor/**`, `vendor/**`, `lib/Cake/**`, `*~`, `*-bkp.*`, `*.zip`, `frontend.{html,md}`, backups datados e arquivos `_*`/`_blankAppController`.

## Navegação em god files
Controllers legados chegam a 5.000+ linhas. **Nunca** leia o arquivo inteiro: localize por nome de método
(`function <metodo>(` no CakePHP; `public function <metodo>(` no Lumen) e leia só o trecho relevante + colaboradores.

---

## Modo A — Pergunta pontual (default)
Argumento é uma pergunta (ex.: *"como era a tela de cadastro de empresa e suas validações?"*):

1. **Cheque o dossiê primeiro.** Se já existe `docs/legado/<domínio>.md` que cobre a dúvida, responda a partir dele (barato) e pare.
2. Senão, **lance UM subagente `Explore`** com escopo na fonte legada (respeitando exclusões), pedindo:
   - localizar controller/model/view `.ctp` relevantes (navegar god files por método);
   - extrair o **fluxo/comportamento concreto** com `arquivo::método` e **trechos curtos**, sem despejar arquivos;
   - identificar validações, máscaras, comportamento por grupo, integrações e gotchas.
3. Devolva uma resposta **concisa** (pontos de entrada + comportamento + gotchas). Ao final, **ofereça persistir**
   o achado como dossiê novo ou seção de um dossiê existente (Modo B).

## Modo B — Gerar/atualizar dossiê de um domínio
Argumento é *"gerar dossiê <domínio>"* (ou parte do build da base):

1. Liste os controllers/plugins/telas do domínio na `../backend/docs/parity-matrix.md` (referência) ou pelo módulo correspondente.
2. Subagente lê os **pontos de entrada** desses arquivos e preenche o **template** abaixo (trechos curtos, não dumps).
3. Grave em `docs/legado/<dominio>.md`. Se passar de ~500 linhas, **quebre** em `docs/legado/<dominio>/<sub>.md`.
4. Atualize `docs/legado/README.md` (índice).

### Template do dossiê
```markdown
# Legado — <Domínio>

> Resumo em 2–3 linhas do que o domínio faz no legado e onde se encaixa na migração (fase/issues).

## Cobertura
- Arquivos legados cobertos: `Controller`, `Model`, `Plugin`, view(s) `.ctp`.

## Pontos de entrada
- `Arquivo::método()` — o que faz; fluxo de chamadas resumido.

## Telas / fluxos (frontend)
- Tela `.ctp`, campos, validações, máscaras, comportamento por grupo, navegação.

## Regras de negócio
## Máquina de estados / status
## Integrações
## Multi-tenant (como empresa_id / saas_id entra)
## Gotchas / decisões kept-bug (o que NÃO replicar)
## Destino (milestone/issues Linear + dossiê(s) relacionados)
```

---

## Regras de qualidade
- **Equivalência/paridade funcional:** documente o comportamento **real** (inclusive bugs relevantes), marcando o que **não** deve ser replicado.
- **Tamanho:** dossiê skimmável (~≤300–500 linhas), resumo no topo; split se maior.
- **Precisão:** sempre cite `arquivo::método`; trechos curtos, não arquivos inteiros.
- **Isolamento:** a exploração roda em subagente — ao agente principal volta só a síntese.
