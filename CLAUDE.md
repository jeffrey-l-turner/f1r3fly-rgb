# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

F1r3fly-RGB is a Bitcoin Signet wallet with RGB protocol integration for issuing and transferring digital assets on Bitcoin. The project consists of:

1. **Rust Backend** (`wallet/`) - HTTP REST API server with Bitcoin and RGB functionality
2. **React Frontend** (`wallet-frontend/`) - Modern web UI built with React + TypeScript + Vite + Tailwind
3. **RGB Submodules** - Multiple RGB protocol libraries included as git submodules

## Repository Structure

```
f1r3fly-rgb/
├── wallet/                    # Rust backend (axum HTTP server)
│   ├── src/
│   │   ├── api/              # HTTP handlers, routes, request/response types
│   │   ├── wallet/           # Core wallet logic
│   │   │   ├── manager.rs    # Main wallet orchestration (1200+ lines)
│   │   │   ├── rgb.rs        # RGB contract management & asset issuance
│   │   │   ├── rgb_runtime.rs # RGB runtime initialization & sync
│   │   │   ├── transaction.rs # Bitcoin transaction building
│   │   │   ├── signer.rs     # Transaction signing with BIP32 keys
│   │   │   ├── balance.rs    # Balance checking via Esplora
│   │   │   └── storage.rs    # File-based wallet persistence
│   │   └── error.rs          # Error types
│   └── assets/               # RGB20 schema bundled at compile time
├── wallet-frontend/          # React frontend
│   └── src/
│       ├── api/             # API client types
│       ├── components/      # React components (modals, lists, etc)
│       └── pages/           # Page components + docs
├── rgb/                     # RGB core library (submodule)
├── rgb-std/                 # RGB standard library (submodule)
├── rgb-wallet/              # RGB wallet library (submodule)
├── bp-std/                  # Bitcoin Protocol standards (submodule)
└── docs/                    # Design docs & implementation plans
```

## Common Development Commands

### Git Interaction

**For LLM assistance in multi-repo workspace:**
See [Git Interaction Policy](../../top-level-gitlab-profile/docs/common/git-interaction-policy.md)

**For reference (GitLab):**
[Git Interaction Policy](https://gitlab.com/smart-assets.io/gitlab-profile/-/blob/master/docs/common/git-interaction-policy.md)

### Git Submodules

```bash
# Initialize submodules after clone
git submodule update --init --recursive

# Update all submodules to latest
git submodule update --remote
```

### Backend (Rust)

```bash
# Start backend server (binds to 127.0.0.1:3000)
cd wallet
cargo run --release

# Run tests
cargo test --release

# Build RGB wallet binary (if needed)
cd rgb
cargo build -p rgb-wallet --release
../rgb/target/release/rgb --help

# Run all tests including submodules (excluding f1r3node)
cargo test --release && git submodule foreach --recursive \
  'if [ "$name" != "f1r3node" ] && [ -f Cargo.toml ]; then \
   echo "Testing $name"; cargo test --release; fi'
```

### Frontend (React)

```bash
cd wallet-frontend

# Install dependencies
npm install

# Start dev server (http://localhost:5173)
npm run dev

# Build for production
npm run build

# Lint
npm run lint
```

## Architecture Overview

### Backend Architecture

The backend is built around a central `WalletManager` struct that coordinates:

- **Bitcoin Operations**: BIP39 mnemonic generation, BIP32 HD derivation (m/84'/1'/0' path for Signet), P2WPKH addresses, transaction building/signing, Esplora API integration
- **RGB Operations**: RGB runtime initialization, contract issuance (RGB20 tokens), invoice generation, asset transfers, consignment file management
- **Storage**: File-based persistence in `./wallets/{name}/` with JSON metadata, mnemonic, descriptor, and state files

#### Key Backend Files

- `wallet/src/wallet/manager.rs` - Central orchestrator (~1200 lines):
  - Wallet creation/import
  - Address derivation with BIP32 multi-key signing
  - Balance checking with RGB asset binding
  - Bitcoin transaction building (send, create UTXO, unlock UTXO)
  - RGB invoice generation & transfer coordination
  - Consignment accept/export logic

- `wallet/src/wallet/rgb.rs` - RGB contract operations:
  - Check if UTXOs are occupied by RGB assets
  - Extract bound assets from UTXOs
  - Issue RGB20 assets using embedded schema
  - Contract state queries

- `wallet/src/wallet/rgb_runtime.rs` - RGB runtime initialization:
  - Creates `RgbpRuntimeDir` with MultiResolver (Esplora backend)
  - Manages FileHolder (wallet-specific RGB state)
  - Loads shared Contracts (stockpile in `rgb_data/`)
  - Provides both sync and no-sync initialization modes

- `wallet/src/api/handlers.rs` - HTTP request handlers for all endpoints

### Frontend Architecture

React SPA with TypeScript, using:

- **React Router** for client-side routing
- **Axios** for HTTP API calls to backend
- **Tailwind CSS** for styling
- **Component Structure**:
  - `pages/` - Main views (Home, WalletDetail, CreateWallet, ImportWallet, Docs pages)
  - `components/` - Reusable UI components, mostly modals for wallet operations
  - `api/` - TypeScript types and API client functions

### RGB Integration

The wallet implements RGB protocol for issuing and transferring digital assets:

1. **Asset Issuance** (`IssueAssetModal.tsx` → `rgb.rs::issue_rgb20_asset`):
   - Uses embedded RGB20-FNA schema
   - Creates genesis contract bound to a Bitcoin UTXO
   - Stores contract in shared stockpile

2. **Invoice Generation** (`GenerateInvoiceModal.tsx` → `manager.rs::generate_rgb_invoice`):
   - Creates blinded UTXO seal from available UTXOs
   - Generates RGB invoice URI
   - Auto-syncs runtime if needed to register UTXOs

3. **Asset Transfer** (`SendTransferModal.tsx` → `manager.rs::send_transfer`):
   - Parses RGB invoice
   - Creates PSBT with RGB commitment
   - Signs with wallet keys using multi-address derivation
   - Broadcasts Bitcoin transaction
   - Generates consignment file for recipient

4. **Accept Transfer** (`AcceptConsignmentModal.tsx` → `manager.rs::accept_consignment`):
   - Validates consignment (genesis or transfer)
   - Imports into RGB runtime
   - Detects transaction status (pending/confirmed)
   - Updates wallet state after sync

### Multi-Key Signing System

**Critical**: The wallet uses BIP32 hierarchical derivation, meaning each address has a different private key. When building transactions:

- `WalletManager::sign_transaction_multi_key` derives the correct private key for EACH input's UTXO based on its `address_index`
- Each UTXO tracks which address index it belongs to
- Transaction signing iterates over inputs and derives keys on-the-fly

This is implemented in `wallet/src/wallet/manager.rs:593-649`.

### Storage Layout

```
./wallets/
└── {wallet_name}/
    ├── descriptor.txt        # BIP32 extended public key
    ├── mnemonic.txt         # BIP39 seed phrase (24 words)
    ├── metadata.json        # Wallet metadata (name, created_at, network)
    ├── state.json           # Sync state (used addresses, last synced height)
    ├── rgb_wallet/          # FileHolder (wallet-specific RGB state)
    ├── consignments/        # Outgoing transfer consignments
    ├── exports/             # Genesis consignment exports
    └── temp_consignments/   # Temporary files during import

./wallets/rgb_data/          # Shared RGB stockpile (all contracts)
```

## API Endpoints

All endpoints use `http://localhost:3000/api` base URL:

- `POST /wallet/create` - Create new wallet
- `POST /wallet/import` - Import from mnemonic
- `GET /wallets` - List all wallets
- `GET /wallet/{name}/addresses` - Get receive addresses
- `GET /wallet/{name}/primary-address` - Get address #0 (always used)
- `GET /wallet/{name}/balance` - Get balance + UTXOs + RGB assets
- `POST /wallet/{name}/sync` - Sync Bitcoin wallet
- `POST /wallet/{name}/sync-rgb` - Sync RGB runtime (updates contract states)
- `POST /wallet/{name}/utxo/create` - Create specific UTXO amount
- `POST /wallet/{name}/utxo/unlock` - Spend a UTXO back to self
- `POST /wallet/{name}/send-bitcoin` - Send Bitcoin to address
- `POST /wallet/{name}/rgb/issue` - Issue RGB20 asset
- `POST /wallet/{name}/rgb/invoice` - Generate RGB invoice
- `POST /wallet/{name}/rgb/transfer` - Send RGB asset transfer
- `POST /wallet/{name}/rgb/accept` - Accept consignment
- `GET /wallet/{name}/rgb/export-genesis/{contract_id}` - Export genesis
- `GET /consignment/{filename}` - Download consignment file
- `GET /genesis/{filename}` - Download genesis file

## Network Configuration

- **Bitcoin Network**: Signet (testnet for RGB development)
- **Blockchain API**: Esplora at `https://mempool.space/signet/api`
- **BIP32 Derivation Path**: `m/84'/1'/0'/0/{index}` (Signet coin type = 1)
- **Address Format**: P2WPKH (native SegWit)

## Development Workflow

### Creating a Wallet
1. Frontend calls `POST /wallet/create`
2. Backend generates 24-word mnemonic
3. Derives BIP32 master key and descriptor
4. Saves to `./wallets/{name}/`
5. Returns mnemonic + first address

### Issuing RGB Assets
1. Wallet must have at least one UTXO (use faucet)
2. Call `POST /wallet/{name}/rgb/issue` with asset metadata
3. RGB contract is bound to specified UTXO
4. Contract ID returned for future operations

### Transferring RGB Assets
1. Receiver generates invoice: `POST /wallet/{name}/rgb/invoice`
2. Sender pays invoice: `POST /wallet/{name}/rgb/transfer`
3. Bitcoin transaction broadcast + consignment created
4. Receiver downloads consignment file
5. Receiver accepts: `POST /wallet/{name}/rgb/accept`
6. Both parties sync: `POST /wallet/{name}/sync-rgb`

## Important Implementation Details

### RGB Runtime Sync Modes

The wallet has two RGB runtime initialization modes:

1. **With Sync** (`init_runtime`) - Queries blockchain for 32 confirmations, slow but accurate
2. **No Sync** (`init_runtime_no_sync`) - Fast startup, uses cached state

Use no-sync for invoice generation and consignment operations. Use sync after transfers to update balances.

### UTXO Occupation

RGB assets are "bound" to Bitcoin UTXOs. The balance checker marks UTXOs as `is_occupied` when they contain RGB allocations. Occupied UTXOs should generally not be spent for regular Bitcoin transactions (they're used for RGB operations or need to be "unlocked" first).

### Address Index Strategy

The wallet uses address index 0 as the "primary address" for consistent development experience (`get_primary_address` always returns index 0). Address derivation gap limit is 20.

### Transaction Fee Handling

Default fee rate is 2 sat/vB. All transaction building functions accept optional `fee_rate_sat_vb` parameter.

### Consignment File Management

- Transfer consignments stored as `transfer_{contract_id}_{timestamp}.rgbc`
- Genesis exports stored as `genesis_{contract_id}.rgbc`
- Files served via direct HTTP download endpoints
- Cleanup of temp files handled automatically

## Testing & Debugging

The backend includes extensive `eprintln!` debug output for RGB operations. Look for:
- [DEBUG] - Operation starting
- [SYNC] - Processing step
- [OK] - Success
- [ERROR] - Error
- [SATS] - Fee/amount info
- [NOTE] - Data/info
- [MSG] - Network operations

Frontend uses console.log for debugging API calls and component state.

## Common Gotchas

1. **RGB Runtime Sync**: After any RGB transfer (send or accept), call `/sync-rgb` to update contract states and balances
2. **Multi-Key Signing**: Always use `sign_transaction_multi_key` for transactions - never derive a single key
3. **UTXO Requirements**: Many RGB operations require available UTXOs - direct users to create UTXOs if needed
4. **Submodule Versions**: RGB libraries are tightly coupled - avoid updating submodules independently
5. **Genesis vs Transfer**: Genesis consignments don't have Bitcoin transactions; transfers do
6. **Invoice Without UTXOs**: Invoice generation fails without UTXOs - the error message guides users through the solution
7. **Descriptor Format**: RGB uses `XpubDerivable` format, different from standard Bitcoin descriptors

## Documentation

The `docs/` directory contains detailed implementation plans and research findings:
- `rgb-transfer-implementation-plan.md` - Complete transfer flow implementation
- `rgb-asset-issuance-plan.md` - Asset issuance implementation
- `rgb-runtime-research-findings.md` - RGB runtime API exploration
- `rgb-transfer-user-flows.md` - User interaction flows
- `phase-3-rgb-integration-plan.md` - Original RGB integration plan
