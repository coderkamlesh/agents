---
name: api-handoff-doc-generator
description: Generates frontend API handoff documentation from recently created or
  modified backend endpoints. This skill should be used when the user asks for API
  documentation, frontend handoff docs, API contract, "API list for frontend",
  "postman docs", or wants the newly built backend APIs written up for the UI team.
  Also trigger on Hinglish requests such as "frontend ke liye api de de",
  "frontend ko dena hai api de de", "api docs bana de", "frontend ke liye api
  documentation bana do", or "api bana de frontend ke liye".
agent_created: true
allowed-tools: Read, Grep, Glob, Bash, Write
---

# API handoff doc generator

## When to use

- The user wants documentation of recently created or modified APIs for the frontend team.
- The user asks for API documentation, frontend handoff, an API contract, or API list for the UI.

Trigger phrases the user actually says, in Hinglish:

- "frontend ke liye api de de"
- "frontend ko dena hai api de de"
- "api docs bana de"
- "frontend ke liye api documentation bana do"
- "api bana de frontend ke liye"
- "frontend handoff doc bana do"

## Core principles

1. **Frontend perspective.** The frontend does not need internal DB structures, class names,
   or implementation details. It needs the exact URL, request payload, headers, and a realistic
   response shape.
2. **Realistic dummy data.** Avoid lazy placeholders such as `string`, `xyz`, or `1`. Use
   real-world domain data: valid JWT structure, ISO dates, realistic amounts, valid enums.
3. **Consistency.** Every endpoint follows exactly the same format so frontend developers can
   parse it without surprises.

## Workflow

1. **Identify the scope.**
   - If the user names a module (for example "wallet APIs" or "user auth APIs"), restrict the
     scan to that module's controllers or routes.
   - If no module is given, extract newly created or modified endpoints from the recent session
     or from `git diff`.

2. **Analyse the project response envelope.**
   - Find the project's global response wrapper: `@RestControllerAdvice` in Java, global
     middleware in Node.js, or a standard `Response<T>` DTO.
   - Make the dummy response follow that exact envelope, for example
     `{ "status": 1, "message": "...", "data": { ... } }`.

3. **Extract endpoint details.** For each endpoint capture:
   - HTTP method and path
   - Auth and role requirements
   - Path variables, query parameters, headers
   - Request body (JSON or form-data)

4. **Generate realistic dummy data.**
   - Use domain-specific values: 10-digit Indian mobile numbers, ISO 8601 timestamps, real
     enum values such as `PENDING` or `APPROVED`.

5. **Save and format.**
   - Group endpoints by module or controller.
   - Save as Markdown at `docs/api-handoff/<module-name>-apis.md`.
   - Create `docs/api-handoff/` automatically if it does not exist.
   - Overwrite existing files with the latest data.

## Strict Mermaid Diagram Rules (Zero-Parse-Error Policy)
When including Mermaid workflow diagrams (`flowchart TD`), strictly follow these compatibility rules to prevent parser crashes on Mermaid 11.x / markdownlivepreview.com:
1. **NO `subgraph`:** Never use `subgraph` blocks. Use a flat, linear flowchart instead.
2. **NO `{}` inside rectangular nodes:** Never put path variables like `{userId}` or curly braces inside `[...]` shapes (e.g. `[GET /users/{id}]` breaks Mermaid tokenizer). Use `:userId` or `<userId>` instead (e.g. `[GET /users/:id]`).
3. **NO quotes inside pipes/delimiters:** Never write `-->|"Yes"|` or `{"Condition"}`.
4. **Clean branch labels:** Use standard `-- Yes, status: 1 -->` or `-->` without nested quotes.
5. **NO Ampersand (`&`):** Never use `&` inside node text or transitions. Always use the word `and`.
6. **Standard node shapes only:**
   - Action/Endpoint: `A[Text here]`
   - Decision/Condition: `C{Is Condition?}`
   - Use `<br/>` for line breaks inside nodes.

## Output format template

### [Number]. [Endpoint purpose]

`[METHOD] /api/v1/[path]` — [One-line purpose]. (Auth: [Requirements, e.g. Bearer Token, Role: `ADMIN`])

**Dummy request**

- **Example URL:** `[METHOD] /api/v1/[path/with-example-values]`
- **Path variables:** (if applicable)
  - `[varName]` ([Type]): [Description] (e.g. `[example-value]`)
- **Query parameters:** (if applicable)
  - `[paramName]` ([Type], [Required/Optional]): [Description] (e.g. `[example-value]`)
- **Headers:**

  ```http
  [Header-Name]: [Example-Value]
  ```

- **Request body:** (for POST / PUT / PATCH)

  ```json
  {
    "field1": "realistic-value"
  }
  ```

**Dummy response ([Status Code] OK / Created)**

```json
{
  "status": "[Project-Specific-Status]",
  "message": "...",
  "data": {
    "exampleField": "realistic-value"
  }
}
```

## Constraints

- **No internal details.** Never include internal class names, database column names, or
  internal method logic.
- **Strict realistic data.**
  - Mobile numbers: 10 digits.
  - Amounts: 2-3 decimal places when currency.
  - Dates: ISO 8601 format (`YYYY-MM-DDTHH:mm:ss`).
- **Confirmation.** After saving the file, print:
  `Saved API handoff docs to docs/api-handoff/<module-name>-apis.md`

## Language

Documentation, headings, and descriptions are written in English.
