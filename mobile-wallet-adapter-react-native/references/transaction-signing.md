# Transaction Signing Implementation

Complete implementation for signing and sending transactions using Mobile Wallet Adapter.

## Basic SOL Transfer

### Simple Transfer Example

```typescript
// app/send.tsx or components/SendButton.tsx
import { useState } from 'react';
import { View, TextInput, Pressable, Text, Alert } from 'react-native';
import { useMobileWalletAdapter } from '@wallet-ui/react-native-web3js';
import { Connection, PublicKey, Transaction, SystemProgram, LAMPORTS_PER_SOL } from '@solana/web3.js';
import { SOLANA_RPC_ENDPOINT } from '@/constants/wallet';

export default function SendScreen() {
  const { account, signAndSendTransaction } = useMobileWalletAdapter();
  const [recipient, setRecipient] = useState('');
  const [amount, setAmount] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSend = async () => {
    if (!account) {
      Alert.alert('Error', 'Please connect your wallet first');
      return;
    }

    try {
      setLoading(true);

      // Connect to Solana
      const connection = new Connection(SOLANA_RPC_ENDPOINT, 'confirmed');

      // Get fresh blockhash
      const { blockhash, lastValidBlockHeight } = 
        await connection.getLatestBlockhash();

      // Convert SOL to lamports
      const lamports = Math.floor(parseFloat(amount) * LAMPORTS_PER_SOL);

      // Create transaction
      const transaction = new Transaction({
        feePayer: account.publicKey,
        blockhash,
        lastValidBlockHeight,
      }).add(
        SystemProgram.transfer({
          fromPubkey: account.publicKey,
          toPubkey: new PublicKey(recipient),
          lamports,
        })
      );

      // Sign and send (SDK handles everything)
      const signature = await signAndSendTransaction(transaction);
      
      console.log('Transaction signature:', signature);

      // Confirm transaction
      const confirmation = await connection.confirmTransaction({
        signature,
        blockhash,
        lastValidBlockHeight,
      });

      if (confirmation.value.err) {
        throw new Error('Transaction failed');
      }

      Alert.alert('Success', `Sent ${amount} SOL!`);
      
      // Clear form
      setRecipient('');
      setAmount('');
    } catch (error: any) {
      console.error('Transaction error:', error);
      Alert.alert('Error', error.message || 'Transaction failed');
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
        disabled={loading || !recipient || !amount}
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

## Transaction Building Deep Dive

### Why Get Fresh Blockhash?

```typescript
const { blockhash, lastValidBlockHeight } = 
  await connection.getLatestBlockhash();
```

**Purpose**:
- Transactions expire after ~150 blocks (~1-2 minutes)
- Prevents replay attacks
- Ensures transaction is recent and valid
- Required for transaction confirmation

### Why Convert to Lamports?

```typescript
const lamports = Math.floor(amountInSol * LAMPORTS_PER_SOL);
```

**Purpose**:
- 1 SOL = 1,000,000,000 lamports
- Prevents floating-point precision issues
- Native unit for Solana runtime
- Always use integers for on-chain amounts

**Example**:
```typescript
0.001 SOL = 1,000,000 lamports
0.5 SOL = 500,000,000 lamports
```

### Why SystemProgram.transfer()?

**Purpose**:
- Built-in Solana instruction for SOL transfers
- Optimized native instruction
- Lower compute units = lower fees
- Simpler than custom programs
- Battle-tested and secure

### Why Confirm Transaction?

```typescript
await connection.confirmTransaction({
  signature,
  blockhash,
  lastValidBlockHeight,
});
```

**Purpose**:
- Transaction might fail after submission
- Network issues could prevent finalization
- User needs to know actual result
- Prevents showing success for failed transactions

## Advanced: Sign Without Sending

If you want to sign a transaction but send it yourself:

```typescript
import { useMobileWalletAdapter } from '@wallet-ui/react-native-web3js';
import { Connection, sendAndConfirmRawTransaction } from '@solana/web3.js';

const { signTransaction } = useMobileWalletAdapter();
const connection = new Connection(SOLANA_RPC_ENDPOINT);

// Build transaction...
const transaction = new Transaction({...}).add(instruction);

// Sign only (doesn't send)
const signedTransaction = await signTransaction(transaction);

// Send yourself
const serializedTx = signedTransaction.serialize();
const signature = await connection.sendRawTransaction(serializedTx);

// Confirm
await connection.confirmTransaction(signature);
```

**When to use**:
- Custom retry logic
- Batch transaction handling
- Advanced error recovery
- Specific timing requirements

## Checking Balance Before Transfer

Always check the user has enough SOL:

```typescript
import { Connection, LAMPORTS_PER_SOL } from '@solana/web3.js';

const checkBalance = async (publicKey: PublicKey, amountSol: number) => {
  const connection = new Connection(SOLANA_RPC_ENDPOINT);
  
  // Get balance in lamports
  const balance = await connection.getBalance(publicKey);
  
  // Convert to SOL
  const balanceSol = balance / LAMPORTS_PER_SOL;
  
  // Check if sufficient (add buffer for fees ~0.000005 SOL)
  const required = amountSol + 0.00001;
  
  if (balanceSol < required) {
    throw new Error(`Insufficient funds. You have ${balanceSol} SOL`);
  }
  
  return true;
};

// Usage
const handleSend = async () => {
  try {
    await checkBalance(account.publicKey, parseFloat(amount));
    // Proceed with transaction...
  } catch (error) {
    Alert.alert('Error', error.message);
  }
};
```

## Error Handling

### Common Transaction Errors

```typescript
const handleSend = async () => {
  try {
    const signature = await signAndSendTransaction(transaction);
    // ...
  } catch (error: any) {
    // User rejected transaction
    if (error.message?.includes('User declined')) {
      Alert.alert('Cancelled', 'You rejected the transaction');
      return;
    }
    
    // Insufficient funds
    if (error.message?.includes('insufficient funds')) {
      Alert.alert('Insufficient Funds', 'You don\'t have enough SOL');
      return;
    }
    
    // Transaction timeout
    if (error.message?.includes('expired')) {
      Alert.alert('Timeout', 'Transaction expired. Please try again.');
      return;
    }
    
    // Network error
    if (error.message?.includes('network')) {
      Alert.alert('Network Error', 'Please check your connection');
      return;
    }
    
    // Generic error
    Alert.alert('Transaction Failed', error.message || 'Unknown error');
    console.error('Transaction error:', error);
  }
};
```

## Transaction Status UI

Show transaction progress to user:

```typescript
import { useState } from 'react';
import { ActivityIndicator, Text, View } from 'react-native';

type TxStatus = 'idle' | 'building' | 'signing' | 'confirming' | 'success' | 'error';

export default function SendWithStatus() {
  const [status, setStatus] = useState<TxStatus>('idle');
  const [signature, setSignature] = useState<string>('');

  const handleSend = async () => {
    try {
      setStatus('building');
      // Build transaction...
      
      setStatus('signing');
      const sig = await signAndSendTransaction(transaction);
      setSignature(sig);
      
      setStatus('confirming');
      await connection.confirmTransaction({...});
      
      setStatus('success');
    } catch (error) {
      setStatus('error');
      console.error(error);
    }
  };

  return (
    <View>
      <Pressable onPress={handleSend} disabled={status !== 'idle'}>
        <Text>Send Transaction</Text>
      </Pressable>
      
      {status === 'building' && (
        <View>
          <ActivityIndicator />
          <Text>Building transaction...</Text>
        </View>
      )}
      
      {status === 'signing' && (
        <View>
          <ActivityIndicator />
          <Text>Waiting for signature...</Text>
        </View>
      )}
      
      {status === 'confirming' && (
        <View>
          <ActivityIndicator />
          <Text>Confirming on blockchain...</Text>
        </View>
      )}
      
      {status === 'success' && (
        <View>
          <Text>✅ Transaction confirmed!</Text>
          <Text>Signature: {signature.slice(0, 8)}...</Text>
        </View>
      )}
      
      {status === 'error' && (
        <Text>❌ Transaction failed</Text>
      )}
    </View>
  );
}
```

## Viewing Transaction on Explorer

```typescript
const openExplorer = (signature: string, cluster: string = 'devnet') => {
  const url = `https://explorer.solana.com/tx/${signature}?cluster=${cluster}`;
  Linking.openURL(url);
};

// Usage
Alert.alert(
  'Transaction Sent!',
  `Signature: ${signature.slice(0, 8)}...`,
  [
    { text: 'View on Explorer', onPress: () => openExplorer(signature, 'devnet') },
    { text: 'OK', style: 'cancel' },
  ]
);
```

## Integration with Connection Provider

If using a Connection provider:

```typescript
// components/providers/ConnectionProvider.tsx
import { createContext, useContext } from 'react';
import { Connection } from '@solana/web3.js';
import { SOLANA_RPC_ENDPOINT } from '@/constants/wallet';

const ConnectionContext = createContext<Connection | null>(null);

export function ConnectionProvider({ children }: { children: React.ReactNode }) {
  const connection = new Connection(SOLANA_RPC_ENDPOINT, 'confirmed');
  
  return (
    <ConnectionContext.Provider value={connection}>
      {children}
    </ConnectionContext.Provider>
  );
}

export function useConnection() {
  const connection = useContext(ConnectionContext);
  if (!connection) {
    throw new Error('useConnection must be used within ConnectionProvider');
  }
  return connection;
}
```

Then in components:

```typescript
import { useConnection } from '@/components/providers/ConnectionProvider';
import { useMobileWalletAdapter } from '@wallet-ui/react-native-web3js';

export default function SendScreen() {
  const connection = useConnection(); // From provider
  const { account, signAndSendTransaction } = useMobileWalletAdapter();

  const handleSend = async () => {
    const { blockhash, lastValidBlockHeight } = 
      await connection.getLatestBlockhash();
    // ...
  };
}
```

## Best Practices

### 1. Always Get Fresh Blockhash
Don't reuse blockhashes - they expire quickly.

### 2. Confirm Transactions
Never assume success without confirmation.

### 3. Show Clear Status
Users need to know what's happening during signing/confirmation.

### 4. Handle All Errors
Transaction failures are common - handle gracefully.

### 5. Add Buffers for Fees
Always leave extra SOL for transaction fees (~0.000005 SOL).

### 6. Validate Inputs
Check recipient address is valid PublicKey before building transaction:

```typescript
try {
  new PublicKey(recipient);
} catch (error) {
  Alert.alert('Invalid Address', 'Please enter a valid Solana address');
  return;
}
```

### 7. Use Proper Commitment Levels
- `confirmed`: Fast, good for most cases
- `finalized`: Slower, but guaranteed (use for high-value transactions)

```typescript
const connection = new Connection(RPC_ENDPOINT, 'confirmed');
```
