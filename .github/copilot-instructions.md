# Copilot Instructions

## Build, Test, and Lint

```bash
npm install
npm run build        # builds both main and cleanup entry points via ncc
npm run build:main   # build src/main.ts → lib/main/index.js
npm run build:cleanup # build src/cleanup.ts → lib/cleanup/index.js
npm test             # run all Jest tests
```

Run a single test file:
```bash
npx jest __tests__/LoginConfig.test.ts
```

Run a single test by name:
```bash
npx jest -t "initialize with creds"
```

The `lib/` directory contains the bundled output (committed to the repo); always run `npm run build` after source changes before committing.

## Architecture

This is a GitHub Action with two entry points:

- **`src/main.ts`** — runs on action start: sets user-agent, optionally pre-cleans Azure accounts, then initializes/validates `LoginConfig` and triggers CLI and optionally PowerShell login.
- **`src/cleanup.ts`** — runs as the `post` step (after the job): logs out by clearing cached Azure CLI and PowerShell accounts.

The `action.yml` wires these to `lib/main/index.js` and `lib/cleanup/index.js` (bundled by `@vercel/ncc`).

### Key classes

| Class | Location | Responsibility |
|---|---|---|
| `LoginConfig` | `src/common/LoginConfig.ts` | Reads action inputs, parses `creds` JSON, fetches OIDC federated token, validates config |
| `AzureCliLogin` | `src/Cli/AzureCliLogin.ts` | Executes `az` CLI commands for all auth flows |
| `AzPSLogin` | `src/PowerShell/AzPSLogin.ts` | Orchestrates PowerShell login via script execution |
| `AzPSScriptBuilder` | `src/PowerShell/AzPSScriptBuilder.ts` | Builds PowerShell `Connect-AzAccount` scripts as strings |
| `AzPSUtils` | `src/PowerShell/AzPSUtils.ts` | Utilities for running PS scripts and setting PS module path |

### Authentication flows

`LoginConfig.authType` drives branching throughout both `AzureCliLogin` and `AzPSScriptBuilder`:

- **`SERVICE_PRINCIPAL`** — with `servicePrincipalSecret` → secret-based; without → OIDC (fetches GitHub ID token via `core.getIDToken`)
- **`IDENTITY`** — with `servicePrincipalId` → user-assigned managed identity; without → system-assigned

## Key Conventions

- **`creds` input is legacy**: When both `creds` and individual parameters (`client-id`, `tenant-id`, `subscription-id`) are provided, individual parameters take precedence and `creds` is ignored with a warning.
- **Sensitive values are masked**: `LoginConfig.mask()` calls `core.setSecret()` for `servicePrincipalId`, `servicePrincipalSecret`, and `federatedToken`.
- **Tests use env vars to simulate action inputs**: `@actions/core` reads inputs from `INPUT_<NAME>` environment variables. Tests set these directly via `process.env[`INPUT_${name.toUpperCase()}`]` and clean up in `beforeEach`.
- **Azure CLI version gating**: `AzureCliLogin.loginWithUserAssignedIdentity` parses the CLI minor version — `--username` is used for `< 2.69.0`, `--client-id` for `>= 2.69.0`.
- **PowerShell scripts return JSON**: All PS scripts are structured with `$output = @{}` and end with `return ConvertTo-Json $output`, with `Success`/`Error`/`Result` keys.
- **`azurestack` environment requires special handling**: Both `AzureCliLogin` (re-registers the cloud endpoint) and `AzPSScriptBuilder` (calls `Add-AzEnvironment`) have special cases for `environment == "azurestack"`.
- **TypeScript strict mode is off**: `"strict": false` in `tsconfig.json` — avoid adding strict-mode-breaking patterns.
