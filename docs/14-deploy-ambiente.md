# 14 — Deploy e Ambiente

> Variáveis de ambiente (validadas por Zod no boot), scripts Yarn, imagem Docker de produção (multi-stage `standalone`, non-root), `next.config.ts`, Husky e CI/CD do frontend weCorp.
>
> Cross-links: [02 — Stack tecnológico](./02-stack-tecnologico.md), [06 — Multi-tenancy/white-label](./06-multitenancy-whitelabel.md) (mapa tenant→backend), [07 — Auth/sessão](./07-auth-sessao-permissoes.md) (cookies, `proxy.ts`), [13 — Testes](./13-testes.md) (CI), [15 — Roadmap](./15-roadmap-fases.md).
>
> **Mapeamento Linear:** F19 — Qualidade, Testes e Deploy: **HUB-309** (Docker standalone + `env.ts` Zod + `.env.example`), **HUB-310** (CI/CD GitHub Actions + Husky).

---

## 1. Variáveis de ambiente validadas por Zod (`src/configs/env.ts`)

**Regra central:** nada lê `process.env` espalhado pelo código. Existe **um** ponto — `src/configs/env.ts` — que valida o ambiente com Zod no boot e **lança erro** se faltar/estiver inválido. O resto do código importa `env`.

```typescript
// src/configs/env.ts
import { z } from 'zod';

const envSchema = z.object({
  // públicas (expostas ao bundle — sem segredo!)
  NEXT_PUBLIC_API_BASE_URL: z.url(),
  NEXT_PUBLIC_API_URL: z.url(),
  NEXT_PUBLIC_TOKEN_KEY: z.string().min(1),
  NEXT_PUBLIC_REFRESH_TOKEN_KEY: z.string().min(1),
  // server-only
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().default(3000),
  TENANT_API_MAP: z.string().optional(), // JSON: { "<tenant>": "<api url>" }
});

const parsed = envSchema.safeParse(process.env);
if (!parsed.success) {
  // falha no boot — não sobe com configuração inválida
  throw new Error(`Configuração de ambiente inválida: ${parsed.error.message}`);
}

export const env = parsed.data;
```

> O `safeParse` + `throw` garante **fail-fast**: a aplicação não inicia com env quebrado, em vez de falhar silenciosamente em runtime.

### Regras de uso (MUST / MUST NOT)

| Regra | Detalhe |
|-------|---------|
| **Segredos NUNCA com `NEXT_PUBLIC_`** | Tudo com prefixo `NEXT_PUBLIC_` vai para o **bundle do browser** — visível a qualquer usuário. Segredos (tokens de integração, secrets) ficam **server-only**, sem o prefixo, e só são lidos em código que roda no servidor (`proxy.ts`, Route Handlers, Server Actions, `api`). |
| **Importar `env`** | `import { env } from '@/configs/env'`. **MUST NOT** `process.env.X \|\| ''` espalhado — quebra a validação e a tipagem. |
| **Chaves públicas = só o necessário** | Nomes de cookie (`NEXT_PUBLIC_TOKEN_KEY`), URL pública da API. Nada além disso vira `NEXT_PUBLIC_`. |

---

## 2. Tabela de variáveis — `.env.example`

`.env`/`.env.local` **nunca** versionados; manter `.env.example` (sem valores reais) versionado.

| Variável | Escopo | Descrição |
|----------|--------|-----------|
| `NEXT_PUBLIC_API_BASE_URL` | pública | URL base do backend (host/porta da API REST) |
| `NEXT_PUBLIC_API_URL` | pública | URL completa da API (ex.: `${BASE}/api`) usada pelo `fetcher` |
| `NEXT_PUBLIC_TOKEN_KEY` | pública | Nome do cookie do **access token** |
| `NEXT_PUBLIC_REFRESH_TOKEN_KEY` | pública | Nome do cookie do **refresh token** |
| `PORT` | server | Porta do servidor Next (ex.: 3000/3001) |
| `NODE_ENV` | server | `development` \| `test` \| `production` |
| `TENANT_API_MAP` | **server** | JSON `tenant → URL do backend` para roteamento white-label por host (ver spec `frontend.md` §1.6 e [06](./06-multitenancy-whitelabel.md)) |

```dotenv
# ──────────────── API ────────────────
NEXT_PUBLIC_API_BASE_URL=https://api.hubstate.com.br
NEXT_PUBLIC_API_URL=https://api.hubstate.com.br/api

# ──────────────── Cookies de sessão ────────────────
NEXT_PUBLIC_TOKEN_KEY=wecorp_at
NEXT_PUBLIC_REFRESH_TOKEN_KEY=wecorp_rt

# ──────────────── App ────────────────
NODE_ENV=development
PORT=3000

# ──────────────── Multi-tenant (white-label) ────────────────
# mapeia o tenant detectado por host → URL do backend correspondente
TENANT_API_MAP={"hubstate":"https://api.hubstate.com.br","wecorp":"https://api.wecorp.com.br"}
```

> **Reconciliação com o spec:** o `frontend.md` §1.6 cita `NEXTAUTH_URL`/`NEXTAUTH_SECRET` e integrações pass-through (`SERASA_API_URL`...). **Superado** — a auth é por **cookies httpOnly + `proxy.ts`** (não NextAuth) e as integrações externas vivem **no backend**, não no frontend. Ver [02 — Stack](./02-stack-tecnologico.md) §1. O que permanece do spec: `NEXT_PUBLIC_API_BASE_URL` e `TENANT_API_MAP`.

---

## 3. Scripts Yarn

```bash
yarn install                 # dependências (lockfile yarn.lock)
yarn dev                     # desenvolvimento — Next 16 com Turbopack
yarn build                   # build de produção (output: 'standalone')
yarn start                   # servidor de produção (node server.js do standalone)
yarn lint                    # ESLint 9 (eslint-config-next)
yarn test                    # Vitest (unit + componente, com MSW)
yarn test:e2e                # Playwright (e2e dos fluxos críticos)
```

Subida local:

```bash
yarn install
cp .env.example .env.local   # preencher valores locais
yarn dev                     # http://localhost:3000
```

---

## 4. Docker de produção (multi-stage, `standalone`, non-root)

`output: 'standalone'` faz o Next emitir um `server.js` autocontido com apenas as dependências usadas — imagem final mínima. Build multi-stage separa toolchain de build do runtime; container roda como usuário **não-root** `nextjs`.

```dockerfile
# ───── Stage 1: deps ─────
FROM node:24-alpine AS deps
WORKDIR /app
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

# ───── Stage 2: build ─────
FROM node:24-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# variáveis NEXT_PUBLIC_* precisam existir em BUILD TIME (entram no bundle)
ARG NEXT_PUBLIC_API_BASE_URL
ARG NEXT_PUBLIC_API_URL
ARG NEXT_PUBLIC_TOKEN_KEY
ARG NEXT_PUBLIC_REFRESH_TOKEN_KEY
ENV NEXT_PUBLIC_API_BASE_URL=$NEXT_PUBLIC_API_BASE_URL \
    NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL \
    NEXT_PUBLIC_TOKEN_KEY=$NEXT_PUBLIC_TOKEN_KEY \
    NEXT_PUBLIC_REFRESH_TOKEN_KEY=$NEXT_PUBLIC_REFRESH_TOKEN_KEY

RUN yarn build

# ───── Stage 3: runtime mínimo ─────
FROM node:24-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

# usuário não-root
RUN addgroup --system --gid 1001 nodejs && adduser --system --uid 1001 nextjs

# artefatos do standalone
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=builder --chown=nextjs:nodejs /app/public ./public

USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

> **Segredos `NEXT_PUBLIC_*` viram públicos no bundle** — não passe nada sensível como build-arg. Variáveis **server-only** (`TENANT_API_MAP`) são injetadas em **runtime** pelo orquestrador, não como build-arg. Acompanhar de `.dockerignore` (`node_modules`, `.next`, `.env*`, `.git`, `e2e`, `coverage`).

### docker-compose local

```yaml
services:
  web:
    build:
      context: .
      args:
        NEXT_PUBLIC_API_BASE_URL: http://localhost:3333
        NEXT_PUBLIC_API_URL: http://localhost:3333/api
        NEXT_PUBLIC_TOKEN_KEY: wecorp_at
        NEXT_PUBLIC_REFRESH_TOKEN_KEY: wecorp_rt
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      TENANT_API_MAP: '{"hubstate":"http://localhost:3333"}'
```

---

## 5. `next.config.ts`

```typescript
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  output: 'standalone',          // imagem Docker mínima (server.js autocontido)
  turbopack: {},                 // dev
  experimental: {
    optimizePackageImports: [    // tree-shaking agressivo de libs pesadas
      'lucide-react',
      'date-fns',
      'recharts',
      '@tanstack/react-table',
    ],
  },
  images: {
    remotePatterns: [
      // apenas hosts reais de produção (logos white-label, anexos)
      { protocol: 'https', hostname: '*.wecorp.com.br' },
      { protocol: 'https', hostname: '*.hubstate.com.br' },
    ],
  },
};

export default nextConfig;
```

- **`output: 'standalone'`** — habilita a imagem Docker da §4.
- **`optimizePackageImports`** — reduz JS enviado ao cliente (objetivo de performance, [00](./00-visao-geral.md) §3).
- **`images.remotePatterns`** — só domínios reais; nada de wildcard aberto.

---

## 6. CI/CD — GitHub Actions

### 6.1 PR — lint + test + build

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 24, cache: yarn }
      - run: yarn install --frozen-lockfile
      - run: yarn lint
      - run: yarn test
      - run: yarn build
        env:
          NEXT_PUBLIC_API_BASE_URL: https://api.ci.local
          NEXT_PUBLIC_API_URL: https://api.ci.local/api
          NEXT_PUBLIC_TOKEN_KEY: wecorp_at
          NEXT_PUBLIC_REFRESH_TOKEN_KEY: wecorp_rt

  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 24, cache: yarn }
      - run: yarn install --frozen-lockfile
      - run: npx playwright install --with-deps
      - run: yarn test:e2e
```

### 6.2 Deploy — build + push da imagem (push em main)

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - name: Build & push imagem Docker
        run: |
          docker build \
            --build-arg NEXT_PUBLIC_API_BASE_URL=${{ vars.NEXT_PUBLIC_API_BASE_URL }} \
            --build-arg NEXT_PUBLIC_API_URL=${{ vars.NEXT_PUBLIC_API_URL }} \
            --build-arg NEXT_PUBLIC_TOKEN_KEY=${{ vars.NEXT_PUBLIC_TOKEN_KEY }} \
            --build-arg NEXT_PUBLIC_REFRESH_TOKEN_KEY=${{ vars.NEXT_PUBLIC_REFRESH_TOKEN_KEY }} \
            -t $REGISTRY/wecorp-frontend:${{ github.sha }} .
          # docker push $REGISTRY/wecorp-frontend:${{ github.sha }} && release no orquestrador
```

> Segredos/variáveis do CI vivem em **GitHub Secrets/Variables** (ou *environments*), nunca no YAML. Variáveis **server-only** (`TENANT_API_MAP`) são injetadas no orquestrador em runtime. CI verde é **pré-requisito de merge**.

---

## 7. Husky e fluxo de branches

```bash
# .husky/pre-commit
yarn lint
```

- **Husky pre-commit** roda `yarn lint` localmente — barra erro óbvio antes do commit. (Testes/build pesados ficam no CI para não travar o commit.)
- **Feature branches:** `feat/HUB-<n>-<slug>` a partir da branch principal (`main`) atualizada. **MUST NOT** commitar direto na branch principal — todo trabalho entra por PR referenciando a issue `HUB-N` ([`../CLAUDE.md`](../CLAUDE.md) §8).

---

## 8. Ambientes

| Ambiente | `NODE_ENV` | Backend consumido | Cookies | Observação |
|----------|------------|-------------------|---------|------------|
| Local | `development` | backend local (Docker) | `secure` relaxado p/ HTTP | `.env.local` |
| Teste/CI | `test`/`production` | MSW / mocks | — | efêmero por job; e2e com Playwright |
| Staging | `production` | API de staging | httpOnly + secure | espelha prod |
| Produção | `production` | API de produção | httpOnly + secure + HTTPS/HSTS | env server-only via orquestrador; white-label por host |

---

## Ver também

- [02 — Stack tecnológico](./02-stack-tecnologico.md) — reconciliação de versões (Next 16, cookies+proxy, sem NextAuth/ky).
- [06 — Multi-tenancy/white-label](./06-multitenancy-whitelabel.md) — `TENANT_API_MAP`, detecção de marca por host.
- [07 — Auth/sessão/permissões](./07-auth-sessao-permissoes.md) — cookies httpOnly, `proxy.ts`.
- [13 — Testes](./13-testes.md) — CI (lint/test/build/e2e), Husky.
- [15 — Roadmap e fases](./15-roadmap-fases.md).
</content>
</invoke>
