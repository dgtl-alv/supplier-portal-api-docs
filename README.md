# Supplier Portal Integration API

OpenAPI contract for Supplier Portal integrations exposed through n8n.

The API provides JSON endpoints for:

- QAD Product Master
- QAD BOM Master
- QAD Supplier List
- Approved supplier list from NSR
- Approved supplier detail from NSR

SOAP transport to QAD and the NSR partner credential stay private behind n8n.

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
x-api-key: ***

{
  "q": "astra",
  "limit": 50,
  "offset": 0
}
```

All fields are optional. Search is case-insensitive and runs in n8n after QAD returns the complete supplier list. It covers supplier code, names, emails, address, city, postal code, type, and active status. `limit` accepts 1–200 and defaults to 50; `offset` defaults to 0.

The response contains paginated rows in `data.suppliers`, filtered count in `data.total`, and unfiltered QAD count in `data.sourceTotal`.

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

Only approved, non-voided suppliers are returned.

### Supplier Detail

```http
GET /supplier-portal/staging/detail-supplier?public_id=a100000000000001
x-api-key: <SUPPLIER_PORTAL_API_KEY>
```

`public_id` is a required opaque identifier containing 16 lowercase hexadecimal characters.

## Architecture

```text
Client
  -> n8n Webhook + Header Auth
     -> QAD bridge through private jump host
     -> NSR partner API through private Header Auth credential
  <- JSON response
```

Clients never send QAD SOAP envelopes or the NSR partner API key.

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
