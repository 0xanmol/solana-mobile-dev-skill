---
name: mobile-wallet-adapter-react-native
description: Integrate Mobile Wallet Adapter (MWA) for wallet connection and transaction signing in React Native Expo apps using Beeman's Wallet UI SDK (@wallet-ui/react-native-web3js). Use when the user requests to add wallet connection, integrate Solana wallet support, add "connect wallet" button, implement transaction signing, send SOL transfers, or set up MWA in their React Native app.
---

# Mobile Wallet Adapter Integration

Add secure wallet connection and transaction signing to React Native Expo apps using Mobile Wallet Adapter protocol.

## What This Skill Does

Integrates Mobile Wallet Adapter (MWA) using Beeman's production-ready SDK for:
- **Wallet Connection**: Connect button that triggers native wallet picker
- **Transaction Signing**: Sign and send SOL transfers and custom transactions
- **Session Persistence**: Automatic reconnection across app restarts
- **Complete Setup**: Provider configuration, crypto polyfills, error handling

## When to Use

Use this skill when the user asks to:
- "Add wallet connection to my React Native app"
- "Integrate Mobile Wallet Adapter"
- "Add a connect wallet button"
- "Implement Solana wallet support"
- "Send SOL transactions from my app"
- "Set up MWA in Expo"

## Prerequisites Check

Before implementing, verify:
1. **Not using Expo Go** (MWA requires development build)
2. **Android development environment** set up (MWA is Android-only currently)

## Implementation Workflow

### Step 1: Assess Existing Setup

Check what's already in place:
- Is there an `app/_layout.tsx` (Expo Router) or `index.tsx` entry point (React Navigation)?
- Are there existing wallet connection buttons or auth screens?
- Is `react-native-get-random-values` already imported?
- Does `constants/wallet.ts` or similar config exist?

Based on findings, integrate cleanly without duplicating code.

### Step 2: Install Dependencies

```bash
npm install @wallet-ui/react-native-web3js @solana/web3.js @tanstack/react-query react-native-get-random-values
```

**Critical**: After installing, must run development build:

```bash
npx expo prebuild --clean
npx expo run:android
```

**Why?** MWA uses native Android modules that Expo Go doesn't include.

### Step 3: Setup Crypto Polyfill

**CRITICAL**: The crypto polyfill must be the **VERY FIRST import** in the app entry point.

**For Expo Router apps** (`app/_layout.tsx`):
```typescript
import 'react-native-get-random-values'; // IMPORTANT: MUST BE FIRST

// Then other imports...
import { Stack } from 'expo-router';
```

**For React Navigation apps** (`index.tsx`):
```typescript
// CRITICAL: Must be first import for crypto polyfill
import 'react-native-get-random-values';

import '@expo/metro-runtime';
import { registerRootComponent } from 'expo';
// ... rest of imports
```

**If already present**, verify it's the first import. If not, move it to the top.

**Why This Matters**:
- Polyfills global `crypto` object
- Other modules check for crypto on import
- Wrong order = "crypto.getRandomValues() not supported" errors
- Transactions will fail without it

### Step 4: Environment Configuration

**Create `.env` file** in project root:

```bash
EXPO_PUBLIC_SOLANA_CLUSTER=devnet
EXPO_PUBLIC_SOLANA_RPC_ENDPOINT=https://api.devnet.solana.com
```

**Create `constants/wallet.ts`** (or add to existing constants):

```typescript
import type { Chain } from '@solana-mobile/mobile-wallet-adapter-protocol';

export const APP_IDENTITY = {
  name: 'Your App Name',
  uri: 'https://yourapp.com',
  icon: 'favicon.ico',
};

export const SOLANA_CLUSTER = (
  process.env.EXPO_PUBLIC_SOLANA_CLUSTER || 'devnet'
) as 'devnet' | 'testnet' | 'mainnet-beta';

export const SOLANA_CHAIN: Chain = `solana:${SOLANA_CLUSTER}`;

export const SOLANA_RPC_ENDPOINT =
  process.env.EXPO_PUBLIC_SOLANA_RPC_ENDPOINT ||
  'https://api.devnet.solana.com';
```

### Step 5: Configure Root Provider

**For Expo Router** - Update `app/_layout.tsx`:

```typescript
import 'react-native-get-random-values'; // IMPORTANT: FIRST

import { Stack } from 'expo-router';
import { MobileWalletProvider } from '@wallet-ui/react-native-web3js';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { APP_IDENTITY, SOLANA_CHAIN, SOLANA_RPC_ENDPOINT } from '@/constants/wallet';

const queryClient = new QueryClient();

export default function RootLayout() {
  return (
    <QueryClientProvider client={queryClient}>
      <MobileWalletProvider
        chain={SOLANA_CHAIN}
        endpoint={SOLANA_RPC_ENDPOINT}
        identity={APP_IDENTITY}
      >
        <Stack>
          {/* Existing screens */}
        </Stack>
      </MobileWalletProvider>
    </QueryClientProvider>
  );
}
```

**For React Navigation** - Update the main App component:

```typescript
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { MobileWalletProvider } from '@wallet-ui/react-native-web3js';
import { APP_IDENTITY, SOLANA_CHAIN, SOLANA_RPC_ENDPOINT } from './config/wallet';

const queryClient = new QueryClient();

export function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <MobileWalletProvider
        chain={SOLANA_CHAIN}
        endpoint={SOLANA_RPC_ENDPOINT}
        identity={APP_IDENTITY}
      >
        {/* Existing providers and navigation */}
      </MobileWalletProvider>
    </QueryClientProvider>
  );
}
```

**See:** [references/setup-and-configuration.md](references/setup-and-configuration.md) for complete setup details

### Step 6A: Add Connect Button (New Button)

If no connect button exists:

```typescript
import { View, Pressable, Text, StyleSheet } from 'react-native';
import { useMobileWallet } from '@wallet-ui/react-native-web3js';

export default function LoginScreen() {
  const { account, connect, disconnect } = useMobileWallet();
  const connected = !!account;

  const handleConnect = async () => {
    try {
      const connectedAccount = await connect();
      console.log('Connected:', connectedAccount.address);
      // Navigate or update auth state...
    } catch (error) {
      console.error('Connection failed:', error);
    }
  };

  return (
    <View style={styles.container}>
      {!connected ? (
        <Pressable style={styles.button} onPress={handleConnect}>
          <Text style={styles.buttonText}>Connect Wallet</Text>
        </Pressable>
      ) : (
        <Pressable style={styles.button} onPress={disconnect}>
          <Text style={styles.buttonText}>Disconnect</Text>
        </Pressable>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  button: { backgroundColor: '#9945FF', padding: 15, borderRadius: 8 },
  buttonText: { color: 'white', fontSize: 16, fontWeight: '600' },
});
```

### Step 6B: Add to Existing Button

If connect button already exists, integrate MWA:

```typescript
import { useMobileWallet } from '@wallet-ui/react-native-web3js';

export function ExistingAuthButton() {
  const { connect } = useMobileWallet();

  const handleConnect = async () => {
    try {
      const account = await connect();
      // account.address - Base58 encoded public key string
      // account.publicKey - PublicKey object from @solana/web3.js
      // Existing auth logic...
    } catch (error) {
      console.error('MWA connection failed:', error);
    }
  };

  return <Button onPress={handleConnect}>Connect Wallet</Button>;
}
```

**See:** [references/wallet-connection.md](references/wallet-connection.md) for complete implementation

### Step 7: Add Transaction Example (Optional)

Add a simple SOL transfer button to demonstrate transaction signing:

```typescript
// app/send.tsx or add to existing screen
import { useState } from 'react';
import { View, TextInput, Pressable, Text, Alert } from 'react-native';
import { useMobileWallet } from '@wallet-ui/react-native-web3js';
import {
  PublicKey,
  Transaction,
  SystemProgram,
  LAMPORTS_PER_SOL
} from '@solana/web3.js';

export default function SendScreen() {
  // connection is provided by the hook - no need to create manually
  const { account, signAndSendTransaction, connection } = useMobileWallet();
  const [recipient, setRecipient] = useState('');
  const [amount, setAmount] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSend = async () => {
    if (!account) return;

    try {
      setLoading(true);

      // Get fresh blockhash
      const { blockhash, lastValidBlockHeight } =
        await connection.getLatestBlockhash();
      
      // Build transaction
      const transaction = new Transaction({
        feePayer: account.publicKey,
        blockhash,
        lastValidBlockHeight,
      }).add(
        SystemProgram.transfer({
          fromPubkey: account.publicKey,
          toPubkey: new PublicKey(recipient),
          lamports: Math.floor(parseFloat(amount) * LAMPORTS_PER_SOL),
        })
      );

      // Sign and send (SDK handles everything)
      const signature = await signAndSendTransaction(transaction);
      
      // Confirm
      await connection.confirmTransaction({
        signature,
        blockhash,
        lastValidBlockHeight,
      });

      Alert.alert('Success!', `Sent ${amount} SOL`);
    } catch (error: any) {
      Alert.alert('Error', error.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <View style={{ padding: 20 }}>
      <TextInput
        placeholder="Recipient Address"
        value={recipient}
        onChangeText={setRecipient}
        style={{ borderWidth: 1, padding: 10, marginBottom: 10 }}
      />
      <TextInput
        placeholder="Amount (SOL)"
        value={amount}
        onChangeText={setAmount}
        keyboardType="decimal-pad"
        style={{ borderWidth: 1, padding: 10, marginBottom: 10 }}
      />
      <Pressable
        onPress={handleSend}
        disabled={loading}
        style={{
          backgroundColor: '#9945FF',
          padding: 15,
          borderRadius: 8,
          opacity: loading ? 0.6 : 1,
        }}
      >
        <Text style={{ color: 'white', textAlign: 'center', fontWeight: 'bold' }}>
          {loading ? 'Sending...' : 'Send SOL'}
        </Text>
      </Pressable>
    </View>
  );
}
```

**See:** [references/transaction-signing.md](references/transaction-signing.md) for complete transaction implementation

### Step 8: Testing

After implementation:

1. **Build and run**: `npx expo run:android`
2. **Install a wallet**: Phantom, Solflare, or another MWA-compatible wallet on the device/emulator
3. **Test connection**: Tap connect button, approve in wallet
4. **Test transaction** (if implemented): Send small amount, confirm in explorer
5. **Test reconnection**: Close and reopen app, should auto-reconnect

## Troubleshooting

### Wallet Address Shows Garbled/Encoded Text

If connected address shows something like "+9pgyt LK...MIiSdpI=" instead of a proper Solana address:

**Cause**: Using `account.address` instead of `account.publicKey.toString()`.

**Fix**: Always use `publicKey.toString()` to display the wallet address:

```typescript
// BAD: May not return proper base58 address
<Text>{account.address}</Text>

// GOOD: Correct - returns base58 address like "7xKXtg2CW87d..."
<Text>{account.publicKey.toString()}</Text>
```

### "Buffer doesn't exist" Error

If you see this error when working with signatures or binary data:
```
ReferenceError: Property 'Buffer' doesn't exist
```

**Cause**: `Buffer` is a Node.js global that doesn't exist in React Native.

**Fix**: Use `btoa` with `String.fromCharCode` for base64 encoding:

```typescript
// BAD: Won't work - Buffer doesn't exist in React Native
const sigBase64 = Buffer.from(signature).toString("base64");

// GOOD: Works in React Native
const sigBase64 = btoa(String.fromCharCode(...signature));
```

For hex encoding:
```typescript
// GOOD: Works in React Native
const sigHex = Array.from(signature)
  .map(b => b.toString(16).padStart(2, '0'))
  .join('');
```

### Transaction Fails When Sending to Random/New Public Keys

If you see errors like `payloads invalid for signing` or `-32002` when sending to a newly generated `Keypair`:

**Cause**: Some wallets require the recipient account to already exist on-chain.

**Fix**: Send to an existing account, not a randomly generated keypair:

```typescript
// May fail - recipient doesn't exist on-chain
const randomRecipient = Keypair.generate().publicKey;

// Works - use an existing account address
const recipient = new PublicKey("ExistingAddress...");
```

**Note**: On Solana, sending SOL to a new address should create the account, but MWA wallets may reject transactions to non-existent accounts. Always test with known existing addresses first.

### "Secure context (https)" Error

If you see this error:
```
SolanaMobileWalletAdapterError: The mobile wallet adapter protocol must be used in a secure context (`https`).
```

**Cause**: The `@solana-mobile/mobile-wallet-adapter-protocol` package is resolving to its ESM web implementation instead of the native CommonJS implementation.

**Fix**: Remove the ESM folder to force native resolution:
```bash
rm -rf node_modules/@solana-mobile/mobile-wallet-adapter-protocol/lib/esm
```

Then rebuild: `npx expo run:android`

**Note**: This is a known issue with Expo SDK 53+ and module resolution. See [GitHub Issue #1179](https://github.com/solana-mobile/mobile-wallet-adapter/issues/1179) for updates.

## Key Implementation Details

### Using Beeman's SDK

This skill uses `@wallet-ui/react-native-web3js` because:
- **Production-ready**: Maintained, tested, used in production apps
- **Simplified API**: Cleaner than raw MWA protocol
- **Auto-persistence**: Handles auth token storage automatically
- **TypeScript**: Full type safety out of the box

### Wallet Connection Flow

1. User taps connect button
2. SDK dispatches intent to wallet apps
3. Android shows wallet picker (if multiple wallets)
4. User approves in wallet app
5. SDK returns account with `address` (string) and `publicKey` (PublicKey object)
6. Auth token automatically stored for next session

### Transaction Flow

1. Use `connection` from the `useMobileWallet()` hook (not `new Connection()`)
2. Build transaction with `Transaction` and instructions
3. Get fresh blockhash (expires in ~150 blocks)
4. Call `signAndSendTransaction(transaction)`
5. Wallet signs and broadcasts to network
6. Confirm with `confirmTransaction()`

### Session Persistence

The SDK automatically handles:
- Storing auth token after first connection
- Checking for token on app startup
- Auto-reconnecting if token is valid
- Clearing token on disconnect

No manual AsyncStorage code needed.

## Resources

### Setup and Configuration
Complete setup guide including:
- All required dependencies
- Crypto polyfill setup (critical!)
- Environment variables
- Provider configuration
- Development build requirements

**File:** [references/setup-and-configuration.md](references/setup-and-configuration.md)

### Wallet Connection
Implementation guide for:
- Basic connect button
- Adding to existing buttons
- Using wallet state in components
- Protected routes
- Error handling
- Session persistence

**File:** [references/wallet-connection.md](references/wallet-connection.md)

### Transaction Signing
Complete transaction implementation:
- SOL transfers
- Transaction building
- Balance checking
- Error handling
- Status UI
- Explorer links

**File:** [references/transaction-signing.md](references/transaction-signing.md)

## External Documentation

- **Wallet UI Docs (Providers)**: https://wallet-ui.dev/docs/react-native/providers
- **Wallet UI Docs (Hooks)**: https://wallet-ui.dev/docs/react-native/hooks/use-mobile-wallet
- **Solana Mobile Docs**: https://docs.solanamobile.com/react-native/using_mobile_wallet_adapter
- **MWA Spec**: https://solana-mobile.github.io/mobile-wallet-adapter/spec/spec.html
- **Solana Web3.js**: https://solana-labs.github.io/solana-web3.js/
