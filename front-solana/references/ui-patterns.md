# UI Patterns Reference

Reusable UI utilities and components for Solana dApp frontends.

## `cn()` utility

```ts
// src/lib/utils.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';
export function cn(...inputs: ClassValue[]) { return twMerge(clsx(inputs)); }
```

---

## Button Component (shadcn-style)

```tsx
// src/components/ui/button.tsx
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils';

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default:     'bg-primary text-primary-foreground shadow hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline:     'border border-input bg-background hover:bg-accent hover:text-accent-foreground',
        ghost:       'hover:bg-accent hover:text-accent-foreground',
      },
      size: {
        default: 'h-9 px-4 py-2',
        sm:      'h-8 rounded-md px-3 text-xs',
        lg:      'h-10 rounded-md px-8',
        icon:    'h-9 w-9',
      },
    },
    defaultVariants: { variant: 'default', size: 'default' },
  },
);

interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {}

export function Button({ className, variant, size, ...props }: ButtonProps) {
  return <button className={cn(buttonVariants({ variant, size }), className)} {...props} />;
}
```

---

## TokenSelect Component

Searchable token picker that shows logo, symbol, and balance. Used for "pay with" / "receive" token selectors in swap or escrow UIs.

```tsx
// src/components/TokenSelect.tsx
import { useState, useRef, useEffect } from 'react';
import { ChevronDown, Search, Coins } from 'lucide-react';
import { cn } from '@/lib/utils';
import type { TokenAccount } from '@/lib/helius'; // { mint, symbol, name, logo?, balance, decimals }

interface TokenSelectProps {
  tokens: TokenAccount[];
  value: TokenAccount | null;
  onChange: (token: TokenAccount) => void;
  placeholder?: string;
  disabled?: boolean;
}

export function TokenSelect({
  tokens,
  value,
  onChange,
  placeholder = 'Select token',
  disabled = false,
}: TokenSelectProps) {
  const [open, setOpen] = useState(false);
  const [query, setQuery] = useState('');
  const ref = useRef<HTMLDivElement>(null);

  // Close on outside click
  useEffect(() => {
    function handle(e: MouseEvent) {
      if (ref.current && !ref.current.contains(e.target as Node)) {
        setOpen(false);
        setQuery('');
      }
    }
    document.addEventListener('mousedown', handle);
    return () => document.removeEventListener('mousedown', handle);
  }, []);

  const filtered = tokens.filter(t =>
    t.symbol.toLowerCase().includes(query.toLowerCase()) ||
    t.name.toLowerCase().includes(query.toLowerCase()) ||
    t.mint.toLowerCase().includes(query.toLowerCase()),
  );

  return (
    <div className="relative" ref={ref}>
      {/* Trigger */}
      <button
        type="button"
        disabled={disabled}
        onClick={() => setOpen(v => !v)}
        className={cn(
          'flex w-full items-center gap-2 rounded-lg border border-slate-700 bg-slate-800 px-3 py-2 text-sm transition-colors',
          'hover:bg-slate-700 focus:outline-none focus:ring-1 focus:ring-slate-500',
          'disabled:cursor-not-allowed disabled:opacity-50',
        )}
      >
        {value ? (
          <>
            <TokenLogo logo={value.logo} symbol={value.symbol} size={20} />
            <span className="font-medium text-slate-100">{value.symbol}</span>
            <span className="ml-auto text-slate-400">{value.balance.toLocaleString()}</span>
          </>
        ) : (
          <span className="text-slate-400">{placeholder}</span>
        )}
        <ChevronDown className={cn('ml-1 h-4 w-4 text-slate-400 transition-transform', open && 'rotate-180')} />
      </button>

      {/* Dropdown */}
      {open && (
        <div className="absolute z-50 mt-1 w-full rounded-xl border border-slate-700 bg-slate-900 shadow-2xl">
          {/* Search */}
          <div className="flex items-center gap-2 border-b border-slate-700 px-3 py-2">
            <Search className="h-4 w-4 text-slate-500" />
            <input
              autoFocus
              value={query}
              onChange={e => setQuery(e.target.value)}
              placeholder="Search by name or mint…"
              className="flex-1 bg-transparent text-sm text-slate-200 placeholder-slate-500 focus:outline-none"
            />
          </div>

          {/* List */}
          <ul className="max-h-60 overflow-y-auto p-1">
            {filtered.length === 0 ? (
              <li className="px-3 py-4 text-center text-sm text-slate-500">No tokens found</li>
            ) : (
              filtered.map(token => (
                <li key={token.mint}>
                  <button
                    type="button"
                    onClick={() => { onChange(token); setOpen(false); setQuery(''); }}
                    className={cn(
                      'flex w-full items-center gap-3 rounded-lg px-3 py-2 text-sm transition-colors hover:bg-slate-800',
                      value?.mint === token.mint && 'bg-slate-800',
                    )}
                  >
                    <TokenLogo logo={token.logo} symbol={token.symbol} size={28} />
                    <div className="flex flex-col items-start">
                      <span className="font-medium text-slate-100">{token.symbol}</span>
                      <span className="text-xs text-slate-500">{token.name}</span>
                    </div>
                    <span className="ml-auto text-slate-300">{token.balance.toLocaleString()}</span>
                  </button>
                </li>
              ))
            )}
          </ul>
        </div>
      )}
    </div>
  );
}

// Internal logo with fallback
function TokenLogo({ logo, symbol, size }: { logo?: string; symbol: string; size: number }) {
  const [error, setError] = useState(false);

  if (logo && !error) {
    return (
      <img
        src={logo}
        alt={symbol}
        width={size}
        height={size}
        className="rounded-full object-cover"
        style={{ width: size, height: size }}
        onError={() => setError(true)}
      />
    );
  }

  return (
    <div
      className="flex items-center justify-center rounded-full bg-slate-700 text-slate-400"
      style={{ width: size, height: size }}
    >
      <Coins style={{ width: size * 0.55, height: size * 0.55 }} />
    </div>
  );
}
```

---

## TransactionStatus Component

Shows pending / success / error states after sending a transaction.

```tsx
// src/components/TransactionStatus.tsx
import { Loader2, CheckCircle2, XCircle, ExternalLink } from 'lucide-react';

type Status = 'idle' | 'pending' | 'success' | 'error';

interface TransactionStatusProps {
  status: Status;
  signature?: string | null;
  error?: string | null;
  cluster?: 'devnet' | 'mainnet-beta';
}

export function TransactionStatus({
  status,
  signature,
  error,
  cluster = 'devnet',
}: TransactionStatusProps) {
  if (status === 'idle') return null;

  return (
    <div className="mt-3 rounded-lg border border-slate-700 bg-slate-800/50 px-4 py-3 text-sm">
      {status === 'pending' && (
        <div className="flex items-center gap-2 text-slate-300">
          <Loader2 className="h-4 w-4 animate-spin" />
          <span>Sending transaction…</span>
        </div>
      )}
      {status === 'success' && signature && (
        <div className="flex items-center gap-2 text-green-400">
          <CheckCircle2 className="h-4 w-4" />
          <span>Transaction confirmed</span>
          <a
            href={`https://explorer.solana.com/tx/${signature}?cluster=${cluster}`}
            target="_blank"
            rel="noreferrer"
            className="ml-auto flex items-center gap-1 text-xs text-slate-400 hover:text-slate-200"
          >
            Explorer <ExternalLink className="h-3 w-3" />
          </a>
        </div>
      )}
      {status === 'error' && (
        <div className="flex items-center gap-2 text-red-400">
          <XCircle className="h-4 w-4" />
          <span>{error ?? 'Transaction failed'}</span>
        </div>
      )}
    </div>
  );
}
```

Usage:

```tsx
const [txStatus, setTxStatus] = useState<'idle'|'pending'|'success'|'error'>('idle');
const [signature, setSignature] = useState<string | null>(null);
const [txError, setTxError] = useState<string | null>(null);

const handleSubmit = async () => {
  setTxStatus('pending');
  try {
    const sig = await executeTransaction(signer, [ix]);
    setSignature(sig);
    setTxStatus('success');
  } catch (e: unknown) {
    setTxError((e as { message?: string })?.message ?? 'Unknown error');
    setTxStatus('error');
  }
};

// In JSX:
<button disabled={txStatus === 'pending'} onClick={handleSubmit}>Send</button>
<TransactionStatus status={txStatus} signature={signature} error={txError} />
```

---

## Dark Theme Base (Tailwind v4)

```css
/* src/index.css */
@import "tailwindcss";

:root {
  color-scheme: dark;
  background-color: #0f172a;
  color: #f1f5f9;
}
```

Common palette used across components:
- Background layers: `slate-900` → `slate-800` → `slate-700`
- Borders: `slate-700`
- Text primary: `slate-100`, secondary: `slate-400`, muted: `slate-500`
- Accent/action: `blue-600` hover `blue-500`
- Success: `green-400`, Error: `red-400`

---

## Common Patterns

### Loading skeleton

```tsx
function Skeleton({ className }: { className?: string }) {
  return <div className={cn('animate-pulse rounded-md bg-slate-700/50', className)} />;
}

// Usage
{loading ? <Skeleton className="h-9 w-full" /> : <TokenSelect ... />}
```

### Empty state

```tsx
function EmptyState({ message }: { message: string }) {
  return (
    <div className="flex flex-col items-center gap-2 py-12 text-slate-500">
      <Coins className="h-8 w-8" />
      <p className="text-sm">{message}</p>
    </div>
  );
}
```

### Amount input with max button

```tsx
function AmountInput({
  value, onChange, max, symbol,
}: { value: string; onChange: (v: string) => void; max?: number; symbol?: string }) {
  return (
    <div className="flex items-center gap-2 rounded-lg border border-slate-700 bg-slate-800 px-3 py-2">
      <input
        type="number"
        min="0"
        step="any"
        value={value}
        onChange={e => onChange(e.target.value)}
        placeholder="0.00"
        className="flex-1 bg-transparent text-lg font-medium text-slate-100 focus:outline-none"
      />
      {symbol && <span className="text-sm text-slate-400">{symbol}</span>}
      {max !== undefined && (
        <button
          type="button"
          onClick={() => onChange(String(max))}
          className="rounded px-2 py-0.5 text-xs text-blue-400 hover:bg-slate-700"
        >
          MAX
        </button>
      )}
    </div>
  );
}
```