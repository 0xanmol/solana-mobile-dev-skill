# Solana Mobile Dev Skills

Claude Code skills for building Solana Mobile apps.

## Available Skills

### Mobile Wallet Adapter (`mwa/`)

Integrates Mobile Wallet Adapter (MWA) for wallet connection and transaction signing in React Native Expo apps using [@wallet-ui/react-native-web3js](https://wallet-ui.dev/).

All MWA skills are located in the `mwa/` directory:

| Folder | Skill | Description |
|--------|-------|-------------|
| `mwa/mobile-wallet-adapter-react-native` | Router | Assesses needs and delegates to sub-skills |
| `mwa/mwa-setup` | Setup | Install dependencies, polyfills, and provider configuration |
| `mwa/mwa-connection` | Connection | Add connect/disconnect wallet functionality |
| `mwa/mwa-transactions` | Transactions | Add SOL transfers and transaction signing |

**Trigger phrases:**
- "Add wallet connection to my React Native app"
- "Integrate Mobile Wallet Adapter"
- "Add a connect wallet button"
- "Send SOL transactions from my app"

## Installation

Copy the skill folders to your Claude Code skills directory:

```bash
cp -r mwa/* ~/.claude/skills/
```

## Requirements

- React Native Expo project
- Development build (not Expo Go - MWA uses native modules)
- Android development environment

## Contributing

Improving skills is an iterative process—prompts can always be refined and enhanced. This repo is open to any issues raised or improvements suggested. Feel free to open an issue or submit a pull request if you have ideas for better prompts, additional features, or bug fixes.
