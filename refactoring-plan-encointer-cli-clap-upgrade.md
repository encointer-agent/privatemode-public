# Refactoring Plan: Encointer CLI — Clap Upgrade & Subcommand Restructuring

## Current State

- **Clap version**: Uses `clap` + `clap_nested` (pre-3.x builder pattern)
- **CLI style**: Flat list of ~80+ subcommands registered via `.add_cmd()` chains on a `Commander`
- **Argument handling**: Custom `EncointerArgs` / `EncointerArgsExtractor` traits wrapping `ArgMatches`
- **Commands**: 14 command modules in `src/commands/`, each exporting async handler functions
- **Shared state**: Node URL, port, CID extracted from `ArgMatches` via trait methods

## Target State (modeled after Integritee CLI pattern)

- **Clap version**: `4.5.x` with `derive` feature
- **CLI style**: Derive-based `#[derive(Parser)]` structs and `#[derive(Subcommand)]` enums
- **Argument handling**: Typed struct fields with clap attributes replace string-keyed `ArgMatches`
- **Commands**: Grouped into subcommand categories with two-level dispatch
- **Shared state**: Global `Cli` struct passed by reference to all command handlers

---

## Step-by-Step Plan

### Phase 1: Project Setup & Dependencies

#### 1. Update `Cargo.toml`
- Replace `clap` and `clap_nested` dependencies with `clap = { version = "4.5", features = ["derive"] }`
- Remove `clap_nested` entirely
- Add `thiserror` for structured error types (if not present)

#### 2. Define result/error types (`src/error.rs`)
- Create `CliError` enum with `#[derive(Debug, thiserror::Error)]`
- Variants: `Extrinsic`, `ApiConnection`, `KeystoreError`, `ParseError`, `IoError`, etc.
- Define `pub type CliResult = Result<CliResultOk, CliError>;`
- Define `CliResultOk` enum if commands need to return typed data, or use `()` if all output is printed directly

### Phase 2: Define the Top-Level CLI Struct & Command Groups

#### 3. Create main `Cli` struct (`src/lib.rs` or `src/cli.rs`)
```rust
#[derive(Parser)]
#[command(name = "encointer-client-notee")]
#[command(version, author, about)]
pub struct Cli {
    /// Substrate node WebSocket URL
    #[arg(short = 'u', long, default_value = "ws://127.0.0.1")]
    pub node_url: String,

    /// Substrate node WebSocket port
    #[arg(short = 'p', long, default_value = "9944")]
    pub node_port: String,

    /// Community identifier (optional, required by most community commands)
    #[arg(long)]
    pub cid: Option<String>,

    #[command(subcommand)]
    pub command: Commands,
}
```

#### 4. Define `Commands` enum (`src/commands/mod.rs`)

Group the ~80 flat subcommands into logical categories:

```rust
#[derive(Subcommand)]
pub enum Commands {
    // Flatten keystore commands to top level (frequently used)
    #[command(flatten)]
    Account(AccountCommand),

    /// Community management
    #[command(subcommand)]
    Community(CommunityCommand),

    /// Ceremony participation and management
    #[command(subcommand)]
    Ceremony(CeremonyCommand),

    /// Scheduler phase queries and control
    #[command(subcommand)]
    Scheduler(SchedulerCommand),

    /// Bazaar marketplace
    #[command(subcommand)]
    Bazaar(BazaarCommand),

    /// Faucet management
    #[command(subcommand)]
    Faucet(FaucetCommand),

    /// Reputation rings (ring-VRF personhood proofs)
    #[command(subcommand)]
    Reputation(ReputationCommand),

    /// Democracy / governance proposals
    #[command(subcommand)]
    Democracy(DemocracyCommand),

    /// Offline payment (ZK proofs)
    #[command(subcommand)]
    OfflinePayment(OfflinePaymentCommand),

    /// Core chain operations (balance, transfer, etc.)
    #[command(flatten)]
    Chain(ChainCommand),
}
```

**Grouping rationale:**

| Group | Current commands | UX |
|-------|-----------------|-----|
| `Account` (flattened) | `new-account`, `list-accounts`, `export-secret` | `cli new-account` |
| `Chain` (flattened) | `balance`, `transfer`, `transfer-all`, `listen`, `print-metadata` | `cli balance ...` |
| `Community` | `new-community`, `add-locations`, `remove-location`, `list-communities`, `list-locations` | `cli community list` |
| `Ceremony` | 16 ceremony commands | `cli ceremony register ...` |
| `Scheduler` | `get-phase`, `get-cindex`, `next-phase` | `cli scheduler get-phase` |
| `Bazaar` | 6 bazaar commands | `cli bazaar list-businesses` |
| `Faucet` | 6 faucet commands | `cli faucet create ...` |
| `Reputation` | 6 ring commands + 2 commitment commands | `cli reputation get-rings` |
| `Democracy` | 8 governance commands | `cli democracy vote ...` |
| `OfflinePayment` | 13 offline payment commands | `cli offline-payment generate ...` |

The `#[command(flatten)]` on `Account` and `Chain` keeps the most common operations at the top level (no extra subcommand word needed).

### Phase 3: Define Per-Group Subcommand Enums

#### 5. For each command group, create a subcommand enum in its module

Example for `CeremonyCommand` (`src/commands/ceremonies.rs`):

```rust
#[derive(Subcommand)]
pub enum CeremonyCommand {
    /// Register as participant for next ceremony
    Register {
        #[arg(long)]
        account: String,
        // ...
    },
    /// Upgrade registration (e.g., from newbie to reputable)
    UpgradeRegistration { /* ... */ },
    /// Unregister from ceremony
    Unregister { /* ... */ },
    /// Endorse newcomers (bootstrapper only)
    EndorseNewcomers { /* ... */ },
    /// Attest attendees after meetup
    AttestAttendees { /* ... */ },
    /// Claim ceremony reward
    ClaimReward { /* ... */ },
    /// List registered participants
    ListParticipants,
    /// List meetup assignments
    ListMeetups,
    /// Print ceremony statistics
    Stats,
    // ... etc
}
```

Each variant's fields replace the old ad-hoc `.arg()` definitions with typed, documented struct fields.

#### 6. Implement `run()` on each command group

```rust
impl CeremonyCommand {
    pub async fn run(&self, cli: &Cli) -> CliResult {
        match self {
            Self::Register { account, .. } => { /* logic */ },
            Self::ListParticipants => { /* logic */ },
            // ...
        }
    }
}
```

### Phase 4: Refactor Shared Utilities

#### 7. Replace `EncointerArgs` / `EncointerArgsExtractor` traits

- These traits added arguments to `App` and extracted values from `ArgMatches` by string key
- With derive-based clap, struct fields handle both definition and extraction
- **Delete** `src/cli_args.rs` entirely
- Move any shared argument groups into reusable clap `Args` structs:

```rust
#[derive(Args)]
pub struct AccountArgs {
    /// Account address or name from keystore
    #[arg(long)]
    pub account: String,
}

#[derive(Args)]
pub struct SeedArgs {
    /// Secret seed for signing
    #[arg(long)]
    pub seed: Option<String>,
}
```

- Embed these as fields in command variants that need them (composition over string keys)

#### 8. Refactor `utils.rs`

- Change `get_chain_api()` to accept `&Cli` instead of `&ArgMatches`
- Change `xt()`, `sudo_call()`, `batch_call()` to accept typed parameters
- Update `ensure_payment()` and send helpers similarly
- The `CallWrapping` enum and governance helpers remain largely the same

### Phase 5: Migrate Command Implementations

#### 9. Migrate each command module (one module at a time, in this order to manage complexity)

| Order | Module | Complexity | Notes |
|-------|--------|-----------|-------|
| 1 | `keystore.rs` → `account.rs` | Low (3 cmds) | Good warm-up, no chain calls |
| 2 | `frame.rs` → merge into `chain.rs` | Low (1 cmd) | Treasury query |
| 3 | `encointer_scheduler.rs` → `scheduler.rs` | Low (3 cmds) | Simple queries |
| 4 | `encointer_core.rs` → `chain.rs` | Medium (5 cmds) | Balance, transfer, listen |
| 5 | `encointer_communities.rs` → `community.rs` | Medium (5 cmds) | Uses `CommunitySpec` |
| 6 | `encointer_bazaar.rs` → `bazaar.rs` | Medium (6 cmds) | Straightforward |
| 7 | `encointer_faucet.rs` → `faucet.rs` | Medium (6 cmds) | Mix of user/governance calls |
| 8 | `encointer_reputation_commitments.rs` → merge into `reputation.rs` | Low (2 cmds) | |
| 9 | `encointer_reputation_rings.rs` → `reputation.rs` | Medium (6 cmds) | Ring-VRF crypto |
| 10 | `encointer_democracy.rs` → `democracy.rs` | Medium (8 cmds) | Governance |
| 11 | `encointer_ceremonies.rs` → `ceremonies.rs` | High (16 cmds) | Largest module |
| 12 | `encointer_offline_payment.rs` → `offline_payment.rs` | High (13 cmds) | ZK proof handling |
| 13 | `encointer_ipfs.rs` → `ipfs.rs` or merge into `community.rs` | Low (1 cmd) | |
| 14 | `encointer_treasuries.rs` → merge into `chain.rs` | Low (1 cmd) | |

**Migration pattern for each module:**
- Define subcommand enum variants with typed fields
- Move handler function bodies into `match` arms of `run()`
- Replace `matches.value_of("arg_name")` with direct struct field access
- Replace `get_chain_api(matches)` with `get_chain_api(cli)`
- Delete old function signatures

### Phase 6: Rewrite `main.rs`

#### 10. Simplify `main.rs`

```rust
use clap::Parser;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    env_logger::init();
    let cli = Cli::parse();
    commands::match_command(&cli).await?;
    Ok(())
}
```

#### 11. Create top-level dispatcher (`src/commands/mod.rs`)

```rust
pub async fn match_command(cli: &Cli) -> CliResult {
    match &cli.command {
        Commands::Account(cmd) => cmd.run(cli).await,
        Commands::Chain(cmd) => cmd.run(cli).await,
        Commands::Community(cmd) => cmd.run(cli).await,
        Commands::Ceremony(cmd) => cmd.run(cli).await,
        Commands::Scheduler(cmd) => cmd.run(cli).await,
        Commands::Bazaar(cmd) => cmd.run(cli).await,
        Commands::Faucet(cmd) => cmd.run(cli).await,
        Commands::Reputation(cmd) => cmd.run(cli).await,
        Commands::Democracy(cmd) => cmd.run(cli).await,
        Commands::OfflinePayment(cmd) => cmd.run(cli).await,
    }
}
```

### Phase 7: Cleanup & Verification

#### 12. Delete removed files
- `src/cli_args.rs` (replaced by derive structs)
- Remove `clap_nested` from `Cargo.toml`

#### 13. Update `community_spec.rs`
- Minimal changes; just ensure it works with the new `Cli` struct instead of `ArgMatches`

#### 14. Verify compilation and test
- `cargo check` after each module migration
- `cargo clippy` for lint issues
- Run existing tests / integration tests
- Verify `--help` output for all subcommand groups

---

## Final File Structure

```
client/src/
├── main.rs                    # Minimal: parse + dispatch
├── lib.rs                     # Cli struct definition
├── error.rs                   # CliError, CliResult types
├── utils.rs                   # Shared utilities (refactored)
├── community_spec.rs          # Community spec parsing (minor changes)
└── commands/
    ├── mod.rs                 # Commands enum + match_command()
    ├── account.rs             # new-account, list-accounts, export-secret
    ├── chain.rs               # balance, transfer, listen, metadata, treasury
    ├── community.rs           # new-community, add/remove locations, list
    ├── ceremonies.rs          # 16 ceremony commands
    ├── scheduler.rs           # get-phase, get-cindex, next-phase
    ├── bazaar.rs              # business & offering CRUD
    ├── faucet.rs              # faucet lifecycle
    ├── reputation.rs          # rings + commitments
    ├── democracy.rs           # proposals, voting, enactment
    ├── offline_payment.rs     # ZK proof generation & trusted setup
    └── ipfs.rs                # IPFS upload (or merge into community)
```

## Breaking Changes Summary

| Before | After |
|--------|-------|
| `encointer-client-notee transfer ...` | `encointer-client-notee transfer ...` (unchanged — flattened) |
| `encointer-client-notee register-participant ...` | `encointer-client-notee ceremony register ...` |
| `encointer-client-notee new-community ...` | `encointer-client-notee community new ...` |
| `encointer-client-notee create-business ...` | `encointer-client-notee bazaar create-business ...` |
| `encointer-client-notee get-phase` | `encointer-client-notee scheduler get-phase` |
| `encointer-client-notee create-faucet ...` | `encointer-client-notee faucet create ...` |
| `encointer-client-notee vote ...` | `encointer-client-notee democracy vote ...` |
| `encointer-client-notee generate-offline-payment ...` | `encointer-client-notee offline-payment generate ...` |

The most common operations (`balance`, `transfer`, `new-account`, `list-accounts`) stay at the top level via `#[command(flatten)]`. Domain-specific commands move under their group prefix, improving discoverability via `--help`.

## Key Design Decisions

1. **Derive over Builder**: All clap configuration via derive macros — declarative, type-safe, less boilerplate
2. **Flatten for frequent commands**: `Account` and `Chain` commands are flattened to top level for ergonomics
3. **`&Cli` passed everywhere**: Global config struct replaces string-keyed `ArgMatches` extraction
4. **Shared `Args` structs**: Common argument patterns (account, seed, CID) defined once as `#[derive(Args)]` and composed into command variants
5. **Async throughout**: All `run()` methods are `async fn` since most commands make RPC calls
6. **One module per group**: Simplifies navigation; the ceremony and offline-payment modules will still be large but are self-contained

## References

- [Encointer CLI (current)](https://github.com/encointer/encointer-node/tree/master/client)
- [Integritee CLI (reference pattern)](https://github.com/integritee-network/worker/tree/master/cli)
- [clap 4.5.x documentation](https://docs.rs/clap/latest/clap/)
- [clap derive tutorial](https://docs.rs/clap/latest/clap/_derive/_tutorial/index.html)
