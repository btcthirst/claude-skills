# Helius Reference

Helius is the recommended data layer for Solana frontends. It provides:
- **DAS API** (Digital Asset Standard) — rich token/NFT metadata including names, symbols, logos, decimals
- **Enhanced RPC** — standard Solana RPC methods with higher rate limits and reliability
- **Webhooks** — real-time event notifications (server-side)

## Endpoints

```
# Devnet
https://devnet.helius-rpc.com/?api-key=YOUR_KEY
wss://devnet.helius-rpc.com/?api-key=YOUR_KEY

# Mainnet
https://mainnet.helius-rpc.com/?api-key=YOUR_KEY
wss://mainnet.helius-rpc.com/?api-key=YOUR_KEY
```

Get a free API key at https://dashboard.helius.dev

## Setup

```ts
// src/lib/helius.ts
const HELIUS_API_KEY = import.meta.env.VITE_HELIUS_API_KEY as string ?? 'YOUR_HELIUS_API_KEY';

export const HELIUS_RPC = `https://devnet.helius-rpc.com/?api-key=${HELIUS_API_KEY}`;

export async function heliusPost(method: string, params: unknown) {
  const resp = await fetch(HELIUS_RPC, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: method, method, params }),
  });
  if (!resp.ok) throw new Error(`Helius HTTP error: ${resp.status}`);
  const json = await resp.json();
  if (json.error) throw new Error(json.error.message ?? 'RPC error');
  return json.result;
}
```

## DAS API Methods

### getAssetsByOwner — Get all tokens/NFTs for a wallet

Best method for building a token selector UI. Returns rich metadata.

```ts
interface DasItem {
  id: string;                    // mint address
  content?: {
    metadata?: { symbol?: string; name?: string };
    links?: { image?: string };
  };
  token_info?: {
    balance?: number;            // raw on-chain balance (int, not divided by decimals)
    decimals?: number;
    symbol?: string;
    associated_token_address?: string;
    token_program?: string;      // "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA" | token-2022
  };
  interface?: string;            // "FungibleToken" | "FungibleAsset" | "ProgrammableNFT" | etc.
}

async function getTokenAccounts(ownerAddress: string): Promise<TokenAccount[]> {
  const result = await heliusPost('getAssetsByOwner', {
    ownerAddress,
    page: 1,
    limit: 100,
    displayOptions: {
      showFungible: true,
      showNativeBalance: false,   // exclude SOL (native balance)
      showInscription: false,
      showCollectionMetadata: false,
    },
  });

  const items: DasItem[] = result?.items ?? [];

  return items
    .filter(item => {
      const ti = item.token_info;
      return ti && (ti.balance ?? 0) > 0;  // only tokens with balance
    })
    .map(item => {
      const ti = item.token_info!;
      const decimals = ti.decimals ?? 6;
      const rawBalance = BigInt(Math.round(ti.balance ?? 0));
      const balance = Number(rawBalance) / 10 ** decimals;
      const symbol = ti.symbol || item.content?.metadata?.symbol || item.id.slice(0, 4) + '…';
      const name = item.content?.metadata?.name || symbol;
      const logo = item.content?.links?.image;

      return {
        mint: item.id,
        symbol,
        name,
        logo,
        balance,
        rawBalance,
        decimals,
        tokenAccount: ti.associated_token_address ?? '',
      };
    });
}
```

### getAsset — Get a single asset by mint

```ts
async function getAsset(mintAddress: string) {
  return await heliusPost('getAsset', { id: mintAddress });
}
```

### getAssetsByMint — Batch fetch multiple assets

```ts
async function getAssetsByMint(mints: string[]) {
  return await heliusPost('getAssetsByMint', { ids: mints });
}
```

### searchAssets — Query assets by various criteria

```ts
async function searchFungibleTokens(ownerAddress: string) {
  return await heliusPost('searchAssets', {
    ownerAddress,
    interface: 'FungibleToken',
    page: 1,
    limit: 50,
  });
}
```

## Standard RPC via Helius

Helius also supports all standard Solana RPC methods. Use it as your RPC endpoint for better reliability:

```ts
import { createSolanaRpc, devnet } from '@solana/kit';
export const rpc = createSolanaRpc(devnet(HELIUS_RPC));
```

This gives you standard RPC calls (`getBalance`, `getAccountInfo`, `getProgramAccounts`, etc.) with Helius's infrastructure.

## useTokenAccounts Hook

```ts
import { useState, useEffect } from 'react';

export interface TokenAccount {
  mint: string;
  symbol: string;
  name: string;
  logo?: string;
  balance: number;      // human-readable
  rawBalance: bigint;   // raw on-chain
  decimals: number;
  tokenAccount: string; // ATA address
}

export function useTokenAccounts(ownerAddress: string | null) {
  const [tokens, setTokens] = useState<TokenAccount[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (!ownerAddress) { setTokens([]); return; }

    let cancelled = false;
    setLoading(true);
    setError(null);

    getTokenAccounts(ownerAddress)
      .then(result => { if (!cancelled) setTokens(result); })
      .catch((e: unknown) => {
        if (!cancelled)
          setError((e as { message?: string })?.message ?? 'Failed to load tokens');
      })
      .finally(() => { if (!cancelled) setLoading(false); });

    return () => { cancelled = true; };
  }, [ownerAddress]);

  return { tokens, loading, error };
}
```

## Common DAS `interface` values

| Value | Meaning |
|-------|---------|
| `FungibleToken` | Standard SPL token (e.g. USDC, BONK) |
| `FungibleAsset` | Token with metadata (often meme coins with Metaplex) |
| `ProgrammableNFT` | pNFT |
| `NonFungible` | Standard NFT |
| `NonFungibleEdition` | NFT edition |

## Notes

- `token_info.balance` is the raw on-chain integer balance — divide by `10 ** decimals` for human-readable
- `token_info.balance` comes back as a JavaScript number, but for large values use `BigInt(Math.round(balance))`
- Some tokens have logos hosted on Arweave, IPFS, or CDNs — always handle `img` `onError` gracefully
- Rate limits: free tier is 10 req/s on devnet, higher on mainnet — add a Helius API key for production