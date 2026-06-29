# 13 — Testes

> Estratégia de testes do frontend weCorp. Regra constitucional ([`../CLAUDE.md`](../CLAUDE.md) §2 / §10): **`hasAccess` sem cobertura de TODOS os branches, schema Zod sem teste de sucesso + CADA erro PT-BR, e fluxo crítico sem e2e não está "pronto".**
>
> Cross-links: [01 — Arquitetura](./01-arquitetura.md) (camadas testáveis), [07 — Auth/permissões](./07-auth-sessao-permissoes.md) (`hasAccess`, JWT), [08 — Consumo de API](./08-consumo-api-dados.md) (envelope/erros), [10 — Formulários](./10-formularios-validacao.md) (RHF + Zod), [14 — Deploy e ambiente](./14-deploy-ambiente.md) (CI + Husky).
>
> **Mapeamento Linear:** F19 — Qualidade, Testes e Deploy: **HUB-307** (suíte Vitest + Testing Library), **HUB-308** (MSW + e2e Playwright dos fluxos críticos).

---

## 1. Pirâmide de testes

A arquitetura em camadas ([01](./01-arquitetura.md)) existe para ser testável: a **lógica pura** (schemas, `hasAccess`, mappers) vive fora da UI e é o alvo predominante; **componentes** validam comportamento de form/estado; **e2e** garante que o fluxo real funciona ponta a ponta no browser.

| Tipo | Local | Ferramentas | O que cobre |
|------|-------|-------------|-------------|
| **Unitário (lógica pura)** | `src/**/*.test.ts` | **Vitest** | Schemas Zod, `hasAccess` (todos os branches), mappers/helpers de `shared/jwt/`, máscaras, `resolveErrorMessage` |
| **Componente** | `src/**/*.test.tsx` | Vitest + **Testing Library** + `user-event` | Forms (RHF + Zod), estados loading/empty/error, dialogs, render condicional (`<If>`), `PermissionGate` |
| **Integração (HTTP isolado)** | componente + **MSW** | Vitest + Testing Library + **MSW** | Hook → action → `api` contra handlers MSW (sucesso + erros do envelope), sem backend real |
| **E2E** | `e2e/**/*.spec.ts` | **Playwright** | Fluxos críticos no browser real: login, análise cadastral, contratação de seguro, vistoria |
| **Acessibilidade** | junto ao componente/e2e | `vitest-axe` / `@axe-core/playwright` | Violações WCAG 2.1 AA por tela/fluxo |

Predomínio de **unitários e de componente** (rápidos, isolam regra de borda e UX); **e2e** poucos e bem escolhidos (lentos, mas provam o fluxo real). **MSW** é o limite entre componente e e2e: o teste exercita as camadas `hook → action → api → fetcher` sem rede real.

---

## 2. Matriz de cobertura por camada

Para **cada** camada, a suíte cobre no mínimo:

| Camada | Cobertura obrigatória | Ferramenta |
|--------|-----------------------|------------|
| **Schemas Zod** | `safeParse` de payload válido (**sucesso**) + **CADA mensagem de erro PT-BR** (um caso por regra: obrigatório, formato, min/max, CPF/CNPJ inválido) | Vitest |
| **`hasAccess`** | **TODOS os branches** — sem sessão, sem requirement, grupo permitido, grupo negado, módulo/ação ausente, role match/no-match (**obrigatório**, §2 CLAUDE) | Vitest |
| **Mappers/helpers `shared/jwt/`** | decode de token válido, token expirado/malformado, extração de claims (`userId`, `empresaId`, `grupoId`), defaults | Vitest |
| **Helpers/máscaras** | CPF/CNPJ/CEP/telefone/moeda — entrada → saída formatada e parse reverso | Vitest |
| **`resolveErrorMessage`** | mapeamento por `code`/`status` (400/401/403/404/422/500) → mensagem PT-BR; fallback padrão | Vitest |
| **Componentes de form** | render, validação client (erros PT-BR aparecem), submit feliz, submit com erro de API (toast), loading/disabled | Testing Library + MSW |
| **Estados de tela** | loading (skeleton), empty, error | Testing Library |
| **E2E (fluxos críticos)** | login, análise cadastral, contratação de seguro, vistoria | Playwright |
| **Acessibilidade** | zero violações `axe` nos fluxos críticos | axe |

> **`hasAccess` é o teste mais importante do frontend.** É a função pura que decide exibição por permissão dos 10 grupos. Cobertura de **todos os branches** é condição de merge — ver [07 — Auth/permissões](./07-auth-sessao-permissoes.md).

---

## 3. Schemas Zod — sucesso + cada erro PT-BR

Regra: para cada schema, um teste de **sucesso** e **um teste por mensagem de erro**. As mensagens são contrato de UX (PT-BR) — não podem regredir.

```typescript
// src/app/(modules)/(pessoas)/_business/schemas/pessoa/pessoa.test.ts
import { describe, it, expect } from 'vitest';
import { pessoaSchema } from './index';

describe('pessoaSchema', () => {
  it('sucesso: payload válido passa', () => {
    const result = pessoaSchema.safeParse({ nome: 'Maria', cpf: '390.533.447-05', email: 'maria@x.com' });
    expect(result.success).toBe(true);
  });

  it('erro: nome obrigatório (PT-BR)', () => {
    const result = pessoaSchema.safeParse({ nome: '', cpf: '390.533.447-05', email: 'maria@x.com' });
    expect(result.success).toBe(false);
    if (!result.success) expect(result.error.issues[0].message).toBe('Campo obrigatório');
  });

  it('erro: CPF inválido (PT-BR)', () => {
    const result = pessoaSchema.safeParse({ nome: 'Maria', cpf: '111.111.111-11', email: 'maria@x.com' });
    expect(result.success).toBe(false);
    if (!result.success) expect(result.error.issues[0].message).toBe('CPF inválido');
  });

  it('erro: e-mail inválido (PT-BR)', () => {
    const result = pessoaSchema.safeParse({ nome: 'Maria', cpf: '390.533.447-05', email: 'nao-email' });
    expect(result.success).toBe(false);
    if (!result.success) expect(result.error.issues[0].message).toBe('Informe um e-mail válido');
  });
});
```

---

## 4. `hasAccess` — todos os branches (obrigatório)

```typescript
// src/shared/permissions/has-access.test.ts
import { describe, it, expect } from 'vitest';
import { hasAccess } from './index';
import { ModulesPermissionEnum, ActionsPermissionEnum, RolesPermissionEnum } from '@/shared';

const baseSession = {
  userId: 'u1',
  empresaId: 'emp-1',
  grupoId: RolesPermissionEnum.IMOBILIARIA,
  permissions: [{ module: ModulesPermissionEnum.ANALISES, action: ActionsPermissionEnum.VIEW }],
};

describe('hasAccess', () => {
  it('sem sessão => false', () => {
    expect(hasAccess(null, { module: ModulesPermissionEnum.ANALISES, action: ActionsPermissionEnum.VIEW })).toBe(false);
  });

  it('sem requirement => true (rota livre para autenticado)', () => {
    expect(hasAccess(baseSession, undefined)).toBe(true);
  });

  it('módulo + ação permitidos => true', () => {
    expect(hasAccess(baseSession, { module: ModulesPermissionEnum.ANALISES, action: ActionsPermissionEnum.VIEW })).toBe(true);
  });

  it('ação ausente no módulo => false', () => {
    expect(hasAccess(baseSession, { module: ModulesPermissionEnum.ANALISES, action: ActionsPermissionEnum.DELETE })).toBe(false);
  });

  it('módulo ausente => false', () => {
    expect(hasAccess(baseSession, { module: ModulesPermissionEnum.FINANCEIRO, action: ActionsPermissionEnum.VIEW })).toBe(false);
  });

  it('requirement por role: grupo correto => true / grupo errado => false', () => {
    expect(hasAccess(baseSession, { role: RolesPermissionEnum.IMOBILIARIA })).toBe(true);
    expect(hasAccess(baseSession, { role: RolesPermissionEnum.SEGURADORA })).toBe(false);
  });
});
```

> Cada `it` cobre **um branch**. Adicionar um branch novo em `hasAccess` sem teste correspondente **MUST NOT** mesclar.

---

## 5. Componentes de form (Testing Library + MSW)

O componente é testado pelo **comportamento**, não pelo markup: o usuário digita, valida, submete; a tela reage (toast, loading, fechar dialog). O HTTP é interceptado por **MSW** — sem backend real, sem `fetch` mockado à mão.

```typescript
// src/test/msw/server.ts
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';
import { env } from '@/configs/env';

export const server = setupServer(
  http.post(`${env.NEXT_PUBLIC_API_URL}/pessoas`, async ({ request }) => {
    const body = (await request.json()) as { nome?: string };
    if (!body.nome) return HttpResponse.json({ error: { code: 'VALIDATION', message: 'inválido' } }, { status: 400 });
    return HttpResponse.json({ data: { id: 'p1', nome: body.nome } }, { status: 201 });
  }),
);
```

```typescript
// src/test/setup.ts — registrado em vitest.config.ts (setupFiles)
import '@testing-library/jest-dom/vitest';
import { afterAll, afterEach, beforeAll } from 'vitest';
import { server } from './msw/server';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

```tsx
// DialogCreatePessoa.test.tsx
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { DialogCreatePessoa } from './index';

describe('DialogCreatePessoa', () => {
  it('happy path: cria pessoa e exibe sucesso', async () => {
    const user = userEvent.setup();
    render(<DialogCreatePessoa open />);
    await user.type(screen.getByLabelText(/nome/i), 'Maria');
    await user.click(screen.getByRole('button', { name: /salvar/i }));
    expect(await screen.findByText(/cadastro realizado/i)).toBeInTheDocument();
  });

  it('validação: nome vazio mostra erro PT-BR e não submete', async () => {
    const user = userEvent.setup();
    render(<DialogCreatePessoa open />);
    await user.click(screen.getByRole('button', { name: /salvar/i }));
    expect(await screen.findByText('Campo obrigatório')).toBeInTheDocument();
  });
});
```

---

## 6. E2E — fluxos críticos (Playwright)

E2E sobe a aplicação real e exercita os fluxos que **não podem quebrar**. Quatro são obrigatórios:

| Fluxo crítico | O que o e2e verifica |
|---------------|----------------------|
| **Login** | autenticação, gravação do cookie httpOnly, redirect para home; sessão inválida volta ao `/sign-in` |
| **Análise cadastral** | criar ficha → adicionar pessoa → submeter → ver status/parecer |
| **Contratação de seguro** | cotação → preencher proposta → enviar → confirmar estado |
| **Vistoria** | abrir vistoria → editar itens/fotos → finalizar (respeitando lock) |

```typescript
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';

test('login redireciona para o painel', async ({ page }) => {
  await page.goto('/sign-in');
  await page.getByLabel(/e-mail/i).fill('user@empresa.com');
  await page.getByLabel(/senha/i).fill('senha-valida');
  await page.getByRole('button', { name: /entrar/i }).click();
  await expect(page).toHaveURL(/\/dashboard/);
});
```

> Em e2e, o backend pode ser **real (ambiente de teste)** ou **MSW/route mocks** conforme o fluxo. Provedores externos (Serasa, Docusign, WBS) são sempre **mockados**. Determinismo: sem dependência de relógio/rede reais.

---

## 7. Acessibilidade (axe)

WCAG 2.1 AA é objetivo do produto ([00](./00-visao-geral.md) §3). Cada fluxo crítico roda `axe` e **MUST** ter zero violações.

```typescript
// componente
import { axe } from 'vitest-axe';
it('sem violações de acessibilidade', async () => {
  const { container } = render(<FormPessoa />);
  expect(await axe(container)).toHaveNoViolations();
});

// e2e (Playwright)
import AxeBuilder from '@axe-core/playwright';
test('análise cadastral acessível', async ({ page }) => {
  await page.goto('/analises');
  const results = await new AxeBuilder({ page }).withTags(['wcag2a', 'wcag2aa']).analyze();
  expect(results.violations).toEqual([]);
});
```

---

## 8. Convenções de teste

- **Arrange / Act / Assert** explícito em cada teste; um comportamento por `it`.
- **Mock do `api`/HTTP via MSW**, nunca `vi.fn()` ad-hoc espalhado — o handler MSW é a fonte única de resposta simulada. Provedores externos sempre mockados.
- **Fixtures e factories** (`src/test/factories.ts`): funções que produzem sessão, pessoa, ficha válidas com overrides (`makeSession({ grupoId })`). Preferir factories a objetos estáticos.
- **Sessão de teste:** `makeSession()` produz claims reais (`userId`, `empresaId`, `grupoId`, `permissions`) para exercitar `hasAccess` e `PermissionGate` de verdade.
- **Queries acessíveis primeiro:** `getByRole`/`getByLabelText` antes de `getByTestId` — espelha como o usuário (e o leitor de tela) percebe a tela.
- **Erros por `code`/`status`**, nunca por texto da mensagem do backend (espelha a regra de [08](./08-consumo-api-dados.md)).
- **Nomear regressões** com o issue: `it('regressão HUB-XXX: grupo X não vê botão excluir', ...)`.

---

## 9. O que NÃO testar

- **Markup trivial** (um `<Card>` que só renderiza children, label estático) — sem lógica, sem teste.
- **Componentes do shadcn/ui** não customizados — já testados upstream.
- **Estilos/Tailwind** (classes CSS, cor da marca) — não é teste de unidade; white-label é validado visualmente.
- **Regra de negócio crítica do backend** (cálculo de prêmio, IOF, máquina de estado) — é testada no backend ([backend/docs/12-testes.md](../../backend/docs/12-testes.md)); o frontend testa a **borda** (validação Zod, exibição por permissão, consumo do contrato).
- **Implementação interna** (estado de um `useState`) — testar o **comportamento observável**, não detalhes.

---

## 10. Integração no CI + Husky

```bash
yarn test            # unitários + componente (Vitest), inclui MSW
yarn test:watch      # Vitest em watch
yarn test:cov        # cobertura — relatório em coverage/
yarn test:e2e        # Playwright (e2e dos fluxos críticos)
```

- **Husky pre-commit** roda `yarn lint` (rápido, barra erro óbvio antes do commit). Ver [14 — Deploy](./14-deploy-ambiente.md) §7.
- **CI (GitHub Actions)** em cada PR roda `yarn lint && yarn test && yarn build` e, em job dedicado, `yarn test:e2e` (Playwright com browsers instalados). PR com teste vermelho **não** mescla — ver [14](./14-deploy-ambiente.md) §6.
- Cobertura de `hasAccess` (todos os branches) e dos schemas é **gate**: queda derruba o PR.

---

## Ver também

- [01 — Arquitetura](./01-arquitetura.md) — camadas testáveis (lógica fora da UI).
- [07 — Auth/permissões](./07-auth-sessao-permissoes.md) — `hasAccess`, sessão, JWT.
- [08 — Consumo de API](./08-consumo-api-dados.md) — envelope/erros por `code`/`status`.
- [10 — Formulários e validação](./10-formularios-validacao.md) — RHF + Zod (mensagens PT-BR).
- [14 — Deploy e ambiente](./14-deploy-ambiente.md) — CI, Husky, scripts.
</content>
</invoke>
