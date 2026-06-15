# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Monorepo Overview

This is a pnpm + Turborepo monorepo for the PancakeSwap frontend. Package manager is **pnpm** (enforced via `preinstall` hook — do not use npm or yarn).

```
root
├── apps/          # Frontend applications
│   ├── web/       # Main EVM app (Next.js, primary target for most work)
│   ├── aptos/     # Aptos chain app
│   ├── blog/      # Blog app
│   ├── bridge/    # Bridge app
│   ├── games/     # Games app
│   ├── gamification/
│   ├── solana/    # Solana frontend
│   ├── ton/       # TON frontend
│   └── e2e/       # End-to-end tests
├── packages/      # ~40 shared packages (all prefixed @pancakeswap/*)
├── apis/          # Cloudflare Workers (farms, proxy-worker, routing)
└── scripts/       # Utility scripts (APR updates, Merkl, etc.)
```

Key version pins are declared in `pnpm-workspace.yaml` catalogs (viem, wagmi, next, react, etc.) — use `catalog:` when referencing these in package.json.

## Common Commands

All commands run from the **repo root** unless noted.

```bash
# Install dependencies
pnpm i

# Development servers
pnpm dev              # Main web app (EVM)
pnpm dev:aptos        # Aptos app
pnpm dev:blog         # Blog
pnpm dev:bridge       # Bridge
pnpm dev:games        # Games
pnpm dev:solana       # Solana

# Build
pnpm build            # Main web app
pnpm build:packages   # All shared packages

# Lint (all packages)
pnpm lint

# Format
pnpm format:check
pnpm format:write

# Tests (run from repo root)
pnpm test:ci          # CI test suite (excludes uikit, only changed packages)

# Tests (run from apps/web)
cd apps/web
pnpm test             # All unit tests (vitest --run)
pnpm test:watch       # Watch mode
pnpm test:config      # Config-specific tests

# Type check (from apps/web)
pnpm typechecks

# E2E
pnpm e2e:ci

# Storybook (uikit)
pnpm storybook

# Clean everything
pnpm clean
```

To run a **single test file**:
```bash
cd apps/web
pnpm vitest run src/path/to/test.test.ts
```

Turborepo is the build orchestrator — tasks like `build` respect `dependsOn` (e.g., packages build before apps). Turbo caches outputs; pass `--force` to bypass cache.

## apps/web Architecture

The main app is a **Next.js 15 Pages Router** application (`apps/web`).

### Directory Structure (`apps/web/src/`)

| Directory | Purpose |
|---|---|
| `pages/` | Next.js route entry points — thin shells that export from `views/` |
| `views/` | Feature page content and logic. Each `views/FeatureName/` maps to a page. |
| `components/` | Generic shared components |
| `state/` | Redux Toolkit slices, organized by feature domain |
| `hooks/` | Generic React hooks |
| `utils/` | Pure utility functions |
| `config/` | Contract ABIs, chain configs, constants |
| `contexts/` | React context providers separate from Redux |
| `queries/` | GraphQL / REST query definitions |
| `quoter/` | Quote computation orchestration (calls into the quote worker) |
| `edge/` | Next.js Edge Runtime code |
| `middlewares/` | Next.js middleware chain modules |

The pattern is: `pages/some-route.tsx` → imports and renders `views/SomePage/index.tsx`.

### State Management

Three layers, used for different purposes:

1. **Redux Toolkit** (`src/state/`) — global app state persisted with `redux-persist`. Slices exist for: swap, farms, farmsV3/V4, pools, lottery, predictions, nftMarket, voting, user preferences, transactions, etc. The combined store type is in `src/state/types.ts`.

2. **Jotai** — component-level and mid-level reactive state (atoms). Used across views for UI-local state.

3. **TanStack Query** (`@tanstack/react-query`) — server/async state (API fetches, on-chain reads). Wrap blockchain calls through wagmi hooks which use React Query internally.

### Blockchain Stack

- **viem** — low-level EVM interactions (pinned version in workspace catalog)
- **wagmi** — React hooks for EVM wallets/chains (pinned version in workspace catalog)
- `@pancakeswap/wagmi` (workspace package) — extends wagmi with BSC chain definition, custom wallet connectors (Trust Wallet, Blocto, etc.)
- `@pancakeswap/chains` — chain constants and configurations
- `@pancakeswap/tokens` — token definitions per chain
- `@pancakeswap/multicall` — gas-safe multicall batching

### Smart Routing (Swap)

Swap quote computation is offloaded to a **Web Worker** (`src/quote-worker.ts`) to avoid blocking the main thread. The worker handles two routing modes:

- `getBestTrade` → `SmartRouter.getBestTrade` (on-chain quote provider via viem)
- `getBestTradeOffchain` → `findBestTrade` from `@pancakeswap/routing-sdk` (Infinity Router)

The `src/quoter/` directory contains the client-side orchestration that communicates with the worker.

### Key Shared Packages

| Package | What it does |
|---|---|
| `@pancakeswap/uikit` | All UI primitives (Button, Modal, etc.) — also has Storybook |
| `@pancakeswap/smart-router` | Best-trade routing across V2/V3/stable pools |
| `@pancakeswap/routing-sdk` | New routing SDK (Infinity Router), with chain-specific addons in `packages/routing-sdk/addons/` |
| `@pancakeswap/v3-sdk` | Uniswap-v3 math for PancakeSwap V3 pools |
| `@pancakeswap/v2-sdk` / `swap-sdk-evm` | V2 AMM math |
| `@pancakeswap/stable-swap-sdk` | StableSwap math |
| `@pancakeswap/localization` | `useTranslation` hook + i18n infra |
| `@pancakeswap/farms` | Farm config and APR helpers |
| `@pancakeswap/pools` | Pool config |
| `@pancakeswap/widgets-internal` | Composed feature widgets (shared across apps) |
| `@pancakeswap/next-config` | Shared Next.js config base |

### APIs (Cloudflare Workers)

`apis/routing` — Serves best-trade API for the smart router.  
`apis/farms` — Serves farm APR data.  
`apis/proxy-worker` — General proxy worker.  
Deployed with `wrangler`.

## Code Conventions

### Imports

Lodash must be imported per-method (enforced by ESLint `lodash/import-scope: [error, 'method']`):
```ts
// ✅ correct
import isEmpty from 'lodash/isEmpty'
// ❌ wrong
import { isEmpty } from 'lodash'
```

Ethereum addresses must be checksummed (ESLint `address/addr-type: warn`).

### Localization

All user-visible strings must go through `useTranslation`:
```ts
import { useTranslation } from '@pancakeswap/localization'
const { t } = useTranslation()
t('You have %num% left', { num: cakeBalance })
```
New keys are added to `locales/` and synced via Crowdin.

### Styling

- **Styled-components v6** is the primary styling approach
- **Vanilla Extract** (`@vanilla-extract/vite-plugin`) is used in some packages for zero-runtime CSS
- Stylelint enforces CSS-in-JS style rules (`.stylelintrc.js`)

### Prettier Config
```json
{ "trailingComma": "all", "semi": false, "singleQuote": true, "printWidth": 120 }
```

### TypeScript

- TypeScript 5.7.3; strict settings via `@pancakeswap/tsconfig`
- `apps/web` has two tsconfig files: `tsconfig.json` (app) and `tsconfig.test.json` (tests with different paths)
- Never edit `next-env.d.ts`

### Redux State Shape

When adding a new Redux slice, add it to `apps/web/src/state/index.ts` (the combined reducer) and declare its type in `apps/web/src/state/types.ts`.

### Pages vs Views

Pages in `src/pages/` are kept minimal — they just import and render the corresponding view. Feature logic, components, and hooks all live in `src/views/FeatureName/`.

## Environment Setup

Copy `apps/web/.env.example` to `apps/web/.env.local` and fill in RPC endpoints. Key variables:
- `NEXT_PUBLIC_NODE_REAL_API_ETH` / `SERVER_NODE_REAL_API_ETH` — NodeReal ETH RPC
- `NEXT_PUBLIC_NODIES_*` — Nodies RPC endpoints per chain
- `NEXT_PUBLIC_DEFAULT_PROJECT_ID` — WalletConnect project ID

The app runs with placeholder values without most keys, but RPC endpoints are needed for on-chain features.

## Changesets (for packages)

When modifying shared packages, add a changeset:
```bash
pnpm changeset      # Interactive prompt
pnpm version-packages  # Bump versions + update lockfile
```
