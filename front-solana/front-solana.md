---
name: front-solana
description: >
  Always use this skill for ANY Solana frontend work with Vite + React + TypeScript.
  Trigger whenever the user mentions: wallet connection, @solana/kit, anza-kit, Phantom/Backpack,
  Helius, Codama, IDL codegen, Jupiter swap, SPL tokens, program accounts, PDAs, transaction sending,
  token balances, escrow UI, dApp scaffold — or any combination of Solana + React/Vite.
  Do NOT attempt Solana frontend tasks without consulting this skill first.
---

# Solana Frontend Skill

Stack: **Vite + React + TypeScript + @solana/kit + Tailwind v4 + shadcn-style UI + Helius DAS + Codama + Jupiter**

## Reference Files

Load the relevant reference before starting. Multiple refs may apply.

| Task | Read |
|------|------|
| Wallet connect, `SolanaProvider`, `WalletButton` | `references/wallet-setup.md` |
| Token balances, NFT metadata, Helius DAS API | `references/helius.md` |
| Codama codegen, instruction builders, account decoders, PDAs | `references/codama.md` |
| Jupiter swap quotes and execution | `references/jupiter.md` |
| UI components: `cn()`, `TokenSelect`, shadcn-style patterns | `references/ui-patterns.md` |

---

## 1. Project Scaffold

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app
```

### Core dependencies

```bash
# Solana
npm install @solana/kit @solana/client @solana/react-hooks
npm install @solana-program/token @solana-program/token-2022 \
            @solana-program/associated-token-account \
            @solana-program/compute-budget

# UI
npm install -D @tailwindcss/vite tailwindcss
npm install @radix-ui/react-dialog @radix-ui/react-dropdown-menu \
            @radix-ui/react-label @radix-ui/react-select @radix-ui/react-slot
npm install class-variance-authority clsx tailwind-merge lucide-react
```

### `vite.config.ts`

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';
import path from 'path';

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') },
  },
  define: { global: 'globalThis' }, // required — Solana deps use `global`
});
```

### `tsconfig.app.json` (critical settings)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "erasableSyntaxOnly": false,
    "paths": { "@/*": ["./src/*"] }
  }
}
```

> `erasableSyntaxOnly: false` — Codama generates regular TS enums; this must stay off.

### `src/index.css`

```css
@import "tailwindcss";

:root {
  color-scheme: dark;
  background-color: #0f172a;
  color: #f1f5f9;
}
```

### `.env`

```
VITE_SOLANA_RPC_URL=https://devnet.helius-rpc.com/?api-key=YOUR_KEY
VITE_SOLANA_WS_URL=wss://devnet.helius-rpc.com/?api-key=YOUR_KEY
VITE_HELIUS_API_KEY=YOUR_KEY
```

---

## 2. App Entry Points

### `src/lib/solana.ts`

```ts
import { createClient, autoDiscover } from '@solana/client';

export const solanaClient = createClient({
  endpoint: import.meta.env.VITE_SOLANA_RPC_URL ?? 'https://api.devnet.solana.com',
  websocketEndpoint: import.meta.env.VITE_SOLANA_WS_URL ?? 'wss://api.devnet.solana.com',
  walletConnectors: autoDiscover(),
});
```

### `src/lib/rpc.ts`

```ts
import { createSolanaRpc, createSolanaRpcSubscriptions, devnet } from '@solana/kit';

export const rpc = createSolanaRpc(devnet(import.meta.env.VITE_SOLANA_RPC_URL));
export const rpcSubscriptions = createSolanaRpcSubscriptions(devnet(import.meta.env.VITE_SOLANA_WS_URL));
```

### `src/main.tsx`

```tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { SolanaProvider } from '@solana/react-hooks';
import { solanaClient } from '@/lib/solana';
import App from './App';
import './index.css';

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <SolanaProvider client={solanaClient}>
      <App />
    </SolanaProvider>
  </StrictMode>,
);
```

---

## 3. Send a Transaction

```ts
import {
  createTransactionMessage,
  appendTransactionMessageInstructions,
  setTransactionMessageFeePayerSigner,
  setTransactionMessageLifetimeUsingBlockhash,
  signTransactionMessageWithSigners,
  getBase58Codec,
  type TransactionSigner,
  type IInstruction,
} from '@solana/kit';
import { rpc, rpcSubscriptions } from '@/lib/rpc';
import { sendAndConfirmTransactionFactory } from '@solana/kit';

const sendAndConfirm = sendAndConfirmTransactionFactory({ rpc, rpcSubscriptions });

export async function executeTransaction(
  signer: TransactionSigner,
  instructions: IInstruction[],
): Promise<string> {
  const { value: latestBlockhash } = await rpc.getLatestBlockhash().send();

  const message = setTransactionMessageFeePayerSigner(
    signer,
    setTransactionMessageLifetimeUsingBlockhash(
      latestBlockhash,
      appendTransactionMessageInstructions(instructions, createTransactionMessage({ version: 0 })),
    ),
  );

  const signed = await signTransactionMessageWithSigners(message);
  await (sendAndConfirm as (tx: typeof signed, opts: object) => Promise<void>)(
    signed, { commitment: 'confirmed' },
  );

  const codec = getBase58Codec();
  return codec.encode(signed.signatures[Object.keys(signed.signatures)[0]]);
}
```

**Transaction UX checklist:**
- Disable submit while `pending`
- Spinner during send/confirm
- Explorer link on success (`https://explorer.solana.com/tx/{sig}?cluster=devnet`)
- Distinct errors: user rejection / insufficient funds / expired blockhash

---

## 4. Core Rules

### Architecture
- One `createClient()` for the entire app — never inside a component
- One `<SolanaProvider>` at the root
- Keep `autoDiscover()` — handles wallets injected after page load

### TypeScript
- `erasableSyntaxOnly: false` — required for Codama enums
- Always wrap RPC URL with `devnet()` / `mainnet()`
- `createWalletTransactionSigner(wallet)` → destructure `{ signer }`
- `wallet.connector?.icon` / `wallet.connector?.name` — not `wallet.icon`

### Accounts & Data
- Always add discriminator `memcmp` filter when using `getProgramAccounts`
- Use Helius DAS (`getAssetsByOwner`) for token metadata — not raw `getTokenAccountsByOwner`
- Store balances as `rawBalance: bigint` + `balance: number` (divided by decimals)

### Mainnet checklist
- Swap `devnet()` → `mainnet()` in `rpc.ts`
- Update `.env` to mainnet Helius endpoints
- Jupiter works mainnet-only (mock or skip on devnet)
- Audit all hardcoded mint addresses