## What is CI (Continuous Integration)?
**What**: Automated checks that run on every commit to main branch  
**Why**: Enables trunk-based development - deploy main anytime, catch issues early, tighter feedback loop  
**Rule**: If any check fails → cannot merge to main  

## Essential CI Steps

### 1. Tests
**What**: Run all unit and integration tests  
**Why**: Core quality check - if this fails, everything else is pointless  

**Commands:**
```bash
cargo test    # Builds project + runs tests automatically
```

### 2. Code Coverage  
**What**: Measure how much code is tested  
**Why**: Spot poorly tested areas (not a quality metric, just info)  

**Commands:**
```bash
# Install
rustup component add llvm-tools-preview
cargo install cargo-llvm-cov

# Run
cargo llvm-cov --all-features --workspace --lcov --output-path lcov.info
cargo llvm-cov report --html --output-dir coverage
```

### 3. Linting (clippy)
**What**: Static analysis for unidiomatic code and common mistakes  
**Why**: Catch code smells, enforce best practices  

**Commands:**
```bash
# Install (if needed)
rustup component add clippy

# Run
cargo clippy                    # Check for issues
cargo clippy -- -D warnings     # Fail CI on any warnings
```

### 4. Formatting (rustfmt)
**What**: Enforce consistent code formatting  
**Why**: Remove formatting bikeshedding from PR reviews  

**Commands:**
```bash
# Install (if needed) 
rustup component add rustfmt

# Run
cargo fmt              # Format code
cargo fmt -- --check   # Fail CI if code is unformatted
```

### 5. Security Audit
**What**: Check dependencies for known vulnerabilities  
**Why**: Prevent deploying code with security holes  

**Commands:**
```bash
# Install
cargo install cargo-deny    # More powerful than cargo-audit

# Run  
cargo deny check advisories    # Scan for vulnerabilities
```

## Complete GitHub Actions Setup

### Main Workflow: `.github/workflows/general.yml`
```yaml
name: Rust

on:
  push:
    branches: [ main ]
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [ main ]

env:
  CARGO_TERM_COLOR: always

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - name: Check out repository code
        uses: actions/checkout@v4
      - name: Install the Rust toolchain
        uses: actions-rust-lang/setup-rust-toolchain@v1
      - name: Run tests
        run: cargo test

  fmt:
    name: Rustfmt
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install the Rust toolchain
        uses: actions-rust-lang/setup-rust-toolchain@v1
        with:
          components: rustfmt
      - name: Enforce formatting
        run: cargo fmt --check

  clippy:
    name: Clippy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install the Rust toolchain
        uses: actions-rust-lang/setup-rust-toolchain@v1
        with:
          components: clippy
      - name: Linting
        run: cargo clippy -- -D warnings

  coverage:
    name: Code coverage
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install the Rust toolchain
        uses: actions-rust-lang/setup-rust-toolchain@v1
        with:
          components: llvm-tools-preview
      - name: Install cargo-llvm-cov
        uses: taiki-e/install-action@cargo-llvm-cov
      - name: Generate code coverage
        run: cargo llvm-cov --all-features --workspace --lcov --output-path lcov.info
      - name: Generate report
        run: cargo llvm-cov report --html --output-dir coverage
      - uses: actions/upload-artifact@v4
        with:
          name: "Coverage report"
          path: coverage/
```

### Security Audit Workflow: `.github/workflows/audit.yml`
```yaml
name: Security audit

on:
  schedule:
    - cron: '0 0 * * *'    # Daily at midnight
  push:
    paths:
      - '**/Cargo.toml'    # When dependencies change
      - '**/Cargo.lock'

jobs:
  security_audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: taiki-e/install-action@cargo-deny
      - name: Scan for vulnerabilities
        run: cargo deny check advisories
```

**What this setup provides**:
- **4 parallel jobs** on every PR: Tests, Formatting, Linting, Coverage
- **Daily security scans** + scans when dependencies change
- **Coverage reports** as downloadable artifacts
- **Automatic caching** via `actions-rust-lang/setup-rust-toolchain@v1`
- **Fail-fast**: Any job failure blocks the PR/merge

**Pipeline execution order**: All jobs run in parallel except security audit (runs separately on schedule/dependency changes)

**Source**: [Zero To Production GitHub Workflows](https://github.com/LukeMathWalker/zero-to-production/tree/root-chapter-03-part0/.github/workflows)
