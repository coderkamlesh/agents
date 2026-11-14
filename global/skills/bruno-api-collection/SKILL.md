---
name: bruno-api-collection
description: Generate or update a complete Bruno API collection inside docs/bruno/ organized by project services or modules using the latest OpenCollection 1.0.0 YAML specification (.yml). Triggers when the user says "bruno api collection bana de", "create bruno collection", "generate bruno apis", "bruno collection update kar de", or asks an agent to scaffold or sync REST API requests for Bruno from backend controllers, routes, or OpenAPI specs.
---
# Bruno API Collection Generator (OpenCollection YAML Standard)

Generate a structured, ready-to-import Bruno API collection at `<project-root>/docs/bruno/` with separate folders for each service or module in the codebase, strictly adhering to Bruno's latest **OpenCollection 1.0.0 (`.yml`)** specification.

## When to Use

- User asks to generate or update a Bruno API collection (e.g., "bruno api collection bana de", "create bruno collection", "docs/bruno me apis bana de").
- User adds or modifies backend modules, microservices, controllers, or REST endpoints and wants `docs/bruno/` synced.
- User wants environment configs, inherited auth, tests, and post-response token extraction scripts configured for Bruno using the new YAML standard.

## Workflow

### Step 1: Discover Project Structure and Endpoints

1. Locate the project root directory and inspect its architecture (monolith modules, microservices, Spring Boot `@RestController`, Express/NestJS/FastAPI/Go routers, or OpenAPI specs).
2. Identify every distinct service or domain module (for example: `auth`, `user`, `wallet`, `kyc`, `aeps`, `hierarchy`).
3. For each endpoint in a module, extract:
   - HTTP method (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`)
   - Full route path (combining controller base path + method path)
   - Path parameters, query parameters, required headers, and auth requirement (public vs authenticated)
   - Request body DTO/schema (generate realistic sample JSON payloads based on field types and validation annotations)
   - Response envelope structure (check if responses wrap data inside `data`, `result`, or return flat JSON)

### Step 2: Scaffold Collection Root at `docs/bruno/`

Create or update the root collection files inside `<project-root>/docs/bruno/`:

```text
docs/bruno/
├── opencollection.yml
├── environments/
│   └── local.yml
└── <module-or-service>/
    ├── folder.yml
    ├── 01-<action>.yml
    └── 02-<action>.yml
```

1. Write `docs/bruno/opencollection.yml` with `opencollection: 1.0.0`, collection metadata, default collection headers (e.g. `Accept: application/json`), default auth, and Bruno extensions.
2. Write `docs/bruno/environments/local.yml` with `baseUrl` (matching configured server port, defaulting to `http://localhost:8080`) and any secret/environment variables.

### Step 3: Create Service or Module Folders

For each service or module discovered in Step 1:

1. Create a kebab-case directory at `docs/bruno/<module-or-service>/`.
2. Create `docs/bruno/<module-or-service>/folder.yml` with `info` (`name`, `type: folder`, `seq`) and `request.auth` (`inherit` or `none`).
3. Create one `.yml` file per endpoint inside that module folder using ordered kebab-case names (`01-register.yml`, `02-login.yml`, `03-get-by-id.yml`).
4. On public endpoints (login, register, health checks, webhooks), explicitly set `auth: none` inside the `http:` block. On protected endpoints, set `auth: inherit`.
5. On login or token-refresh endpoints, attach an `after-response` script inside `runtime.scripts` to extract the access token and store it in environment variables via `bru.setEnvVar`.

### Step 4: Verify OpenCollection YAML Syntax and Completeness

Before finishing, verify every generated file against these rules:

1. **Strictly YAML**: All collection files must use `.yml` extension. Never generate legacy `.bru` files or deprecated JSON formats (`bruno.json`, `collection.bru`, Postman JSON).
2. **Root specification**: `docs/bruno/opencollection.yml` must start with `opencollection: 1.0.0`.
3. **Folder structure**: Every module folder must have a valid `folder.yml`.
4. **Path parameters**: Use `:paramName` in the URL (e.g., `/api/v1/users/:userId`), and declare each in `http.params` with `type: path`. Never use `{paramName}` or `{{paramName}}` in the route template.
5. **Valid JSON in data block**: Every `body.data` block with `type: json` must contain strictly valid JSON (double-quoted keys, no trailing commas) inside a YAML block scalar (`|-`).
6. **Consistent settings**: Include standard Bruno execution settings (`encodeUrl`, `timeout`, `followRedirects`, `forwardAuthorizationHeader`).

---

## OpenCollection 1.0.0 (`.yml`) File Templates

### 1. Collection Root (`docs/bruno/opencollection.yml`)

```yaml
opencollection: 1.0.0

info:
  name: Project API Collection

request:
  headers:
    - name: Accept
      value: application/json
  auth:
    type: bearer
    token: "{{accessToken}}"
bundled: false
extensions:
  bruno:
    ignore:
      - node_modules
      - .git
```

### 2. Environment Config (`docs/bruno/environments/local.yml`)

```yaml
variables:
  - name: baseUrl
    value: http://localhost:8080
  - name: defaultUserId
    value: 00000000-0000-0000-0000-000000000001
  - secret: true
    name: accessToken
  - secret: true
    name: refreshToken
```

### 3. Folder Config (`docs/bruno/<module>/folder.yml`)

```yaml
info:
  name: User
  type: folder
  seq: 1

request:
  auth: inherit
```

### 4. Public POST Request with Token Extraction (`docs/bruno/auth/01-login.yml`)

```yaml
info:
  name: Login
  type: http
  seq: 1

http:
  method: POST
  url: "{{baseUrl}}/api/v1/auth/login"
  body:
    type: json
    data: |-
      {
        "email": "user@example.com",
        "password": "Password@123"
      }
  auth: none

runtime:
  scripts:
    - type: after-response
      code: |-
        if (res.getStatus() === 200 || res.getStatus() === 201) {
          const body = res.getBody();
          const token = body?.data?.accessToken ?? body?.data?.token ?? body?.accessToken ?? body?.token;
          if (token) {
            bru.setEnvVar("accessToken", token);
          }
        }
    - type: tests
      code: |-
        test("Status code is 200", function() {
          expect(res.getStatus()).to.equal(200);
        });

settings:
  encodeUrl: true
  timeout: 0
  followRedirects: true
  maxRedirects: 5
  forwardAuthorizationHeader: true
```

### 5. Authenticated GET Request with Path & Query Params (`docs/bruno/user/02-get-user.yml`)

```yaml
info:
  name: Get User Details
  type: http
  seq: 2

http:
  method: GET
  url: "{{baseUrl}}/api/v1/users/:userId"
  headers:
    - name: X-User-Id
      value: "{{defaultUserId}}"
  params:
    - name: userId
      value: "{{defaultUserId}}"
      type: path
    - name: includeDetails
      value: "true"
      type: query
  auth: inherit

runtime:
  scripts:
    - type: tests
      code: |-
        test("Status code is 200", function() {
          expect(res.getStatus()).to.equal(200);
        });

settings:
  encodeUrl: true
  timeout: 0
  followRedirects: true
  maxRedirects: 5
  forwardAuthorizationHeader: true
```

### 6. Authenticated POST Request with JSON Body (`docs/bruno/wallet/01-debit.yml`)

```yaml
info:
  name: Debit Wallet
  type: http
  seq: 1

http:
  method: POST
  url: "{{baseUrl}}/api/v1/wallets/debit"
  headers:
    - name: X-User-Id
      value: "{{defaultUserId}}"
    - name: Idempotency-Key
      value: txn-idempotency-001
  body:
    type: json
    data: |-
      {
        "amount": 500.00,
        "serviceType": "AEPS_CASH_WITHDRAWAL",
        "narration": "AEPS Cash Withdrawal"
      }
  auth: inherit

runtime:
  scripts:
    - type: tests
      code: |-
        test("Status code is 200", function() {
          expect(res.getStatus()).to.equal(200);
        });

settings:
  encodeUrl: true
  timeout: 0
  followRedirects: true
  maxRedirects: 5
  forwardAuthorizationHeader: true
```

### 7. Multipart File Upload Request (`docs/bruno/kyc/01-upload-document.yml`)

```yaml
info:
  name: Upload Document
  type: http
  seq: 1

http:
  method: POST
  url: "{{baseUrl}}/api/v1/kyc/documents"
  headers:
    - name: X-User-Id
      value: "{{defaultUserId}}"
  body:
    type: multipartForm
    data:
      - name: documentType
        value: PAN_CARD
        type: text
      - name: file
        value: ./sample-document.pdf
        type: file
  auth: inherit

runtime:
  scripts:
    - type: tests
      code: |-
        test("Status code is 201", function() {
          expect(res.getStatus()).to.equal(201);
        });

settings:
  encodeUrl: true
  timeout: 0
  followRedirects: true
  maxRedirects: 5
  forwardAuthorizationHeader: true
```

---

## Key Gotchas & Rules

- **Strictly No `.bru` Files**: Do not generate `.bru` or `bruno.json` files under any circumstance. All files are YAML (`.yml`) conforming to `opencollection: 1.0.0`.
- **Single Root Spec**: Root configuration is handled entirely by `opencollection.yml`. There is no separate `collection.yml`.
- **Param Types**: Bruno parses path params and query params from the `http.params` list via `type: path` and `type: query`.
- **Scripts Location**: All lifecycle scripts (`before-request`, `after-response`, `tests`) reside inside `runtime.scripts` as a list with explicit `type:` and `code: |-`.
- **Non-Destructive Sync**: When updating an existing collection, keep existing environment variables and user-defined tokens intact.
