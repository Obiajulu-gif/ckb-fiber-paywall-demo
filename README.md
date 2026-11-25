# Pay to Decrypt — Encrypted Data Payments over Fiber

Browser-only demo that puts a data report behind a Lightning-style payment. A **Seller** encrypts a report with AES-256-GCM and issues a 1 CKB Fiber invoice whose **payment preimage is the AES key**. A **Buyer** sees a free preview, pays 1 CKB over a Fiber payment channel, and the payment itself reveals the key that decrypts the full report. No escrow, no server, no trusted third party.

Built on the official [Fiber encrypted-data-payment starter](https://www.fiber.world/docs/build/encrypted-data-payment) for **CKBoost Campaign #8 — "Unlock Encrypted Data with a Fiber Payment" (Advanced track)**.

![Next.js](https://img.shields.io/badge/Next.js-16.2.6-000000)
![React](https://img.shields.io/badge/React-19-61DAFB)
![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178C6)
![Network](https://img.shields.io/badge/CKB-Testnet-00CC9B)
![Crypto](https://img.shields.io/badge/AES--256--GCM-WebCrypto-6E57E0)

> **Demo video:** _unlisted link pending publish_ — replace this line with the YouTube URL.

---

## How it works

Both the Buyer and the Seller are Fiber WASM nodes that run **entirely in the browser**. Payment is routed over a public intermediary node (Bottle), so neither side needs a direct channel to the other.

```
Seller                                             Buyer
  │  encrypt report  ──►  AES-256-GCM (key = K)
  │  invoice: 1 CKB, payment_preimage = K, hash = SHA-256(K)
  │
  │ ───────────── public offer (title, summary, ciphertext, iv, invoice) ─────────────►
  │                                              validate offer  →  free preview only
  │                                              decrypt without key  →  rejected
  │                                              pay 1 CKB ─► Fiber channel ─► Bottle ─► Seller
  │                                              invoice settles → preimage K revealed
  │                                              decrypt(ciphertext, K)  →  full report
```

The security property is that the ciphertext and the invoice travel together in public, but the AES key never does — it only becomes knowable to the Buyer as a side effect of paying the invoice.

## What this build adds — research brief + free preview

The upstream starter encrypts an opaque blob. This variant turns the offer into a **paywalled research brief**:

- The offer now carries a public **`title`** and **`summary`** alongside the encrypted body.
- The Buyer reads the title and free summary **before** paying, so the purchase is informed rather than blind.
- Paying the 1 CKB invoice reveals the preimage (the AES key) and unlocks the full report.

Fiber routing, AES-256-GCM, and the invoice ⇄ preimage link are unchanged from the starter.

**Changed files**

| File | Change |
| --- | --- |
| `lib/offer.ts` | `EncryptedOffer` gains `title` / `summary`; `createOffer` and `parseOffer` validate and bound them. |
| `app/page.tsx` | Seller inputs a title, a free summary, and the full report; Buyer renders the free preview after validation and the full body only after pay + decrypt. |

## Tech stack

- **Next.js 16.2.6** (Turbopack) · **React 19** · **TypeScript** (strict)
- **[`@fiber-pay/sdk`](https://www.fiber.world)** — browser Fiber nodes, channels, invoices
- **`@nervosnetwork/fiber-js`** — Fiber primitives
- **Web Crypto** — AES-256-GCM encrypt/decrypt (`lib/crypto.ts`)
- **CKB Testnet** — channel funding and settlement

## Prerequisites

- Node.js **22.18+** and npm
- A Chromium or Firefox browser that supports `SharedArrayBuffer`, Web Locks, and cross-origin isolation
- Testnet CKB from the [Nervos Pudge faucet](https://faucet.nervos.org)

## Run locally

```sh
npm install
npm run dev
```

Open **http://localhost:3001**. Serve over `localhost` or HTTPS only: the WASM runtime requires a secure context, `SharedArrayBuffer`, Web Locks, and cross-origin isolation. `next.config.mjs` supplies the isolation headers (COOP/COEP). A browser extension or proxy that strips those headers will prevent the nodes from starting.

## Walkthrough

1. **Start both nodes.** Start the Buyer and the Seller node; each prints a funding address.
2. **Fund.** Copy each address to the faucet and request enough Testnet CKB for 500 CKB of channel funding plus the on-chain fee. Channel V2 reserves 100 CKB, leaving 400 CKB — the minimum for Bottle's auto-accept.
3. **Open channels.** Open both channels and refresh status until each reports **`CHANNEL_READY`** (confirmation can take several minutes). Channel balances also reveal whether Bottle already holds inbound liquidity; the optional **5 CKB transfer** is only needed if it does not.
4. **Create an offer.** As the Seller, enter a title, a free summary, and the full report, then **Encrypt & create a 1 CKB invoice**. Copy the public offer JSON.
5. **Validate (Buyer).** Paste the offer and **Validate**. The free preview (title + summary) appears; the full body stays locked. Decrypting without the key is rejected.
6. **Pay & unlock.** Pay the 1 CKB invoice. On settlement the preimage is revealed, the report decrypts, and the receipt shows the actual routing fee (configured maximum: 1 CKB).

## Resume & troubleshooting

- **After a refresh**, restart the same saved nodes, revalidate the restored offer, and choose **Check payment status**. A completed or pending attempt is never resubmitted automatically; an explicit retry is offered only after the node confirms failure.
- **Expired invoice** — blocked before a new attempt is recorded. If the node rejects submission and confirms no payment exists, the rejected attempt is cleared; choose **Start another offer** for a fresh invoice. Timeouts, worker failures, and unavailable status reads keep the attempt locked for status checking.
- **One tab per saved node.** Browser storage holds node identities and public offer/payment progress; clearing it can lose access to funded channels. AES keys and decrypted output are never persisted — a settled key is recovered via `getPayment`.
- **Interrupted 5 CKB transfer** — the button blocks a second transfer until resolved. Review the Seller's history (`listPayments`) before deliberately clearing its `fiber-docs:encrypted-inbound:<seller-id>` session entry.
- **Slow channel** — check status or reconnect; the existing channel is reused. Do not clear storage or fund a replacement to resolve a wait.
- **Faucet/confirmation latency** — check the faucet transaction and refresh channel status once funds arrive.

## Project structure

| Module | Responsibility |
| --- | --- |
| `lib/offer.ts` | Builds and validates the public offer payload (title, summary, ciphertext, invoice, routing hint). |
| `lib/crypto.ts` | AES-256-GCM encrypt/decrypt via Web Crypto. |
| `lib/payment.ts` | Validates the decoded invoice (network, amount, payment hash) and records an attempt before submission. |
| `lib/progress.ts` | Restores public data only for matching saved node identities. |
| `lib/node-session.ts` | Prevents two runtimes from opening the same saved node. |
| `app/page.tsx` | Seller and Buyer UI. |

## Checks

```sh
npm run typecheck
npm test
npm run build
```

The app catches action errors and surfaces them inline. When a payment outcome is unknown, it checks the existing attempt rather than starting a replacement.

## Security & scope

- **Testnet only.** No real funds are involved; addresses and invoices are CKB Testnet.
- Private keys and seed phrases are never entered, displayed, or submitted.
- AES keys and decrypted plaintext live only in memory and are not written to browser storage.

## Acknowledgements

Based on the official Fiber **encrypted-data-payment** starter and walkthrough: <https://www.fiber.world/docs/build/encrypted-data-payment>. Built for the CKBoost "Build on CKB" developer campaign.
