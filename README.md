# Tranche LP

Structured liquidity for Uniswap v4 — split LP cash flows into **senior** (fixed yield, IL-protected) and **junior** (leveraged fees, first-loss) tranches via a custom hook.

| Package | Description |
|---------|-------------|
| [`packages/hook`](packages/hook) | `TrancheLPHook` — Solidity contracts, Foundry tests, scenario runner ([submodule](https://github.com/jimmychu0807/uhi9-hook)) |
| [`packages/web`](packages/web) | Next.js frontend (early scaffold) |

## How it works

A standard concentrated-liquidity position earns **swap fees** and bears **impermanent loss (IL)**. Tranche LP separates those flows:

```
┌─────────────────────────────────────┐
│         POOL TOTAL VALUE            │
├─────────────────────────────────────┤
│  SENIOR — fixed APY, fees first,    │
│           no IL while buffer holds  │
├─────────────────────────────────────┤
│  JUNIOR — residual fees, absorbs    │
│           all IL (first-loss)       │
└─────────────────────────────────────┘
```

On deposit, LPs choose a personal senior fraction `φ` (0–100%). The hook mints **Senior Receipt Tokens (SRT)** and **Junior Receipt Tokens (JRT)** as ERC-1155 shares. Accrual runs on swaps using TWAP-based IL and a configurable senior fixed rate (default 8% APY).

## Repository structure

```
tranche-lp/
├── packages/
│   ├── hook/          # Uniswap v4 hook (git submodule)
│   └── web/           # Next.js app
├── package.json       # Bun workspaces root
└── .github/workflows/ # CI per package
```

> [!NOTE]
> Clone with submodules so the hook package is populated:
>
> ```bash
> git clone --recurse-submodules <repo-url>
> # or after a plain clone:
> git submodule update --init --recursive
> ```

## Prerequisites

- [Bun](https://bun.sh) (workspace package manager)
- [Foundry](https://book.getfoundry.sh/getting-started/installation) (for `packages/hook`)

## Quick start

```bash
# Install JS dependencies (root + workspaces)
bun install

# Run the web app
bun run dev
# → http://localhost:3000

# Build / lint web
bun run build
bun run lint
```

### Hook contracts

```bash
cd packages/hook
forge install          # if lib/ is empty
forge build
forge test -vvv
```

### Scenario tests

Declarative `.scenario` files drive end-to-end Foundry tests (deposits, swaps, time passes, assertions):

```bash
cd packages/hook
bun run test-scenario -- simple01
bun run test-all-scenarios
```

## CI

| Workflow | Trigger | Command |
|----------|---------|---------|
| `ci-hook.yml` | `packages/hook/**` | `forge build` + `forge test` |
| `ci-web.yml` | `packages/web/**` | `bun run build` + `bun run lint` |

## Status

- **Hook**: MVP implementation with unit tests, scenario harness, and deploy scripts
- **Web**: Next.js 16 scaffold — onchain integration not yet wired

## Related

- Hook submodule: [jimmychu0807/uhi9-hook](https://github.com/jimmychu0807/uhi9-hook)
- [Uniswap v4 docs](https://docs.uniswap.org/contracts/v4/overview)
