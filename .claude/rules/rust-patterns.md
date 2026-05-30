# Rust Patterns — RTK Development Rules

RTK-specific Rust idioms and constraints. Applied to all code in this repository.

## Non-Negotiable RTK Rules

These override general Rust conventions:

1. **No async** — Zero `tokio`, `async-std`, `futures`. Single-threaded by design. Async adds 5-10ms startup.
2. **No `unwrap()` in production** — Use `.context("description")?`. Tests: use `expect("reason")`.
3. **Lazy regex** — `Regex::new()` inside a function recompiles on every call. Always `lazy_static!`.
4. **Fallback pattern** — If filter fails, execute raw command unchanged. Never block the user.
5. **Exit code propagation** — `std::process::exit(code)` if underlying command fails.

## Error Handling

### Always context, always anyhow

```rust
use anyhow::{Context, Result};

// ✅ Correct
fn read_config(path: &Path) -> Result<Config> {
    let content = fs::read_to_string(path)
        .with_context(|| format!("Failed to read config: {}", path.display()))?;
    toml::from_str(&content)
        .context("Failed to parse config TOML")
}

// ❌ Wrong — no context
fn read_config(path: &Path) -> Result<Config> {
    let content = fs::read_to_string(path)?;
    Ok(toml::from_str(&content)?)
}

// ❌ Wrong — panic in production
fn read_config(path: &Path) -> Config {
    let content = fs::read_to_string(path).unwrap();
    toml::from_str(&content).unwrap()
}
```

### Fallback pattern (mandatory for all filters)

```rust
pub fn run(args: MyArgs) -> Result<()> {
    let output = execute_command("mycmd", &args.to_cmd_args())
        .context("Failed to execute mycmd")?;

    let filtered = filter_output(&output.stdout)
        .unwrap_or_else(|e| {
            eprintln!("rtk: filter warning: {}", e);
            output.stdout.clone()  // Passthrough on failure
        });

    tracking::record("mycmd", &output.stdout, &filtered)?;
    print!("{}", filtered);

    if !output.status.success() {
        std::process::exit(output.status.code().unwrap_or(1));
    }
    Ok(())
}
```

## Regex — Always lazy_static

```rust
use lazy_static::lazy_static;
use regex::Regex;

lazy_static! {
    static ref ERROR_RE: Regex = Regex::new(r"^error\[").unwrap();
    static ref HASH_RE: Regex = Regex::new(r"^[0-9a-f]{7,40}").unwrap();
}

// ✅ Correct — regex compiled once at first use
fn is_error_line(line: &str) -> bool {
    ERROR_RE.is_match(line)
}

// ❌ Wrong — recompiles every call (kills performance)
fn is_error_line(line: &str) -> bool {
    let re = Regex::new(r"^error\[").unwrap();
    re.is_match(line)
}
```

Note: `lazy_static!` with `.unwrap()` for initialization is the **established RTK pattern** — it's acceptable because a bad regex literal is a programming error caught at first use.

## Ownership — Borrow Over Clone

```rust
// ✅ Prefer borrows in filter functions
fn filter_lines<'a>(input: &'a str) -> Vec<&'a str> {
    input.lines()
        .filter(|line| !line.is_empty())
        .collect()
}

// ✅ Clone only when you need to own the data
fn filter_output(input: &str) -> String {
    input.lines()
        .filter(|line| !line.trim().is_empty())
        .collect::<Vec<_>>()
        .join("\n")
}

// ❌ Unnecessary clone
fn filter_output(input: &str) -> String {
    let owned = input.to_string();  // Clone for no reason
    owned.lines()
        .filter(|line| !line.is_empty())
        .collect::<Vec<_>>()
        .join("\n")
}
```

## Iterators Over Loops

```rust
// ✅ Iterator chain — idiomatic
let errors: Vec<&str> = output.lines()
    .filter(|l| l.starts_with("error"))
    .take(20)
    .collect();

// ❌ Manual loop — verbose
let mut errors = Vec::new();
for line in output.lines() {
    if line.starts_with("error") {
        errors.push(line);
        if errors.len() >= 20 { break; }
    }
}
```

## Struct Patterns

### Builder for complex args

```rust
// Use Builder when struct has >5 optional fields
pub struct FilterConfig {
    max_lines: usize,
    show_warnings: bool,
    strip_ansi: bool,
}

impl FilterConfig {
    pub fn new() -> Self {
        Self { max_lines: 100, show_warnings: false, strip_ansi: true }
    }
    pub fn max_lines(mut self, n: usize) -> Self { self.max_lines = n; self }
    pub fn show_warnings(mut self, v: bool) -> Self { self.show_warnings = v; self }
}

// Usage: FilterConfig::new().max_lines(50).show_warnings(true)
```

### Newtype for validation

```rust
// Newtype prevents misuse of raw strings
pub struct CommandName(String);

impl CommandName {
    pub fn new(name: &str) -> Result<Self> {
        if name.contains(';') || name.contains('|') {
            anyhow::bail!("Invalid command name: contains shell metacharacters");
        }
        Ok(Self(name.to_string()))
    }
}
```

## String Handling

```rust
// String: owned, heap-allocated
// &str: borrowed slice (prefer in function signatures)
// &String: almost never — use &str instead

fn process(input: &str) -> String {  // ✅ &str in, String out
    input.trim().to_uppercase()
}

fn process(input: &String) -> String {  // ❌ Unnecessary &String
    input.trim().to_uppercase()
}
```

## Match — Exhaustive and Explicit

```rust
// ✅ Exhaustive match with explicit cases
match result {
    Ok(output) => process(output),
    Err(e) => {
        eprintln!("rtk: filter warning: {}", e);
        fallback()
    }
}

// ❌ Silent swallow — catastrophic in RTK (user gets no output)
match result {
    Ok(output) => process(output),
    Err(_) => {}
}
```

## Module Structure

Every `*_cmd.rs` follows this pattern:

```rust
// 1. Imports
use anyhow::{Context, Result};
use lazy_static::lazy_static;
use regex::Regex;

// 2. Types (args struct)
pub struct MyArgs { ... }

// 3. Lazy regexes
lazy_static! { static ref MY_RE: Regex = ...; }

// 4. Public entry point
pub fn run(args: MyArgs) -> Result<()> { ... }

// 5. Private filter functions
fn filter_output(input: &str) -> Result<String> { ... }

// 6. Tests (always present)
#[cfg(test)]
mod tests {
    use super::*;
    fn count_tokens(s: &str) -> usize { s.split_whitespace().count() }
    // ... snapshot tests, savings tests
}
```

## Anti-Patterns (RTK-Specific)

| Pattern | Problem | Fix |
|---------|---------|-----|
| `Regex::new()` in function | Recompiles every call | `lazy_static!` |
| `unwrap()` in production | Panic breaks user workflow | `.context()?` |
| `tokio::main` or `async fn` | +5-10ms startup | Blocking I/O only |
| Silent match `Err(_) => {}` | User gets no output | Log warning + fallback |
| `println!` in filter path | Debug artifact in output | Remove or `eprintln!` |
| Returning early without exit code | CI/CD thinks command succeeded | `std::process::exit(code)` |
| `clone()` of large strings | Extra allocation in hot path | Borrow with `&str` |

## Merge & Refactor Safety

A conflict-free `git merge` is **not** a verified merge. When two branches edit
the same function on non-conflicting lines, git stitches them textually — the
result can fail to compile, resurrect dead code, or **silently revert a feature**.

**After merging branches that touched the same files, always run the full gate:**

```bash
cargo build && cargo clippy --all-targets && cargo test --bin rtk
```

> RTK is a **binary-only** crate. `cargo test --lib` fails with "no library
> targets found" — use `cargo test --bin rtk`.

### Three semantic-conflict classes to scan for

| Class | Symptom | Caught by |
|-------|---------|-----------|
| Function rename | other branch's call site uses the old name | `cargo build` (loud) |
| Added parameter | other branch's call sites / tests use the old arity | `cargo build` (loud) |
| **Silent revert** | a sibling edit uses a now-stale variable — compiles, lint-clean, classify still "passes" | **only behavioral tests** |

The silent class is the dangerous one: it ships unless the feature has end-to-end
tests *and* you run them. This is the real reason to write a test *with* every
feature — the payoff lands at merge time, not write time.

### RTK invariant: classify ↔ rewrite symmetry

`classify_command` and `rewrite_segment_inner` (`src/discover/registry.rs`) MUST
apply **identical** normalizations before matching: env-prefix strip, absolute
binary-path strip (`/usr/bin/grep` → `grep`, #485), git global-opt strip
(`git -C /tmp` → `git`), golangci global-opt strip. Add a normalization to one
path without the other and rewriting breaks silently — classify says "supported,"
then the rewrite loop fails to match. When you touch one, touch the other (with a test).
