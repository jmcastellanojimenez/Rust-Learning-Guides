### Quick Reference

**Vec** - Use for lists of the same type that grow/shrink

**String** - Use for text that changes (for read-only text, use &str)  

**HashMap** - Use for key-value lookups and counting

✅ All three store data on the heap and can grow at runtime. All three follow Rust's ownership rules strictly.

---

## VEC<T> - "Dynamic Lists"

```rust
// Create and modify
let mut scores = vec![85, 92, 78];
scores.push(95);                    // Add to end

// Access safely
let third = scores.get(2);          // Returns Option<&i32>
let third = &scores[2];             // Panics if index doesn't exist

// Loop through
for score in &scores {
    println!("{}", score);
}
```

✅ Why Vec matters: Grows automatically, stores data next to each other in memory for speed, enforces ownership rules.

---

## STRING - "Text That Can Change"

```rust
// Build strings
let mut message = String::from("Hello");
message.push_str(", world!");

// Combine strings
let name = "Alice";
let greeting = format!("Hi, {}", name);     // Doesn't take ownership

// Can't do this - no indexing
// let first = text[0];                     // ERROR

// Do this instead
let first = text.chars().nth(0);            // Returns Option<char>
```

✅ Why String is tricky: UTF-8 encoding means one character might be multiple bytes. Rust prevents crashes by disallowing direct indexing.

---

## HASHMAP<K,V> - "Key-Value Storage"

```rust
use std::collections::HashMap;

// Store and retrieve
let mut scores = HashMap::new();
scores.insert("Alice", 100);
scores.insert("Bob", 85);

let alice_score = scores.get("Alice");      // Returns Option<&i32>

// Count words example
let mut word_count = HashMap::new();
for word in "hello world hello".split_whitespace() {
    let count = word_count.entry(word).or_insert(0);
    *count += 1;
}
```

✅ Why HashMap is useful: Fast lookups by key, perfect for counting things or storing configuration data.

---

## 🔥 The Real Issues You'll Hit

**Borrowing while iterating:**
```rust
let mut v = vec![1, 2, 3];
for item in &v {
    v.push(4);                      // ERROR: can't modify while iterating
}
```

**String ownership confusion:**
```rust
let s1 = String::from("hello");
let s2 = s1 + " world";             // s1 is moved, no longer usable
```

**HashMap takes ownership:**
```rust
let key = String::from("name");
map.insert(key, "Alice");           // key is moved into map
// println!("{}", key);             // ERROR: key no longer exists
```
