# Private Memory API - Development Plan

**Scope**: Private Memory service implementation
**Assumption**: Vector stores, files, and RAG pipelines are handled by Cloud API
**Starting Point**: Fork of NEAR AI Chat API with existing auth, conversations, and files infrastructure

---

## Architecture Position

Private Memory API sits **on top of Cloud API** and owns:
- User identity & authentication
- OAuth **provider** (third-party app authorization)
- Projects/Clients management
- Memory search aggregation layer

```
Developer/App → Private Memory API → Cloud API → RAG Service
```

---

## Current State (What Already Exists)

The fork includes a substantial foundation:

### ✅ Already Implemented

| Component | Status | Notes |
|-----------|--------|-------|
| **User Authentication** | ✅ Complete | Google, GitHub, NEAR (NEP-413) OAuth |
| **Session Management** | ✅ Complete | `sess_` tokens, SHA-256 hashing, expiration |
| **Users Table** | ✅ Complete | `id`, `email`, `name`, `avatar_url`, timestamps |
| **OAuth Accounts** | ✅ Complete | Linked credentials (provider + provider_user_id) |
| **User Settings** | ✅ Complete | JSONB flexible storage |
| **Conversations CRUD** | ✅ Complete | Create, list, get, delete, archive, pin, clone |
| **Conversation Items** | ✅ Complete | Proxy to Cloud API |
| **Files CRUD** | ✅ Complete | Upload, list, get, delete (proxied) |
| **Response Proxy** | ✅ Complete | `POST /v1/responses` forwarding to Cloud API |
| **Sharing Infrastructure** | ✅ Complete | Direct, group, organization, public shares |
| **Rate Limiting** | ✅ Complete | Database-backed, hot-reloadable |
| **Activity Logging** | ✅ Complete | User activity tracking with metadata |
| **Admin Routes** | ✅ Complete | User management, analytics, system config |
| **OpenAPI Docs** | ✅ Complete | utoipa auto-generated at `/docs` |

### ❌ Not Yet Implemented (Core Memory API Features)

| Component | Status | Required For |
|-----------|--------|--------------|
| **OAuth Provider** | ❌ Missing | Third-party apps authenticating users |
| **Projects/Clients** | ❌ Missing | Developer management |
| **Scopes System** | ❌ Missing | Fine-grained permissions |
| **Vector Stores Proxy** | ❌ Missing | Memory storage |
| **Memory Search** | ❌ Missing | Aggregation layer (key feature) |
| **CLI Auth Flow** | ❌ Missing | Moltbot integration |

---

## Phase 1: OAuth Provider Implementation

**Goal**: Transform from OAuth *consumer* (users login via Google) to also being an OAuth *provider* (apps get tokens to access user data).

### 1.1 Database Schema Extensions
- [ ] `projects` table (developer projects)
  ```sql
  id UUID PRIMARY KEY,
  owner_user_id UUID REFERENCES users(id),
  name VARCHAR(255) NOT NULL,
  description TEXT,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
  ```
- [ ] `oauth_clients` table
  ```sql
  id UUID PRIMARY KEY,
  project_id UUID REFERENCES projects(id),
  client_id VARCHAR(64) UNIQUE NOT NULL,
  client_secret_hash VARCHAR(255),  -- NULL for public clients
  client_type VARCHAR(20) NOT NULL, -- 'confidential' | 'public'
  allowed_scopes TEXT[] NOT NULL,
  redirect_uris TEXT[] NOT NULL,
  name VARCHAR(255) NOT NULL,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
  ```
- [ ] `oauth_authorization_codes` table (short-lived, ~10 min)
  ```sql
  code_hash VARCHAR(255) PRIMARY KEY,
  client_id VARCHAR(64) NOT NULL,
  user_id UUID NOT NULL,
  redirect_uri TEXT NOT NULL,
  scopes TEXT[] NOT NULL,
  code_challenge VARCHAR(128),      -- PKCE
  code_challenge_method VARCHAR(10), -- 'S256'
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP
  ```
- [ ] `user_access_tokens` table (issued to apps, not sessions)
  ```sql
  id UUID PRIMARY KEY,
  token_hash VARCHAR(255) UNIQUE NOT NULL,
  user_id UUID REFERENCES users(id),
  client_id VARCHAR(64) NOT NULL,
  scopes TEXT[] NOT NULL,
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP
  ```
- [ ] `user_refresh_tokens` table
  ```sql
  id UUID PRIMARY KEY,
  token_hash VARCHAR(255) UNIQUE NOT NULL,
  user_id UUID REFERENCES users(id),
  client_id VARCHAR(64) NOT NULL,
  scopes TEXT[] NOT NULL,
  expires_at TIMESTAMP,             -- NULL = no absolute expiry
  last_used_at TIMESTAMP,
  created_at TIMESTAMP
  ```
- [ ] `user_app_authorizations` table (consent records)
  ```sql
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  client_id VARCHAR(64) NOT NULL,
  granted_scopes TEXT[] NOT NULL,
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  UNIQUE(user_id, client_id)
  ```

### 1.2 Scopes Definition
Define scope constants and validation:
- [ ] `memory.read` - App's own data + user's global memory
- [ ] `memory.read.all` - Cross-app search (requires explicit consent)
- [ ] `files.read` - Read user files
- [ ] `files.write` - Upload files for user
- [ ] `conversations.read` - Read conversations
- [ ] `conversations.write` - Create conversations/messages
- [ ] `profile.read` - Read user profile (email, name)

### 1.3 Authorization Endpoint
```
GET /v1/oauth/authorize
```
- [ ] Query params: `client_id`, `redirect_uri`, `response_type=code`, `scope`, `state`, `code_challenge`, `code_challenge_method`
- [ ] Validate client_id exists and redirect_uri matches registered URIs
- [ ] Validate requested scopes ⊆ client's `allowed_scopes`
- [ ] PKCE required for all clients (`code_challenge_method=S256`)
- [ ] If user not logged in → redirect to login with return URL
- [ ] If user logged in but no prior consent → show consent screen
- [ ] If user has prior consent with same/superset scopes → auto-approve
- [ ] Generate authorization code, store hashed, redirect with code

### 1.4 Token Endpoint
```
POST /v1/oauth/token
```
- [ ] Grant type: `authorization_code`
  - Validate code, client_id, redirect_uri, code_verifier (PKCE)
  - Issue access token (5-15 min TTL)
  - Issue refresh token (confidential clients, or if explicitly enabled)
- [ ] Grant type: `refresh_token`
  - Validate refresh token, client_id
  - Rotate refresh token (return new one, invalidate old)
  - Reuse detection → revoke all tokens for user+client
- [ ] Response format: `{ access_token, token_type, expires_in, refresh_token?, scope }`

### 1.5 Token Validation Middleware
- [ ] New middleware: `app_token_auth_middleware`
  - Extract Bearer token from Authorization header
  - Validate against `user_access_tokens` table
  - Extract `user_id`, `client_id`, `scopes` into request extensions
- [ ] Scope-checking helpers: `require_scope("memory.read")`
- [ ] Distinguish from session auth (existing `auth_middleware`)

### 1.6 User Token Management
```
GET /v1/users/me/authorized-apps
DELETE /v1/users/me/authorized-apps/{client_id}
```
- [ ] List apps user has authorized (from `user_app_authorizations`)
- [ ] Revoke access: delete authorization + all tokens for user+client

---

## Phase 2: Developer/Project Management

### 2.1 Developer Authentication
- [ ] Reuse existing session auth for developers
- [ ] Any authenticated user can create projects (or restrict to verified developers)

### 2.2 Project Endpoints
```
GET|POST    /v1/projects
GET|PATCH|DELETE /v1/projects/{project_id}
```
- [ ] CRUD operations, owner-only access
- [ ] List projects for authenticated user

### 2.3 OAuth Client Management
```
GET|POST    /v1/projects/{project_id}/clients
GET|PATCH|DELETE /v1/projects/{project_id}/clients/{client_id}
POST /v1/projects/{project_id}/clients/{client_id}/rotate-secret
```
- [ ] Create client: generate `client_id`, `client_secret` (for confidential)
- [ ] Configure `allowed_scopes`, `redirect_uris`, `client_type`
- [ ] Secret rotation without invalidating existing tokens
- [ ] Delete client: revoke all tokens

---

## Phase 3: Vector Stores Proxy

### 3.1 Vector Store Endpoints (Proxy to Cloud API)
```
GET|POST    /v1/vector_stores
GET|PATCH|DELETE /v1/vector_stores/{vector_store_id}
GET|POST    /v1/vector_stores/{vector_store_id}/files
GET|DELETE  /v1/vector_stores/{vector_store_id}/files/{file_id}
POST /v1/vector_stores/{vector_store_id}/file_batches
GET  /v1/vector_stores/{vector_store_id}/file_batches/{batch_id}
```
- [ ] Proxy all requests to Cloud API
- [ ] Track vector store ownership in local DB (for access control)
- [ ] Scope validation (`files.write` for mutations)
- [ ] Tag stores with `user_id` and optionally `client_id` (app-specific vs global)

### 3.2 Vector Store Tracking Table
```sql
CREATE TABLE vector_store_ownership (
  vector_store_id VARCHAR(64) PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id),
  client_id VARCHAR(64),  -- NULL = user's global memory
  created_at TIMESTAMP,
  INDEX (user_id, client_id)
);
```

---

## Phase 4: Memory Search Aggregation

**The key value-add of Private Memory API**

### 4.1 Memory Search Endpoint
```
POST /v1/memory/search
```

Request:
```json
{
  "query": "food preferences",
  "limit": 10,
  "sources": ["global", "app"],  // optional filter
  "include_metadata": false
}
```

Response:
```json
{
  "data": [
    {
      "content": "User likes Italian food",
      "score": 0.92,
      "file_id": "file_abc123",
      "chunk_id": "chunk_xyz"
    }
  ]
}
```

### 4.2 Search Logic
- [ ] Parse access token → extract `user_id`, `client_id`, `scopes`
- [ ] Determine searchable stores based on scopes:
  - `memory.read` → user's global stores + current app's stores
  - `memory.read.all` → all stores across all apps
- [ ] Query `vector_store_ownership` to find relevant store IDs
- [ ] Fan out search to Cloud API `/v1/vector_stores/{id}/search` for each store
- [ ] Aggregate results across sources
- [ ] Apply reranking:
  - Exponential decay for recency
  - Boost exact matches
- [ ] Deduplicate (by content hash or semantic similarity)
- [ ] Return top K results

### 4.3 Memory Write (via Responses)
```
POST /v1/responses
POST /v1/chat/completions
```
- [ ] Existing proxy already works
- [ ] Add option: `store_in_memory: true` (default: false)
- [ ] When enabled, extract and index conversation into user's global memory
- [ ] Tag with `client_id` if app-specific storage desired

---

## Phase 5: Enhanced User Data APIs

### 5.1 Conversations (Update Existing)
- [ ] Add scope validation to existing endpoints
- [ ] `conversations.read` required for GET operations
- [ ] `conversations.write` required for POST/PATCH/DELETE
- [ ] Filter list by `client_id` when app token used

### 5.2 Files (Update Existing)
- [ ] Add scope validation
- [ ] `files.read` for GET
- [ ] `files.write` for POST/DELETE
- [ ] Track `client_id` on uploads for app-scoped files

### 5.3 Responses (Update Existing)
- [ ] Add `use_memory` option (inject relevant context from memory search)
- [ ] Add `store_in_memory` option
- [ ] When `use_memory: true`:
  - Call `/v1/memory/search` internally
  - Prepend results to system prompt or context

---

## Phase 6: CLI/Gateway Support (Moltbot Integration)

### 6.1 Device Authorization Flow
Alternative to localhost callback for CLI environments:

```
POST /v1/oauth/device/code
```
- [ ] Returns `device_code`, `user_code`, `verification_uri`, `expires_in`, `interval`

```
POST /v1/oauth/device/token
```
- [ ] Poll with `device_code` until user completes auth
- [ ] Returns tokens when authorized

### 6.2 Localhost Callback Flow (Primary)
```bash
moltbot memory auth
```
- [ ] Document the flow:
  1. CLI starts local server on random port
  2. Opens browser to `/v1/oauth/authorize?redirect_uri=http://localhost:PORT/callback`
  3. User authenticates, consents
  4. Browser redirects to localhost with code
  5. CLI exchanges code for tokens
  6. Store in `~/.moltbot/credentials.json`

### 6.3 CLI Token Refresh
- [ ] Auto-refresh before expiration
- [ ] Handle refresh token rotation
- [ ] Re-auth flow if refresh fails

---

## Phase 7: Migration Support

### 7.1 Bulk Upload Endpoint
```
POST /v1/memory/import
```
- [ ] Accept multipart form with files
- [ ] Process MEMORY.md files (parse structure)
- [ ] Process session transcripts (extract key facts)
- [ ] Batch upload to vector stores
- [ ] Return progress/status

### 7.2 Migration CLI Command
```bash
moltbot memory migrate --source ./MEMORY.md
moltbot memory migrate --source ./sessions/ --format transcript
```
- [ ] Read local files
- [ ] POST to `/v1/memory/import`
- [ ] Progress bar, error handling

---

## Technical Decisions Summary

| Decision | Choice | Notes |
|----------|--------|-------|
| OAuth flow | Authorization Code + PKCE | Required for all clients |
| Refresh tokens | Confidential clients by default | Can enable for public with rotation |
| Access token TTL | 15 minutes | Short-lived for security |
| Refresh token TTL | 30 days absolute | + 7 day inactivity timeout |
| Memory scoping | App's data + user's global | Default for `memory.read` |
| Cross-app search | Requires `memory.read.all` | Explicit consent required |
| Write scoping | All writes to global memory | Unless app requests isolation |
| Deduplication | By content hash | Transparent to caller |
| Reranking | Exponential decay (recency) | + semantic relevance score |
| CLI auth | Localhost callback + device flow | Fallback for environments without browser |

---

## Database Migration Plan

### New Tables (in order)
1. `V16__projects.sql` - Projects table
2. `V17__oauth_clients.sql` - OAuth client registration
3. `V18__oauth_authorization_codes.sql` - Auth codes (ephemeral)
4. `V19__user_tokens.sql` - Access & refresh tokens
5. `V20__user_app_authorizations.sql` - Consent records
6. `V21__vector_store_ownership.sql` - Memory store tracking

### Existing Table Modifications
- `files`: Add `client_id` column (nullable) for app-scoped files
- `conversations`: Add `client_id` column (nullable) for app-scoped conversations

---

## API Routes Summary

```
# Existing (session auth) - No changes needed
GET  /v1/auth/google
GET  /v1/auth/github
POST /v1/auth/near
POST /v1/auth/logout
GET  /v1/users/me
GET|POST|PATCH /v1/users/me/settings

# New: OAuth Provider
GET  /v1/oauth/authorize           # Authorization endpoint
POST /v1/oauth/token               # Token endpoint
POST /v1/oauth/device/code         # Device flow (CLI)
POST /v1/oauth/device/token        # Device flow polling

# New: User Token Management
GET    /v1/users/me/authorized-apps
DELETE /v1/users/me/authorized-apps/{client_id}

# New: Developer Management (session auth)
GET|POST    /v1/projects
GET|PATCH|DELETE /v1/projects/{project_id}
GET|POST    /v1/projects/{project_id}/clients
GET|PATCH|DELETE /v1/projects/{project_id}/clients/{client_id}
POST /v1/projects/{project_id}/clients/{client_id}/rotate-secret

# Updated: User Data (session OR app token auth)
GET|POST    /v1/conversations                    # + scope validation
GET|PATCH|DELETE /v1/conversations/{id}          # + scope validation
GET|POST    /v1/conversations/{id}/items         # + scope validation
POST /v1/responses                               # + use_memory, store_in_memory
POST /v1/chat/completions                        # + use_memory, store_in_memory
GET|POST    /v1/files                            # + scope validation
GET|DELETE  /v1/files/{file_id}                  # + scope validation

# New: Vector Stores (proxy to Cloud API)
GET|POST    /v1/vector_stores
GET|PATCH|DELETE /v1/vector_stores/{vector_store_id}
GET|POST    /v1/vector_stores/{vector_store_id}/files
GET|DELETE  /v1/vector_stores/{vector_store_id}/files/{file_id}

# New: Memory Aggregation (the key feature)
POST /v1/memory/search
POST /v1/memory/import
```

---

## Implementation Priority

### MVP (Minimum Viable Product)
1. **Phase 1.1-1.5**: OAuth provider core (authorization, tokens)
2. **Phase 3**: Vector stores proxy
3. **Phase 4.1-4.2**: Memory search endpoint
4. **Phase 6.2**: CLI localhost auth

### Post-MVP
- Phase 2: Developer portal (projects, client management)
- Phase 4.3: Memory injection into responses
- Phase 5: Scope validation on existing endpoints
- Phase 6.1: Device authorization flow
- Phase 7: Migration tools

---

## Out of Scope (Post-POC)

- Billing & metering
- Privacy audit trail / data export (GDPR)
- Multi-modal memory (images, audio, video)
- Memory sharing (family/team)
- Fine-grained ACLs beyond scopes
- Temporal queries ("what did I say last week")
- Memory versioning/history
- Cross-app suggestions
- Hybrid local+cloud sync mode
- Rate limiting per client (currently per user)
