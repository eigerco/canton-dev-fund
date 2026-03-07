# Next-Generation Developer Experience: An Integrated CLI Toolchain for Canton

### Authors:
### Status: Drafty McDraftFace
### Created:


## TODO: remove - NARRATIVE / FRAMING NOTE

The core value proposition of this proposal is bridging the UX/DX gap between broader Web3 development and Canton (Daml). Modern developers expect single-line CLI interactions, instant local networks with pre-funded accounts, and fast native compilation. Canton's enterprise-grade architecture currently requires developers to manage multiple containers, explicit hexadecimal IDs, and multi-step API workflows. By pitching a comprehensive, developer-first toolchain, we sell an ecosystem-growth narrative: to attract a wider pool of Web3 builders, Canton needs modern, frictionless developer ergonomics.


## Abstract

(TODO: Insert Entity Name) requests a Development Fund grant to build an ergonomic CLI toolchain for Canton Network, directly addressing friction points identified in the [2026 Developer Survey](https://discuss.daml.com/t/canton-network-developer-experience-and-tooling-survey-analysis-2026/8412).

The survey reveals that 71% of Canton developers come from Ethereum backgrounds, where tools like Foundry and Hardhat provide single-command workflows. Meanwhile, 41% of respondents cite environment setup as their biggest hurdle, and "Local Development Frameworks" was rated the most critical tooling gap. Developers specifically requested familiar unified CLIs (11+ mentions) and Tenderly-like transaction simulation.

This proposal delivers a CLI toolchain that brings Canton to parity with modern Web3 developer expectations. By providing familiar, powerful workflows inspired by Foundry, this toolchain lets developers focus on building applications rather than managing infrastructure. Quick prototyping, easy scripting, seamless CI integration, and first-class AI agent support lower the barrier to entry and accelerate Canton ecosystem growth. Even though APIs exposed with JSON-RPC, gRPC, console or even Postgres form a comprehensive and fully functional stack, they require bindings or other scaffolding on top of them to be used comfortably and ergonomically. This is where simple CLI tools shine: they require no additional setup and prioritize convenience over explicitness. Interaction is as simple as a command call, and most, if not all, environments can execute shell commands with no extra libraries or frameworks. Moreover, an executed command will look the same regardless of the language or environment it was run from. This makes them a perfect tool for testing, automated workflows, scripting and experimentation. That is also the main goal of the proposal. While for production use developers will likely choose gRPC or other well-established, mature protocols, having a way to inspect the state or make an interaction with a simple shell command is priceless, aiding debugging, prototyping and learning.

The main focus of the proposal is to complement existing tools in the ecosystem, like DPM, and provide developers with:

- **Instant Networks**: One-command launch of localnet, or multi-domain topologies with pre-allocated parties, auto-managed OAuth2 credentials, and the global domain Splice stack (Super Validator, Scan, Wallet).
- **One-Liner Interactions**: Query packages, parties, templates, and contracts. Create new contracts and exercise choices. All with simple one-line commands replacing multi-step API workflows.
- **Intelligent Aliasing**: Human-readable names for parties, contracts, and packages replace 68-136 character hex IDs. Auto-aliasing whenever a new package, party, or contract is created. Automatic resolving of IDs based on prefix. Choice template and interface deduction if possible. Errors listing all possibilities on ambiguity.
- **Token Support**: Manage Amulet (Canton Coin) and CIP-56 compliant tokens - balances, transfers and allocations. Integrate with Splice stack's Scan/Registry APIs on localnet.
- **Native Runtime** (exploratory): Rust-based Daml interpreter for dry-running, simulation and debugging. Speed is an additional benefit due to elimination of JVM startup time, which becomes even more important when designing feedback loops for AI workflows.
- **DPM Integration**: Complements existing DPM workflows - delegates builds to DPM, uses it to inspect DARs and validate packages.
- **AI Agent Ready**: Structured JSON output, field masking to limit response size, dry-run modes for safe exploration, and queryable schemas - designed for agentic workflows from the start.


## Specification

### Objective

Deliver a unified CLI toolchain that streamlines Canton development, complementing the existing DPM stack. Core deliverables:

- **Ledger CLI** (`canton call`, `canton query`): Single-command contract interactions with intelligent ID resolution. Auto-resolves package IDs, interface hierarchies, and party aliases. Supports `--dry-run` for transaction simulation.
- **Network & Auth Manager** (`canton up`, `canton down`, `canton auth`): One-command launch of localnet, or multi-domain topologies. Auto-provisions parties, manages OAuth2/JWT tokens transparently, supports `--with-splice` for Amulet/Scan/Wallet stack. Profile support for multiple environments.
- **Alias System**: Persistent human-readable aliases for contracts, parties, and packages. Auto-aliasing on upload/create, prefix resolution (`00abc` -> full ID), interface-to-package discovery.
- **Token Management** (`canton wallet`): CIP-56 compliant token operations - balance queries, transfers (FOP), allocations (DVP), UTXO merging. Integrates with Scan/Registry APIs.
- **Party Management** (`canton party`): Create parties and manage their rights. Get-or-create semantics (`canton party ensure alice`) solving collision issues.
- **Native Runtime** (exploratory): Prototype Rust-based Daml interpreter for `--dry-run` simulation and reduced JVM startup overhead. Enables Tenderly-like transaction tracing locally.

### Implementation mechanics

The toolchain acts as an intelligent layer between developers and Canton's Ledger/Scan APIs.

**Alias & Resolution System**: The core differentiator. When a user runs `canton call @ore-token GetView --as alice`:
1. Resolve `alice` -> `alice::1220abc...` (party alias)
2. Resolve `@ore-token` -> `00abc123...` (contract alias or prefix match)
3. Discover `GetView` is on `Asset` interface, resolve interface package ID
4. Execute via console, grpc or JSON api, return labeled output

Aliases exist per network and their lifetimes are tied. Auto-aliasing occurs on `canton upload` (packages), localnet up / party allocation (parties) and `canton call [--create]` (contracts).

Prefix-based resolution allows less typing whenever there is a single valid answer, the same way git allows using shorthand form of SHA-1 hashes.

Choice resolution should happen whenever an unambiguous answer exists. Template choices should be prioritized over interfaces, but choices from interfaces should be resolved whenever possible. This can be achieved e.g. by inspecting the DALF artifacts of uploaded packages, indexed whenever `canton upload` is called.

**DPM Integration**: The CLI complements DPM rather than replacing it:
- `canton upload --build ./main/` delegates to `dpm build`, then uploads
- Uses `dpm inspect-dar` output for package metadata indexing
- Same DAR artifacts, same ledger APIs - enhanced interaction ergonomics

**Network Orchestration**: `canton up` wraps Docker Compose with presets (`--with-oauth`, `--topology multi-domain`). Credentials auto-provisioned and stored.

### Examples

This section illustrates the vision and capabilities of the proposed toolkit, demonstrating the benefits and ease of use a native shell-driven approach can achieve.

**Network Management:**
```bash
# quick localnet with single domain and automatic participant assignment
$ canton up
✓ Canton localnet started on localhost:6865
✓ JSON API on localhost:7575
✓ Parties allocated:
  alice → alice::1220abc... (alias: alice)
✓ Auth token stored in ~/.canton/networks/bold-isaac/credentials
# named localnet with single domain and automatic participant assignment
$ canton up --name mynetwork
# localnet with single domain and specific participants pre-allocated
$ canton up --party alice --party bob --party bank
...
✓ Parties allocated:
  alice → alice::1220abc... (alias: alice)
  bob   → bob::1220def...   (alias: bob)
  bank  → bank::1220789...  (alias: bank)
# shut down (if only one network is up or some is set as default)
$ canton down

# localnet with OAuth, global domain, amulet/scan/wallet
# and specific participants pre-allocated
$ canton up --with-splice --with-oauth --party alice --sv bob-sv1
# localnet with global domain, two user domains and some participants pre-allocated
$ canton up \
  --name multi-domain-1 \
  --with-splice \
  --domain trading:alice,bob \
  --domain settlement:alice,bank
# targeted shutdown
$ canton down multi-domain-1
```

**Configuration management**:
```bash
# custom aliases for anything, automatic ID resolving by prefix (if unambiguous)
$ canton alias set alice-asset 00def456
# set default participant to interact with
$ canton config set default-participant localhost:6865
# set default party to act as
$ canton config set default-party alice
# set default output format for commands
$ canton config set output-format json
```

**Package upload:**
```bash
# upload a package from the current directory
$ canton upload
✓ Uploaded: ore-bank-main-0.0.1.dar
  Package ID: 8b4c3d2e5f6a... (alias: @ore-bank-main)
  Templates: OreToken
# upload specific dars to given participant
$ canton upload .daml/dist/*.dar --participant alice
```

**Ledger Interaction:**
```bash
# exercise the choice on the contract (or create it) with automatic authentication
# and participant / domain deduction (if possible, e.g. by party)
$ canton call [--create] MyTemplate arg1 arg2 ... --as myparty
# create a contract from the template acting on behalf of two parties
$ canton call --create OreToken @bank @alice 100.00 --as bank,alice
✓ Contract created: OreToken
  Contract: 00abc123... (alias: @ore-token)
  Transaction ID: tx-123456
# exercise a choice on a contract
$ canton call @ore-token Split 30.0 --as alice
✓ Choice exercised: Split
  Contract: 00abc123...
  Result: (newCid1, newCid2)
    newCid1: 00abc789... (alias: @ore-token-1)
    newCid2: 00def456... (alias: @ore-token-2)
  Transaction ID: tx-654321
# exercise a non-consuming choice on the contract and show output as JSON
$ canton call @ore-token-1 GetView --as alice --json
{
  "assetOwner": "alice::1220abc...",
  "description": "Magic Ore",
  "quantity": 70.0
}
# exercise a choice with an observer
$ canton call @proposal Accept --as alice,bob --read-as auditor
```

**Queries:**
```bash
# query all contracts of the given template owned by alice
$ canton query contracts --template OreToken --as alice --json
[
  { "contractId": "00abc...", "grams": 100.0, "owner": "alice::1220abc..." },
  { "contractId": "00def...", "grams": 75.0, "owner": "alice::1220abc..." }
]
# query last 10 alice transactions
$ canton query transactions --as alice --limit 10
# query all parties in the network
$ canton query parties
# query all uploaded packages
$ canton query packages
```

**Party Management:**
```bash
# allocate a new party with the hint
$ canton party new alice --hint "Alice the Trader"
✓ Party created: "alice::1220abc..."
  Aliases: @alice @alice-the-trader # automatic and from slugified hint
# get or create party, infallible
$ canton party ensure alice
```

**Auth Management:**
```bash
# get current token (auto-refreshes if expired)
$ canton auth token
# show credential info
$ canton auth status
✓ Authenticated
  Token expires: 2026-03-05 15:30:00 (29m remaining)
  Endpoint: localhost:8080
  Client: alice_wallet
# login (for localnet/production)
$ canton auth login --client alice
# logout / clear credentials
$ canton auth logout
```

**Token & Wallet:**
```bash
# amulet/CC balance
$ canton wallet balance --as alice
ASSET          BALANCE      LOCKED    AVAILABLE
Canton Coin    1,250.50 CC  100.00    1,150.50
# Detailed holdings (UTXO view)
$ canton wallet balance --as alice --detailed
CONTRACT_ID     AMOUNT      LOCKED_UNTIL    CREATED
@holding-1      500.00 CC   -               2026-03-05 10:00
@holding-2      450.50 CC   -               2026-03-05 11:30
@holding-3      300.00 CC   2026-03-10      2026-03-05 12:00
# faucet
$ canton wallet tap --as alice
# amulet transfer
$ canton wallet transfer --to @bob --amount 50 --as alice
# all holdings, amulet + cip-56
$ canton wallet holdings --token "Gold Token" --as alice
# utxo merging
$ canton wallet merge --token "Canton Coin" --as alice
```

**Verbose mode:**
```bash
$ canton call @ore-token GetView --as alice --verbose
[resolve] @ore-token → 00abc123def456789...
[resolve] alice → alice::1220abc123def456789...
[infer] Template: Main:OreToken from package ore-bank-main (8b4c...)
[infer] Choice GetView from interface Asset (ore-bank-interfaces, 7a3b...)
[infer] Participant: participant1 (single participant)
[infer] Offset: 00000000000000a5
[auth] Using token from profile: default (expires in 28m)
[exec] POST /v2/commands/submit-and-wait
```

**Dry-Run & AI Agent Support:**
```bash
# dry-running a consuming choice by creating a temporary environment with current state, "cloning"
canton call @token Split 30.0 --dry-run --as alice
# limiting the output fields
canton query contracts --as alice --output json --fields contractId,payload.grams
# getting tool schema, help but for agents
canton schema call # JSON schema for command parameters
```

**For comparison, exercising GetView via JSON API today:**
```bash
# 1. Auth
$ source .env.alice_validator_wallet
$ TOKEN=$(curl -s -X POST "http://keycloak.localhost:8082/realms/AppProvider/protocol/openid-connect/token" \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -d "client_id=$CLIENT_ID" \
    -d "client_secret=$CLIENT_SECRET" \
    -d "grant_type=client_credentials" \
    -d "scope=openid" | jq -r .access_token)

# 2. Get party ID (68 chars)
$ curl -s http://localhost:7575/v2/parties \
  -H "Authorization: Bearer $TOKEN"
{
  "partyDetails": [
    {"party": "Alice-9b3970be::1220..."},
    {"party": "OreBank-d4d95138::1220..."}
  ]
}

# 3. Query alice contract
$ curl -X POST http://localhost:7575/v2/state/active-contracts \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"filter":{"filtersByParty":{"alice::1220abc...":{"templateFilters":[...]}}}}'
{
  "contractEntry": {
    "createdEvent": {
      "contractId": "0038aa6ee4aee9838897aba6c1685fa10b19a4c8937c2f7ec5ad0ad447d691b0b1ca12122060df4f4a6d33a5886739c128381d8d6bf79b73eccaf6a5499cfc86c248a4ee86",
      "templateId": "f6f30cd775711b761f6f06f3b7e0df9c9ab033f9c82f4c06184fab988059aa7c:Main:OreToken",
      "createArguments": {
        "issuer": "OreBank-d4d95138::...",
        "owner": "Alice-9b3970be::...",
        "grams": "100.0"
      }
    }
  }
}

# 4. Get package ID (GetView comes from interface which is different than contract's template ID)
# information about which package corresponds to a given ID is only available in canton-console
$ curl -s http://localhost:7575/v2/packages \
  -H "Authorization: Bearer $TOKEN"
{
  "packageIds": [
    "f6f30cd775711b761f6f06f3b7e0df9c9ab033f9c82f4c06184fab988059aa7c",
    "e503a7f44411aea3e430b355ea395525e147bd3b07bfae52aabf4b64d6049447",
    ...
  ]
}

# 5. Exercise
$ curl -s -X POST http://localhost:7575/v2/commands/submit-and-wait-for-transaction \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "commands": {
      "userId": "json-user",
      "commandId": "getview-001",
      "actAs": ["Alice-9b3970be::1220b2f08ab0588eb8591ca46eec6ba9a54013134633098b355e96a64787db27ae9a"],
      "commands": [{
        "ExerciseCommand": {
          "templateId": "e503a7f44411aea3e430b355ea395525e147bd3b07bfae52aabf4b64d6049447:Asset:Asset",
          "contractId": "0038aa6ee4aee9838897aba6c1685fa10b19a4c8937c2f7ec5ad0ad447d691b0b1ca12122060df4f4a6d33a5886739c128381d8d6bf79b73eccaf6a5499cfc86c248a4ee86",
          "choice": "GetView",
          "choiceArgument": {}
        }
      }]
    },
    "transactionFormat": {
      "eventFormat": {
        "filtersByParty": {
          "Alice-9b3970be::1220b2f08ab0588eb8591ca46eec6ba9a54013134633098b355e96a64787db27ae9a": {}
        },
        "verbose": true
      },
      "transactionShape": "TRANSACTION_SHAPE_LEDGER_EFFECTS"
    }
  }'
{
  "transaction": {
    "events": [{
      "ExercisedEvent": {
        "choice": "GetView",
        "exerciseResult": {
          "assetOwner": "Alice-9b3970be::1220b2f08ab0588eb8591ca46eec6ba9a54013134633098b355e96a64787db27ae9a",
          "description": "Magic Ore",
          "quantity": "100.0000000000"
        }
      }
    }]
  }
}
```

This workflow is not much different when using gRPC or canton console, except that with console it is possible to get names of the packages together with their IDs, saving some trial-and-error time.

The proposed CLI equivalent:

```bash
# 0.0. Auth, automatically handled within a network
$ canton up --party alice --party bank

# 0.1. Deployment, not even included above
$ canton upload --build
$ canton call --create OreToken @bank @alice 100.00

# 1. Exercise the choice right away, using automatic aliases and party defaults
$ canton call @ore-token-alice GetView --json
{
  "assetOwner": "Alice-9b3970be::1220b2f08ab0588eb8591ca46eec6ba9a54013134633098b355e96a64787db27ae9a",
  "description": "Magic Ore",
  "quantity": "100.0000000000"
}
```

### Architectural alignment

This toolchain extends DPM's capabilities to runtime interaction without fragmenting the ecosystem:

```
Developer Workflow:
  dpm build        ->  canton upload  ->  canton call  ->  canton query
  (compile DAR)        (deploy)           (interact)       (inspect)
```

**Complementary, not competing**: DPM handles compilation, testing, and package management. This CLI handles network orchestration and ledger interaction. Same artifacts, same APIs, enhanced ergonomics. Proposed solution aims to fill the gaps rather than partitioning the ecosystem.

**CLI next to RPC/gRPC/Console**: While Canton's existing API ecosystem is robust, efficient programmatic use requires language bindings, client libraries, or framework integration. A CLI enables quick prototyping without constructing complex JSON schemas - any shell script, CI pipeline, or development environment can invoke CLI commands directly. This lowers the barrier for exploration and rapid iteration.

**AI Agent Ready**: Designed for agentic workflows where LLMs invoke CLI commands programmatically:
- Structured `--output json` for all commands - machine-readable responses that map directly to schemas
- Field masking (`--fields`) to limit response size and protect context windows
- Pagination via NDJSON for streaming large result sets without buffering
- Dry-run modes (`--dry-run`) for safe exploration before mutations
- Queryable schemas (`canton schema <command>`) exposing parameters and types as JSON
- Input validation rejecting malformed parameters before API calls

**Strategic alignment**: With 71% of Canton developers coming from Ethereum and "Local Development Frameworks" rated critical, this toolchain directly addresses the ecosystem growth mandate. It makes Canton accessible to Web3 developers accustomed to Foundry/Hardhat workflows while preserving Canton's enterprise-grade architecture.

**Technology tailored to the problem**: The CLI toolchain will be developed in Rust, a current go-to language for shell tools. Chosen for its performance characteristics, static linkage, memory safety guarantees, and excellent cross-platform support. Rust enables fast startup times critical for iterative development workflows, produces single-binary distributions without runtime dependencies at native speed, and aligns with modern CLI tooling trends (Foundry's forge/cast, ripgrep, etc.). The native Daml runtime prototype will also leverage Rust for reduced JVM startup overhead.

### Backward Compatibility

Fully backward compatible. This is purely an off-chain developer tooling proposal. It interacts with standard Canton APIs (gRPC and JSON) and provides an ergonomic layer over them. Existing Canton deployments, ledgers, and standard DPM workflows are completely unaffected.


## Milestones and deliverables

### (TODO: Open Question)
The template uses a recurring quarterly evaluation framework. Should we structure this as a continuous development grant (like the Zenith one) or a fixed-scope milestone grant? Drafted below as a continuous grant based on quarterly metrics.

### Evaluation metric 1: Streamlined Ledger CLI

Focus: Single-command contract interactions with intelligent resolution.
Deliverables:
- `canton call` for choice execution and contract creation
- `canton query` for contracts, parties, packages, transactions
- **Alias & Resolution System**:
  - Persistent aliases for contracts (`@ore-token`), parties (`alice`), packages (`@ore-bank-main`)
  - Auto-aliasing on `canton upload` and `canton call --create` or from choices returning new contracts.
  - Prefix resolution (`00abc` -> full contract ID if unique)
  - Interface-to-package discovery (if feasible, by inspecting uploaded DALF files)
  - Meaningful errors when resolution would be ambiguous
- `canton upload` with optional `--build` delegation to DPM
- `canton config` for further optimizing / tailoring the toolkit
- Multi-party authorization via `--as` and `--read-as` flags

### Evaluation metric 2: Local Environment Manager

Focus: Zero-configuration local network environments.
Deliverables:
- One-click startup of ephemeral Canton environments with `canton up / down / auth`, including OAuth
- Automatic provisioning of pre-funded, pre-allocated development parties.
- Support for distinct multi-party, multi-domain topology presets without manual configuration file editing.

### Evaluation metric 3: Token & Wallet Support

Focus: CIP-56 compliant token operations and Splice stack integration.
Deliverables:
- `canton wallet balance` and `canton wallet holdings` for Amulet (CC) and CIP-56 tokens
- `canton wallet transfer` for FOP (Free of Payment) transfers
- `canton wallet allocate` for DVP (Delivery vs Payment) settlements
- `canton wallet merge` for UTXO consolidation
- `canton up --with-splice` for deploying Splice stack (Super Validator, Scan, Wallet)
- Integration with Scan/Registry APIs for token metadata and transfer instructions

### Evaluation metric 4: Native Runtime, Debugging & AI Agent Support

Focus: Speed, developer observability, and first-class support for agentic workflows.
Deliverables:
- Prototype/Release of a native (Rust-based) Daml test runner to significantly reduce base startup times.
- Implementation of highly readable, modern error messaging.
- "Dev-mode" execution tracing: exposing comprehensive call stacks and temporarily bypassing privacy-features locally to allow developers to rapidly diagnose failed transactions.
- `--output json` flag on all commands for structured, machine-readable responses.
- `--fields` flag for field masking to limit response size and protect context windows.
- `--dry-run` on mutation commands for safe exploration before commits.
- `canton schema <command>` exposing parameters, types, and constraints as queryable JSON.
- Input validation with clear error messages for malformed parameters.


## Acceptance criteria

### Evaluation metric 1: Streamlined Ledger CLI

- Developer can query and exercise choices using single CLI commands with human-readable aliases.
- Authorization is properly handled based on the `--as` and `--read-as` params.
- Queries support filtering: `canton query contracts --template OreToken --as alice` lists contracts by party and template.
- Aliases auto-created on: `canton upload` (packages), `canton party new` (parties), and any `canton call` returning new contract IDs.
- Prefix resolution works: `canton call 00abc GetView` resolves to full contract ID if unique.
- Interface choices resolve to correct package ID (e.g., `GetView` on `Asset` interface).
- Ability to choose default networks, participants and parties using `canton config`.

### Evaluation metric 2: Local Environment Manager

- `canton up` launches localnet with pre-allocated parties in a single command.
- `canton up --with-oauth` launches full OAuth2-enabled environment with auto-managed credentials.
- `canton party ensure alice` succeeds idempotently (no "party already exists" errors).
- Multi-domain topologies configurable via `--domain` flags.

### Evaluation metric 3: Token & Wallet Support

- `canton wallet balance` returns token holdings for a party.
- `canton wallet transfer` executes FOP transfers via Scan/Registry APIs.
- `canton up --with-splice` launches functional Splice stack (SV, Scan, Wallet).
- Operations work with both Amulet (CC) and custom CIP-56 tokens.

### Evaluation metric 4: Native Runtime, Debugging & AI Agent Support

- Benchmarked reduction in test suite startup and execution time.
- `canton call --dry-run` simulates transaction without committing.
- Developer can view execution trace of failed transactions on local network.
- All commands support `--output json` returning structured, parseable responses.
- `--fields` limits output to specified fields (e.g., `--fields contractId,payload.grams`).
- `canton schema query` returns JSON describing available parameters and types.
- AI agent can invoke commands and parse responses without custom integration code.


## Funding

Monthly funding request: (TODO: Determine budget. e.g., X,XXX,XXX CC ($XXX,XXX at 0.15 CC/USD))

- The exact CC amount is subject to volatility adjustment as described below.

Grant payout: takes place on a quarterly basis.

- TODO: Need to calculate team size, and duration.

Volatility handling: The monthly grants are denominated in Canton Coin (CC) using an initial reference price of 0.15 USD per CC. For every month, the 30-day moving average (MA) of the CC/USD price will be evaluated.

- If the 30-day MA remains within the band of 33.3% (0.10–0.22.5 USD for the initial reference price of 0.15), no adjustment will be made.
- If the 30-day MA leaves the above band, the milestone tranche will be recalculated to preserve its value using the applicable 30-day moving average price.


## Co-Marketing

Upon each quarterly review period, (`TODO:` Team Name) will collaborate with Canton Foundation on:

- Developer-focused technical blog posts demonstrating the new "web3-native" Canton workflows.
- Video tutorials showcasing the CLI interacting with the ledger (e.g., "From Zero to Daml Deployment in 60 Seconds").


## Motivation

The [2026 Developer Survey](https://discuss.daml.com/t/canton-network-developer-experience-and-tooling-survey-analysis-2026/8412) identified concrete friction points:

- **ID Cognitive Overload**: Developers must manually track 68-char party IDs, 136-char contract IDs, and 64-char package IDs. The survey specifically noted package ID discovery as "opaque".
- **Multi-Step API Workflows**: A simple contract interaction requires multiple API calls - fetch party, query ACS, resolve package ID, construct JSON payload, submit transaction.
- **Localnet Complexity**: Running a production-like environment involves 15+ containers, hunting for OAuth2 secrets across env files, and manual credential management.
- **Token Transfer Friction**: CIP-56 token operations require coordinating Scan APIs, Registry APIs, and Ledger APIs - typically 4+ calls for a single transfer.
- **JVM Startup Overhead**: The JVM-based runtime introduces startup latency that slows iterative development cycles.

With 71% of developers coming from Ethereum (where Foundry/Hardhat provide single-command workflows), Canton needs equivalent ergonomics to capture this developer base. This toolchain removes these hurdles while preserving Canton's enterprise-grade architecture.


## Rationale

Our team has deep, hands-on experience in the Web3 developer tooling space and has conducted extensive technical audits of the current Canton development lifecycle. We understand exactly what modern protocol developers expect, and we have directly mapped Canton's existing workflows to proven DX solutions in the broader software ecosystem.

By bundling CLI interaction, localnet orchestration, and performance enhancements into a single, cohesive project, we can allocate a dedicated, long-term engineering squad to elevate Canton's tooling and provide a truly world-class developer experience.
