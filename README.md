# Supplier Portal Integration API

OpenAPI contract for Supplier Portal integrations exposed through n8n.

The API provides JSON endpoints for:

- QAD Product Master
- QAD BOM Master
- QAD Supplier List
- ailox Product-Supplier List
- ailox Supplier Master List
- ailox Supplier Detail
- Approved supplier list from NSR
- Approved supplier detail from NSR

SOAP transport to QAD, database access, and the NSR partner credential stay private behind n8n.

## Documentation

- Swagger UI: https://hfkjs.alvaauto.com/webhook/supplier-portal/staging/api-docs
- OpenAPI JSON: https://hfkjs.alvaauto.com/webhook/supplier-portal/staging/openapi.json
- Versioned contract: [`supplier-portal-openapi.json`](./supplier-portal-openapi.json)

The OpenAPI contract can be imported into Swagger UI, Postman, Insomnia, or another OpenAPI 3.1-compatible tool.

## Base URL

```text
https://hfkjs.alvaauto.com/webhook
```

## Authentication

Every business endpoint requires native n8n Header Auth:

```http
x-api-key: <SUPPLIER_PORTAL_API_KEY>
```

Do not commit API keys, place them in URLs, or include them in logs and tickets.

The documentation endpoints are public and contain no credentials.

## Endpoints

### Product Master

```http
POST /supplier-portal/staging/product-master
Content-Type: application/json
x-api-key: <SUPPLIER_PORTAL_API_KEY>

{}
```

Returns normalized QAD product rows in `data.products`.

### BOM Master

```http
POST /supplier-portal/staging/bom-master
Content-Type: application/json
x-api-key: <SUPPLIER_PORTAL_API_KEY>

{
  "productcode": "PRODUCT_CODE"
}
```

`productcode` is required and limited to 100 characters. Returns normalized QAD component rows in `data.components`.

QAD business failures may return HTTP `200` with `success: false`. Clients must inspect the `success` field.

### QAD Supplier List

```http
POST /supplier-portal/staging/qad-supplier-list
Content-Type: application/json
x-api-key: <SUPPLIER_PORTAL_API_KEY>

{
  "q": "astra",
  "limit": 50,
  "offset": 0
}
```

All fields are optional. Search is case-insensitive and runs in n8n after QAD returns the complete supplier list. It covers supplier code, names, emails, address, city, postal code, type, and active status. `limit` accepts 1–200 and defaults to 50; `offset` defaults to 0.

The response contains paginated rows in `data.suppliers`, filtered count in `data.total`, and unfiltered QAD count in `data.sourceTotal`. QAD contracts remain unchanged until connectivity is repaired.

### ailox Product-Supplier List

```http
POST /supplier-portal/staging/list-product
Content-Type: application/json
x-api-key: <SUPPLIER_PORTAL_API_KEY>

{
  "q": "bolt",
  "item_number": "",
  "item_name": "",
  "model": "P2.0",
  "supplier_code": "",
  "supplier_name": "astra",
  "limit": 50,
  "offset": 0
}
```

All fields are optional. `q` searches item number, item name, model, supplier code, and supplier name. Specific fields, including `model`, apply additional case-insensitive substring filters. The response includes `model` as comma-separated model names. `limit` accepts 1–200 and defaults to 50; `offset` defaults to 0. Supplier names containing `vmoto` are excluded.

The response returns rows in `data` and pagination/filter details in `meta`. Empty results return HTTP `200` with `data: []`, `total: 0`, and `count: 0`.

### ailox Supplier Master List

```http
POST /supplier-portal/staging/aliox-supplier-list
Content-Type: application/json
x-api-key: ***

{
  "q": "rubberindo",
  "supplier_code": "",
  "supplier_name": "PT. Aneka Rubberindo Piranti Sejati",
  "city": "",
  "country": "",
  "status": "ACTIVE",
  "linked_to_pfep": true,
  "limit": 50,
  "offset": 0
}
```

All fields are optional. `q` searches supplier identity, location, business contact, type, and status fields. Specific fields apply additional case-insensitive substring filters. `linked_to_pfep` selects suppliers with or without PFEP mappings. `limit` accepts 1–200 and defaults to 50; `offset` defaults to 0. Supplier names containing `vmoto` are excluded.

Rows include supplier business details, operational lead-time fields, source update timestamp, and `pfep_rows`. Empty results return HTTP `200` with `data: []`, `total: 0`, and `count: 0`.

### ailox Supplier Detail

```http
GET /supplier-portal/staging/aliox-supplier-detail?supplier_code=A1D001
x-api-key: ***
```

`supplier_code` is required, case-insensitive, and limited to 100 characters. The response includes supplier business/contact fields, logistics settings, source timestamps, and `pfep_rows`.

Known suppliers return HTTP `200` with `success: true`. Unknown suppliers return HTTP `404` with error code `SUPPLIER_NOT_FOUND`.

### List Suppliers

```http
POST /supplier-portal/staging/list-supplier
Content-Type: application/json
x-api-key: <SUPPLIER_PORTAL_API_KEY>

{
  "q": "Contoh",
  "limit": 50,
  "offset": 0
}
```

All fields are optional:

- `q`: case-insensitive company-name search
- `limit`: integer from 1 to 200; default 50
- `offset`: non-negative integer; default 0

Only approved, non-voided suppliers are returned. Rows are in `data`; pagination and filters are in `meta`.

### Supplier Detail

```http
GET /supplier-portal/staging/detail-supplier?public_id=a100000000000001
x-api-key: <SUPPLIER_PORTAL_API_KEY>
```

`public_id` is a required opaque identifier containing 16 lowercase hexadecimal characters. Successful details are in `data`; unknown suppliers return the standard error envelope with HTTP `404`.

## Architecture

```text
Client
  -> n8n Webhook + Header Auth
     -> QAD bridge through private jump host
     -> NSR partner API through private Header Auth credential
  <- JSON response
```

Clients never send QAD SOAP envelopes or the NSR partner API key.

## Standard Envelope (NSR and ailox)

Successful NSR and ailox endpoints return:

```json
{
  "success": true,
  "data": {},
  "meta": {}
}
```

Workflow-handled errors return:

```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "..."
  },
  "meta": {}
}
```

Validation errors use HTTP `400`; unknown suppliers use `404`; NSR upstream failures use `401`, `429`, `502`, or `504`; internal database failures use `500`. Native n8n Header Auth rejects missing or invalid credentials before workflow execution with HTTP `403` and a plain-text body.

QAD endpoints are excluded from this envelope migration while QAD connectivity is unresolved.

## Error Behavior

- `403`: missing or invalid n8n Header Auth
- `400`: invalid request parameter where supported by the upstream API
- `401`: NSR partner credential missing, invalid, or revoked
- `404`: supplier is unknown, not approved, or voided
- `429`: NSR rate limit exceeded
- `500`: request or upstream failure

QAD application errors use these JSON codes:

- `QAD_REJECTED`
- `QAD_SOAP_FAULT`
- `QAD_HTTP_ERROR`
- `QAD_INVALID_RESPONSE`

See [`supplier-portal-openapi.json`](./supplier-portal-openapi.json) for complete request, response, and schema definitions.

## Updating Documentation

1. Update `supplier-portal-openapi.json` when an endpoint contract changes.
2. Validate the JSON and OpenAPI references.
3. Update the manual OpenAPI copy in the n8n API Docs workflow.
4. Verify both documentation URLs return HTTP `200`.
5. Confirm business endpoints still reject requests without `x-api-key`.
