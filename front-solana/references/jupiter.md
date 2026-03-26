# Jupiter Swap Integration

Jupiter is the leading DEX aggregator on Solana. Use it when users need to swap one token for another.

## Install

```bash
npm install @jup-ag/api
```

## Setup

```ts
// src/lib/jupiter.ts
import { createJupiterApiClient } from '@jup-ag/api';

export const jupiterApi = createJupiterApiClient({
  basePath: 'https://quote-api.jup.ag/v6',  // default, can omit
});
```

## Get a Swap Quote

```ts
import { jupiterApi } from '@/lib/jupiter';

async function getSwapQuote({
  inputMint,
  outputMint,
  amount,         // in base units (e.g. lamports for SOL, 1_000_000 for 1 USDC)
  slippageBps = 50,  // 0.5% slippage
}: {
  inputMint: string;
  outputMint: string;
  amount: number;
  slippageBps?: number;
}) {
  return await jupiterApi.quoteGet({
    inputMint,
    outputMint,
    amount,
    slippageBps,
    onlyDirectRoutes: false,  // allow multi-hop routes for better prices
    asLegacyTransaction: false,  // use versioned transactions
  });
}
```

Common mints:
- SOL: `So11111111111111111111111111111111111111112`
- USDC: `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`
- USDT: `Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB`

## Execute a Swap

```ts
import { createWalletTransactionSigner } from '@solana/client';
import { useWalletConnection } from '@solana/react-hooks';
import { jupiterApi } from '@/lib/jupiter';
import { rpc, rpcSubscriptions } from '@/lib/rpc';
import {
  getBase64Decoder,
  sendAndConfirmTransactionFactory,
  signTransactionMessageWithSigners,
} from '@solana/kit';

async function executeSwap(quoteResponse: Awaited<ReturnType<typeof jupiterApi.quoteGet>>, wallet: NonNullable<ReturnType<typeof useWalletConnection>['wallet']>) {
  const { signer } = createWalletTransactionSigner(wallet);
  const userPublicKey = String(wallet.account.address);

  // Get the swap transaction from Jupiter
  const swapResult = await jupiterApi.swapPost({
    swapRequest: {
      quoteResponse,
      userPublicKey,
      wrapAndUnwrapSol: true,   // auto-wrap SOL to WSOL and unwrap after
      dynamicComputeUnitLimit: true,  // optimize CU limit
      prioritizationFeeLamports: 'auto', // auto-set priority fee
    },
  });

  // swapResult.swapTransaction is a base64-encoded versioned transaction
  const txBytes = Buffer.from(swapResult.swapTransaction, 'base64');

  // The wallet signs the pre-built transaction bytes directly
  const signedTx = await (signer as { signTransaction: (tx: Uint8Array) => Promise<Uint8Array> })
    .signTransaction(new Uint8Array(txBytes));

  const signature = await rpc.sendTransaction(
    Buffer.from(signedTx).toString('base64') as `${string}`,
    { encoding: 'base64', skipPreflight: false }
  ).send();

  // Confirm
  const { value: latestBlockhash } = await rpc.getLatestBlockhash().send();
  await rpc.confirmTransaction({
    signature: signature as `${string}`,
    ...latestBlockhash,
  }, { commitment: 'confirmed' }).send();

  return signature;
}
```

## useSwap Hook

```tsx
import { useState } from 'react';
import { useWalletConnection } from '@solana/react-hooks';
import { jupiterApi } from '@/lib/jupiter';

export function useSwap() {
  const { wallet } = useWalletConnection();
  const [pending, setPending] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [signature, setSignature] = useState<string | null>(null);

  const swap = async ({
    inputMint,
    outputMint,
    amount,
    slippageBps = 50,
  }: {
    inputMint: string;
    outputMint: string;
    amount: number;
    slippageBps?: number;
  }) => {
    if (!wallet) throw new Error('Wallet not connected');
    setPending(true);
    setError(null);
    setSignature(null);

    try {
      const quote = await jupiterApi.quoteGet({ inputMint, outputMint, amount, slippageBps });
      const sig = await executeSwap(quote, wallet);
      setSignature(sig);
      return sig;
    } catch (e: unknown) {
      const msg = (e as { message?: string })?.message ?? 'Swap failed';
      setError(msg);
      throw e;
    } finally {
      setPending(false);
    }
  };

  return { swap, pending, error, signature };
}
```

## Swap UI Component

```tsx
import { useState } from 'react';
import { useWalletConnection } from '@solana/react-hooks';
import { useSwap } from '@/hooks/useSwap';
import { jupiterApi } from '@/lib/jupiter';

export function SwapForm() {
  const { connected } = useWalletConnection();
  const { swap, pending, error, signature } = useSwap();

  const [inputMint, setInputMint] = useState('So11111111111111111111111111111111111111112');
  const [outputMint, setOutputMint] = useState('EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v');
  const [amount, setAmount] = useState('');
  const [quote, setQuote] = useState<Awaited<ReturnType<typeof jupiterApi.quoteGet>> | null>(null);
  const [quoteLoading, setQuoteLoading] = useState(false);

  const fetchQuote = async () => {
    if (!amount || !inputMint || !outputMint) return;
    setQuoteLoading(true);
    try {
      const q = await jupiterApi.quoteGet({
        inputMint,
        outputMint,
        amount: Math.round(parseFloat(amount) * 1e9), // e.g. SOL → lamports
        slippageBps: 50,
      });
      setQuote(q);
    } catch (e) {
      console.error('Quote error', e);
    } finally {
      setQuoteLoading(false);
    }
  };

  const handleSwap = async () => {
    if (!quote) return;
    await swap({
      inputMint,
      outputMint,
      amount: Math.round(parseFloat(amount) * 1e9),
    });
  };

  if (!connected) return <p>Connect wallet to swap</p>;

  return (
    <div>
      <input value={amount} onChange={e => setAmount(e.target.value)} placeholder="Amount" type="number" />
      <button onClick={fetchQuote} disabled={quoteLoading}>
        {quoteLoading ? 'Getting quote…' : 'Get Quote'}
      </button>
      {quote && (
        <div>
          <p>You'll receive: {Number(quote.outAmount) / 1e6} USDC</p>
          <p>Price impact: {quote.priceImpactPct}%</p>
          <button onClick={handleSwap} disabled={pending}>
            {pending ? 'Swapping…' : 'Swap'}
          </button>
        </div>
      )}
      {error && <p className="text-red-500">{error}</p>}
      {signature && (
        <a href={`https://explorer.solana.com/tx/${signature}?cluster=devnet`} target="_blank" rel="noreferrer">
          View transaction
        </a>
      )}
    </div>
  );
}
```

## Quote Response Fields

Key fields on the `quoteResponse` object:

```ts
quote.inputMint             // input token mint
quote.inAmount              // amount in (base units, string)
quote.outputMint            // output token mint
quote.outAmount             // amount out (base units, string) — after fees/slippage
quote.otherAmountThreshold  // minimum amount out given slippage
quote.swapMode              // "ExactIn" | "ExactOut"
quote.slippageBps           // configured slippage
quote.priceImpactPct        // price impact as percentage string
quote.routePlan             // array of DEXes used in the route
quote.contextSlot           // the slot this quote was computed at
```

## Key Points

- Jupiter V6 API is the current version (`quote-api.jup.ag/v6`)
- `slippageBps: 50` = 0.5% slippage — reasonable default; increase for illiquid tokens
- `wrapAndUnwrapSol: true` — always set this when the user might be swapping native SOL; it auto-wraps/unwraps WSOL
- `dynamicComputeUnitLimit: true` — lets Jupiter optimize the compute budget
- Jupiter returns a fully-built versioned transaction — you just sign and send it
- For devnet testing, Jupiter only works with tokens that have real liquidity pools; most test tokens won't have routes

## Environment

Jupiter's API is mainnet-only by design (no devnet). For development:
- Test swap UI logic on mainnet with small amounts
- Use mock/placeholder swap logic on devnet
- Point your API client at the devnet cluster only for non-Jupiter on-chain calls