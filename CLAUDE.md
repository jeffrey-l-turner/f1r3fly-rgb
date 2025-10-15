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

## Related Documentation

### Parent Project

**For LLM assistance in multi-repo workspace:**
See [SATCHEL GitLab Profile](../gitlab-profile/CLAUDE.md) for subgroup coordination and project relationships.

**For reference (GitLab):**
[SATCHEL GitLab Profile](https://gitlab.com/smart-assets.io/SATCHEL/gitlab-profile/-/blob/master/CLAUDE.md)

### Related SATCHEL Projects

This Bitcoin/RGB wallet integrates with other SATCHEL projects for a complete cross-chain experience:

**satchel_ux** - Cross-chain wallet user interface that provides the DApp frontend for wallet operations.

**For LLM assistance in multi-repo workspace:**
See [satchel_ux](../satchel_ux/CLAUDE.md)

**For reference (GitLab):**
[satchel_ux](https://gitlab.com/smart-assets.io/SATCHEL/satchel_ux/-/blob/master/CLAUDE.md)

**satchelprotocol** - Smart contract protocol implementation for cross-chain asset management that f1r3fly-rgb uses for RGB asset operations.

**For LLM assistance in multi-repo workspace:**
See [satchelprotocol](../satchelprotocol/CLAUDE.md)

**For reference (GitLab):**
[satchelprotocol](https://gitlab.com/smart-assets.io/SATCHEL/satchelprotocol/-/blob/master/CLAUDE.md)

## Security Considerations

This project handles Bitcoin and RGB assets, requiring exceptional security practices. The following guidelines are CRITICAL for maintaining wallet security and protecting user funds.

### Wallet Security Best Practices

**Private Key Management:**
- Private keys are derived from BIP39 mnemonics and NEVER exposed in logs or API responses
- Mnemonics are stored in plaintext files (`wallets/{name}/mnemonic.txt`) - these files MUST be protected with file system permissions
- The wallet uses BIP32 hierarchical derivation (m/84'/1'/0'/0/{index}) - each address has a unique private key
- Never log or expose derived private keys in any form
- All signing operations must occur server-side with keys kept in secure storage

**Hardware Wallet Support:**
- Current implementation uses software-based key derivation and signing
- For production use, consider integrating hardware wallet support (Ledger, Trezor)
- Hardware wallets should be the recommended approach for mainnet deployments
- Document hardware wallet integration paths for future enhancement

**Multi-Signature Considerations:**
- Current wallet is single-signature (one key per address)
- For high-value operations, consider implementing multi-signature requirements
- RGB protocol supports multi-sig through Bitcoin's native script capabilities
- Document multi-sig patterns for enterprise or institutional deployments

**Key Derivation and HD Wallet Security:**
- Uses BIP32 HD wallet structure with gap limit of 20 addresses
- Address reuse is minimized through HD derivation
- Master key is never directly used - all operations use derived keys
- Ensure proper entropy sources for mnemonic generation (see `WalletManager::create_wallet`)
- Never use weak or predictable entropy sources

### Transaction Signing Security

**Transaction Verification Before Signing:**
- Always verify transaction details before signing (amounts, recipients, fees)
- Implement user confirmation dialogs for all transaction signing operations
- Display all inputs, outputs, and fee calculations clearly
- For RGB transfers, show both Bitcoin transaction details AND RGB asset movement

**Double-Spend Prevention:**
- Use Esplora API to verify UTXO availability before building transactions
- Implement proper UTXO locking during transaction construction
- RGB asset UTXOs must be tracked separately to prevent asset loss
- Mark UTXOs as "occupied" when they contain RGB allocations

**Fee Estimation and Validation:**
- Current default fee rate: 2 sat/vB (configurable per transaction)
- Always validate fee rates against current network conditions
- Prevent excessively high fees that could drain wallet funds
- Implement fee estimation using mempool data for optimal transaction timing
- For RGB transfers, account for both Bitcoin fees AND RGB commitment overhead

**Transaction Malleability Concerns:**
- Use native SegWit (P2WPKH) addresses exclusively to prevent malleability
- PSBT (Partially Signed Bitcoin Transaction) format is used for RGB transfers
- RGB commitments are bound to specific transaction IDs - malleability would break RGB transfers
- Always use BIP141/BIP143 signing for witness transactions

### Private Key Management

**Never Log or Expose Private Keys:**
- CRITICAL: Private keys MUST NEVER appear in logs, debug output, or API responses
- Review all `eprintln!` debug statements to ensure no key material is logged
- Use secure memory handling practices (consider zeroizing keys after use)
- Implement audit logging that explicitly excludes sensitive data

**Secure Key Generation:**
- Use cryptographically secure random number generators for mnemonic generation
- Ensure sufficient entropy (256 bits for 24-word mnemonics)
- Never use predictable seeds, timestamps, or user-provided data as entropy
- Validate mnemonic checksums on import to prevent corruption

**Key Backup and Recovery:**
- Users must securely backup their 24-word mnemonic phrase
- Implement secure mnemonic display (one-time view with user confirmation)
- Warn users about secure storage: paper wallets, metal backups, encrypted digital storage
- Never store mnemonics in plaintext on shared/cloud storage
- Consider implementing Shamir Secret Sharing (SLIP-39) for advanced backup scenarios

**Memory Handling for Sensitive Data:**
- Minimize time that private keys exist in memory
- Consider using secure memory (mlock/mprotect) for key material
- Zeroize memory containing keys immediately after use
- Rust's memory safety helps, but explicit zeroization is still recommended
- Review all key derivation paths for potential memory leaks

### RGB Asset Handling Security

**RGB Protocol Security Considerations:**
- RGB uses client-side validation - each party must independently validate state
- Contract state is stored locally in RGB runtime (not on Bitcoin blockchain)
- Consensus rules are enforced by RGB schema (not Bitcoin network)
- Invalid RGB operations can result in asset loss without blockchain-level protection

**Asset Validation and Verification:**
- Always validate consignment files before accepting transfers
- Verify RGB contract genesis matches expected schema (RGB20-FNA)
- Check that asset allocations match invoice amounts
- Validate that Bitcoin transactions are properly confirmed before considering RGB transfer complete
- Implement consignment validation using `RgbRuntime::validate_consignment`

**State Transition Security:**
- RGB state transitions must be validated against contract schema
- Each state transition is cryptographically bound to a Bitcoin UTXO
- Spent UTXOs must be tracked to prevent double-spending of RGB assets
- Use RGB runtime sync (`sync-rgb` endpoint) to update contract states after transfers

**Client-Side Validation Requirements:**
- Both sender and receiver must validate RGB consignments independently
- Invalid consignments should be rejected immediately with clear error messages
- Maintain local RGB state consistency through proper sync operations
- Never trust remote RGB state without local validation
- Genesis consignments establish initial contract state and must be validated first

### Bitcoin Security Best Practices

**UTXO Management:**
- Track all UTXOs with their address indices for proper key derivation during signing
- Implement coin selection strategies that optimize privacy and fee efficiency
- Mark UTXOs as "occupied" when they contain RGB assets to prevent accidental spending
- Use UTXO unlocking feature (`/utxo/unlock`) carefully - it can expose RGB assets to loss
- Consider implementing UTXO consolidation strategies for wallet maintenance

**Address Reuse Considerations:**
- HD wallet derivation minimizes address reuse automatically
- Each receive operation should use a fresh address from the derivation path
- Address index 0 is used as "primary address" for convenience - consider privacy implications
- For maximum privacy, encourage users to generate new addresses for each receive
- RGB operations may require specific address handling - document RGB-specific address usage

**Network Security:**
- All Bitcoin and RGB operations should use secure connections (HTTPS for Esplora)
- Consider Tor support for enhanced privacy (hide IP addresses from Esplora/network)
- Implement VPN recommendations for users in restrictive jurisdictions
- Validate SSL certificates when connecting to blockchain APIs
- Consider running a local Bitcoin/Esplora node for enhanced security and privacy

**Reference Bitcoin Core Security Guidelines:**
- Follow Bitcoin Core's security best practices: https://bitcoin.org/en/secure-your-wallet
- Stay updated on Bitcoin security advisories and CVEs
- Review Bitcoin Improvement Proposals (BIPs) for security-relevant changes
- Test wallet behavior against Bitcoin Core reference implementation
- Participate in Bitcoin security community discussions and audits

### Common Security Practices

**For LLM assistance in multi-repo workspace:**
See [Security Best Practices](../../top-level-gitlab-profile/docs/common/security-best-practices.md)

**For reference (GitLab):**
[Security Best Practices](https://gitlab.com/smart-assets.io/gitlab-profile/-/blob/master/docs/common/security-best-practices.md)

### Production Deployment Checklist

Before deploying this wallet to production:

1. Conduct comprehensive security audit of all wallet and RGB code
2. Implement hardware wallet support for key management
3. Add transaction confirmation dialogs with detailed breakdowns
4. Implement proper logging that excludes all sensitive data
5. Set up secure backup and recovery mechanisms
6. Test against mainnet with small amounts first
7. Implement rate limiting for API endpoints
8. Add intrusion detection and monitoring
9. Secure file system permissions for wallet directories
10. Implement disaster recovery procedures
