# Gr4vy Seed Studio

A browser-based tool for generating realistic transaction data in Gr4vy sandbox instances. Useful for demoing Insights, testing flow rules, validating reporting before go-live, and reproducing customer issues.

Open `seed-studio.html` directly in any browser — no build step, no dependencies, no server required.

---

## Features

### Batch tab
Generates a standalone Python script that fires hundreds or thousands of transactions across a configured duration.

- **Processors** — configure multiple Gr4vy connections with PSP-specific test cards and weighted traffic split. Includes the Gr4vy Simulator (amount-driven, no PSP credentials needed), Adyen, Checkout.com, Stripe, Braintree, Worldpay, and custom.
- **Buyer pool & vaulting** — creates a pool of buyers at script startup and vaults one card per buyer. Transactions are split between returning customers (vaulted payment method) and new customers (raw card + `store=True`). Fully idempotent — re-running against the same instance returns existing buyers and payment methods rather than creating duplicates.
- **Country & currency distribution** — weighted country mix drives transaction currency, amount range, and a `country` metadata field. Type a known ISO code to auto-fill currency and suggested amount range.
- **Payment method mix** — configurable split between card, Apple Pay, and Google Pay (wallet methods use card PAN as sandbox fallback).
- **Metadata fields** — add any key with weighted values to drive flow rules, Insights filters, or reporting.
- **Schedule** — configure duration, transactions per day, decline rate, random seed, and optional peak-hour time distribution.
- **Production safeguard** — switching to production requires actively acknowledging a warning modal before the script is generated.

### Single tab
Craft one transaction exactly as it happened — useful for recreating customer issues or testing specific scenarios.

- Full card details, amount, currency, intent, country, external identifier
- Pin to a specific payment service ID (connection)
- Buyer fields — buyer ID, external identifier, email
- Complete 3DS data injection — version, status, CAVV, ECI, XID, directory response, authentication response
- Free-form metadata key/value pairs
- Live-updating Python snippet, ready to copy and run

### Guide tab
Reference documentation built into the tool — PSP test card numbers, simulator amount codes, country/currency reference, and setup instructions.

---

## Usage

### 1. Open the tool
```
open seed-studio.html
```
No installation required. Works in Chrome, Firefox, Safari, and Edge.

### 2. Install the Gr4vy Python SDK
```bash
pip install gr4vy
```

### 3. Export your private key
Find it in the merchant dashboard under **Developers → API Keys**.
```bash
export GR4VY_PRIVATE_KEY="$(cat path/to/private_key.pem)"
```

### 4. Configure and download
Set your instance ID, processors, countries, and any other options in the **Batch** tab, then click **download** to get a generated Python script.

### 5. Run
```bash
python seed_merchant.py
```

For multi-day runs, use `nohup` to keep it running after closing the terminal:
```bash
nohup python seed_merchant.py > seed.log 2>&1 &
tail -f seed.log
```

A healthy run looks like:
```
2026-05-11 09:04 INFO  starting: 7d x 150/day = 1050 txns (576.0s interval)
2026-05-11 09:04 INFO  setting up 20 buyers and vaulting cards...
2026-05-11 09:04 INFO    seed-buyer-001 ready — pm a1b2c3d4-...
2026-05-11 09:04 INFO  buyer pool ready: 20 buyers
2026-05-11 09:04 INFO  day 1/7 -- 2026-05-11
2026-05-11 09:04 INFO    OK  FR  | Adyen_Main           | card       | EUR    12.50 | capture_succeeded
2026-05-11 09:14 INFO    OK  BR  | Checkout_Main        | apple-pay  | BRL    47.20 | capture_succeeded
2026-05-11 09:25 INFO    --  DE  | Adyen_Main           | card       | EUR    89.00 | authorization_failed
```

---

## Instance ID vs Merchant Account ID

The **Instance ID** is the top-level Gr4vy instance identifier (e.g. `blablacar`) — used to connect to the right API host.

The **Merchant Account ID** is only needed for instances with multiple submerchants. Leave it blank for single-account instances and the API will use the default merchant account automatically.

---

## Simulator connector

The Gr4vy Simulator is a sandbox-only connector that returns mocked responses based on the transaction amount. No real PSP is involved — no credentials needed, just activate it in the sandbox dashboard under Connections.

Any card number works. The amount determines the outcome:

| Amount | Outcome |
|--------|---------|
| Any other value | `capture_succeeded` |
| `200009` | `incorrect_cvv` |
| `200010` | `incorrect_expiry_date` |
| `200011` | `insufficient_funds` |
| `200012` | `issuer_decline` |
| `200014` | `requires_buyer_authentication` |
| `200015` | `refused_transaction` |
| `200017` | `suspected_fraud` |
| `300005` | `invalid_service_credentials` |
| `400001` | `internal_error` |

See the [full simulator reference](https://docs.gr4vy.com/guides/features/simulators/overview) for all codes including capture, void, and refund scenarios.

---

## PSP test card reference

| PSP | Scheme | Number | Exp | CVV | Result |
|-----|--------|--------|-----|-----|--------|
| Simulator | any | any valid number | any future | any | amount-driven |
| Adyen | Visa | `4111111111111111` | 03/30 | 737 | Success |
| Adyen | Mastercard | `5500005555555559` | 03/30 | 737 | Success |
| Adyen | Visa | `4000000000000002` | 03/30 | 737 | Decline |
| Checkout.com | Visa | `4242424242424242` | 01/29 | 100 | Success |
| Checkout.com | Mastercard | `5436031030606378` | 12/30 | 257 | Success |
| Stripe | Visa | `4242424242424242` | 12/34 | 123 | Success |
| Stripe | Mastercard | `5555555555554444` | 12/34 | 123 | Success |
| Braintree | Visa | `4111111111111111` | 12/30 | 123 | Success |
| Worldpay | Visa | `4444333322221111` | 12/30 | 123 | Success |

---

## Buyers & vaulting

When the buyer pool is enabled, the generated script:

1. Creates N buyers via `client.buyers.create()` using `external_identifier` like `seed-buyer-001`
2. Vaults one card per buyer via `client.payment_methods.create()` using `CardPaymentMethodCreate`
3. On each transaction, picks a buyer from the pool and either uses their vaulted `payment_method_id` (returning customer) or a raw card with `store=True` (new customer, also vaults for future runs)

Both buyer creation and card vaulting are idempotent — if the same `external_identifier` already exists, the script fetches the existing record rather than failing.

---

## Troubleshooting

**All transactions failing**
Most likely an inactive PSP connection or wrong test card for that PSP. Confirm the connection is active in the dashboard and that you've selected the right PSP type.

**Bearer token expiry on long runs**
Tokens expire after 1 hour. For multi-day runs, move the client initialisation inside the day loop — the generated script includes a comment flagging this location.

**Transactions not appearing in Insights**
Insights has a processing delay of a few minutes, sometimes up to 15–20 min in sandbox. If still missing after 30 minutes, confirm Insights is enabled on the merchant account and the date range filter includes today.

**Buyer pool empty on re-run**
If buyers were created on a previous run, the `409 Conflict` on buyer create is handled automatically — the script falls back to `client.buyers.list()` to retrieve the existing buyer. Same for payment methods.

---

## Notes

- The generated script always adds `gr4vy_seeded: true` to transaction metadata. Tell the merchant to filter this out when they start looking at real traffic.
- Apple Pay and Google Pay wallet sessions require device validation and can't be scripted end-to-end via API. The script uses card PANs as a sandbox fallback and preserves the intended method in the `original_method` metadata field.
- Keep the random seed fixed for reproducible runs. Change it for a different spread.
- Don't exceed ~500 txns/day per PSP in sandbox — Adyen and Checkout.com sandbox environments can throttle under load.

---

## Requirements

- Python 3.8+
- [`gr4vy`](https://pypi.org/project/gr4vy/) Python SDK (`pip install gr4vy`)
- A Gr4vy sandbox instance with at least one active connection
- A merchant private key exported as `GR4VY_PRIVATE_KEY`
