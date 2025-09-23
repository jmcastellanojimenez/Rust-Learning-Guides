# 📁 RUST MODULES & PROJECT ORGANIZATION
**"Organizing Code as Projects Grow"**

---

## 🏗️ THE MODULE SYSTEM HIERARCHY

```rust
// PACKAGE = Bundle of crates (has Cargo.toml)
my-project/
├── Cargo.toml          // Package manifest
├── src/
│   ├── main.rs         // Binary crate root
│   ├── lib.rs          // Library crate root (optional)
│   └── bin/            // Additional binary crates
│       └── other.rs

// CRATE = Tree of modules (smallest compilation unit)
// - Binary crate: Has main(), compiles to executable
// - Library crate: No main(), shared functionality

// MODULE = Code organization within a crate
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}
```

💡 **Think of it like a file system:**
- Package = Project folder
- Crate = Individual programs/libraries  
- Module = Folders organizing code
- Functions/Structs = Files

---

## 🛣️ PATHS & PRIVACY

```rust
// ABSOLUTE PATH - starts from crate root
crate::front_of_house::hosting::add_to_waitlist();

// RELATIVE PATH - starts from current module
front_of_house::hosting::add_to_waitlist();

// PARENT PATH - go up one level
super::deliver_order();

// PRIVACY RULES
mod restaurant {
    fn private_function() {}           // ❌ Private by default
    pub fn public_function() {}        // ✅ Explicitly public
    
    pub mod dining {                   // Module must be pub
        pub fn seat_customer() {}      // AND function must be pub
    }
}
```

🧠 **Privacy follows parent-child rules:**
- Child modules can see parent items
- Parent modules CANNOT see child private items  
- Siblings can see each other only if public

---

## 🔗 USE KEYWORD - "Import Shortcuts"

```rust
// WITHOUT use - repetitive
restaurant::front_of_house::hosting::add_to_waitlist();
restaurant::front_of_house::hosting::seat_at_table();

// WITH use - create shortcut once
use restaurant::front_of_house::hosting;
hosting::add_to_waitlist();
hosting::seat_at_table();

// IDIOMATIC PATTERNS
use std::collections::HashMap;        // ✅ Structs: full path
use std::io::Write;                   // ✅ Traits: full path  
use std::io;                          // ✅ Functions: parent module
io::read_to_string();                 // Then specify parent

// ADVANCED USE
use std::io::{self, Write};           // Multiple imports
use std::fmt::Result as FmtResult;    // Rename conflicts
use std::collections::*;              // Import all (use sparingly)
```

---

## 📂 FILE ORGANIZATION

```rust
// SINGLE FILE - small projects
// src/lib.rs
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

// SEPARATE FILES - larger projects  
// src/lib.rs
mod front_of_house;  // Declares module, looks for file

// src/front_of_house.rs
pub mod hosting {
    pub fn add_to_waitlist() {}
}

// DIRECTORY STRUCTURE - complex modules
// src/front_of_house/mod.rs  (older style)
// src/front_of_house.rs      (modern style)
```

---

## 🌟 RE-EXPORTING WITH PUB USE

```rust
// INTERNAL STRUCTURE
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

// RE-EXPORT for simpler external API
pub use crate::front_of_house::hosting;

// EXTERNAL CODE can now use:
restaurant::hosting::add_to_waitlist();  // ✅ Short path
// Instead of:
restaurant::front_of_house::hosting::add_to_waitlist();  // Long path
```

🧠 **When to use pub use:**
- Hide complex internal organization
- Provide clean external APIs
- Keep backward compatibility when refactoring

---

## 🎯 PRACTICAL GUIDELINES

**File Organization:**
- Start with single file until it gets large (>200-300 lines)
- Split related functions into modules
- Move large modules to separate files
- Use directories for complex module hierarchies

**Privacy Strategy:**
- Default to private, expose only what's needed
- Public APIs are contracts - harder to change later
- Use pub use to create clean interfaces

**Import Style:**
- Functions: import parent module, call with module::function()
- Structs/Enums: import directly, use bare name
- Avoid glob imports (*) except in tests

---

## 🔄 THE COMPLETE WORKFLOW

```
1. Start: Single file with functions
2. Grow: Group related functions in modules  
3. Split: Move modules to separate files
4. Organize: Create directory structure
5. Export: Use pub use for clean external API
6. Package: Add to Cargo.toml for distribution
```

**Bottom Line:** Modules are organizational tools that grow with your project. Start simple, add complexity only when needed. Focus on clear boundaries and minimal public interfaces.
