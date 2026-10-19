---
name: update-openapi
description: Update OpenAPI specifications based on API changes in the current PR. Use when new API endpoints are added, existing endpoints are modified, or user asks to update/generate OpenAPI/Swagger docs.
---

> **Target project configuration.** Read the target repository's `CLAUDE.md` /
> `AGENTS.md` first. Resolve GitHub `{owner}/{repo}` from its remote or selected PR,
> the Jira account from the authenticated connector, and the local health URL from
> project configuration (ask if missing; never substitute a production URL).
> Commands and paths below describe a reference PHP/CodeIgniter + Prisma + Docker
> layout. Verify services, schemas, base branch and Playwright paths against the
> target project before executing; use its documented equivalents where different.


# Update OpenAPI Specs

Update OpenAPI specifications based on API changes in the current PR.

## File locations

- Main spec: `frontend/apidocs/openapi.yaml`
- Domain-specific paths: `frontend/apidocs/paths/{domain}.yaml`

## Process

### 1. Identify API changes

Find changes in:
- Routes in `application/config/routes.php` matching `api/v1/*` pattern
- Controllers with `api*` methods
- DTOs in `application/Modules/**/DTO/`

**If a PR exists**, use `gh pr diff` for accurate PR-only changes:

```bash
# Check if PR exists for current branch
gh pr view --json number 2>/dev/null

# If PR exists, use gh pr diff (shows exact PR changes)
gh pr diff -- application/config/routes.php
gh pr diff --name-only | grep -E '(controllers/|DTO/)'
```

**Fallback** (no PR yet): use git diff against develop:

```bash
git diff develop...HEAD -- application/config/routes.php
git diff develop...HEAD --name-only | grep -E '(controllers/|DTO/)'
```

Note: `git diff develop...HEAD` can include noise if develop was merged into the branch. Prefer `gh pr diff` when available.

### 2. Extract endpoint information

For each new/modified API endpoint:

**From routes.php:**
- HTTP method (GET, POST, PUT, DELETE)
- Path pattern (e.g., `/api/v1/phases`, `/api/v1/phases/(:num)`)
- Controller method mapping

**From controller methods:**
- Request validation: `$this->validateBody(DTO\ClassName::class)` → request body schema
- Query validation: `$this->validateQuery(DTO\ClassName::class)` → query parameters
- Response serialization: `$this->serializeDto($result)` → response schema
- Status codes: `$this->output->set_status_header(204)` → response codes

**From DTOs:**
- Properties and types from constructor parameters
- Constraints from `#[Assert\*]` attributes:
  - `NotBlank` → required field, minLength: 1
  - `NotNull` → required field
  - `Positive` → minimum: 1
  - `Range(min: X, max: Y)` → minimum/maximum
- Nullable types (`?string`) → `nullable: true`
- PHPDoc `@property` annotations for descriptions

### 3. Generate/update OpenAPI files

**For new domains**, create `frontend/apidocs/paths/{domain}.yaml`:

```yaml
paths:
  /api/v1/{domain}:
    get:
      summary: List {domain}
      operationId: list{Domain}
      tags:
        - {Domain}
      parameters:
        # From query DTO
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Paginated{Domain}'
        '401':
          description: Unauthorized
        '403':
          description: Forbidden

    post:
      summary: Create {item}
      operationId: create{Item}
      tags:
        - {Domain}
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/New{Item}'
      responses:
        '200':
          description: Created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/{Item}'
        '400':
          description: Validation error
        '401':
          description: Unauthorized
        '403':
          description: Forbidden

  /api/v1/{domain}/{id}:
    put:
      summary: Update {item}
      operationId: update{Item}
      tags:
        - {Domain}
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Update{Item}'
      responses:
        '200':
          description: Updated successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/{Item}'
        '400':
          description: Validation error
        '401':
          description: Unauthorized
        '403':
          description: Forbidden
        '404':
          description: Not found

    delete:
      summary: Delete {item}
      operationId: delete{Item}
      tags:
        - {Domain}
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '204':
          description: Deleted successfully
        '401':
          description: Unauthorized
        '403':
          description: Forbidden
        '404':
          description: Not found

components:
  schemas:
    {Item}:
      type: object
      required:
        # Fields without nullable/default
      properties:
        # From output DTO

    New{Item}:
      type: object
      required:
        # Fields with NotBlank/NotNull
      properties:
        # From input DTO

    Update{Item}:
      type: object
      required:
        # Fields with NotBlank/NotNull
      properties:
        # From update DTO

    Paginated{Domain}:
      type: object
      required:
        - page
        - perPage
        - totalItems
        - totalPages
        - items
      properties:
        page:
          type: integer
        perPage:
          type: integer
        totalItems:
          type: integer
        totalPages:
          type: integer
        items:
          type: array
          items:
            $ref: '#/components/schemas/{Item}'
```

**Update `frontend/apidocs/openapi.yaml`:**
- Add path references under `paths:`
- Add new tag under `tags:` if new domain

### 4. Type mappings

| PHP Type | OpenAPI Type |
|----------|--------------|
| `int` | `type: integer` |
| `string` | `type: string` |
| `bool` | `type: boolean` |
| `float` | `type: number` |
| `array` | `type: array` |
| `?T` | add `nullable: true` |
| `DateTime` | `type: string, format: date-time` |

### 5. Validate

Run the linter to check for errors:

```bash
npx @redocly/cli lint frontend/apidocs/openapi.yaml
```

Fix any issues before proceeding.

### 6. Verify alignment with PHP

Spawn a separate agent to verify the OpenAPI spec matches the PHP implementation. The agent should compare:

**Files to check:**
- OpenAPI: `frontend/apidocs/openapi.yaml` and `frontend/apidocs/paths/{domain}.yaml`
- PHP: routes in `application/config/routes.php`, controller methods, and DTOs

**Verification checklist:**

1. **Routes alignment:**
   - Each `api/v1/*` route in routes.php has corresponding OpenAPI path
   - HTTP methods match (GET, POST, PUT, DELETE)
   - Path parameters match (`(:num)` → `{id}`)

2. **Request bodies:**
   - POST/PUT: check `$this->validateBody(DTO\Class::class)` in controller
   - OpenAPI request schema properties match DTO constructor parameters
   - `required` fields match `#[Assert\NotBlank]` or `#[Assert\NotNull]`

3. **Query parameters:**
   - GET: check `$this->validateQuery(DTO\Class::class)` in controller
   - OpenAPI parameters match DTO properties
   - Constraints: `#[Assert\Positive]` → `minimum: 1`, `#[Assert\Range(min, max)]` → min/max

4. **Response schemas:**
   - Check `$this->serializeDto($result)` to identify response DTO
   - OpenAPI response properties match DTO properties
   - List endpoints: verify paginated structure matches PaginatedResult

5. **Types:**
   - `int` → `integer`, `string` → `string`, `bool` → `boolean`
   - `?Type` → `nullable: true`

6. **Status codes:**
   - Check `$this->output->set_status_header()` calls
   - DELETE → 204, others → 200

**Output:** List discrepancies found (file, line, issue) or confirm alignment.

Fix any discrepancies before completing.

### 7. Generate TypeScript API client

After alignment is verified, regenerate the typed API client:

```bash
cd frontend && pnpm generate:api
```

This generates typed SDK functions and types at `frontend/src/core/api/`.

## Conventions

- Security is defined globally in openapi.yaml (sessionAuth cookie) - don't add to individual operations
- Use `$ref` syntax for path references: `'./paths/{domain}.yaml#/paths/~1api~1v1~1{domain}'`
- Path parameter `(:num)` in routes becomes `{id}` with `type: integer`
- operationId format: `list{Domain}`, `create{Item}`, `update{Item}`, `delete{Item}`
- Tags use PascalCase domain name
- All endpoints should have 401/403 responses (auth is required)

## Example

See `frontend/apidocs/paths/phases.yaml` as reference implementation.
