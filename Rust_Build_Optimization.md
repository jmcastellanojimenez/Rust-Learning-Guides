## 1. Faster Linking (lld)
**What**: Replace default linker with faster LLVM lld linker  
**Why**: Reduces linking time during incremental compilation  

**Commands:**
```bash
# Install
brew install llvm

# Add to .cargo/config.toml (project root or ~/.cargo/config.toml for global)
[target.x86_64-apple-darwin]
rustflags = ["-C", "link-arg=-fuse-ld=lld"]

[target.aarch64-apple-darwin]
rustflags = ["-C", "link-arg=-fuse-ld=/opt/homebrew/opt/llvm/bin/ld64.lld"]
```

## 2. Auto-rebuild (cargo-watch)
**What**: Automatically runs cargo commands when files change  
**Why**: Reduces perceived compilation time - builds while you're coding  

**Commands:**
```bash
# Install
cargo install cargo-watch

# Usage
cargo watch -x check    # Auto cargo check
cargo watch -x run      # Auto cargo run  
cargo watch -x test     # Auto cargo test
```

**Result**: Compilation starts automatically when you save, so it's done when you check terminal.
