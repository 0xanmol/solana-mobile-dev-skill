# Wallet Connection Implementation

Complete implementation for wallet connection using Beeman's Wallet UI SDK.

## Basic Connect Button

### Simple Implementation

```typescript
// app/login.tsx
import { View, Text, Pressable, StyleSheet } from 'react-native';
import { useMobileWalletAdapter } from '@wallet-ui/react-native-web3js';
import { router } from 'expo-router';

export default function LoginScreen() {
  const { account, connect, disconnect } = useMobileWalletAdapter();
  const connected = !!account;

  const handleConnect = async () => {
    try {
      const connectedAccount = await connect();
      console.log('Connected:', connectedAccount.publicKey.toString());
      
      // Navigate to main app after connection
      router.replace('/(tabs)');
    } catch (error) {
      console.error('Connection failed:', error);
      // Handle error - show alert to user
    }
  };

  const handleDisconnect = async () => {
    try {
      await disconnect();
      console.log('Disconnected');
    } catch (error) {
      console.error('Disconnect failed:', error);
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Welcome to Your App</Text>
      
      {!connected ? (
        <Pressable style={styles.button} onPress={handleConnect}>
          <Text style={styles.buttonText}>Connect Wallet</Text>
        </Pressable>
      ) : (
        <View>
          <Text>Connected: {account.publicKey.toString().slice(0, 8)}...</Text>
          <Pressable style={styles.button} onPress={handleDisconnect}>
            <Text style={styles.buttonText}>Disconnect</Text>
          </Pressable>
        </View>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 20,
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    marginBottom: 40,
  },
  button: {
    backgroundColor: '#9945FF',
    paddingHorizontal: 24,
    paddingVertical: 12,
    borderRadius: 8,
  },
  buttonText: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
});
```

### Adding to Existing Button

If the app already has a connect button, add MWA connection to it:

```typescript
// Existing button component
import { useMobileWalletAdapter } from '@wallet-ui/react-native-web3js';

export function ExistingConnectButton() {
  const { connect } = useMobileWalletAdapter();

  const handleConnect = async () => {
    try {
      const account = await connect();
      // Your existing connection logic...
    } catch (error) {
      console.error('MWA connection failed:', error);
    }
  };

  return (
    <Button onPress={handleConnect}>
      Connect with Mobile Wallet
    </Button>
  );
}
```

## Using Wallet State in Components

### Access Connected Account

```typescript
import { useMobileWalletAdapter } from '@wallet-ui/react-native-web3js';

export default function ProfileScreen() {
  const { account } = useMobileWalletAdapter();
  const connected = !!account;

  if (!connected) {
    return <Text>Please connect your wallet</Text>;
  }

  return (
    <View>
      <Text>Wallet Address:</Text>
      <Text>{account.publicKey.toString()}</Text>
    </View>
  );
}
```

### Protected Routes

```typescript
// app/(tabs)/_layout.tsx
import { useMobileWalletAdapter } from '@wallet-ui/react-native-web3js';
import { Redirect } from 'expo-router';

export default function TabsLayout() {
  const { account } = useMobileWalletAdapter();

  // Redirect to login if not connected
  if (!account) {
    return <Redirect href="/login" />;
  }

  return (
    <Tabs>
      {/* Your tab screens */}
    </Tabs>
  );
}
```

## Wallet Connection Flow

### 1. User Taps Connect Button
- App calls `connect()` from `useMobileWalletAdapter()`
- SDK dispatches intent to wallet apps on device

### 2. User Selects Wallet
- Android shows wallet picker if multiple wallets installed
- User selects their preferred wallet app

### 3. Wallet Shows Approval Dialog
- Wallet displays your app's identity (name, icon, URI)
- User approves or rejects connection
- Wallet may show terms/conditions

### 4. Connection Complete
- If approved: SDK returns `account` object with `publicKey`
- If rejected: Promise rejects with error
- SDK automatically stores auth token for future sessions

## SDK Hook API

### useMobileWalletAdapter()

Returns:

```typescript
{
  account: {
    publicKey: PublicKey;
    address: string;
  } | null;
  
  connect: () => Promise<{
    publicKey: PublicKey;
    address: string;
  }>;
  
  disconnect: () => Promise<void>;
  
  signAndSendTransaction: (transaction: Transaction) => Promise<string>;
  
  signTransaction: (transaction: Transaction) => Promise<Transaction>;
  
  signMessage: (message: Uint8Array) => Promise<Uint8Array>;
}
```

### Working with signMessage

The `signMessage` function returns a `Uint8Array`. To convert to base64 for display or storage:

```typescript
const handleSignMessage = async () => {
  const message = new TextEncoder().encode("Hello from MyApp!");
  const signature = await signMessage(message);

  // ⚠️ Buffer doesn't exist in React Native - use btoa instead
  const signatureBase64 = btoa(String.fromCharCode(...signature));

  console.log('Signature:', signatureBase64);
};
```
```

### Key Properties

**account**:
- `null` when not connected
- Object with `publicKey` and `address` when connected
- `publicKey` is a `PublicKey` instance from `@solana/web3.js`
- **Always use `account.publicKey.toString()`** to get the base58 wallet address for display
- Note: `account.address` may not return the expected base58 format - prefer `publicKey.toString()`

**connect()**:
- Initiates wallet connection flow
- Returns account info on success
- Throws error on rejection or failure
- Automatically stores auth token

**disconnect()**:
- Disconnects current session
- Clears stored auth token
- Sets `account` to `null`

## Error Handling

### Common Errors

```typescript
const handleConnect = async () => {
  try {
    await connect();
  } catch (error: any) {
    if (error.message?.includes('User declined')) {
      // User rejected connection
      Alert.alert('Connection Cancelled', 'You declined the wallet connection');
    } else if (error.message?.includes('No wallet found')) {
      // No wallet app installed
      Alert.alert(
        'No Wallet Found',
        'Please install a Solana wallet app like Phantom or Solflare'
      );
    } else {
      // Other errors
      Alert.alert('Connection Failed', 'Please try again');
      console.error('Connection error:', error);
    }
  }
};
```

### No Wallet Installed

```typescript
import { Linking } from 'react-native';

const handleNoWalletFound = () => {
  Alert.alert(
    'Install a Wallet',
    'You need a Solana wallet to use this app',
    [
      {
        text: 'Get Phantom',
        onPress: () => {
          Linking.openURL('https://phantom.app/download');
        },
      },
      { text: 'Cancel', style: 'cancel' },
    ]
  );
};
```

## Session Persistence

### Automatic Reconnection

The SDK automatically handles session persistence:

```typescript
// On app startup, SDK checks for stored auth token
// If found and valid, automatically reconnects

export default function App() {
  const { account } = useMobileWalletAdapter();
  
  useEffect(() => {
    if (account) {
      console.log('Auto-reconnected to:', account.publicKey.toString());
    }
  }, [account]);
  
  // ...
}
```

### Manual Check on Mount

```typescript
import { useEffect } from 'react';
import { useMobileWalletAdapter } from '@wallet-ui/react-native-web3js';

export default function HomeScreen() {
  const { account } = useMobileWalletAdapter();

  useEffect(() => {
    // Check connection state on mount
    if (account) {
      console.log('Wallet connected:', account.publicKey.toString());
      // Fetch user data, balances, etc.
    } else {
      console.log('No wallet connected');
      // Show connect prompt
    }
  }, []);

  // ...
}
```

## Integration with Backend

### Send Public Key to Backend

```typescript
import { useMobileWalletAdapter } from '@wallet-ui/react-native-web3js';

export default function LoginScreen() {
  const { connect } = useMobileWalletAdapter();

  const handleLogin = async () => {
    try {
      // Connect wallet
      const account = await connect();
      const walletAddress = account.publicKey.toString();
      
      // Send to backend
      const response = await fetch('https://api.yourapp.com/auth/connect', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ walletAddress }),
      });
      
      const { token, user } = await response.json();
      
      // Store auth token, navigate to app, etc.
      await AsyncStorage.setItem('authToken', token);
      router.replace('/(tabs)');
    } catch (error) {
      console.error('Login failed:', error);
    }
  };

  return (
    <Pressable onPress={handleLogin}>
      <Text>Login with Wallet</Text>
    </Pressable>
  );
}
```

## Best Practices

### 1. Show Connection State
Always show users whether they're connected:

```typescript
const { account } = useMobileWalletAdapter();

return (
  <View>
    {account ? (
      <Text>🟢 Connected: {ellipsify(account.publicKey.toString())}</Text>
    ) : (
      <Text>🔴 Not Connected</Text>
    )}
  </View>
);
```

### 2. Handle Connection Errors Gracefully
Don't let connection failures crash the app - show helpful messages.

### 3. Provide Disconnect Option
Always give users a way to disconnect:

```typescript
<Button onPress={disconnect}>Disconnect Wallet</Button>
```

### 4. Check Connection Before Actions
```typescript
const handleAction = async () => {
  if (!account) {
    Alert.alert('Please connect your wallet first');
    return;
  }
  
  // Proceed with action...
};
```

### 5. Use TypeScript
The SDK is fully typed - leverage TypeScript for better DX:

```typescript
import type { PublicKey } from '@solana/web3.js';

const publicKey: PublicKey | null = account?.publicKey || null;
```
