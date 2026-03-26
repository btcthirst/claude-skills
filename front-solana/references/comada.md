# Codama Reference

Codama generates fully-typed TypeScript clients from Anchor IDL files. It produces instruction builders, account decoders, PDA derivation helpers, and error types — so you never write raw instruction data by hand.

## Install

```bash
npm install -D codama @codama/nodes-from-anchor @codama/renderers-js
```

## Codegen Script

```js
// scripts/codegen.mjs
import { readFileSync } from 'fs';
import { createFromRoot } from 'codama';
import { rootNodeFromAnchor } from '@codama/nodes-from-anchor';
import { renderVisitor } from '@codama/renderers-js';

const idlPath = './your-program.json';  // path to Anchor IDL
const outputPath = './src/generated';  // where to write the client

const idl = JSON.parse(readFileSync(idlPath, 'utf-8'));
const codama = createFromRoot(rootNodeFromAnchor(idl));
codama.accept(renderVisitor(outputPath, { deleteFolderBeforeRendering: true }));
console.log('✅ Generated Codama client');
```

Run it:

```bash
node scripts/codegen.mjs
```

Add to `package.json` for convenience:

```json
{
  "scripts": {
    "codegen": "node scripts/codegen.mjs"
  }
}
```

## Output Structure

Codama generates files nested under the output path. The actual generated code lands at `<outputPath>/src/generated/`:

```
src/generated/
└── src/
    └── generated/
        ├── accounts/
        │   └── offer.ts          → decodeOffer(), OFFER_DISCRIMINATOR, type Offer
        ├── instructions/
        │   ├── makeOffer.ts      → getMakeOfferInstruction(), getMakeOfferInstructionAsync()
        │   └── takeOffer.ts      → getTakeOfferInstruction(), getTakeOfferInstructionAsync()
        ├── programs/
        │   └── escrow.ts         → ESCROW_PROGRAM_ADDRESS, enum EscrowInstruction
        ├── errors/
        │   └── escrow.ts         → error codes/messages
        └── index.ts
```

### tsconfig.app.json path alias

```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"],
      "@generated/*": ["./src/generated/src/generated/*"]
    }
  }
}
```

### vite.config.ts alias

```ts
resolve: {
  alias: {
    '@': path.resolve(__dirname, './src'),
    '@generated': path.resolve(__dirname, './src/generated/src/generated'),
  },
},
```

## Using Generated Code

### Account decoder

```ts
import { decodeOffer, OFFER_DISCRIMINATOR, type Offer } from '@generated/accounts/offer';

// Decode a raw account
const encodedAccount = {
  address: address(pubkey),
  data: new Uint8Array(rawBytes),
  executable: false,
  lamports: lamportsBigint,
  programAddress: ESCROW_PROGRAM_ADDRESS,
};

const result = decodeOffer(encodedAccount);
if (result.exists) {
  const offer: Offer = result.data;
  console.log(offer.maker, offer.tokenMintA, offer.tokenBWantedAmount);
}
```

### Sync instruction builder

Use when no PDA derivation is needed:

```ts
import { getMakeOfferInstruction } from '@generated/instructions/makeOffer';

const ix = getMakeOfferInstruction({
  maker: signer,
  tokenMintA: mintAAddress,
  tokenMintB: mintBAddress,
  offer: offerPda,  // must provide PDA manually
  vault: vaultAta,  // must provide vault ATA manually
  id: BigInt(offerId),
  tokenAOfferedAmount: BigInt(rawAmount),
  tokenBWantedAmount: BigInt(wantedRaw),
});
```

### Async instruction builder (recommended)

The `Async` variant automatically derives PDAs defined in the IDL:

```ts
import { getMakeOfferInstructionAsync } from '@generated/instructions/makeOffer';

// PDAs (offer account PDA, vault ATA) are derived automatically
const ix = await getMakeOfferInstructionAsync({
  maker: signer,
  tokenMintA: address(mintA) as Address,
  tokenMintB: address(mintB) as Address,
  id: BigInt(offerId),
  tokenAOfferedAmount: BigInt(Math.round(amountA * 10 ** decimalsA)),
  tokenBWantedAmount: BigInt(Math.round(amountB * 10 ** decimalsB)),
});
```

### Deriving PDAs manually

Sometimes you need the PDA address before calling the instruction builder (e.g., to fetch the account first):

```ts
import {
  getProgramDerivedAddress,
  getUtf8Encoder,
  getU64Encoder,
  getBase58Codec,
  address,
} from '@solana/kit';
import { ESCROW_PROGRAM_ADDRESS } from '@generated/programs/escrow';

async function deriveOfferPda(maker: string, id: bigint): Promise<Address> {
  const [pda] = await getProgramDerivedAddress({
    programAddress: ESCROW_PROGRAM_ADDRESS,
    seeds: [
      getUtf8Encoder().encode('offer'),        // string seed
      getBase58Codec().encode(address(maker)), // pubkey seed (32 bytes)
      getU64Encoder().encode(id),              // u64 seed (8 bytes LE)
    ],
  });
  return pda;
}
```

The seeds must match your program's PDA derivation exactly. Check the IDL's `seeds` array for the account definition.

### Fetching all accounts of a type

```ts
import {
  createSolanaRpc,
  devnet,
  address,
  getBase58Codec,
  type Base58EncodedBytes,
} from '@solana/kit';
import { decodeOffer, OFFER_DISCRIMINATOR } from '@generated/accounts/offer';
import { ESCROW_PROGRAM_ADDRESS } from '@generated/programs/escrow';

const rpc = createSolanaRpc(devnet(RPC_URL));

async function fetchAllOffers() {
  // Encode discriminator to base58 (required format for memcmp)
  const discriminatorB58 = getBase58Codec().decode(OFFER_DISCRIMINATOR) as Base58EncodedBytes;

  const response = await rpc.getProgramAccounts(address(ESCROW_PROGRAM_ADDRESS), {
    encoding: 'base64',
    filters: [{
      memcmp: {
        offset: 0n,
        bytes: discriminatorB58,
        encoding: 'base58',
      },
    }],
  }).send();

  const items = response as unknown as Array<{
    pubkey: string;
    account: {
      data: [string, string];
      lamports: bigint;
      executable: boolean;
      owner: string;
    };
  }>;

  return items.flatMap(item => {
    try {
      const dataBytes = Buffer.from(item.account.data[0], 'base64');
      const encoded = {
        address: address(item.pubkey),
        data: new Uint8Array(dataBytes),
        executable: item.account.executable,
        lamports: item.account.lamports,
        programAddress: address(ESCROW_PROGRAM_ADDRESS),
      };
      const decoded = decodeOffer(encoded as Parameters<typeof decodeOffer>[0]);
      return decoded.exists ? [{ address: item.pubkey, data: decoded.data }] : [];
    } catch {
      return [];  // skip malformed accounts
    }
  });
}
```

## TypeScript Gotchas

### `erasableSyntaxOnly` must be false

Codama generates regular TypeScript `enum`s (e.g., in `programs/escrow.ts`). If `erasableSyntaxOnly` is `true` in tsconfig, you'll get:

```
error TS5117: This syntax is not allowed when 'erasableSyntaxOnly' is enabled.
```

Fix: set `"erasableSyntaxOnly": false` in `tsconfig.app.json`.

### Nested output path

The generated output lands at `<outputPath>/src/generated/`, not directly at `<outputPath>/`. Set up path aliases accordingly.

### Instruction type inference

When the compiler complains about the input type for a generated instruction:

```ts
// If you get a type error on the encoded account:
const decoded = decodeOffer(encoded as Parameters<typeof decodeOffer>[0]);
```

### `takeOffer` requires explicit `offer` field

Some instructions reference another account's field in their PDA seeds (e.g., `offer.id`). Codama requires you to provide the PDA explicitly in these cases rather than deriving it automatically. Derive it with `getProgramDerivedAddress` and pass it in:

```ts
const offerPda = await deriveOfferPda(makerAddress, offerId);
const ix = await getTakeOfferInstructionAsync({
  taker: signer,
  maker: address(makerAddress),
  offer: offerPda,  // explicit — required when IDL uses cross-account seeds
  tokenMintA: address(tokenMintA),
  tokenMintB: address(tokenMintB),
});
```

## Full Workflow

1. Write your Anchor program
2. Build with `anchor build` (or obtain the IDL JSON)
3. Run `node scripts/codegen.mjs`
4. Import from `@generated/instructions/...` and `@generated/accounts/...`
5. Re-run codegen whenever you change the program interface