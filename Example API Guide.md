# Product Catalog & Pricing API Guide 

A comprehensive reference for integrating with the Product Catalog, Price Books, Rates, and Promotions. This guide covers authentication, versioning, resources, request/response schemas, errors, webhooks, pagination, filtering, and example calls in cURL and JavaScript/TypeScript (fetch/axios).

Use this guide on your GitHub site to help developers build, automate, and audit catalog and pricing workflows.

---

## Getting Started

### Base URL and Versioning

- Base URL: `https://api.example.com`
- Versioning: URI-based and header-based are supported
  - URI versioning (recommended): `/v1/...`
  - Header versioning: `Accept: application/vnd.example+json; version=1`

### Authentication

- Scheme: Bearer token (OAuth 2.0 Client Credentials or PAT)
- Header: `Authorization: Bearer <token>`
- Token scopes:
  - `catalog.read`, `catalog.write`
  - `pricing.read`, `pricing.write`
  - `promotions.write`
  - `admin` (privileged operations)

Example token retrieval (OAuth 2.0 Client Credentials):
```bash
curl -X POST https://auth.example.com/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=$CLIENT_ID&client_secret=$CLIENT_SECRET&scope=catalog.read catalog.write pricing.read pricing.write"
```

### Content Types

- Requests: `Content-Type: application/json`
- Responses: `application/json`
- CSV endpoints (for bulk upload): `text/csv` for downloads, `multipart/form-data` for uploads

### Idempotency

- Header: `Idempotency-Key: <uuid>` for POST/PUT/PATCH calls that create or modify resources
- Idempotency window: 24 hours

---

## Conventions

- Timestamps: ISO 8601 UTC, e.g., `2025-01-31T23:59:59Z`
- Money: Minor units or decimal with currency; fields indicate which
- Currencies: ISO 4217 uppercase, e.g., `USD`, `EUR`
- Pagination: Cursor-based
  - Params: `limit`, `cursor`
  - Headers: `X-Next-Cursor`, `X-Prev-Cursor`
- Filtering: RSQL-like `filter` query or discrete params (documented per endpoint)
- Sorting: `sort=field_name` or `sort=-field_name` (desc)
- Expand: `expand=field1,field2` to include nested related entities
- Errors: Problem+JSON (`application/problem+json`)

---

## Resource Overview

| Resource | Path | Description |
|---|---|---|
| Products | `/v1/products` | Create, read, update, archive products |
| Price Books | `/v1/price-books` | Manage price books and precedence |
| Rates | `/v1/rates` | Effective-dated rates per product/price book/currency |
| Promotions | `/v1/promotions` | Time-bound promotional prices and rules |
| Rate Upload Jobs | `/v1/rate-upload-jobs` | Configure and run bulk CSV uploads |
| Tax Categories | `/v1/tax-categories` | Map products to tax rules |
| Attributes | `/v1/products/{id}/attributes` | Product-level custom attributes |
| Audit Logs | `/v1/audit-logs` | Change history and event feeds |

---

## Products

### Product Schema

```json
{
  "id": "prod_12345",
  "code": "STORAGE-SSD-PREMIUM",
  "name": "Premium SSD Storage",
  "description": "High-performance SSD storage tier.",
  "status": "ACTIVE",
  "lifecycle": "ACTIVE",
  "category": "STORAGE",
  "tags": ["ssd", "premium", "iaas"],
  "visibility": {
    "regions": ["EU", "US"],
    "segments": ["ENTERPRISE", "SMB"],
    "channels": ["DIRECT", "PARTNER"]
  },
  "tax_category_id": "tax_software_saas",
  "gl_account": "4000-REVENUE-STORAGE",
  "created_at": "2025-01-01T12:00:00Z",
  "updated_at": "2025-03-15T08:30:10Z",
  "links": {
    "self": "/v1/products/prod_12345"
  }
}
```

Status values: `DRAFT`, `ACTIVE`, `INACTIVE`, `DEPRECATED`

### List Products

GET `/v1/products?limit=50&cursor=...&filter=status==ACTIVE;category==STORAGE&sort=code`

```bash
curl -H "Authorization: Bearer $TOKEN" \
  "https://api.example.com/v1/products?limit=50&filter=status==ACTIVE;category==STORAGE&sort=code"
```

Response headers may include `X-Next-Cursor`.

### Create Product

POST `/v1/products`
```bash
curl -X POST https://api.example.com/v1/products \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "code": "STORAGE-SSD-PREMIUM",
    "name": "Premium SSD Storage",
    "description": "High-performance SSD storage tier.",
    "category": "STORAGE",
    "status": "DRAFT",
    "visibility": {"regions": ["US","EU"]},
    "tax_category_id": "tax_software_saas",
    "gl_account": "4000-REVENUE-STORAGE"
  }'
```

### Update Product

PATCH `/v1/products/{id}`
```bash
curl -X PATCH https://api.example.com/v1/products/prod_12345 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "ACTIVE", "visibility": {"regions": ["US","EU","APAC"]}}'
```

### Delete/Archive Product

DELETE `/v1/products/{id}` supports soft-delete: `lifecycle` → `DEPRECATED`.
```bash
curl -X DELETE https://api.example.com/v1/products/prod_12345 \
  -H "Authorization: Bearer $TOKEN"
```

### Manage Product Attributes

PUT `/v1/products/{id}/attributes` (replace) or PATCH (merge)
```bash
curl -X PATCH https://api.example.com/v1/products/prod_12345/attributes \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"rate_per_gb": 0.12, "feature_flags": ["snapshots","encryption"], "unit": "GB"}'
```

---

## Price Books

### Price Book Schema

```json
{
  "id": "pb_us_enterprise",
  "name": "US Enterprise Price Book",
  "region": "US",
  "segment": "ENTERPRISE",
  "currency_policy": {
    "allowed": ["USD"],
    "default": "USD"
  },
  "precedence": 200,
  "status": "ACTIVE",
  "created_at": "2025-01-10T10:20:00Z",
  "updated_at": "2025-03-01T11:00:00Z"
}
```

- Precedence: higher number overrides lower ones in conflicts.

### List/Create/Update

- GET `/v1/price-books`
- POST `/v1/price-books`
- PATCH `/v1/price-books/{id}`

Example create:
```bash
curl -X POST https://api.example.com/v1/price-books \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"id":"pb_eu_smb","name":"EU SMB","region":"EU","segment":"SMB","currency_policy":{"allowed":["EUR"],"default":"EUR"},"precedence":150,"status":"ACTIVE"}'
```

---

## Rates

Rates are effective-dated prices per product, price book, and currency.

### Rate Schema

```json
{
  "id": "rate_abc123",
  "product_id": "prod_12345",
  "price_book_id": "pb_us_enterprise",
  "currency": "USD",
  "amount": 125.00,
  "amount_precision": 2,
  "effective_start": "2025-04-01T00:00:00Z",
  "effective_end": null,
  "reason_code": "ANNUAL_REPRICE",
  "created_by": "user_789",
  "created_at": "2025-03-15T12:00:00Z"
}
```

### List Rates

GET `/v1/rates?product_id=prod_12345&price_book_id=pb_us_enterprise&as_of=2025-04-01T00:00:00Z`

```bash
curl -H "Authorization: Bearer $TOKEN" \
  "https://api.example.com/v1/rates?product_id=prod_12345&price_book_id=pb_us_enterprise&as_of=now"
```

### Create or Insert-Replace Rate

POST `/v1/rates`
```bash
curl -X POST https://api.example.com/v1/rates \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "product_id": "prod_12345",
    "price_book_id": "pb_us_enterprise",
    "currency": "USD",
    "amount": 125.00,
    "effective_start": "2025-04-01T00:00:00Z",
    "conflict_handling": "INSERT_END_DATE_PREVIOUS",
    "reason_code": "ANNUAL_REPRICE"
  }'
```

- `conflict_handling`: `OVERWRITE`, `INSERT_END_DATE_PREVIOUS`, `SKIP`

### End-date a Rate

PATCH `/v1/rates/{id}`
```bash
curl -X PATCH https://api.example.com/v1/rates/rate_abc123 \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{ "effective_end": "2025-12-31T23:59:59Z", "reason_code": "PROMO_END" }'
```

---

## Promotions

Promotions define time-bound discount or alternate price logic.

### Promotion Schema

```json
{
  "id": "promo_spring_2025",
  "name": "Spring 2025 Discount",
  "products": ["prod_12345", "prod_67890"],
  "predicate": {
    "segment": ["SMB"],
    "region": ["US"]
  },
  "rules": [
    {"type": "FIXED_PRICE", "amount": 99.00, "currency": "USD"},
    {"type": "MAX_QUANTITY", "value": 100}
  ],
  "stacking_rule": "NO_STACK",
  "start_at": "2025-04-15T00:00:00Z",
  "end_at": "2025-05-15T23:59:59Z",
  "status": "SCHEDULED"
}
```

- Rule types: `PERCENT_OFF`, `AMOUNT_OFF`, `FIXED_PRICE`, `MAX_QUANTITY`
- Stacking: `NO_STACK`, `STACK_PERCENT`, `STACK_LOWEST_PRICE`

### Create Promotion

POST `/v1/promotions`
```bash
curl -X POST https://api.example.com/v1/promotions \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{
    "id":"promo_spring_2025",
    "name":"Spring 2025 Discount",
    "products":["prod_12345"],
    "predicate":{"segment":["SMB"],"region":["US"]},
    "rules":[{"type":"
