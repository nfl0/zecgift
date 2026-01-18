# ZecGift

**Send Zcash as a gift card. Private, simple, no signup required.**

ZecGift is a fully client-side web application that lets you create and send Zcash gift cards to anyone. All cryptographic operations happen in the browser using WebAssembly - no server ever sees your keys.

## How It Works

![ZecGift Architecture](static/architecture.png)

### Gift Creation (Sender)
1. Visit ZecGift - a new ephemeral wallet is generated in your browser
2. A 24-word BIP39 seed phrase and Zcash Unified Address are created client-side
3. Fund the address with any amount of ZEC (minimum 0.001 ZEC recommended)
4. Once funded, copy the gift link and share it with your recipient

### The Gift Link
- Contains a base64-encoded payload with the seed phrase and wallet birthday height
- Format: `https://zecgift.app?gift=base64({seed, birthday})`
- The seed is encoded in the URL and **never stored on any server**
- Share via any channel: text message, email, chat, QR code, etc.

### Claim Flow (Recipient)
1. Open the gift link in a browser
2. The ephemeral wallet is restored from the seed in the URL
3. Enter your Zcash wallet address
4. Click "Claim Gift" - funds are sent to your address minus network fees
5. View the transaction on Zcash Explorer

## Features

- **100% Client-Side**: All cryptography runs in WebAssembly in your browser
- **Fully Private**: Uses Zcash Orchard shielded pool for all transactions
- **No Signup**: No accounts, no registration, no KYC
- **No Custody**: We never hold or have access to any funds
- **Open Source**: Fully auditable code

## Technical Details

| Component | Technology |
|-----------|------------|
| Frontend | SvelteKit 5 |
| Wallet | WebZjs (WASM) |
| Cryptography | librustzcash (Rust → WASM) |
| Network | Zcash Mainnet |
| Protocol | Orchard shielded transactions |
| Fees | ZIP-317 (0.0003 ZEC) |
| Sync | lightwalletd via gRPC-web |

### Key Derivation
- BIP39 seed phrase (24 words)
- ZIP-32 HD key derivation
- Orchard spending and viewing keys
- Unified Address generation (u1...)

### Transaction Flow
1. Wallet syncs from birthday height via lightwalletd
2. Detects shielded notes in Orchard pool
3. Creates shielded transaction to recipient address
4. Signs transaction with spending key derived from seed
5. Broadcasts via lightwalletd to Zcash network

## Development

### Prerequisites
- [Bun](https://bun.sh/) (recommended) or Node.js 18+

### Setup
```bash
# Clone the repository
git clone https://github.com/bajpai244/zecgift.git
cd zecgift

# Install dependencies
bun install

# Start development server
bun run dev
```

### Environment Variables
Create a `.env` file:
```
PUBLIC_ZCASH_NETWORK=main
PUBLIC_LIGHTWALLETD_URL=https://zcash-mainnet.chainsafe.dev
```

### Build
```bash
bun run build
```

### Deploy
The app is configured for static deployment (Vercel, Netlify, etc.). The build output is in the `build/` directory.

## Security Considerations

- **Gift links contain the seed phrase** - treat them like cash. Anyone with the link can claim the funds.
- **Share links securely** - use encrypted messaging when possible.
- **Claim promptly** - funds in unclaimed gift cards are at risk if the link is compromised.
- **Browser security** - ensure you're on the official ZecGift domain and your browser is secure.

## Fees

- **Network Fee**: 0.0003 ZEC (ZIP-317 standard fee for shielded transactions)
- **Service Fee**: None - ZecGift is free to use

## Timing

- **Deposit Confirmation**: 75-90 seconds (1 Zcash block)
- **Claim Transaction**: Near-instant broadcast, 75-90 seconds for confirmation

## Limitations

- Minimum gift amount: Must be greater than 0.0003 ZEC (to cover fees)
- Recommended minimum: 0.001 ZEC
- Only supports Zcash mainnet
- Requires a modern browser with WebAssembly and SharedArrayBuffer support

## License

MIT

## Acknowledgments

- [ChainSafe WebZjs](https://github.com/chainsafe/webzjs) - WASM Zcash wallet library
- [Zcash](https://z.cash/) - Privacy-preserving cryptocurrency
- [Electric Coin Company](https://electriccoin.co/) - librustzcash
