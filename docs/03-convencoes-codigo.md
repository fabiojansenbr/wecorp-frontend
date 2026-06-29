# 03 — Convenções de Código

> Padrões de escrita de código do **frontend weCorp** (Next.js 16 + React 19 + TypeScript strict).
> Este documento é a referência prática para **nomear, estruturar, importar e revisar** código. As regras não-negociáveis estão na [constituição](../CLAUDE.md) (§2, §4, §7); aqui detalhamos o **como**, com exemplos.
> Cross-links: [01 — Arquitetura](./01-arquitetura.md) · [02 — Stack](./02-stack-tecnologico.md) · [04 — Design system e UI](./04-design-system-ui.md) · [10 — Formulários e validação](./10-formularios-validacao.md).

---

## 1. Nomenclatura

Regra geral: **código e identificadores em inglês; UI e mensagens de validação em PT-BR.** Domínio de negócio pode ficar em PT-BR quando reflete o contrato do backend (ex.: `empresa`, `analise`), mas mantenha consistência dentro de cada módulo.

| Elemento | Convenção | Exemplo |
|----------|-----------|---------|
| Componente (React) | PascalCase | `DialogCreateResource`, `AnaliseStatusBadge` |
| Hook | prefixo `use` + camelCase | `useFormCreateResource`, `useTableAnalises` |
| Action | **verbo + entidade** (camelCase) | `getResources`, `createResource`, `updateAnalise`, `deleteVistoria` |
| Schema Zod | `xSchema` → tipo `XSchemaProps` | `resourceSchema` → `ResourceSchemaProps` |
| Enum | sufixo `Enum` | `ResourceStatusEnum`, `ModulesPermissionEnum` |
| Type / Interface | PascalCase (sem prefixo `I`) | `Session`, `ApiError`, `CreateResourceProps` |
| Constante de módulo | SCREAMING_SNAKE_CASE | `DEFAULT_PAGE_SIZE`, `STORAGE_KEY` |
| Arquivos | kebab-case | `create-resource/index.ts`, `use-form-create-resource/index.ts` |
| Pasta de artefato | kebab-case | `create-resource/`, `analise-status-badge/` |
| Route group | `(kebab-case)` | `(modules)`, `(analises)`, `(auth)`, `(public)` |
| Página / rota | kebab-case na URL | `analises/page.tsx` → `/analises` |
| Branch | `feat/HUB-<n>-<slug>` | `feat/HUB-228-listagem-empresas` |
| Commit | `HUB-<n>: <descrição imperativa>` | `HUB-228: cria tela de listagem de empresas` |

**Convenções de actions por operação:**

| Operação | Verbo | Exemplo |
|----------|-------|---------|
| Listar (coleção) | `get` + plural | `getResources(filters)` |
| Detalhe (item) | `get` + singular | `getResource(id)` |
| Criar | `create` | `createResource(props)` |
| Atualizar | `update` | `updateResource(id, props)` |
| Remover | `delete` | `deleteResource(id)` |

> O `xSchema` é a fonte do tipo: nunca declare manualmente uma interface duplicando o schema — use `z.infer` (`export type XSchemaProps = z.infer<typeof xSchema>`). O schema é o contrato; o tipo é derivado dele.

---

## 2. Estrutura de um artefato

Todo **componente, hook, action ou schema** é uma **pasta** com no máximo dois arquivos:

```text
NomeDoArtefato/
├── index.ts(x)       # implementação (a única coisa exportada)
└── interfaces.ts     # Props, Response e types do artefato
```

- `index.tsx` para componentes (JSX); `index.ts` para hooks/actions/schemas.
- `interfaces.ts` concentra `Props`, payloads e tipos de resposta. Quando o tipo vem do schema (`z.infer`), ele mora no próprio `index.ts` do schema, não em `interfaces.ts`.
- Cada módulo expõe um **barrel público** `_business/index.ts` que reexporta actions, hooks, schemas e helpers. **Fora do módulo, importa-se só pelo barrel** — nunca caminho interno.

```text
src/app/(modules)/(resources)/
├── _business/
│   ├── actions/
│   │   ├── get-resources/
│   │   │   ├── index.ts
│   │   │   └── interfaces.ts
│   │   └── create-resource/
│   │       ├── index.ts
│   │       └── interfaces.ts
│   ├── hooks/
│   │   └── use-form-create-resource/
│   │       └── index.ts
│   ├── schemas/
│   │   └── resource/
│   │       └── index.ts          # resourceSchema + ResourceSchemaProps
│   ├── helpers/                  # mappers, constants, máscaras
│   └── index.ts                  # BARREL público do módulo
└── (web)/
    ├── (pages)/                  # rotas App Router
    └── _components/              # Form, Dialog, Table... da tela
```

---

## 3. Padrões por camada

O fluxo é **UI → hook (`useXxx`) → action → `api` (`@/infra`) → fetcher → backend** ([01 — Arquitetura](./01-arquitetura.md)). Cada camada tem um padrão de escrita.

### 3.1 Action — só chama `api`, sem `try/catch`

A action recebe **um objeto `props` tipado** e delega ao `api`. **MUST NOT** conter `try/catch` (quem trata o erro é o hook) nem regra de negócio.

```typescript
// _business/actions/create-resource/interfaces.ts
import type { ResourceSchemaProps } from '@/app/(modules)/(resources)/_business';

export type CreateResourceProps = ResourceSchemaProps;
```

```typescript
// _business/actions/create-resource/index.ts
import { api } from '@/infra';
import type { CreateResourceProps } from './interfaces';

export async function createResource(props: CreateResourceProps) {
  return api.post('/resources', props); // sem try/catch
}
```

```typescript
// _business/actions/get-resources/index.ts
import { api } from '@/infra';
import type { ListResourcesProps } from './interfaces';

export async function getResources(props: ListResourcesProps) {
  return api.get('/resources', { params: props });
}
```

### 3.2 Hook de formulário — RHF + zodResolver + toast + revalidatePath

O hook (`'use client'`) orquestra `useForm`, submit, loading e tratamento de erro. O erro é resolvido por `resolveErrorMessage` (mapeia por `code`/`status`, nunca por texto). Após mutação que altera dados do servidor: `revalidatePath`.

```typescript
// _business/hooks/use-form-create-resource/index.ts
'use client';

import { zodResolver } from '@hookform/resolvers/zod';
import { useState } from 'react';
import { useForm } from 'react-hook-form';
import {
  resourceSchema,
  createResource,
  type ResourceSchemaProps,
} from '@/app/(modules)/(resources)/_business';
import { revalidatePath, toast, resolveErrorMessage } from '@/shared';

export function useFormCreateResource() {
  const [isOpen, setIsOpen] = useState(false);

  const form = useForm<ResourceSchemaProps>({
    resolver: zodResolver(resourceSchema),
    defaultValues: { name: '', email: '' },
  });

  async function handleSubmit(data: ResourceSchemaProps) {
    try {
      await createResource(data);
      setIsOpen(false);
      form.reset();
      revalidatePath('/resources');
      toast.success('Cadastro realizado com sucesso');
    } catch (error) {
      toast.error(resolveErrorMessage(error));
    }
  }

  return {
    form,
    isOpen,
    setIsOpen,
    isSubmitting: form.formState.isSubmitting,
    onSubmit: form.handleSubmit(handleSubmit),
  };
}
```

> O `try/catch` vive **no hook**, nunca na action. O hook é a única camada que conhece `toast`. Detalhes de erro de campo (`error.details[]` → RHF) em [10 — Formulários e validação](./10-formularios-validacao.md).

### 3.3 Schema Zod — contrato do domínio, mensagens PT-BR

O schema é a **borda de validação** e a fonte do tipo. Mensagens **sempre em PT-BR**; o tipo vem de `z.infer`.

```typescript
// _business/schemas/resource/index.ts
import { z } from 'zod';

export const resourceSchema = z.object({
  name: z.string().min(1, 'Campo obrigatório'),
  email: z.email('Informe um e-mail válido'),
});

export type ResourceSchemaProps = z.infer<typeof resourceSchema>;
```

### 3.4 Página — Server Component com Suspense

A página é **Server Component por padrão**: faz o fetch inicial via `api` e isola a tabela num `<Suspense>` com fallback. `params`/`searchParams` são **`Promise`** no Next 16 — sempre `await`.

```tsx
// (web)/(pages)/resources/page.tsx
import { Suspense } from 'react';
import { Card, Table, TableFallback } from '@/components';
import { getResources } from '@/app/(modules)/(resources)/_business';
import { columns } from './columns';

async function TableContent({ filters }: { filters: Record<string, string> }) {
  const { data, meta } = await getResources(filters);
  return <Table columns={columns} data={data} total={meta.total} />;
}

export default async function ResourcesPage(props: {
  searchParams: Promise<Record<string, string>>;
}) {
  const filters = await props.searchParams;

  return (
    <Card title="Recursos">
      <Suspense fallback={<TableFallback columns={columns} />}>
        <TableContent filters={filters} />
      </Suspense>
    </Card>
  );
}
```

> Componentes client (`Dialog`, `Form`) são ilhas orquestradas pelo hook; a página os compõe, mas o fetch e o permission gate ficam no servidor. Ver [04 — Design system e UI](./04-design-system-ui.md) e [07 — Auth/permissões](./07-auth-sessao-permissoes.md).

---

## 4. Barrels e imports

A fronteira de import é o **barrel público** de cada área. Importa-se **sempre via alias `@/`** — **MUST NOT** usar `../../..`.

| Origem | Importar de |
|--------|-------------|
| Design system (shadcn + custom) | `@/components` |
| HTTP (api + fetcher) | `@/infra` |
| Cross-cutting (types, enums, utils, cookies, jwt, errors, toast, revalidate) | `@/shared` |
| `cn()` e utilitários | `@/lib/utils` |
| Domínio (actions/hooks/schemas/helpers do módulo) | `@/app/(modules)/(dominio)/_business` |
| Componentes de tela do módulo | `@/app/(modules)/(dominio)/(web)/_components` |

```typescript
// ✅ Import via barrels públicos, com alias e import type
import {
  useFormCreateResource,
  createResource,
  resourceSchema,
  type ResourceSchemaProps,
} from '@/app/(modules)/(resources)/_business';
import { Button, Card, Dialog } from '@/components';
import { api } from '@/infra';
import { toast, resolveErrorMessage } from '@/shared';

// ❌ NUNCA — caminho interno e relativo profundo
import { createResource } from '../../_business/actions/create-resource';
```

Regras:

- **`import type { X }`** quando o uso é só de tipo (melhora tree-shaking e deixa claro o que é runtime vs tipo).
- O barrel reexporta **apenas o que é público**. Detalhes internos (`interfaces.ts`, helpers privados) não saem do módulo.
- Não crie ciclos entre barrels: `_business` não importa de `(web)`.

---

## 5. Idioma: código em inglês, UI em PT-BR

| Onde | Idioma | Exemplo |
|------|--------|---------|
| Identificadores (funções, vars, tipos) | Inglês | `createResource`, `isSubmitting` |
| Mensagens de validação (Zod) | **PT-BR** | `'Campo obrigatório'`, `'CPF inválido'` |
| Textos de UI (labels, botões, toasts) | **PT-BR** | `'Salvar'`, `'Cadastro realizado com sucesso'` |
| Comentários | Inglês ou PT-BR (curtos; evite ruído) | — |
| Nomes de domínio do backend | PT-BR (espelha contrato) | `empresa`, `analise`, `vistoria` |

---

## 6. Regras de ESLint relevantes

Base: `eslint-config-next/core-web-vitals` + `typescript`. Regras adicionais aplicadas a `src/**/*.{ts,tsx}` (ver [02 — Stack](./02-stack-tecnologico.md) e [CLAUDE.md](../CLAUDE.md) §10):

| Regra | Efeito | Motivo |
|-------|--------|--------|
| `@typescript-eslint/no-explicit-any: error` | Proíbe `any` | Type-safety ponta a ponta |
| `no-console: ['error', { allow: ['error'] }]` | Só `console.error` permitido | Sem logs de debug em produção |
| `no-restricted-imports` (padrão `../../*`) | Bloqueia import relativo profundo | Força alias `@/` |
| `@typescript-eslint/naming-convention` (enum sufixo `Enum`) | Enums devem terminar em `Enum` | Consistência |
| `no-warning-comments` (TODO/FIXME) | Avisa sobre comentários pendentes | Evita débito esquecido |

```js
// eslint.config.mjs (trecho ilustrativo)
{
  rules: {
    '@typescript-eslint/no-explicit-any': 'error',
    'no-console': ['error', { allow: ['error'] }],
    'no-restricted-imports': ['error', { patterns: ['../../*'] }],
    'no-warning-comments': ['warn', { terms: ['todo', 'fixme'], location: 'anywhere' }],
    '@typescript-eslint/naming-convention': [
      'error',
      { selector: 'enum', format: ['PascalCase'], suffix: ['Enum'] },
    ],
  },
}
```

> `yarn lint && yarn build` **MUST** estar verde antes de qualquer PR ([Definition of Done](../CLAUDE.md#10-definition-of-done-checklist-universal)). O Husky roda `yarn lint` no pre-commit.

---

## 7. Boas práticas e anti-padrões

Síntese dos anti-padrões da base técnica (`codefirst/api/frontend.md` §13), adaptada ao weCorp:

| Evitar ❌ | Usar ✅ |
|-----------|---------|
| `condition && <Component />` ou ternário inline no JSX | `<If condition={} elseRender={}>` ([04](./04-design-system-ui.md)) |
| `useState`/`useEffect`/`useForm` no componente de UI | Hook `useXxx` que orquestra o estado |
| `fetch` direto no código de aplicação | `api` de `@/infra` |
| `try/catch` dentro da action | Tratamento no hook (`toast` + `resolveErrorMessage`) |
| `console.log` para depurar | Remover; só `console.error` é permitido |
| Mensagens Zod em inglês | Mensagens **PT-BR** |
| Cores hex inline em componentes | Variáveis CSS / tokens Tailwind ([04](./04-design-system-ui.md)) |
| Strings de permissão hardcoded (`'admin'`) | Enums de `@/shared` (`RolesPermissionEnum`) |
| Imports `../../../` | Alias `@/` + barrels públicos |
| Token em `localStorage`/`document.cookie` | Cookies httpOnly ([07](./07-auth-sessao-permissoes.md)) |
| Sessão/JWT duplicados no Zustand | `getServerSession()`; Zustand só UI |
| Erro tratado por texto da mensagem do backend | Tratar por `error.code` / `status` |
| `process.env.X \|\| ''` espalhado | `env` validado por Zod em `@/configs/env` |
| Mock importado em código de produção | Flag `USE_MOCKS` / MSW |
| `any` para silenciar o compilador | `unknown` + narrowing, generics, tipos do contrato |
| Interface duplicando um schema Zod | `z.infer<typeof xSchema>` |

---

## 8. Checklist de revisão de código

- [ ] Arquivos/pastas kebab-case; componentes PascalCase; hooks `useXxx`; actions verbo+entidade.
- [ ] Artefato em pasta com `index.ts(x)` (+ `interfaces.ts`); export só pelo barrel público.
- [ ] Imports via alias `@/` e barrels (`@/shared`/`@/components`/`@/infra`/`_business`); zero `../../..`; `import type` onde cabe.
- [ ] Camadas respeitadas: UI sem `fetch`/estado direto; action sem `try/catch`; hook trata erro.
- [ ] Server Component por padrão; `"use client"` justificado; `params`/`searchParams` com `await`.
- [ ] Schema Zod PT-BR; tipo via `z.infer`; enum com sufixo `Enum`.
- [ ] `revalidatePath` após mutação; envelope `{data}`/`{data,meta}`/`{error}` consumido corretamente.
- [ ] Zero `any`; sem `console.log`; sem hex inline; sem string de permissão hardcoded.
- [ ] `yarn lint && yarn build` verdes.

## Ver também

- [01 — Arquitetura](./01-arquitetura.md) · [02 — Stack tecnológico](./02-stack-tecnologico.md)
- [04 — Design system e UI](./04-design-system-ui.md) · [10 — Formulários e validação](./10-formularios-validacao.md)
- [08 — Consumo de API e data fetching](./08-consumo-api-dados.md) · [CLAUDE.md](../CLAUDE.md)
</content>
</invoke>
