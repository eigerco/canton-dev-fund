# Canton Forge: A Developer-Centric CLI Toolset for Canton Network

**Design Proposal** - Addressing the DevEx gaps identified in documentation and the 2026 Developer Survey

---

## Executive Summary

Canton Network's current developer experience requires developers to become "Infrastructure Engineers before Product Builders" (survey finding). With 71% of Canton developers coming from Ethereum and 11+ survey mentions requesting Hardhat/Foundry-like tooling, there's a clear mandate for a unified CLI.

This proposal introduces **Canton Forge** (`forge` or `canton`), a comprehensive CLI that:
- Reduces setup friction from hours to seconds
- Eliminates 60+ character hex IDs with aliases and prefix resolution
- Hides JWT/OAuth complexity behind automatic credential management
- Provides `cast`-equivalent one-liner interactions
- Unifies fragmented tooling (JSON API, gRPC, Console, daml-shell) into one interface

---

## Pain Points Addressed

| # | Pain Point | Severity | Solution |
|---|-----------|----------|----------|
| 1 | Long hex IDs (68-136 chars) | High | Aliases + prefix resolution |
| 2 | No `cast call` equivalent | High | `canton call` command |
| 3 | OAuth2 token hunting | High | Auto-managed credential store |
| 4 | Party collision on sandbox restart | Medium | `getOrAllocate` semantics |
| 5 | 4 API calls for one exercise | High | Single `canton call` command |
| 6 | Package ID discovery is "opaque" | High | `canton pkg list --names` |
| 7 | Undocumented magic flags | High | Sensible defaults + warnings |
| 8 | Multi-party authorization ceremony | Medium | `--act-as` / `--read-as` flags |
| 9 | Environment setup friction | High | `canton up` one-liner |
| 10 | Token transfers require 4+ API calls | High | `canton wallet transfer` |
| 11 | UTXO management is manual | Medium | `canton wallet merge` |
| 12 | Scan/Registry API complexity | High | `canton wallet` abstractions |

---

## Tool Architecture

```
canton (or forge)
├── up          # Start local networks (anvil equivalent)
├── down        # Stop networks
├── call        # Exercise choices on contracts (read views, split, merge, etc.)
├── query       # Query contracts, parties, packages (passive reads)
├── upload      # Deploy DAR packages
├── party       # Party management
├── wallet      # Wallet & token management (Amulet/CC, CIP-56 tokens)
├── auth        # Credential management
└── config      # Configuration
```

---

## 1. `canton up` - Network Launcher (Anvil Equivalent)

The biggest friction point: spinning up a development network. Current state requires understanding docker-compose, 15+ containers, OAuth2 setup, and env file hunting.

### Basic Usage

```bash
# Quick sandbox (replaces: dpm sandbox)
canton up

# Named sandbox with persistence
canton up --name myproject

# With specific parties pre-allocated
canton up --party alice --party bob --party bank

# Output:
# ✓ Canton sandbox started on localhost:6865
# ✓ JSON API on localhost:7575
# ✓ Parties allocated:
#   alice → alice::1220abc... (alias: alice)
#   bob   → bob::1220def...   (alias: bob)
#   bank  → bank::1220789...  (alias: bank)
# ✓ Auth token stored in ~/.canton/credentials
```

### Multi-Domain Networks

Canton's killer feature is multi-domain privacy. This should be easy to set up:

```bash
# Two domains with automatic participant assignment
canton up --domain trading --domain settlement

# Explicit topology with sync domain for cross-domain settlement
canton up \
  --domain trading:alice,bob \
  --domain settlement:bank,custodian \
  --sync-domain global

# Complex topology with separate sequencers
canton up \
  --topology config/network.yaml

# Output:
# ✓ Domains:
#   trading    → localhost:10018 (sequencer: localhost:10028)
#   settlement → localhost:10019 (sequencer: localhost:10029)
#   global     → localhost:10020 (sync domain)
# ✓ Participants:
#   alice → connected to: trading
#   bob   → connected to: trading
#   bank  → connected to: settlement, global
#   custodian → connected to: settlement
```

### Sync Domain with Amulet & Super Validators

For production-like deployments with native token support, use `--with-splice` to enable the full Splice stack (Amulet, Scan, Wallet, SV). At least one Super Validator is required on the sync domain.

```bash
# Sync domain with Splice stack (Amulet + Scan + Wallet)
# Automatically designates first party on sync domain as SV
canton up \
  --domain trading:alice,bob \
  --domain settlement:bank,custodian \
  --sync-domain global:bank \
  --with-splice

# Explicit SV designation (bank is the Super Validator)
canton up \
  --domain trading:alice,bob \
  --domain settlement:bank,custodian \
  --sync-domain global \
  --with-splice \
  --sv bank

# Multiple SVs for quorum (production-like)
canton up \
  --domain trading:alice,bob \
  --domain settlement:bank,custodian \
  --sync-domain global \
  --with-splice \
  --sv bank --sv custodian

# Output (with --with-splice --sv bank):
# ✓ Domains:
#   trading    → localhost:10018 (sequencer: localhost:10028)
#   settlement → localhost:10019 (sequencer: localhost:10029)
#   global     → localhost:10020 (sync domain)
# ✓ Participants:
#   alice     → connected to: trading
#   bob       → connected to: trading
#   bank      → connected to: settlement, global (SV)
#   custodian → connected to: settlement
# ✓ Splice Stack:
#   Super Validator: bank (localhost:5014)
#   Scan (Registry): localhost:5012
#   Wallet Web UI:   localhost:3000
# ✓ Amulet: Canton Coin (CC)
# ✓ Token Standard APIs: enabled
# ✓ Faucet: canton wallet tap --as bank
```

**Sync Domain & Super Validator Notes:**
- The sync domain (`--sync-domain`) is the global domain connecting private domains
- `--with-splice` enables: Super Validator, Scan (data indexing), Wallet, Amulet (native token)
- At least one `--sv` is required when using `--with-splice`
- SVs must be participants connected to the sync domain
- SVs control Amulet price, holding fees, and network governance
- **Scan** indexes all Token Standard contracts and serves the registry APIs
- CIP-56 contracts can be deployed without `--with-splice`, but Token Standard APIs require Scan

### Production-Like (Localnet)

```bash
# Full stack with OAuth2, PQS, etc (replaces docker-compose + env hunting)
canton up --mode localnet

# With Splice stack (Amulet + Scan + Wallet + SV)
canton up --mode localnet --with-splice

# Lighter deployment without Splice
canton up --mode localnet --no-splice

# With custom Keycloak config
canton up --mode localnet --oauth config/keycloak.json

# Specific versions
canton up --mode localnet --canton-version 3.4.11

# Output (default localnet includes Splice):
# ✓ Localnet started (18 containers)
# ✓ Canton: localhost:6865
# ✓ JSON API: localhost:7575
# ✓ Keycloak: localhost:8080
# ✓ PQS: localhost:4000
# ✓ Splice Stack:
#   Super Validator: localhost:5014
#   Scan (Registry): localhost:5012
#   Wallet Web UI:   localhost:3000
# ✓ Amulet: Canton Coin (CC)
# ✓ OAuth2 credentials auto-configured
# ✓ Faucet: canton wallet tap
```

### Why This Matters

**Current workflow:**
```bash
cd quickstart
docker-compose up -d                    # Start 15 containers
# Wait... hunt for env files...
source .env.alice_validator_wallet      # Find credentials
curl -X POST http://localhost:8080/...  # Get OAuth token manually
export AUTH_TOKEN=...                   # Set in environment
# Now you can make API calls
```

**Proposed workflow:**
```bash
canton up --mode localnet
# Done. Credentials auto-managed.
```

---

## 2. `canton call` - Exercise Choices on Contracts

The biggest gap vs EVM tooling. Currently requires 4 API calls and copy-pasting 130+ character IDs. `canton call` exercises any choice on a contract - whether reading state via interface views or modifying state via Split/Merge/etc.

### Basic Usage

```bash
# Read contract state via interface view
canton call @ore-token GetView --as alice

# Output (JSON by default):
{
  "assetOwner": "alice::1220abc...",
  "description": "Magic Ore",
  "quantity": 100.0
}

# Exercise a state-changing choice
canton call @ore-token Split --args '{"amount": 30.0}' --as alice

# Output:
✓ Choice exercised: Split
  Contract: 00abc123...
  Result:
    newCid1: 00xyz789... (alias: @ore-token-1)
    newCid2: 00uvw456... (alias: @ore-token-2)
  Transaction ID: tx-123456

# Contract ID with prefix resolution
canton call 00abc GetView --as alice

# Full contract ID also works
canton call 00abc123def456... GetView --as alice
```

### Multi-Party Authorization

```bash
# Multi-party authorization
canton call @proposal AcceptTransfer \
  --act-as alice,bob \
  --read-as bank

# With JSON args file
canton call @token Merge --args-file merge-params.json --as alice
```

### Alias Support

```bash
# Set up aliases for readability
canton alias set ore-token 00abc123
canton alias set alice-token 00def456

# Use aliases anywhere
canton call @ore-token GetView
canton call @alice-token Split --args '{"amount": 30.0}'
```

### Package Resolution

```bash
# Specify package by name (not 64-char hex)
canton call 00abc GetView --package ore-bank-interfaces

# Or by prefix
canton call 00abc GetView --package 7a3b
```

### Create Contracts

```bash
# Create a new contract
canton call --create OreToken \
  --args '{"issuer": "@bank", "owner": "@alice", "grams": 100.0}' \
  --act-as bank,alice

# With package specification
canton call --create Main:OreToken \
  --package ore-bank-main \
  --args '{"issuer": "@bank", "owner": "@alice", "grams": 100.0}' \
  --act-as bank,alice
```

### Dry Run / Simulation

```bash
# Simulate without committing (like Tenderly - survey request!)
canton call @token Split --args '{"amount": 30.0}' --dry-run --as alice

# Output:
DRY RUN - Transaction not committed
  Would archive: 00abc123...
  Would create:
    - OreToken { grams: 30.0, owner: alice }
    - OreToken { grams: 70.0, owner: alice }
  Estimated disclosure: alice, bank
```

**Note:** Dry-run requires either implementing a Daml interpreter + PQS state reader, or a native Canton simulation API (feature request). This is a significant implementation effort.

### Why This Matters

**Current workflow (JSON API):**
```bash
# 1. Get party ID
curl -s http://localhost:7575/v2/parties | jq '.result[] | select(.displayName=="Alice")'
# Copy 68-char party ID

# 2. Get current offset
curl -s http://localhost:7575/v2/state/end

# 3. Construct 30-line JSON filter, make query
curl -X POST http://localhost:7575/v2/state/acs \
  -H "Content-Type: application/json" \
  -d '{"filter":{"filtersByParty":{...}}}'

# 4. Parse result, find contract ID (136 chars)

# 5. Finally exercise choice
curl -X POST http://localhost:7575/v2/commands/submit-and-wait \
  -d '{"commands":[{"exerciseCommand":{...20 lines...}}]}'
```

**Proposed workflow:**
```bash
canton call @my-contract GetView --as alice
canton call @my-contract Split --args '{"amount": 30.0}' --as alice
```

---

## 3. `canton query` - Unified Query Interface

Replaces separate JSON API, gRPC, and daml-shell queries.

### Parties

```bash
# List all parties
canton query parties

# Output:
ALIAS     DISPLAY_NAME    PARTY_ID                      PARTICIPANT
alice     Alice           alice::1220abc123...          participant1
bob       Bob             bob::1220def456...            participant1
bank      OreBank         bank::1220789xyz...           participant1

# Find party by prefix or alias
canton query party alice
canton query party 1220abc

# Check if party exists
canton query party --exists alice && echo "exists"
```

### Contracts

```bash
# Active contracts for a party
canton query contracts --as alice

# Filter by template
canton query contracts --template OreToken --as alice

# With filter expression
canton query contracts --template OreToken --filter 'grams > 50' --as alice

# Output:
[
  { "contractId": "00abc...", "grams": 100.0, "owner": "alice" },
  { "contractId": "00def...", "grams": 75.0, "owner": "alice" }
]

# SQL-like syntax (inspired by daml-shell, but CLI-friendly)
canton query contracts --where "template = 'OreToken' AND grams > 50" --as alice

# Output format options
canton query contracts --as alice --format table
canton query contracts --as alice --format json
canton query contracts --as alice --format csv
```

### Packages

```bash
# List uploaded packages
canton query packages

# Output:
NAME                  PACKAGE_ID                          VERSION   UPLOADED
ore-bank-interfaces   7a3b2c1d4e5f...                    0.0.1     2026-03-05
ore-bank-main         8b4c3d2e5f6a...                    0.0.1     2026-03-05
ore-bank-test         9c5d4e3f6a7b...                    0.0.1     2026-03-05

# Inspect package contents
canton query package ore-bank-main --templates
canton query package 7a3b --choices OreToken
```

### Transactions

```bash
# Recent transactions
canton query transactions --as alice --limit 10

# Specific transaction
canton query transaction tx-123456

# Events in a transaction
canton query events --transaction tx-123456
```

---

## 4. `canton upload` - Package Deployment

```bash
# Upload a DAR
canton upload ./dist/ore-bank-main-0.0.1.dar

# Upload all DARs in directory
canton upload ./dist/

# Upload with alias
canton upload ./dist/ore-bank-main-0.0.1.dar --as ore-bank

# To specific participant (multi-node)
canton upload ./dist/*.dar --participant alice-participant

# Output:
✓ Uploaded: ore-bank-main-0.0.1.dar
  Package ID: 8b4c3d2e5f6a... (alias: @ore-bank-main)
  Templates: OreToken, Split, Merge
```

### Build + Upload

```bash
# Build and upload in one step
canton upload --build ./main/

# Output:
✓ Building ./main/
  Running: dpm build
✓ Uploaded: main-0.0.1.dar
```

---

## 5. `canton party` - Party Management

```bash
# Allocate a party
canton party new alice

# With hint (display name)
canton party new alice --hint "Alice the Trader"

# Get-or-create semantics (solves sandbox collision issue!)
canton party ensure alice

# Output:
✓ Party: alice
  ID: alice::1220abc123def456...
  Participant: participant1

# List parties
canton party list

# Party rights (multi-party workflows)
canton party grant alice --act-as bob
canton party grant alice --read-as bank
```

### Why `ensure` Matters

**Current pain point:** Running the same script twice on sandbox fails because `allocateParty "Alice"` is deterministic and errors on duplicate.

**Solution:**
```bash
canton party ensure alice  # Creates if missing, returns existing if present
```

---

## 6. `canton wallet` - Wallet & Token Management

Manage Amulet (Canton Coin) and CIP-56 compliant tokens. Requires `--with-splice` to be enabled on the network.

### Amulet (Native Token) Operations

```bash
# Check Amulet/CC balance
canton wallet balance --as alice

# Output:
ASSET          BALANCE      LOCKED    AVAILABLE
Canton Coin    1,250.50 CC  100.00    1,150.50

# Detailed holdings (UTXO view)
canton wallet balance --as alice --detailed

# Output:
CONTRACT_ID     AMOUNT      LOCKED_UNTIL    CREATED
@holding-1      500.00 CC   -               2026-03-05 10:00
@holding-2      450.50 CC   -               2026-03-05 11:30
@holding-3      300.00 CC   2026-03-10      2026-03-05 12:00

# Get CC from faucet (devnet/localnet only)
canton wallet tap --as alice

# Output:
✓ Tapped faucet
  Received: 100.00 CC
  New balance: 1,350.50 CC
  Holding: @holding-4

# Tap specific amount
canton wallet tap --amount 500 --as alice
```

### Transfers (FOP - Free of Payment)

```bash
# Transfer Amulet to another party
canton wallet transfer --to bob --amount 50 --as alice

# Output:
✓ Transfer initiated
  From: alice
  To: bob
  Amount: 50.00 CC
  Transaction: tx-789xyz
  Status: Completed

# Transfer with deadline
canton wallet transfer --to bob --amount 50 --expires 1h --as alice

# Check pending transfers
canton wallet transfers --pending --as alice
```

### CIP-56 Token Operations

CIP-56 is Canton's token standard (like ERC-20). Any token implementing the Holding interface can be managed.

```bash
# List all token holdings (not just Amulet)
canton wallet holdings --as alice

# Output:
TOKEN           REGISTRY        BALANCE      SYMBOL
Canton Coin     splice          1,250.50     CC
Project Token   acme-registry   5,000.00     PTK
Gold Token      ore-bank        100.0g       GOLD

# Filter by token/registry
canton wallet holdings --token "Gold Token" --as alice
canton wallet holdings --registry ore-bank --as alice

# Transfer CIP-56 token
canton wallet transfer \
  --token "Gold Token" \
  --to bob \
  --amount 25.5 \
  --as alice

# Output:
✓ Transfer initiated
  Token: Gold Token (ore-bank registry)
  From: alice
  To: bob
  Amount: 25.50 GOLD
  Status: Completed
```

### Allocations (DVP - Delivery vs Payment)

For atomic multi-asset settlements:

```bash
# Create allocation for DVP settlement
canton wallet allocate \
  --token "Gold Token" \
  --amount 50 \
  --for @settlement-proposal \
  --expires 24h \
  --as alice

# Output:
✓ Allocation created
  Allocation ID: @alloc-123
  Token: Gold Token
  Amount: 50.00 GOLD
  Locked until: 2026-03-07 12:00:00
  Settlement: @settlement-proposal

# List allocations
canton wallet allocations --as alice

# Cancel allocation (if not yet settled)
canton wallet allocate --cancel @alloc-123 --as alice
```

### Merge Holdings (UTXO Consolidation)

CIP-56 uses UTXO model. Consolidate small holdings for efficiency:

```bash
# Merge all holdings of a token into one
canton wallet merge --token "Canton Coin" --as alice

# Output:
✓ Merged 5 holdings into 1
  Previous: @holding-1, @holding-2, @holding-3, @holding-4, @holding-5
  New: @holding-6 (1,350.50 CC)

# Merge holdings for specific registry
canton wallet merge --registry ore-bank --as alice
```

### Token Registry Queries

Query the Scan service for token metadata:

```bash
# List known token registries
canton wallet registries

# Output:
REGISTRY        URL                         TOKENS
splice          http://localhost:5012       Canton Coin
ore-bank        http://localhost:5020       Gold Token, Silver Token
acme-registry   http://localhost:5030       Project Token

# Get token metadata
canton wallet token-info "Gold Token"

# Output:
Token: Gold Token
Symbol: GOLD
Registry: ore-bank (http://localhost:5020)
Total Supply: 10,000.00 GOLD
Decimals: 10
Admin: bank::1220789...
```

### Why `canton wallet` Matters

**Current workflow (complex):**
```bash
# 1. Query Scan API for holdings
curl -s http://localhost:5012/api/scan/v0/holdings \
  -H "Authorization: Bearer $TOKEN" | jq

# 2. Find registry URL from CNS
curl -s http://localhost:5012/api/scan/v0/ans-entries/...

# 3. Get transfer instruction context from registry
curl -X POST http://localhost:5012/registry/transfer/v1/create-context \
  -H "Content-Type: application/json" \
  -d '{"sender": "...", "receiver": "...", ...}'

# 4. Execute transfer via Ledger API
curl -X POST http://localhost:7575/v2/commands/submit-and-wait \
  -d '{"commands": [{"exerciseCommand": {...}}]}'
```

**Proposed workflow:**
```bash
canton wallet transfer --to bob --amount 50 --as alice
```

---

## 7. `canton auth` - Credential Management

Hides OAuth2/JWT complexity entirely.

```bash
# Get current token (auto-refreshes if expired)
canton auth token

# Show credential info
canton auth status
# Output:
✓ Authenticated
  Token expires: 2026-03-05 15:30:00 (29m remaining)
  Endpoint: localhost:8080
  Client: alice_wallet

# Login (for localnet/production)
canton auth login --client alice_wallet

# Logout / clear credentials
canton auth logout

# Use different credential store
canton auth login --profile production
canton auth token --profile production
```

### Automatic Token Injection

All `canton` commands automatically inject the current token:
```bash
# These work without manual token management
canton call @token GetView --as alice      # Token auto-injected
canton call @token Split --as alice        # Token auto-injected
canton upload ./my.dar                     # Token auto-injected
```

---

## 8. `canton config` - Configuration Management

```bash
# View current config
canton config show

# Set defaults
canton config set default-party alice
canton config set default-participant localhost:6865
canton config set output-format json

# Project-level config (canton.yaml)
canton config init

# Output creates:
# canton.yaml
network:
  endpoint: localhost:6865
  json-api: localhost:7575

defaults:
  party: alice
  package: ore-bank-main

aliases:
  bank: "bank::1220789xyz..."
  alice: "alice::1220abc..."
```

### Environment Profiles

```bash
# Switch between environments
canton config use sandbox
canton config use localnet
canton config use testnet

# Custom profile
canton config add production \
  --endpoint canton.mycompany.com:6865 \
  --auth-endpoint auth.mycompany.com
```

---

## Complete Workflow Example

### Current State (Painful)

```bash
# 1. Start localnet (hunt for docker-compose, wait for 15 containers)
cd quickstart && docker-compose up -d
# Wait 2-3 minutes...

# 2. Hunt for credentials
source .env.alice_validator_wallet
# What's the client ID? Secret? Token endpoint?

# 3. Get OAuth token (construct curl manually)
TOKEN=$(curl -s -X POST http://localhost:8080/realms/... \
  -d "grant_type=client_credentials" \
  -d "client_id=$CLIENT_ID" \
  -d "client_secret=$CLIENT_SECRET" | jq -r '.access_token')

# 4. Find party ID (68 characters)
PARTY=$(curl -s http://localhost:7575/v2/parties \
  -H "Authorization: Bearer $TOKEN" | \
  jq -r '.result[] | select(.displayName=="Alice") | .party')

# 5. Upload DAR
curl -X POST http://localhost:7575/v2/packages \
  -H "Authorization: Bearer $TOKEN" \
  -F "darFile=@./dist/my-package.dar"

# 6. Find package ID (64 characters)
PKG=$(curl -s http://localhost:7575/v2/packages \
  -H "Authorization: Bearer $TOKEN" | jq -r '...')

# 7. Finally create a contract (30+ line JSON payload)
curl -X POST http://localhost:7575/v2/commands/submit-and-wait \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "commands": [{
      "createCommand": {
        "templateId": {
          "packageId": "'$PKG'",
          "moduleName": "Main",
          "entityName": "OreToken"
        },
        ...20 more lines...
      }
    }]
  }'
```

### Proposed Workflow (Pleasant)

```bash
# 1. Start localnet (one command, credentials auto-managed)
canton up --mode localnet --party alice --party bank

# 2. Upload package (alias auto-created)
canton upload ./dist/ore-bank-main.dar

# 3. Create contract (human-readable)
canton call --create OreToken \
  --args '{"issuer": "@bank", "owner": "@alice", "grams": 100.0}' \
  --act-as bank,alice

# 4. Read state (one-liner)
canton call @ore-token GetView --as alice

# 5. Exercise choice
canton call @ore-token Split --args '{"amount": 30.0}' --as alice
```

**Result: 5 simple commands vs 7 complex multi-step operations**

---

## Automatic Inference & Ergonomic Defaults

A core design principle: **the CLI should figure out what you mean whenever possible**.

### 1. Choice/Interface Package Inference

When you call a choice, the CLI automatically finds the right package:

```bash
# User types:
canton call @ore-token GetView --as alice

# CLI internally:
# 1. Fetch contract @ore-token → discovers template is Main:OreToken from package 8b4c...
# 2. Check if OreToken has GetView choice → No, it's from interface
# 3. Scan interfaces implemented by OreToken → finds Asset interface
# 4. Resolve Asset interface → package 7a3b... (ore-bank-interfaces)
# 5. Execute against correct package automatically

# Output (shows resolution for transparency):
{
  "assetOwner": "alice::1220abc...",
  "description": "Magic Ore",
  "quantity": 100.0
}
# [resolved: GetView via Asset interface from ore-bank-interfaces]
```

**Ambiguity handling:**
```bash
# If multiple interfaces provide GetView:
canton call @token GetView --as alice

# Error:
✗ Ambiguous choice: GetView
  Found in:
    1. Asset (ore-bank-interfaces)
    2. Viewable (some-other-package)
  Specify: canton call @token Asset.GetView --as alice
          or: canton call @token GetView --interface Asset
```

### 2. Auto-Aliasing on Upload

```bash
# Upload creates aliases automatically:
canton upload ./dist/ore-bank-main-0.0.1.dar

# Output:
✓ Uploaded: ore-bank-main-0.0.1.dar
  Package ID: 8b4c3d2e5f6a...
  Aliases created:
    @ore-bank-main      → 8b4c3d2e5f6a...
    @ore-bank-main-0.0.1 → 8b4c3d2e5f6a...
  Templates: OreToken, Split, Merge
```

**Multiple versions:**
```bash
canton upload ./dist/ore-bank-main-0.0.2.dar

# Output:
✓ Uploaded: ore-bank-main-0.0.2.dar
  Aliases created:
    @ore-bank-main       → 9c5d... (updated to latest)
    @ore-bank-main-0.0.2 → 9c5d... (new)
    @ore-bank-main-0.0.1 → 8b4c... (unchanged)
```

### 3. Auto-Aliasing on Contract Creation

```bash
canton call --create OreToken \
  --args '{"issuer": "@bank", "owner": "@alice", "grams": 100.0}' \
  --act-as bank,alice

# Output:
✓ Created: OreToken
  Contract ID: 00abc123def456...
  Aliases created:
    @OreToken-1          → 00abc123... (auto-incremented)
    @alice-OreToken-1    → 00abc123... (owner-prefixed)

  Use: canton call @OreToken-1 GetView --as alice
```

### 4. Party Alias Inference

```bash
# Party names are auto-aliased on allocation:
canton party new alice --hint "Alice the Trader"

# Creates aliases:
@alice              → alice::1220abc...
@Alice              → alice::1220abc... (case variations)
@alice-the-trader   → alice::1220abc... (from hint, slugified)
```

**In arguments, party resolution is automatic:**
```bash
canton call --create OreToken \
  --args '{"issuer": "bank", "owner": "alice", "grams": 100.0}'

# CLI resolves:
#   "bank" → bank::1220789xyz... (party alias lookup)
#   "alice" → alice::1220abc... (party alias lookup)

# Explicit @ prefix optional but clearer:
  --args '{"issuer": "@bank", "owner": "@alice", "grams": 100.0}'
```

### 5. Template Package Inference

```bash
# User doesn't know which package has OreToken:
canton call --create OreToken --args '{...}' --act-as bank,alice

# CLI:
# 1. Scan all uploaded packages for template "OreToken"
# 2. Found in: ore-bank-main (8b4c...)
# 3. Use that package automatically

# If ambiguous:
✗ Ambiguous template: OreToken
  Found in:
    1. ore-bank-main (8b4c...) - OreToken
    2. ore-bank-v2 (9d5e...) - OreToken
  Specify: canton call --create ore-bank-main:OreToken
```

### 6. Module Name Inference

```bash
# Full qualified name not required if unambiguous:
canton call --create OreToken     # Works (infers Main:OreToken)
canton call --create Main:OreToken  # Also works (explicit)

# If multiple modules have OreToken:
canton call --create OreToken
# Error:
✗ Ambiguous template: OreToken
  Found in modules:
    1. Main:OreToken (ore-bank-main)
    2. Legacy:OreToken (ore-bank-main)
  Specify: canton call --create Main:OreToken
```

### 7. Participant Inference

```bash
# Single participant mode (sandbox):
canton upload ./my.dar
# Auto-selects the only participant

# Multi-participant:
canton upload ./my.dar
# CLI checks: Is there only one participant? Use it.
# Multiple participants? Show helpful error:
✗ Multiple participants available
  Specify: canton upload ./my.dar --participant alice-participant
  Available: alice-participant, bob-participant
```

### 8. Domain Inference for Commands

```bash
# Canton auto-selects domain based on parties involved:
canton call @token Transfer --act-as alice,bob

# CLI:
# 1. alice is on domain "trading"
# 2. bob is on domain "trading"
# 3. Both on same domain → use "trading"
# 4. If cross-domain needed, use sync domain automatically

# If no common domain and no sync domain:
✗ Cannot execute: parties on different domains
  alice: trading
  bob: settlement
  No sync domain configured.
  Options:
    1. Connect both to a sync domain: canton domain connect alice settlement
    2. Specify domain: canton call @token Transfer --domain trading
```

### 9. Contract State Inference for Choices

```bash
# When exercising a choice, validate contract is active:
canton call @old-token Split --args '{...}' --as alice

# CLI:
# 1. Check if @old-token is active
# 2. If archived:
✗ Contract @old-token is archived
  Archived in transaction: tx-456789
  Archived at: 2026-03-05 12:45:00

  Active contracts of same template:
    @OreToken-2 (grams: 70.0)
    @OreToken-3 (grams: 50.0)
```

### 10. Offset/Pagination Auto-Management

```bash
# User never needs to manage offsets:
canton query contracts --as alice

# CLI internally:
# 1. Get current ledger end offset
# 2. Query ACS with proper offset
# 3. Handle pagination transparently
# 4. Return complete result set
```

### 11. Read-As Inference from Observers

```bash
# When calling a view, auto-determine read-as parties:
canton call @token GetView --as alice

# CLI:
# 1. alice is owner (signatory) → can read
# 2. Auto-populate read-as if needed based on contract observers
```

### 12. CommandId Auto-Generation

```bash
# User never specifies commandId:
canton call @token Split --args '{...}'

# CLI generates: cmd-{timestamp}-{random}
# Or uses idempotency key from config if set
```

### 13. Auto-Inference Summary Table

| What | Inferred From | Fallback |
|------|--------------|----------|
| Package for choice | Contract's template + interface hierarchy | Require `--package` |
| Package for create | Uploaded packages with matching template | Require `--package` |
| Module name | Unique template name across modules | Require `Module:Template` |
| Party ID | Party alias → display name → prefix | Require full ID |
| Contract ID | Alias → prefix | Require full ID |
| Participant | Single participant → auto-select | Require `--participant` |
| Domain | Common domain of parties | Require `--domain` |
| Read-as parties | Contract observers + signatories | Require `--read-as` |
| Offset | Current ledger end | N/A (always inferred) |
| CommandId | Auto-generated UUID | N/A (always inferred) |
| JWT token | Credential store | Require `canton auth login` |

### 14. Verbose Mode for Transparency

```bash
# See all inferences:
canton call @ore-token GetView --as alice --verbose

# Output:
[resolve] @ore-token → 00abc123def456789...
[resolve] alice → alice::1220abc123def456789...
[infer] Template: Main:OreToken from package ore-bank-main (8b4c...)
[infer] Choice GetView from interface Asset (ore-bank-interfaces, 7a3b...)
[infer] Participant: participant1 (single participant)
[infer] Offset: 00000000000000a5
[auth] Using token from profile: default (expires in 28m)
[exec] POST /v2/commands/submit-and-wait

{
  "assetOwner": "alice::1220abc...",
  "description": "Magic Ore",
  "quantity": 100.0
}
```

---

## Technical Implementation Notes

### Alias Resolution Order

1. Exact alias match (`@ore-token`)
2. Unambiguous prefix (`00abc` matches `00abc123...` if unique)
3. Display name match (`alice` matches `alice::1220abc...`)
4. Full ID (fallback)

### Multi-Party Authorization

```bash
# --act-as: Submit commands as these parties (signatories)
# --read-as: Read visibility only (observers)

canton call @proposal Accept \
  --act-as alice,bob \      # Both sign
  --read-as auditor          # Auditor can see but not sign
```

### Configuration Hierarchy

1. Command-line flags (highest priority)
2. Environment variables (`CANTON_*`)
3. Project config (`./canton.yaml`)
4. User config (`~/.canton/config.yaml`)
5. System defaults (lowest priority)

### Credential Storage

- Tokens stored in `~/.canton/credentials/`
- Auto-refresh before expiration
- Profile-based for multiple environments
- Never logged or displayed

---

## Survey Alignment

| Survey Request | Canton Forge Feature |
|---------------|---------------------|
| "Unified CLI Framework" (11+ mentions) | Full `canton` CLI suite |
| "Tenderly-like debugger" | `canton call --dry-run` |
| "Typed SDKs" | `canton query --format` with structured output |
| "Consolidated documentation" | `canton help`, `canton docs` |
| "Cargo-like package manager" | `canton upload`, `canton query packages` |
| "Pre-flight resource profiler" | `canton call --dry-run` |
| "Token/wallet management" | `canton wallet` command suite |

---

## Implementation Phases

### Phase 1: Core CLI (MVP)
- `canton up` (sandbox only)
- `canton call`
- `canton query parties/contracts`
- `canton upload`
- Alias system
- Prefix resolution

### Phase 2: Production Features
- `canton up --mode localnet`
- `canton auth` (OAuth2 management)
- `canton config` (profiles)
- Multi-domain support (`--domain`, `--sync-domain`)

### Phase 3: Splice & Token Support
- `canton up --with-splice` (SV, Scan, Wallet)
- `canton wallet` command suite
  - Balance/holdings queries
  - Amulet transfers (FOP)
  - CIP-56 token operations
  - Allocations (DVP)
  - UTXO merging

---

## Technical Feasibility Analysis

### The Interface Resolution Challenge

The documentation shows the pain: "GetView is defined on the Asset interface, not OreToken. We need the interface's package ID." This raises the question: **can the CLI actually determine interface implementations?**

#### What the API Provides

| Data | JSON API | gRPC | Canton Console |
|------|----------|------|----------------|
| Contract → Template | Yes (`templateId` field) | Yes | Yes |
| Template → Package | Yes (in `templateId`) | Yes | Yes |
| Package → Templates | No (just ID list) | No | Yes (`.packages.list()`) |
| Template → Interfaces | **No** | **No** | **No** |

**The gap**: The Ledger API doesn't expose "which interfaces does this template implement?"

#### Solutions (Ranked by Feasibility)

**1. DAR Metadata Index (Best UX, Medium Complexity)**

When uploading a DAR, parse it and store interface relationships:

```bash
canton upload ./ore-bank-main.dar
# Internally:
# 1. Parse Daml-LF in DAR file
# 2. Extract: OreToken implements Asset
# 3. Store in local metadata: ~/.canton/packages/ore-bank-main.json
#    {
#      "templates": {
#        "Main:OreToken": {
#          "implements": ["Asset:Asset"],
#          "choices": ["Split", "Merge"]
#        }
#      }
#    }

# Later:
canton call @ore-token GetView
# 1. Lookup template Main:OreToken
# 2. GetView not in choices → check implements
# 3. Found in Asset:Asset interface
# 4. Resolve Asset package ID from index
# 5. Execute
```

**Required:** Daml-LF parser in CLI. Canton's `dpm inspect-dar` already does this, so the parsing logic exists.

**2. Runtime Trial (Simplest, Acceptable UX)**

If metadata isn't available, try exercising against each candidate package:

```bash
canton call @ore-token GetView
# 1. GetView not a native OreToken choice
# 2. Scan uploaded packages for "GetView" choice on interfaces
# 3. Try each interface package that has GetView
# 4. First success → cache the mapping
# 5. Future calls use cached mapping

# With --verbose:
[resolve] Trying GetView on interface Asset from ore-bank-interfaces... success
[cache] Mapping: OreToken.GetView → Asset (ore-bank-interfaces)
```

**Downside:** First call is slower (tries multiple packages). Could fail ambiguously.

**3. Source Introspection (Most Complete, Most Complex)**

Include `.daml` source in the index (like TypeScript's `.d.ts` files):

```bash
canton upload ./main.dar --with-sources ./main/daml/

# Creates enhanced index with source info
# Can show actual choice signatures, doc comments, etc.
```

**4. Explicit Interface Specification (Fallback)**

When ambiguous, require explicit interface:

```bash
# If resolution fails:
✗ Cannot resolve GetView for OreToken
  Specify interface: canton call @ore-token Asset.GetView

# Or interactive:
canton call @ore-token GetView
? GetView found in multiple interfaces:
  1. Asset (ore-bank-interfaces)
  2. Viewable (other-package)
```

#### Recommended Implementation

**Phase 1 (MVP):**
- Build local metadata index on `canton upload`
- Use `dpm inspect-dar` output as bootstrap
- Store in `~/.canton/packages/`
- Fall back to "explicit interface required" if not indexed

**Phase 2:**
- Add runtime trial for non-indexed packages
- Cache successful resolutions

**Phase 3:**
- Full Daml-LF parser for complete introspection
- Source attachment support

### Other Technical Considerations

**1. Contract ID Stability**

Contract IDs are deterministic based on content + transaction. The CLI can safely create aliases because IDs don't change.

**2. Party ID Determinism**

On sandbox, `allocateParty "alice"` generates a deterministic ID. This is why re-running scripts fails. The `canton party ensure` command would:
1. Check if party exists (search by display name prefix)
2. Return existing if found
3. Allocate new if not

This is fully achievable via the existing `/v2/parties` endpoint.

**3. JWT Auto-Refresh**

OAuth2 tokens include `expires_in`. The CLI stores:
```json
{
  "access_token": "eyJ...",
  "expires_at": "2026-03-05T15:30:00Z",
  "refresh_token": "..." // if available
}
```

Before each request, check expiry and refresh if needed. Standard OAuth2 client behavior.

**4. Prefix Resolution Uniqueness**

```bash
canton call 00abc GetView
```

Implementation:
1. Query all contracts visible to acting party
2. Filter by prefix match
3. If exactly 1 match → use it
4. If 0 matches → error with suggestions
5. If >1 matches → error listing ambiguous IDs

This is achievable via the existing ACS query API.
