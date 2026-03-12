# trustMeBro — Non-Custodial Lightning Escrow

A non-custodial escrow service for Bitcoin Lightning Network, using [Fedimint](https://fedimint.org) as the trustless escrow layer.

**Live:** [trustbro.trade](https://trustbro.trade)

## How It Works

```
Buyer (Lightning) → Fedimint Escrow → Release/Refund → Seller/Buyer (Lightning)
```

1. **Seller creates a deal** — sets title, price, conditions, timeout
2. **Buyer joins** — authenticates with Lightning wallet (LNURL-auth)
3. **Buyer funds** — pays a Lightning invoice, funds are locked in Fedimint escrow
4. **Release** — buyer confirms delivery, seller gets paid via Lightning
5. **Dispute** — if disagreement, 2-of-3 oracle arbitration resolves it
6. **Timeout** — if no action by deadline, funds auto-release per deal terms

Users only interact with Lightning. Fedimint is invisible.

## Non-Custodial Architecture

The service **cannot move funds unilaterally**:

- **Buyer-generated secret**: only the buyer holds the `secret_code` needed to release funds
- **Ephemeral keys**: both parties derive keys from LNURL-auth, stored only in their browser
- **Delegated Schnorr signatures**: Fedimint escrow requires user signatures for all claim operations
- **Oracle dispute resolution**: 2-of-3 independent arbitrators, service operator is not one of them
- **Timeout recovery**: if the service disappears, users can claim via any Fedimint client after timeout

See [docs/non-custodial-architecture.md](docs/non-custodial-architecture.md) for the full security model.

## Stack

| Layer | Technology |
|-------|-----------|
| Frontend | SvelteKit (Svelte 5), static build |
| Backend | Python FastAPI |
| Escrow | Custom Fedimint module ([fedimint-escrow](https://github.com/Antisys/fedimint-escrow)) |
| Auth | LNURL-auth (LUD-04) — wallet-based, no accounts |
| Payments | Lightning Network via Fedimint gateway + LND |
| Database | SQLite |
| Crypto | secp256k1 Schnorr signatures, AES-256-GCM encrypted backups |

## Quick Start

### Prerequisites

- Python 3.10+
- Node.js 18+
- A Fedimint federation with the escrow module
- LND node (for Lightning payments)

### Setup

```bash
# Clone
git clone https://github.com/Antisys/ln-escrow.git
cd ln-escrow

# Backend
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Frontend
cd frontend-svelte
npm install
cd ..

# Configure
cp .env.example .env
# Edit .env — see comments in .env.example for required settings
```

### Run

```bash
# Terminal 1: Backend (port 8001)
source venv/bin/activate
python -m backend.api.main

# Terminal 2: Frontend (port 5173)
cd frontend-svelte
npm run dev
```

### Deploy

```bash
DEPLOY_PASS=your-password bash deploy.sh
```

## Project Structure

```
ln-escrow/
├── backend/
│   ├── api/routes/        # FastAPI endpoints (deals, auth, funding, payout, admin)
│   ├── auth/              # LNURL-auth + Schnorr signature verification
│   ├── fedimint/          # Fedimint escrow client (CLI + HTTP)
│   ├── lightning/         # LND REST client
│   ├── database/          # SQLAlchemy models, deal storage
│   └── tasks/             # Background: timeout handler, payout retries
├── frontend-svelte/
│   ├── src/routes/        # Pages: home, create, join, deal, admin, how
│   ├── src/lib/api/       # API client modules
│   ├── src/lib/components/# Svelte components (deal panels, auth, admin)
│   ├── src/lib/crypto.js  # Key derivation, Schnorr signing, vault encryption
│   └── src/lib/stores/    # Svelte stores (vault, nav)
├── docs/                  # Architecture docs, Fedimint module specs
├── tests/                 # Backend tests (smoke, payout flows, E2E)
├── tools/                 # Oracle signing, recovery, monitoring
└── deploy/                # systemd, nginx, Cloudflare tunnel configs
```

## API

| Endpoint | Description |
|----------|-------------|
| `POST /deals` | Create new deal |
| `GET /deals/{id}` | Get deal details |
| `POST /deals/{id}/create-ln-invoice` | Generate funding invoice |
| `POST /deals/{id}/release` | Release funds to seller |
| `POST /deals/{id}/refund` | Refund funds to buyer |
| `POST /deals/{id}/dispute` | Open dispute |
| `GET /auth/lnurl/challenge/{token}` | LNURL-auth challenge |
| `GET /health` | Health check |
| `GET /system-status` | System status (federation, gateway, LND) |

## Testing

```bash
source venv/bin/activate
pytest tests/ -x -q
```

## Security

- `secret_code` never stored on server — only SHA256 hash
- No private keys on server that can move escrow funds
- All payout operations require user Schnorr signatures
- Oracle keys must be distributed to 3 independent parties for production
- Rate limiting on all endpoints
- Input sanitization on all user content

## License

MIT
