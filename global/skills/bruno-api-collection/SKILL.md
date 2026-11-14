---
name: bruno-api-collection
description: Generate or update a complete Bruno API collection inside docs/bruno/ organized by project services or modules using the latest Bru markup specification. Triggers when the user says "bruno api collection bana de", "create bruno collection", "generate bruno apis", "bruno collection update kar de", or asks an agent to scaffold or sync REST API requests for Bruno from backend controllers, routes, or OpenAPI specs.
---
# Bruno API Collection Generator

Generate a structured, ready-to-import Bruno API collection at `<project-root>/docs/bruno/` with separate folders for each service or module in the codebase, strictly adhering to Bruno's latest `.bru` file format and collection rules.

## When to Use

- User asks to generate a Bruno API collection (e.g., "bruno api collection bana de", "create bruno collection", "docs/bruno me apis bana de").
- User adds or modifies backend modules, microservices, controllers, or REST endpoints and wants `docs/bruno/` synced.
- User wants environment configs, inherited bearer auth, and post-response token extraction scripts configured for Bruno.

## Workflow

### Step 1: Discover Project Structure and Endpoints

1. Locate the project root directory and inspect its architecture (monolith modules, microservices, Spring Boot `@RestController`, Express/NestJS/FastAPI/Go routers, or OpenAPI/Swagger specs).
2. Identify every distinct service or domain module (for example: `auth`, `user`, `wallet`, `kyc`, `orders`).
3. For each endpoint in a module, extract:

   - HTTP method (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`)
   - Full route path (combining controller base path + method path)
   - Path parameters, query parameters, required headers, and auth requirement (public vs bearer token)
   - Request body DTO/schema (generate realistic sample JSON payloads based on field types and validation annotations)
   - Response envelope structure (check if responses wrap data inside `data`, `result`, or return flat JSON)

### Step 2: Scaffold Collection Root at `docs/bruno/`

Create or update the root collection files inside `<project-root>/docs/bruno/`:

```text
docs/bruno/
├── bruno.json
├── collection.bru
├── environments/
│   └── local.bru
└── <module-or-service>/
    ├── folder.bru
    ├── 01-<action>.bru
    └── 02-<action>.bru
```

1. Write `docs/bruno/bruno.json` with mandatory `version: "1"` and `type: "collection"`.
2. Write `docs/bruno/collection.bru` configuring collection-wide `auth { mode: bearer }`, default `Accept: application/json` header, and common variables.
3. Write `docs/bruno/environments/local.bru` with `baseUrl` (matching the project's configured server port, defaulting to `http://localhost:8080`) and `vars:secret` for sensitive tokens.

### Step 3: Create Service or Module Folders

For each service or module discovered in Step 1:

1. Create a kebab-case directory at `docs/bruno/<module-or-service>/`.
2. Create `docs/bruno/<module-or-service>/folder.bru` with `meta` (`name`, `seq`) and `auth { mode: inherit }`.
3. Create one `.bru` file per endpoint inside that module folder using ordered kebab-case names (`01-register.bru`, `02-login.bru`, `03-get-by-id.bru`).
4. On public endpoints (such as login, register, health check, or webhooks), explicitly set `auth: none` inside the HTTP method block. On protected endpoints, set `auth: inherit`.
5. On login or token-refresh endpoints, attach a `script:post-response` block that extracts the access token and stores it in `accessToken` via `bru.setEnvVar`.

### Step 4: Verify Bru Syntax and Completeness

Before finishing, verify every generated file against these rules:

1. `docs/bruno/bruno.json` exists and is valid JSON with `"version": "1"` and `"type": "collection"`.
2. Every module folder has a `folder.bru` file and valid `.bru` request files.
3. No file uses deprecated JSON collection formats (Postman v2.1 or Bruno JSON export); all requests use native `.bru` markup.
4. All path parameters use `:paramName` in the URL paired with a `params:path` block (never `{paramName}` or `{{paramName}}`).
5. Every `body:json` block contains strictly valid JSON (double-quoted keys, no trailing commas).

## Latest Bruno (`.bru`) File Templates

### 1. `docs/bruno/bruno.json`

```json
{
  "version": "1",
  "name": "Project API Collection",
  "type": "collection",
  "ignore": [
    "node_modules",
    ".git"
  ]
}
```

### 2. `docs/bruno/collection.bru`

```bru
meta {
  name: Project API Collection
}

auth {
  mode: bearer
}

auth:bearer {
  token: {{accessToken}}
}

headers {
  Accept: application/json
}
```

### 3. `docs/bruno/environments/local.bru`

```bru
vars {
  baseUrl: http://localhost:8080
}

vars:secret [
  accessToken,
  refreshToken
]
```

### 4. `docs/bruno/<module>/folder.bru`

```bru
meta {
  name: User
  seq: 1
}

auth {
  mode: inherit
}
```

### 5. Public POST Request with Token Extraction (`docs/bruno/auth/01-login.bru`)

```bru
meta {
  name: Login
  type: http
  seq: 1
}

post {
  url: {{baseUrl}}/api/v1/auth/login
  body: json
  auth: none
}

body:json {
  {
    "email": "admin@example.com",
    "password": "Password@123"
  }
}

script:post-response {
  if (res.getStatus() === 200 || res.getStatus() === 201) {
    const body = res.getBody();
    const token = body?.data?.accessToken ?? body?.data?.token ?? body?.accessToken ?? body?.token;
    if (token) {
      bru.setEnvVar("accessToken", token);
    }
  }
}

tests {
  test("Status code is 200", function() {
    expect(res.getStatus()).to.equal(200);
  });
}
```

### 6. Authenticated GET Request with Path and Query Params (`docs/bruno/user/02-get-user.bru`)

```bru
meta {
  name: Get User Details
  type: http
  seq: 2
}

get {
  url: {{baseUrl}}/api/v1/users/:userId?includeRoles=true
  body: none
  auth: inherit
}

params:query {
  includeRoles: true
  ~includeAudit: false
}

params:path {
  userId: 1
}
```

### 7. Multipart File Upload Request (`docs/bruno/kyc/01-upload-document.bru`)

```bru
meta {
  name: Upload KYC Document
  type: http
  seq: 1
}

post {
  url: {{baseUrl}}/api/v1/kyc/documents
  body: multipartForm
  auth: inherit
}

body:multipart-form {
  documentType: PAN
  file: @file(./sample.pdf)
}
```

## Gotchas

- **Auth syntax differs by file type**: In `collection.bru` and `folder.bru`, auth is a top-level block (`auth { mode: bearer }` or `auth { mode: inherit }`). Inside request `.bru` files, `auth: inherit` or `auth: none` is a property inside the HTTP method block (`get { ... }`, `post { ... }`), NOT inside `meta`.
- **Body value naming mismatch**: Inside the HTTP method block, write camelCase (`body: multipartForm`, `body: formUrlEncoded`), while the corresponding body block header uses kebab-case (`body:multipart-form { ... }`, `body:form-urlencoded { ... }`).
- **Path parameter syntax**: Bruno requires `:id` in the URL plus a `params:path { id: 1 }` block. Using Spring/OpenAPI `{id}` style breaks path variable substitution in Bruno.
- **Disabled parameters**: Prefix optional query parameters or headers with `~` (for example, `~pageSize: 20`) instead of commenting them out with `//`.
- **Non-destructive updates**: When regenerating or updating an existing `docs/bruno/` collection, preserve custom environment values and existing `seq` ordering where possible rather than wiping user-configured secrets.
