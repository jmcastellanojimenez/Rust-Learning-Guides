# Complete JWT Authentication Architecture Guide
*From Pure Stateless to Production-Grade Hybrid*

Here's your comprehensive JWT architecture reference with all components integrated:

## JWT Evolution: The Three Approaches Explained

### The Authentication Spectrum

```rust
// 🟢 SIMPLE (Your current 03-web-server)
// Pure stateless - no external dependencies
async fn validate_simple_jwt(token: &str, secret: &str) -> Result<Claims, Error> {
    // Just cryptographic validation, nothing else
    decode::<Claims>(token, &DecodingKey::from_secret(secret.as_bytes()), &Validation::default())
        .map(|data| data.claims)
}

// 🟡 TRADITIONAL (Database sessions)
// Stateful - every request hits database
async fn validate_session(session_id: &str, db: &Database) -> Result<User, Error> {
    // Database lookup on EVERY request (slow but secure)
    db.get_session(session_id).await
}

// 🟢 HYBRID (Production best practice)
// Stateless validation + optional revocation
async fn validate_hybrid_jwt(token: &str, secret: &str, redis: &RedisClient) -> Result<Claims, Error> {
    // 1. Fast cryptographic validation (pure Rust)
    let claims = decode::<Claims>(token, &DecodingKey::from_secret(secret.as_bytes()), &Validation::default())?
        .claims;
    
    // 2. Quick Redis check for revocation (only if needed)
    let is_valid = redis.exists(format!("jwt:{}", claims.jti)).await?;
    if !is_valid {
        return Err(Error::TokenRevoked);
    }
    
    Ok(claims)
}
```


### Why External Components Are Worth It

**Pure Rust (Stateless JWT):**
✅ No external dependencies
✅ Infinite scaling
❌ Can't revoke tokens until expiry
❌ Logout doesn't work immediately

**Rust + Redis (Hybrid):**
✅ Instant token revocation
✅ Still scales to millions of requests
✅ Redis is battle-tested and reliable
❌ One more service to manage (but worth it)

## Components Needed for Hybrid JWT in Rust

### 1. **JWT Library** ✅ Essential Foundation
- 📦 **Crate**: `jsonwebtoken = "9.2"` (most popular, battle-tested)
- **Purpose**: Generate and validate JWTs cryptographically
- **Algorithm Choice**: HS256 for single-service, RS256 for microservices
- **Why This One**: Mature, well-documented, zero-cost abstractions

### 2. **Redis Client** ⚠️ Critical for Production
- 📦 **Crate**: `redis = { version = "0.24", features = ["tokio-comp"] }`
- **Critical Feature**: Must include `tokio-comp` feature for async support
- **Purpose**: Store JWT ID (`jti`) for instant revocation and session management
- **Why not `fred`**: While `fred` is newer, `redis` crate is more mature and widely adopted

### 3. **Web Framework** ✅ Already Optimal
- 📦 **Crate**: `axum = "0.7"` (recommended) or `actix-web`
- **Why Axum**: Better ecosystem integration, more modern design
- **Your Current Setup**: Already using Axum in 03-web-server ✅
- **Middleware Support**: Perfect for JWT authentication layers

### 4. **Database Layer** ✅ Production Storage
- 📦 **Crate**: `sqlx` (recommended for async)
- **Features Needed**: `["runtime-tokio-rustls", "postgres", "uuid", "chrono", "migrate"]`
- **Not Optional**: You need persistent storage for user accounts in production
- **Why not Diesel**: SQLx has better async support and compile-time query verification

### 5. **Configuration Management** ⚠️ Security Critical
- 📦 **Crate**: `config = "0.14"` or built-in `std::env` (avoid `dotenv` in production)
- **Why not `dotenv`**: Only for development; production uses environment variables directly
- **Security**: Never commit secrets to git, even in `.env` files

## Updated Production-Ready Crates Setup

```toml
[dependencies]
# Core web framework
axum = "0.7"
tokio = { version = "1.0", features = ["full"] }
tower = { version = "0.4", features = ["full"] }
tower-http = { version = "0.5", features = ["fs", "trace", "cors", "compression"] }

# JWT and Redis (Hybrid Architecture)
jsonwebtoken = "9.2"  # Updated version
redis = { version = "0.24", features = ["tokio-comp"] }  # Async support

# Database
sqlx = { version = "0.7", features = [
    "runtime-tokio-rustls",  # Async runtime
    "postgres",              # PostgreSQL support
    "uuid",                  # UUID type support
    "chrono",               # DateTime support
    "migrate"               # Database migrations
]}

# Password hashing
bcrypt = "0.15"

# Configuration (production-ready)
config = "0.14"  # Better than dotenv
serde = { version = "1.0", features = ["derive"] }

# Error handling
thiserror = "1.0"
anyhow = "1.0"

# Async traits
async-trait = "0.1"

# Logging
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

# Utilities
uuid = { version = "1.0", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
```


## Complete JWT Claims Structure

```rust
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Claims {
    // Standard JWT claims (RFC 7519)
    pub sub: String,        // Subject (user ID)
    pub iat: i64,          // Issued At (timestamp)
    pub exp: i64,          // Expiration (timestamp)
    pub jti: String,       // JWT ID (unique identifier for this token)
    
    // Custom claims (your application data)
    pub user_id: Uuid,     // Typed user ID
    pub email: String,     // User email
    pub roles: Vec<String>, // User permissions ["admin", "user"]
    
    // Optional enhanced security
    pub session_id: Option<String>, // Link to session if needed
    pub device_id: Option<String>,  // For device tracking
}

impl Claims {
    pub fn new(user_id: Uuid, email: String, roles: Vec<String>, exp_hours: i64) -> Self {
        let now = chrono::Utc::now();
        Self {
            sub: user_id.to_string(),
            iat: now.timestamp(),
            exp: (now + chrono::Duration::hours(exp_hours)).timestamp(),
            jti: Uuid::new_v4().to_string(), // Unique for each token
            user_id,
            email,
            roles,
            session_id: None,
            device_id: None,
        }
    }
    
    pub fn is_expired(&self) -> bool {
        chrono::Utc::now().timestamp() > self.exp
    }
    
    pub fn has_role(&self, role: &str) -> bool {
        self.roles.iter().any(|r| r == role)
    }
}
```


## Implementation Flow (Production Architecture)

### **Login Route** ✅ Complete Implementation
```rust
async fn login(
    State(state): State<AppState>,
    Json(req): Json<LoginRequest>
) -> Result<impl IntoResponse, AppError> {
    // 1. Validate credentials against PostgreSQL
    let user = state.repo.find_by_email(&req.email).await
        .map_err(|_| AppError::Unauthorized("Invalid credentials".into()))?;
    
    let valid = state.auth.verify_password(req.password, user.password_hash.clone()).await?;
    if !valid {
        return Err(AppError::Unauthorized("Invalid credentials".into()));
    }
    
    // 2. Generate JWT with unique `jti` (JWT ID)
    let roles = vec!["user".to_string()]; // Could come from database
    let claims = Claims::new(user.id, user.email.clone(), roles, 24);
    let token = encode(&Header::new(Algorithm::HS256), &claims, &state.auth.encoding_key)?;
    
    // 3. Store `jti` in Redis with TTL = JWT expiry
    let mut redis_conn = state.redis.get_async_connection().await?;
    redis_conn.set_ex(
        format!("jwt:{}", claims.jti),
        serde_json::to_string(&claims)?, // Store full claims for features
        86400 // 24 hours TTL
    ).await?;
    
    // 4. Return JWT to client
    Ok(Json(serde_json::json!({
        "token": token,
        "user": UserResponse::from(user),
        "expires_in": 86400
    })))
}
```


### **Request Middleware** ⚠️ Critical Validation Layer
```rust
async fn jwt_auth_middleware(
    State(state): State<AppState>,
    mut request: Request,
    next: Next,
) -> Result<Response, AppError> {
    // 1. Extract JWT from Authorization header
    let token = extract_bearer_token(request.headers())
        .ok_or_else(|| AppError::Unauthorized("Missing authorization header".into()))?;
    
    // 2. Validate JWT signature (cryptographic verification)
    let token_data = decode::<Claims>(
        &token,
        &state.auth.decoding_key,
        &Validation::new(Algorithm::HS256)
    )?;
    
    let claims = token_data.claims;
    
    // 3. Check if `jti` EXISTS in Redis (whitelist approach)
    //    ✅ Correct: Check if jti exists (whitelist approach)
    //    ❌ Wrong: Check blacklist
    let mut redis_conn = state.redis.get_async_connection().await?;
    let exists: bool = redis_conn.exists(format!("jwt:{}", claims.jti)).await?;
    
    // 4. If jti missing from Redis = token was revoked
    if !exists {
        return Err(AppError::Unauthorized("Token has been revoked".into()));
    }
    
    // Inject claims into request for handlers
    request.extensions_mut().insert(claims);
    Ok(next.run(request).await)
}
```


### **Logout Route** ⚠️ Instant Revocation
```rust
async fn logout(
    Extension(claims): Extension<Claims>,
    State(state): State<AppState>,
) -> Result<impl IntoResponse, AppError> {
    // ✅ Correct: DELETE jti from Redis
    // ❌ Wrong: Add jti to blacklist
    // This immediately invalidates the token
    let mut redis_conn = state.redis.get_async_connection().await?;
    redis_conn.del(format!("jwt:{}", claims.jti)).await?;
    
    Ok(Json(serde_json::json!({
        "message": "Logged out successfully"
    })))
}
```


## Critical Architecture Pattern: Whitelist vs Blacklist

**❌ WRONG - Blacklist Approach:**
```rust
async fn validate_token_wrong(jti: &str, redis: &RedisClient) -> bool {
    // Check if jti is in blacklist
    !redis.exists(format!("blacklist:{}", jti)).await.unwrap_or(false)
    // Problem: Memory grows forever, never cleaned up
}
```


**✅ CORRECT - Whitelist Approach:**
```rust
async fn validate_token_correct(jti: &str, redis: &RedisClient) -> bool {
    // Check if jti exists in Redis (with TTL)
    redis.exists(format!("jwt:{}", jti)).await.unwrap_or(false)
    // Benefit: Auto-cleanup via TTL, constant memory usage
}
```


## Production Configuration Management

```rust
#[derive(Debug, Clone)]
pub struct Config {
    pub server: ServerConfig,
    pub database: DatabaseConfig,
    pub redis: RedisConfig,
    pub jwt: JwtConfig,
}

#[derive(Debug, Clone)]
pub struct JwtConfig {
    pub secret: String,
    pub expiry_hours: i64,
    pub algorithm: Algorithm,
}

impl Config {
    pub fn from_env() -> Result<Self, ConfigError> {
        // Validate JWT secret strength
        let jwt_secret = std::env::var("JWT_SECRET")?;
        if jwt_secret.len() < 32 {
            return Err(ConfigError::WeakJwtSecret);
        }
        
        Ok(Self {
            jwt: JwtConfig {
                secret: jwt_secret,
                expiry_hours: std::env::var("JWT_EXPIRY_HOURS")?.parse().unwrap_or(24),
                algorithm: Algorithm::HS256,
            },
            database: DatabaseConfig {
                url: std::env::var("DATABASE_URL")?,
                max_connections: std::env::var("DB_MAX_CONNECTIONS")?.parse().unwrap_or(20),
            },
            redis: RedisConfig {
                url: std::env::var("REDIS_URL").unwrap_or_else(|_| "redis://127.0.0.1:6379".to_string()),
            },
            server: ServerConfig {
                port: std::env::var("PORT")?.parse().unwrap_or(8080),
                host: std::env::var("HOST").unwrap_or_else(|_| "0.0.0.0".to_string()),
            },
        })
    }
}
```


## Production Security Enhancements

### **Redis TTL Strategy**
```rust
// Set TTL to match JWT expiry for automatic cleanup
redis_conn.set_ex(
    format!("jwt:{}", jti),
    "valid",  // Value doesn't matter, existence matters
    Duration::from_secs(86400)  // 24 hours = JWT expiry
).await?;
```


### **Security Hardening**
- **JWT Secret**: Use 256-bit random key (not a string like "secret123")
- **Redis Security**: Enable AUTH and use TLS in production
- **Rate Limiting**: Prevent brute force on login endpoints

### **Performance Optimizations**
- **Connection Pooling**: Both PostgreSQL and Redis need connection pools
- **Redis Pipelining**: Batch multiple Redis operations
- **JWT Caching**: Cache decoded JWTs briefly to avoid repeated crypto operations

## What This Architecture Enables (Business Value)

```rust
// Before (Stateless only):
// ✅ Infinite scaling
// ❌ Can't logout users
// ❌ Can't revoke compromised tokens
// ❌ No rate limiting per user

// After (Hybrid with Redis):
// ✅ Infinite scaling (still stateless validation)
// ✅ Instant logout/token revocation
// ✅ Rate limiting by user ID
// ✅ Session management capabilities
// ✅ Security incident response (revoke all tokens for user)
```


## Advanced Features Enabled

### **Admin Security Actions**
```rust
// Revoke all tokens for a compromised user
async fn revoke_all_user_tokens(user_id: Uuid, redis: &RedisClient) -> Result<(), AppError> {
    let keys: Vec<String> = redis.keys("jwt:*").await?;
    for key in keys {
        let claims_json: String = redis.get(&key).await.unwrap_or_default();
        if let Ok(claims) = serde_json::from_str::<Claims>(&claims_json) {
            if claims.user_id == user_id {
                redis.del(&key).await?;
            }
        }
    }
    Ok(())
}
```


### **Role-Based Access Control**
```rust
pub async fn admin_only(Extension(claims): Extension<Claims>) -> Result<impl IntoResponse, AppError> {
    if !claims.has_role("admin") {
        return Err(AppError::Forbidden("Admin access required".into()));
    }
    Ok(Json(serde_json::json!({"message": "Welcome, admin!"})))
}
```


This hybrid approach delivers **99% of stateless benefits** with **100% of security control** — the perfect balance for production systems that need infinite scale with complete security management.
