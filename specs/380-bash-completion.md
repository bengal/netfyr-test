# SPEC-380: Bash Completion

## What

Generate and install bash completion for the `netfyr` CLI. All subcommands (`apply`, `query`, `history`, `revert`), their flags, and flag values are completed by the shell when the user presses Tab.

## Why

Tab completion is essential for CLI usability. Without it, users must remember exact subcommand names and flag spellings. `netfyr` has four subcommands with many flags (e.g., `--selector`, `--trigger`, `--since`, `--dry-run`, `--output`, `--show`, `--color`), and some flags accept a fixed set of values (`--output yaml|json`, `--trigger apply|dhcp|external|startup|revert`, `--color auto|always|never`). Completion makes discovery instant and reduces typos.

## User interaction

```bash
# Complete subcommands
netfyr <Tab>
# offers: apply  query  history  revert

# Complete flags for a subcommand
netfyr query --<Tab>
# offers: --selector  --output  --color

# Complete flag values
netfyr query --output <Tab>
# offers: yaml  json

netfyr history --trigger <Tab>
# offers: apply  dhcp  external  startup  revert

netfyr apply --color <Tab>
# offers: auto  always  never

# Complete file paths for apply
netfyr apply /etc/netfyr/<Tab>
# offers: files and directories (default bash path completion)

# Generate completions to stdout
netfyr completions bash
```

## Implementation details

- Crate: `netfyr-cli`
- Files:
  - `crates/netfyr-cli/src/lib.rs` — add `Completions` variant to `Commands` enum
  - `crates/netfyr-cli/src/completions.rs` — (new) completion generation logic
  - `crates/netfyr-cli/src/main.rs` — handle `Completions` command
- Dependencies (external crates): `clap_complete` (companion crate to clap for shell completion generation)

### Completion subcommand

Add a `netfyr completions <shell>` subcommand that prints the completion script to stdout. This follows the standard pattern used by `rustup`, `cargo`, and similar Rust CLI tools.

```rust
#[derive(Args)]
pub struct CompletionsArgs {
    /// Shell to generate completions for
    #[arg(value_enum)]
    pub shell: clap_complete::Shell,
}
```

The subcommand outputs the completion script via `clap_complete::generate()` to stdout. Users install it by redirecting to the appropriate directory:

```bash
# Install for the current user
netfyr completions bash > ~/.local/share/bash-completion/completions/netfyr

# Install system-wide
sudo netfyr completions bash > /usr/share/bash-completion/completions/netfyr
```

### RPM integration

The RPM spec (SPEC-502) should be updated to include the bash completion file. During the `%install` phase, run `netfyr completions bash` and place the output in `%{buildroot}/usr/share/bash-completion/completions/netfyr`. The file is then owned by the package and installed automatically.

### How clap_complete works

`clap_complete::generate()` introspects the clap `Command` structure at runtime and emits a bash completion script that handles:
- Subcommand names from the `Commands` enum
- Long and short flags from `#[arg]` attributes
- `ValueEnum` variants (e.g., `OutputFormat`, `ColorMode`) as value completions
- File path completion for positional arguments (default bash behavior)

No manual completion logic is needed — the completions are derived entirely from the existing clap definitions.

## Depends on

- SPEC-301 (CLI framework — provides the `Cli` struct and `Commands` enum)
- SPEC-302 (CLI query — adds `Query` subcommand and its flags)
- SPEC-352 (History CLI — adds `History` subcommand and its flags)
- SPEC-354 (State revert — adds `Revert` subcommand and its flags)

## Acceptance criteria

```gherkin
Feature: Bash completion generation
  Scenario: Generate bash completion script
    When `netfyr completions bash` is run
    Then the exit code is 0
    And the output is a non-empty bash script
    And the output contains "netfyr" (the command name)

  Scenario: Completion script contains all subcommands
    When `netfyr completions bash` is run
    Then the output contains "apply"
    And the output contains "query"
    And the output contains "history"
    And the output contains "revert"
    And the output contains "diagnose"
    And the output contains "completions"

  Scenario: Completion script contains global flags
    When `netfyr completions bash` is run
    Then the output contains "--color"

  Scenario: Completion script contains subcommand flags
    When `netfyr completions bash` is run
    Then the output contains "--selector"
    And the output contains "--output"
    And the output contains "--dry-run"
    And the output contains "--trigger"
    And the output contains "--since"
    And the output contains "--show"
    And the output contains "--count"
    And the output contains "--absolute-timestamps"

  Scenario: Invalid shell argument shows error
    When `netfyr completions invalid_shell` is run
    Then the exit code is non-zero
    And the error output indicates the value is not valid

Feature: Bash completion works interactively
  Scenario: Subcommand completion
    Given bash completion for netfyr is loaded
    When the user types "netfyr " and presses Tab
    Then "apply", "query", "history", "revert", "diagnose", and "completions" are offered

  Scenario: Flag completion for query
    Given bash completion for netfyr is loaded
    When the user types "netfyr query --" and presses Tab
    Then "--selector", "--output", and "--color" are offered

  Scenario: Value completion for --output
    Given bash completion for netfyr is loaded
    When the user types "netfyr query --output " and presses Tab
    Then "yaml" and "json" are offered

  Scenario: Value completion for --color
    Given bash completion for netfyr is loaded
    When the user types "netfyr apply --color " and presses Tab
    Then "auto", "always", and "never" are offered
```
