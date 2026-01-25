# Solana Mobile Dev Skills

Claude Code skills for building Solana Mobile apps.

## Available Skills

### Mobile Wallet Adapter

Integrates Mobile Wallet Adapter (MWA) for wallet connection and transaction signing in React Native Expo apps using [@wallet-ui/react-native-web3js](https://wallet-ui.dev/).

**Trigger phrases:**
- "Add wallet connection to my React Native app"
- "Integrate Mobile Wallet Adapter"
- "Add a connect wallet button"
- "Send SOL transactions from my app"

**What it does:**
- Wallet connection with native wallet picker
- Transaction signing and sending
- Session persistence across app restarts
- Complete provider and polyfill setup

## Installation

Copy the skill folder to your Claude Code skills directory:

```bash
cp -r mobile-wallet-adapter-react-native ~/.claude/skills/
```

The skill will be automatically discovered and available as `/mobile-wallet-adapter-react-native`.

## Requirements

- React Native Expo project
- Development build (not Expo Go - MWA uses native modules)
- Android development environment

## Contributing

Improving skills is an iterative process—prompts can always be refined and enhanced. This repo is open to any issues raised or improvements suggested. Feel free to open an issue or submit a pull request if you have ideas for better prompts, additional features, or bug fixes.
