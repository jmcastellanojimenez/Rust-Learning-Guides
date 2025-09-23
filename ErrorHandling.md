🚨 Rust forces you to think about what can go wrong. Your programs become more reliable because you must handle every possible failure case.

**"Two Types of Errors, Two Ways to Handle Them"**

---

## 💥 PANIC! - "Unrecoverable Errors"

```rust
// When things go very wrong - program crashes immediately
panic!("Something terrible happened!");

// Array bounds panic automatically
let v = vec![1, 2, 3];
let crash = v[99];  // panic!: index out of bounds

// Use when continuing would be dangerous
fn divide_by_zero(x: i32) {
    if x == 0 {
        panic!("Cannot divide by zero!");
    }
}
```

**When to panic:**
- 🐛 Programming bugs (array out of bounds, invalid state)
- ⚖️ Contract violations that should never happen
- 🛡️ Security risks if you continue

---

## ✅ RESULT<T, E> - "Recoverable Errors"

```rust
// Function that might fail
use std::fs::File;

fn open_file(filename: &str) -> Result<File, std::io::Error> {
    File::open(filename)  // Returns Result automatically
}

// Handle the Result - Method 1: Match
let result = File::open("config.txt");
match result {
    Ok(file) => println!("File opened successfully"),
    Err(error) => println!("Failed to open file: {}", error),
}

// Handle the Result - Method 2: unwrap (crashes if error)
let file = File::open("config.txt").unwrap();  // panic! on error

// Handle the Result - Method 3: expect (crashes with custom message)
let file = File::open("config.txt")
    .expect("Config file should exist");
```

**When to use Result:**
- 📁 File operations, network calls, parsing
- 👤 User input that might be invalid
- ⚡ Any operation that can fail normally

---

## ❓ THE ? OPERATOR - "Error Propagation Made Easy"

```rust
// The hard way - manual error handling
fn read_username() -> Result<String, std::io::Error> {
    let result = File::open("username.txt");
    let mut file = match result {
        Ok(file) => file,
        Err(e) => return Err(e),  // Pass error up
    };
    
    let mut username = String::new();
    match file.read_to_string(&mut username) {
        Ok(_) => Ok(username),
        Err(e) => Err(e),  // Pass error up
    }
}

// The easy way - ? operator
fn read_username() -> Result<String, std::io::Error> {
    let mut file = File::open("username.txt")?;  // Return error if fails
    let mut username = String::new();
    file.read_to_string(&mut username)?;  // Return error if fails
    Ok(username)
}

// Even easier
fn read_username() -> Result<String, std::io::Error> {
    std::fs::read_to_string("username.txt")  // One line!
}
```

**How ? works:** If Result is Ok, unwrap the value. If Result is Err, return the error immediately.

---

## 🛠️ PRACTICAL PATTERNS

**Validate user input:**
```rust
fn parse_age(input: &str) -> Result<u8, String> {
    match input.parse::<u8>() {
        Ok(age) if age <= 120 => Ok(age),
        Ok(_) => Err("Age must be 120 or less".to_string()),
        Err(_) => Err("Please enter a valid number".to_string()),
    }
}
```

**Chain operations with ?:**
```rust
fn process_data() -> Result<String, Box<dyn std::error::Error>> {
    let data = std::fs::read_to_string("input.txt")?;
    let processed = data.trim().parse::<i32>()?;
    let result = format!("Result: {}", processed * 2);
    Ok(result)
}
```

**Custom error types:**
```rust
#[derive(Debug)]
enum MyError {
    FileError(std::io::Error),
    ParseError(std::num::ParseIntError),
}

fn risky_operation() -> Result<i32, MyError> {
    let content = std::fs::read_to_string("data.txt")
        .map_err(MyError::FileError)?;
    
    let number = content.trim().parse()
        .map_err(MyError::ParseError)?;
    
    Ok(number)
}
```

---

## 🎯 MAIN FUNCTION ERROR HANDLING

```rust
// main() can return Result too
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {
    let file = File::open("important.txt")?;  // ? works in main
    // do something with file
    Ok(())
}
```

---

## 🧭 DECISION FLOWCHART

**Use panic! when:**
- 🐛 Programming error (bug in your code)
- 🚫 Security violation if you continue
- 🚧 Prototyping (use unwrap/expect temporarily)
- 🧪 Tests (failure should stop the test)

**Use Result when:**
- 🌐 External operations (files, network, parsing)
- 👥 User input validation
- ⚠️ Any expected failure scenario
- 📚 Library code (let caller decide what to do)

---

## ⚠️ COMMON MISTAKES

**Don't ignore errors:**
```rust
// Bad
let _ = File::open("config.txt");  // Ignores potential error

// Good  
match File::open("config.txt") {
    Ok(_) => println!("Config loaded"),
    Err(e) => eprintln!("Warning: No config file: {}", e),
}
```

**Don't panic in libraries:**
```rust
// Bad - library crashes user's program
pub fn parse_config(data: &str) -> Config {
    data.parse().unwrap()  // panics on invalid input
}

// Good - let caller decide what to do
pub fn parse_config(data: &str) -> Result<Config, ParseError> {
    data.parse()
}
```
