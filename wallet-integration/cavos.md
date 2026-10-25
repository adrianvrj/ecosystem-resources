# Cavos

Cavos is an embedded, self-custodial wallet SDK for web and React Native. Users sign in with Google, Apple, or an emailed code. There is no seed phrase and no browser extension. On Stellar it creates a classic `G…` account that can make native payments and invoke Soroban contracts. The same SDK also supports Solana and Starknet.

> **Version note:** `@cavos/kit` is pre-1.0 (v0.4.x). Check snippets against the [repo README](https://github.com/cavos-labs/kit) for the version you install.

## Overview

- **Type:** Embedded self-custodial wallet SDK
- **User Experience:** OAuth or email login, silent signing within app limits
- **Best for:** Consumer apps that want in-app wallets without extensions or seed phrases
- **Package:** [`@cavos/kit`](https://www.npmjs.com/package/@cavos/kit)
- **GitHub:** [cavos-labs/kit](https://github.com/cavos-labs/kit)
- **Docs:** [docs.cavos.xyz](https://docs.cavos.xyz)

## Key Model

- **Spend key:** The Stellar Ed25519 key is derived (HKDF) from one MasterDEK. The `G…` address is that key's public key.
- **Restore on a new device:** The MasterDEK is restored by a passkey the user added, or by an attested AWS Nitro enclave (Google, Apple, or email). The enclave restores the DEK. It does not sign.
- **Browser:** Keys live in the Cavos vault, an iframe on `vault.cavos.xyz`, not in the app's page. The vault signs silently within limits set in the Cavos dashboard and asks the user beyond them.
- **React Native:** Keys stay in Secure Enclave or Android Keystore storage.
- **Registry:** Cavos maps `(userId, appId, chain, network)` to an address. It holds that map, not the key.

## Key Features

- **Login without a seed phrase:** Google, Apple, or email code
- **Classic Stellar account:** A `G…` address that works with existing Stellar tools
- **Lazy deploy:** `connect` never creates the account. The first `execute()` creates it.
- **Soroban calls:** `invokeContract` with auth-entry signing
- **External XDR:** `signXdr` for APIs that return unsigned transactions
- **Optional sponsorship:** The Cavos relayer can sponsor reserves and fee-bump transactions. It never holds the key.
- **Multi-chain sessions:** One login can connect Stellar, Solana, and Starknet

## Installation

```bash
npm install @cavos/kit
```

Create an app in the [Cavos dashboard](https://cavos.xyz) to get an App ID. Register your web origins and callback URLs there, or the vault will not load.

## Connect

```typescript
import { Cavos } from "@cavos/kit";

const wallet = await Cavos.connect({
  chain: "stellar",
  network: "testnet",              // "testnet" | "mainnet"
  appSalt: "my-app",               // part of key derivation; never change it
  identity: { userId: user.id },
  appId: process.env.NEXT_PUBLIC_CAVOS_APP_ID, // enables vault and relayer
});

console.log(wallet.address); // G…
console.log(wallet.status);  // "undeployed" | "ready" | "needs-device-approval"
```

`needs-device-approval` means this device does not hold the spend key yet. Restore it with the enclave or a passkey.

## Sign and Send Transactions

Native XLM payment, in stroops. The first call also creates the account:

```typescript
if (wallet.chain === "stellar") {
  await wallet.execute(1_000_000n, recipient); // 0.1 XLM
}
```

Soroban contract call:

```typescript
if (wallet.chain === "stellar" && wallet.status === "ready") {
  await wallet.invokeContract({
    contractId: "C...",
    method: "approve",
    args: [approverAddress, amount],
  });
}
```

Sign XDR built elsewhere:

```typescript
const signedXdr = await wallet.signXdr(unsignedXdr);
```

## React Integration

```tsx
import { CavosProvider, useCavos } from "@cavos/kit/react";

<CavosProvider
  config={{
    appId,
    chains: ["stellar"],
    defaultChain: "stellar",
    network: "testnet",
    appSalt: "my-app",
  }}
  modal={{ appName: "My App" }}
>
  <App />
</CavosProvider>
```

```tsx
function App() {
  const { openModal, wallet, walletStatus, enrollPasskeyDefault } = useCavos();
  // openModal() signs the user in; enrollPasskeyDefault() adds a passkey for restore
  return null;
}
```

Signing up never asks for a passkey. Call `enrollPasskeyDefault()` after connect to add one.

To restore with the enclave instead, set `socialRecovery: true` (or `deviceApproval: "enclave"`) in `config`.

## React Native

Supported on iOS 16+ and Android 9+ with a bare React Native app or an Expo development build. Expo Go is not supported.

```bash
npm install @cavos/kit expo-modules-core expo-web-browser expo-linking
```

Add the `@cavos/kit` Expo config plugin with your `rpId` and `scheme`, rebuild, then import `CavosProvider` from `@cavos/kit/react-native` and pass `redirectUri` and `rpId` in `config`. See the [kit README](https://github.com/cavos-labs/kit#react-native--expo) for the full setup.

## Fees and Sponsorship

With `appId` set, Stellar transactions are sponsored by default: the relayer fee-bumps them and can sponsor reserves. To have the account pay its own fee:

```typescript
await wallet.execute(amount, destination, { fee: "self" });
```

## Gas Sponsorship in Any Token

[`@cavos/reserve`](https://github.com/cavos-labs/reserve) is a separate Cavos package. It lets a user pay Stellar fees and account reserves in a token they already hold instead of XLM. Any token the service accepts works; `reserve.tokens()` lists them (USDC is one example). It uses sponsored reserves (CAP-33) and fee-bumps (CAP-15). The fee is paid in the same transaction through a path payment. It works with any classic `G…` account, not only Cavos wallets. It covers classic operations only, not Soroban.

```bash
npm install @cavos/reserve @stellar/stellar-sdk
```

```typescript
import { Reserve } from "@cavos/reserve";

const reserve = await Reserve.connect("testnet"); // or "mainnet"
const { tokens } = await reserve.tokens();      // accepted fee tokens

const { hash } = await reserve.pay(
  {
    source: wallet.address,
    destination,
    amount: "10",
    token: tokens[0],   // e.g. a USDC asset string
    maxSend: "0.05",    // fee ceiling, in the fee token
  },
  (xdr) => wallet.signXdr(xdr),
);
```

`reserve.activate(...)` creates an account that holds no XLM from a claimable balance, paying its reserves from the claimed token. See the [Reserve docs](https://docs.cavos.xyz/docs/reserve).

## Resources

- [GitHub Repository](https://github.com/cavos-labs/kit)
- [Cavos Docs](https://docs.cavos.xyz)
- [Stellar Guide](https://docs.cavos.xyz/docs/stellar)
- [Reserve Docs](https://docs.cavos.xyz/docs/reserve)
