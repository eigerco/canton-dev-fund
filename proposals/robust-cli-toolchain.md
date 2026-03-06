# Next-Generation Developer Experience: An Integrated CLI Toolchain for Canton

### Authors:
### Status: Drafty McDraftFace
### Created:

## NARRATIVE / FRAMING NOTE

The core value proposition of this proposal is bridging the UX/DX gap between broader Web3 development and Canton (Daml). Modern developers expect single-line CLI interactions, instant local networks with pre-funded accounts, and fast native compilation. Canton's enterprise-grade architecture currently requires developers to manage multiple containers, explicit hexadecimal IDs, and multi-step API workflows. By pitching a comprehensive, developer-first toolchain, we sell an ecosystem-growth narrative: to attract a wider pool of Web3 builders, Canton needs modern, frictionless developer ergonomics.

## Abstract

(TODO: Insert Entity Name) requests a Development Fund grant to build an ergonomic CLI toolchain for Canton Network, directly addressing friction points identified in the [2026 Developer Survey](https://discuss.daml.com/t/canton-network-developer-experience-and-tooling-survey-analysis-2026/8412).

The survey reveals that 71% of Canton developers come from Ethereum backgrounds, where tools like Foundry and Hardhat provide single-command workflows. Meanwhile, 41% of respondents cite environment setup as their biggest hurdle, and "Local Development Frameworks" was rated the most critical tooling gap. Developers specifically requested Hardhat/Anchor-style unified CLIs (11+ mentions) and Tenderly-like transaction simulation.

This proposal delivers a CLI toolchain that brings Canton to parity with modern Web3 developer expectations:

- **Instant Networks**: One-command launch of sandbox, localnet, or multi-domain topologies with pre-allocated parties, auto-managed OAuth2 credentials, and optional Splice stack (Super Validator, Scan, Wallet).
- **One-Liner Interactions**: Query packages, parties, templates, and contracts. Create contracts and exercise choices. All with single commands replacing multi-step API workflows.
- **Intelligent Aliasing**: Human-readable names for parties, contracts, and packages replace 68-136 character hex IDs. Auto-aliasing on upload/create, prefix resolution, and interface-to-package discovery.
- **Token Support**: Manage Amulet (Canton Coin) and CIP-56 compliant tokens - balances, transfers, allocations. Integrates with Splice stack's Scan/Registry APIs on localnet.
- **Native Runtime** (exploratory): Rust-based Daml interpreter for dry-run simulation and reduced JVM startup overhead.
- **DPM Integration**: Complements existing DPM workflows - delegates builds to DPM, extends reach to runtime interaction.
- **AI Agent Ready**: Structured JSON output, field masking to limit response size, dry-run modes for safe exploration, and queryable schemas - designed for agentic workflows from the start.

By providing familiar, single-command workflows inspired by Foundry, this toolchain lets developers focus on building applications rather than managing infrastructure. Quick prototyping, easy scripting, seamless CI integration, and first-class AI agent support lower the barrier to entry and accelerate Canton ecosystem growth.

## Specification

### Objective

Deliver a unified CLI toolchain that streamlines Canton development, complementing the existing DPM stack. Core deliverables:

- **Ledger CLI** (`canton call`, `canton query`): Single-command contract interactions with intelligent ID resolution. Auto-resolves package IDs, interface hierarchies, and party aliases. Supports `--dry-run` for transaction simulation.
- **Network & Auth Manager** (`canton up`, `canton down`, `canton auth`): One-command launch of sandbox, localnet, or multi-domain topologies. Auto-provisions parties, manages OAuth2/JWT tokens transparently, supports `--with-splice` for Amulet/Scan/Wallet stack. Profile support for multiple environments.
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

Aliases are stored in persistent directory with project-level `.canton.yaml` overrides. Auto-aliasing occurs on `canton upload` (packages) and `canton call --create` (contracts). Automatic cleanup on fresh network setups.

**DPM Integration**: The CLI complements DPM rather than replacing it:
- `canton upload --build ./main/` delegates to `dpm build`, then uploads
- Uses `dpm inspect-dar` output for package metadata indexing
- Same DAR artifacts, same ledger APIs - enhanced interaction ergonomics

**Network Orchestration**: `canton up` wraps Docker Compose with presets (`--mode sandbox`, `--mode localnet`, `--topology multi-domain`). Credentials auto-provisioned and stored.

### Architectural alignment

This toolchain extends DPM's capabilities to runtime interaction without fragmenting the ecosystem:

```
Developer Workflow:
  dpm build        ->  canton upload  ->  canton call  ->  canton query
  (compile DAR)        (deploy)           (interact)       (inspect)
```

**Complementary, not competing**: DPM handles compilation, testing, and package management. This CLI handles network orchestration and ledger interaction. Same artifacts, same APIs, enhanced ergonomics.

**CLI next to RPC/gRPC/Console**: While Canton's existing API ecosystem is robust, efficient programmatic use requires language bindings, client libraries, or framework integration. A CLI enables quick prototyping without constructing complex JSON schemas - any shell script, CI pipeline, or development environment can invoke CLI commands directly. This lowers the barrier for exploration and rapid iteration.

**AI Agent Ready**: Designed for agentic workflows where LLMs invoke CLI commands programmatically:
- Structured `--output json` for all commands - machine-readable responses that map directly to schemas
- Field masking (`--fields`) to limit response size and protect context windows
- Pagination via NDJSON for streaming large result sets without buffering
- Dry-run modes (`--dry-run`) for safe exploration before mutations
- Queryable schemas (`canton schema <command>`) exposing parameters and types as JSON
- Input validation rejecting malformed parameters before API calls

**Strategic alignment**: With 71% of Canton developers coming from Ethereum and "Local Development Frameworks" rated critical, this toolchain directly addresses the ecosystem growth mandate. It makes Canton accessible to Web3 developers accustomed to Foundry/Hardhat workflows while preserving Canton's enterprise-grade architecture.

The CLI toolchain will be developed in Rust, chosen for its performance characteristics, memory safety guarantees, and excellent cross-platform support. Rust enables fast startup times critical for iterative development workflows, produces single-binary distributions without runtime dependencies, and aligns with modern CLI tooling trends (Foundry's forge/cast, ripgrep, etc.). The native Daml runtime prototype will also leverage Rust for reduced JVM startup overhead.


### Backward Compatibility

Fully backward compatible. This is purely an off-chain developer tooling proposal. It interacts with standard Canton APIs (gRPC and JSON) and outputs standard DAR files. Existing Canton deployments, ledgers, and standard dpm workflows are completely unaffected.

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
- Multi-party authorization via `--act-as` and `--read-as` flags

### Evaluation metric 2: Local Environment Manager
Focus: Zero-configuration local networks and sandbox environments.
Deliverables:
- 1-click startup of ephemeral Canton environments.
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
- Queries support filtering: `canton query contracts --template OreToken --as alice` lists contracts by party and template.
- Aliases auto-created on: `canton upload` (packages), `canton party new` (parties), and any `canton call` returning new contract IDs.
- Prefix resolution works: `canton call 00abc GetView` resolves to full contract ID if unique.
- Interface choices resolve to correct package ID (e.g., `GetView` on `Asset` interface).
- Output includes clearly labeled fields in JSON or table format.

### Evaluation metric 2: Local Environment Manager

- `canton up` launches sandbox with pre-allocated parties in a single command.
- `canton up --mode localnet` launches full OAuth2-enabled environment with auto-managed credentials.
- `canton party ensure alice` succeeds idempotently (no "party already exists" errors).
- Multi-domain topologies configurable via `--domain` and `--sync-domain` flags.

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

Upon each quarterly review period, [Team Name] will collaborate with Canton Foundation on:

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
