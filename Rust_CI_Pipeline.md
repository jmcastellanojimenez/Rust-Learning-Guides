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
cargo llvm-cov    # Generate coverage report
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
cargo install cargo-audit

# Run  
cargo audit    # Scan dependency tree for vulnerabilities
```

## Ready-to-Use GitHub Actions Workflow

Create `.github/workflows/general.yml` in your repository with the following configuration from the Zero To Production book:

```yaml
name: Rust

on:
  push:
    branches: [ main ]
  pull_request:
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
```

**What this does**: 
- **3 parallel jobs**: Tests, Formatting check, Linting
- **Automatic caching**: The `actions-rust-lang/setup-rust-toolchain@v1` action handles dependency caching
- **Fail-fast**: Any job failure blocks the PR/merge

**Typical CI Pipeline Order**: Tests → Coverage → Linting → Formatting → Security Audit

**Source**: [Zero To Production GitHub Workflows](https://github.com/LukeMathWalker/zero-to-production/tree/root-chapter-03-part0/.github/workflows)
