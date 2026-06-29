# 10 — Formulários e Validação

> Como construímos formulários no **frontend weCorp**: React Hook Form + Zod (`zodResolver`), mensagens **sempre PT-BR**, validação brasileira (CPF/CNPJ/PIS) com `zod-br`, máscaras com `react-imask`, seções colapsáveis, autosave de rascunho, dirty-state e mapeamento dos erros do envelope da API para erros de campo.
> Regras não-negociáveis em [`../CLAUDE.md`](../CLAUDE.md) §8. Padrão de hooks/actions em [03 — Convenções](./03-convencoes-codigo.md); componentes de UI em [04 — Design system](./04-design-system-ui.md); contrato/erros da API em [08 — Consumo de API](./08-consumo-api-dados.md).

---

## 1. Padrão base: RHF + zodResolver + hook `useFormXxx`

Todo formulário segue o mesmo esqueleto:

1. **Schema Zod** (`xSchema`) define forma + validação + tipo (`z.infer`).
2. **Hook `useFormXxx`** (`'use client'`) cria o `useForm` com `zodResolver`, gerencia submit/loading e trata erro com `toast` + `resolveErrorMessage`.
3. **Componente de UI** consome o hook e só compõe markup (sem `useForm` direto — [03 §3](./03-convencoes-codigo.md)).

```tsx
// _business/hooks/use-form-create-pessoa/index.ts
'use client';

import { zodResolver } from '@hookform/resolvers/zod';
import { useForm } from 'react-hook-form';
import {
  pessoaSchema,
  createPessoa,
  type PessoaSchemaProps,
} from '@/app/(modules)/(pessoas)/_business';
import { revalidatePath, toast, resolveErrorMessage } from '@/shared';
import { applyApiFieldErrors } from '@/shared'; // mapeia error.details[] -> RHF (ver §7)

export function useFormCreatePessoa() {
  const form = useForm<PessoaSchemaProps>({
    resolver: zodResolver(pessoaSchema),
    defaultValues: { tipo_pessoa: 'PF', nome: '', documento: '' },
  });

  async function handleSubmit(data: PessoaSchemaProps) {
    try {
      await createPessoa(data);
      form.reset();
      revalidatePath('/pessoas');
      toast.success('Pessoa cadastrada com sucesso');
    } catch (error) {
      // erros de campo do backend -> RHF; mensagem geral -> toast
      const handled = applyApiFieldErrors(error, form.setError);
      if (!handled) toast.error(resolveErrorMessage(error));
    }
  }

  return {
    form,
    isSubmitting: form.formState.isSubmitting,
    isDirty: form.formState.isDirty,
    onSubmit: form.handleSubmit(handleSubmit),
  };
}
```

```tsx
// (web)/_components/FormPessoa/index.tsx — UI só compõe markup
'use client';

import { Form, FormField, Input, Button } from '@/components';
import { useFormCreatePessoa } from '@/app/(modules)/(pessoas)/_business';

export function FormPessoa() {
  const { form, isSubmitting, onSubmit } = useFormCreatePessoa();
  return (
    <Form {...form}>
      <form onSubmit={onSubmit} className="space-y-4" noValidate>
        <FormField control={form.control} name="nome" label="Nome" component={Input} />
        <Button type="submit" disabled={isSubmitting}>Salvar</Button>
      </form>
    </Form>
  );
}
```

---

## 2. Zod com mensagens SEMPRE em PT-BR

A validação é a **borda do formulário**. Mensagens **MUST** ser PT-BR; o tipo vem de `z.infer`. Enums com sufixo `Enum`.

```typescript
// _business/schemas/resource/index.ts
import { z } from 'zod';

export const resourceSchema = z.object({
  nome: z.string().min(1, 'Campo obrigatório').max(255, 'Máximo de 255 caracteres'),
  email: z.email('Informe um e-mail válido'),
  prazoContrato: z.coerce
    .number({ message: 'Informe um número' })
    .int('Deve ser inteiro')
    .min(1, 'Mínimo de 1 mês')
    .max(120, 'Máximo de 120 meses'),
});

export type ResourceSchemaProps = z.infer<typeof resourceSchema>;
```

> Erros de submit são tratados por `code`/`status` (§6), **nunca** pelo texto da mensagem do backend. Erros de **campo** (cliente) vêm do Zod; erros de **campo vindos do servidor** são mapeados em §7.

---

## 3. Validação brasileira com `zod-br`

Documentos brasileiros (CPF, CNPJ, PIS) usam `zod-br`, que já traz validação de dígito verificador e mensagens PT-BR. O valor validado é sempre o **cru** (somente dígitos) — a máscara é só apresentação (§4, §8).

```typescript
import { z } from 'zod';
import { cpf, cnpj, pis } from 'zod-br';

export const pessoaPFSchema = z.object({
  tipo_pessoa: z.literal('PF'),
  nome: z.string().min(1, 'Campo obrigatório'),
  documento: cpf(), // valida CPF (dígitos verificadores) + mensagem PT-BR
  pis: pis().optional(),
});

export const pessoaPJSchema = z.object({
  tipo_pessoa: z.literal('PJ'),
  razao_social: z.string().min(1, 'Campo obrigatório'),
  documento: cnpj(), // valida CNPJ
});

// união discriminada PF/PJ (espelha o domínio do backend — ver §5)
export const pessoaSchema = z.discriminatedUnion('tipo_pessoa', [
  pessoaPFSchema,
  pessoaPJSchema,
]);

export type PessoaSchemaProps = z.infer<typeof pessoaSchema>;
```

> Se `zod-br` não cobrir um caso, use `.refine(isValidCPF, 'CPF inválido')` com um validador próprio em `helpers/` — mas a mensagem continua PT-BR.

---

## 4. Máscaras com `react-imask`

Máscaras (CPF, CNPJ, CEP, telefone, moeda, data) são **apresentação**. O componente exibe o valor mascarado, mas o RHF guarda o **valor cru** que vai para a API. Centralize um `MaskedInput` que integra `react-imask` ao `Controller` do RHF.

```tsx
// src/components/masked-input/index.tsx
'use client';

import { IMaskInput } from 'react-imask';
import { Controller, type Control } from 'react-hook-form';

const MASKS = {
  cpf: '000.000.000-00',
  cnpj: '00.000.000/0000-00',
  cep: '00000-000',
  phone: '(00) 00000-0000',
  date: '00/00/0000',
} as const;

interface MaskedInputProps {
  control: Control<any>;
  name: string;
  mask: keyof typeof MASKS;
  label: string;
}

export function MaskedInput({ control, name, mask, label }: MaskedInputProps) {
  return (
    <Controller
      control={control}
      name={name}
      render={({ field, fieldState }) => (
        <div className="space-y-1">
          <label htmlFor={name}>{label}</label>
          <IMaskInput
            id={name}
            mask={MASKS[mask]}
            value={field.value ?? ''}
            unmask // entrega ao RHF o valor CRU (só dígitos)
            onAccept={(value) => field.onChange(value)}
            onBlur={field.onBlur}
            aria-invalid={!!fieldState.error}
          />
          {fieldState.error ? (
            <p role="alert" className="text-destructive text-sm">
              {fieldState.error.message}
            </p>
          ) : null}
        </div>
      )}
    />
  );
}
```

| Máscara | Exibição | Valor cru enviado |
|---------|----------|-------------------|
| CPF | `123.456.789-00` | `12345678900` |
| CNPJ | `12.345.678/0001-90` | `12345678000190` |
| CEP | `01310-100` | `01310100` |
| Telefone | `(11) 99999-8888` | `11999998888` |
| Data | `28/06/2026` | `2026-06-28` (após parse — §8) |
| Moeda | `R$ 1.234,56` | `1234.56` (número — §8) |

> Para **moeda**, use o modo numérico do IMask (`scale: 2`, `radix: ','`) e converta para `number` antes de enviar. O schema recebe `z.coerce.number()`.

---

## 5. Schemas Zod compartilhados como contrato de domínio

Os schemas espelham o domínio do backend (spec `frontend.md` parte 8.2). O **mesmo schema** valida o formulário e descreve o payload — type-safety de ponta a ponta. Schemas reaproveitados entre módulos (ex.: `pessoaSchema`, `enderecoSchema`) ficam em `_business/schemas/` do domínio dono e são reexportados pelo barrel.

```typescript
// espelha o contrato do spec (parte 8.2): SeguroFiancaCreateSchema
import { z } from 'zod';

export const seguroFiancaCreateSchema = z.object({
  empresaparceira_id: z.uuid('Empresa parceira inválida'),
  seguradora_id: z.uuid('Seguradora inválida'),
  tipolocacao_id: z.coerce.number().int('Selecione o tipo de locação'),
  valor_aluguel: z.coerce.number().positive('Valor deve ser maior que zero'),
  prazo_contrato: z.coerce.number().int().min(1).max(120, 'Máximo de 120 meses'),
  data_inicio: z.coerce.date({ message: 'Informe a data de início' }),
});

export type SeguroFiancaCreateProps = z.infer<typeof seguroFiancaCreateSchema>;
```

> Convenção do contrato: campos de domínio em snake_case (espelham o backend); identificadores de código em inglês. Ver [03 — Convenções](./03-convencoes-codigo.md) §1 e [08 — Consumo de API](./08-consumo-api-dados.md).

---

## 6. Formulários longos: seções colapsáveis

Formulários extensos (ex.: Análise Cadastral, ficha de empresa com ~120 campos) são divididos em **seções colapsáveis** (um `<Card>`/`Accordion` por seção), mantendo **um único `useForm`** por trás. Erros do RHF abrem automaticamente a seção que contém o campo inválido.

```tsx
'use client';

import { Accordion, AccordionItem, Card } from '@/components';
import { useFormAnaliseCadastral } from '@/app/(modules)/(analises)/_business';

export function FormAnaliseCadastral() {
  const { form, onSubmit } = useFormAnaliseCadastral();

  return (
    <form onSubmit={onSubmit} className="space-y-4">
      <Accordion type="multiple" defaultValue={['identificacao']}>
        <AccordionItem value="identificacao" title="Identificação">
          {/* campos da seção */}
        </AccordionItem>
        <AccordionItem value="renda" title="Renda e ocupação">
          {/* ... */}
        </AccordionItem>
        <AccordionItem value="referencias" title="Referências">
          {/* ... */}
        </AccordionItem>
      </Accordion>
    </form>
  );
}
```

> Um schema grande pode ser composto por sub-schemas (`identificacaoSchema.merge(rendaSchema)...`), mantendo um `useForm` único e um único submit.

---

## 7. Autosave de rascunho (Análise Cadastral)

Formulários longos como a **Análise Cadastral** salvam rascunho automaticamente para não perder progresso. O autosave observa o form (`form.watch`) com debounce e persiste via action de rascunho; ao montar, o hook hidrata os `defaultValues` a partir do rascunho.

```typescript
// dentro do hook useFormAnaliseCadastral
import { useEffect } from 'react';
import { useDebouncedCallback } from '@/hooks';

const saveDraft = useDebouncedCallback(async (data: AnaliseSchemaProps) => {
  await upsertAnaliseDraft(data); // action -> api; sem try/catch (hook trata)
}, 1500);

useEffect(() => {
  const subscription = form.watch((value) => {
    if (form.formState.isDirty) void saveDraft(value as AnaliseSchemaProps);
  });
  return () => subscription.unsubscribe();
}, [form, saveDraft]);
```

> O rascunho não substitui o submit final; é uma rede de segurança. O envio definitivo continua passando pela validação completa do schema.

---

## 8. Dirty-state bloqueia navegação

Com alterações não salvas (`form.formState.isDirty`), avise antes de sair. No App Router, combine o evento `beforeunload` (recarregar/fechar aba) com confirmação na navegação interna.

```tsx
'use client';

import { useEffect } from 'react';

export function useUnsavedChangesWarning(isDirty: boolean) {
  useEffect(() => {
    if (!isDirty) return;
    function handler(e: BeforeUnloadEvent) {
      e.preventDefault();
      e.returnValue = '';
    }
    window.addEventListener('beforeunload', handler);
    return () => window.removeEventListener('beforeunload', handler);
  }, [isDirty]);
}
```

> Após submit/autosave bem-sucedido, chame `form.reset(data)` para zerar o dirty-state (o form passa a considerar os valores atuais como "limpos").

---

## 9. defaultValues e reset

| Cenário | Padrão |
|---------|--------|
| Criar | `defaultValues` com valores vazios/sensatos; `form.reset()` após sucesso |
| Editar | `defaultValues` a partir do recurso carregado (Server Component passa por prop) |
| Limpar dirty pós-save | `form.reset(novosValores)` |
| Discriminated union (PF/PJ) | `defaultValues` inclui o discriminador (`tipo_pessoa: 'PF'`) |

```typescript
// editar: hidratar defaultValues e resetar quando o dado chega
const form = useForm<PessoaSchemaProps>({
  resolver: zodResolver(pessoaSchema),
  defaultValues: pessoa, // recurso vindo do servidor
});
```

---

## 10. Tratamento de erro de submit

O hook trata exceções da action (que **não** tem `try/catch` — [03 §3](./03-convencoes-codigo.md)). A regra é: **erro de campo → RHF; erro geral → toast**, sempre mapeando por `code`/`status`, nunca por texto.

### 10.1 `resolveErrorMessage` + toast

```typescript
// src/shared/utils/errors/resolve-error-message.ts
import type { ApiError } from '@/shared';

const MESSAGES_BY_CODE: Record<string, string> = {
  CONFLICT: 'Registro já existe.',
  VALIDATION_ERROR: 'Verifique os campos destacados.',
  FORBIDDEN: 'Você não tem permissão para esta ação.',
};

const DEFAULT_MESSAGE = 'Não foi possível concluir, tente novamente ou contate o suporte.';

export function resolveErrorMessage(error: unknown): string {
  const err = error as ApiError;
  if (err?.code && MESSAGES_BY_CODE[err.code]) return MESSAGES_BY_CODE[err.code];
  return DEFAULT_MESSAGE;
}
```

### 10.2 Mapear `error.details[]` do envelope para erros de campo do RHF

O envelope de erro da API é `{ error: { code, message, details: [{ field, code, message }] } }` ([08 — Consumo de API](./08-consumo-api-dados.md)). Em `VALIDATION_ERROR`, cada item de `details` vira um `form.setError(field, ...)`.

```typescript
// src/shared/utils/errors/apply-api-field-errors.ts
import type { UseFormSetError, FieldValues, Path } from 'react-hook-form';
import type { ApiError } from '@/shared';

/** Retorna true se aplicou erros de campo (logo o hook NÃO deve disparar toast). */
export function applyApiFieldErrors<T extends FieldValues>(
  error: unknown,
  setError: UseFormSetError<T>,
): boolean {
  const err = error as ApiError;
  if (err?.code !== 'VALIDATION_ERROR' || !err.details?.length) return false;

  for (const detail of err.details) {
    setError(detail.field as Path<T>, {
      type: detail.code ?? 'server',
      message: detail.message, // mensagem PT-BR vinda do backend
    });
  }
  return true;
}
```

| Situação | `code`/`status` | Tratamento |
|----------|-----------------|------------|
| Validação de campo no servidor | `VALIDATION_ERROR` / 400 | `details[]` → `form.setError` por campo |
| Conflito (documento duplicado) | `CONFLICT` / 409 | `toast.error` com mensagem específica |
| Sem permissão | `FORBIDDEN` / 403 | `toast.error` + (opcional) bloquear ação |
| Sessão expirada | `UNAUTHORIZED` / 401 | redirect para login (proxy/fetcher) |
| Erro inesperado | — / 500 | `toast.error` com mensagem padrão |

---

## 11. Integração máscara × valor cru (resumo)

| Camada | Vê o quê |
|--------|----------|
| UI (`MaskedInput`) | valor **mascarado** (`123.456.789-00`) |
| RHF (`field.value`) | valor **cru** (`12345678900`), via `unmask`/`onAccept` |
| Zod (`cpf()`) | valida o cru (dígitos verificadores) |
| Action → `api` | envia o **cru** no payload |

> Regra de ouro: **a máscara nunca chega à API**. Se algum campo precisa do valor formatado também (raro), envie-o num campo separado, deixando o cru como canônico.

## Ver também

- [03 — Convenções de código](./03-convencoes-codigo.md) · [04 — Design system e UI](./04-design-system-ui.md)
- [08 — Consumo de API e data fetching](./08-consumo-api-dados.md) · [02 — Stack tecnológico](./02-stack-tecnologico.md)
- [13 — Testes](./13-testes.md) · [CLAUDE.md](../CLAUDE.md)
</content>
