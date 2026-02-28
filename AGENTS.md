# Documentation project instructions

## About this project

- This is the **external developer documentation** for [Fordje Connect](https://fordje.com), the public API gateway for the Fordje platform
- Built on [Mintlify](https://mintlify.com) — pages are MDX files with YAML frontmatter
- Configuration lives in `docs.json`
- Run `mint dev` to preview locally
- Run `mint broken-links` to check links
- The API source code lives in the sibling `fordje-public-api/` directory — consult it for accurate endpoint behavior, schemas, and error codes

## Audience

External developers integrating with Fordje Connect via M2M (machine-to-machine) API calls. They are building automated systems, not using a browser UI. Assume they are comfortable with REST APIs, JSON, and API key authentication, but may not know the construction/permitting domain.

## Terminology

Use these terms consistently:

| Term | Usage | Notes |
|------|-------|-------|
| **Fordje Connect** | Product name | Not "Connect API", "Fordje API", or "the API" |
| **AHJ** | Authority Having Jurisdiction | Spell out on first use per page, then use "AHJ" |
| **data point** | A single regulatory data value for an AHJ | Two words, lowercase. Not "datapoint" or "data-point" |
| **data point type** | The definition/category of a data point | e.g., "Permit Fee", "Inspection Required" |
| **collection** | A saved configuration of data point types | Owned by the backend, read-only in Connect |
| **category** | A regulatory topic that classifies data point types | e.g., Permitting, Inspections, Building Standards |
| **organization** | The API consumer's company/entity | Not "tenant", "account", or "workspace" |
| **subscription** | The set of AHJs an organization has purchased access to | Not "entitlements" or "plan" |
| **API key** | Authentication credential | Passed via `X-API-Key` header. Not "token" or "secret" |
| **hierarchy** | The parent jurisdiction chain of an AHJ | e.g., city → county → state |

## Style preferences

- Use active voice and second person ("you")
- Keep sentences concise — one idea per sentence
- Use sentence case for headings
- Bold for UI elements: Click **Settings**
- Code formatting for file names, commands, paths, code references, header names, and query parameters
- All endpoint paths in code formatting: `GET /v1/ahjs`
- Show request/response examples as JSON code blocks with language hints
- Always include the `X-API-Key` header in request examples
- Use realistic but fictional data in examples (not "foo", "bar", "test")
- AHJ IDs are integers (e.g., `42`), organization IDs are UUIDs

## API documentation conventions

- Every endpoint page should include: description, authentication, parameters, example request, example response, and error cases
- Document the standard response envelope (`response_type`, `description`, `data`, `warnings`)
- Document partial-access behavior: when some requested AHJs are outside the subscription, the response includes data for authorized AHJs and a `warnings` array listing excluded IDs
- Rate limit headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`) should be documented once in a shared section and referenced from endpoint pages

## Content boundaries

**Do document:**
- All public API endpoints under `/v1/`
- Authentication (API key setup and usage)
- Rate limiting behavior and headers
- Error codes and response formats
- AHJ access control and partial-access warnings
- Collections (listing, detail, execution, export)
- Quickstart / getting started guides
- Code examples in common languages (Python, JavaScript/Node, cURL)

**Do NOT document:**
- Internal admin features or backend endpoints
- How API keys are created/revoked (this happens in the Fordje admin portal, not via the API)
- Backend architecture or database schema
- Internal services (Redis caching, usage logging)
- Code query / chatbot features
- Data runs / approval workflows
- File uploads / document processing pipelines
