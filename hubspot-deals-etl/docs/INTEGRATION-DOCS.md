# 📋 HubSpot Deals ETL - HubSpot CRM API v3 Integration (Deals)

This document describes the HubSpot REST API endpoints required to extract **Deals** data from a HubSpot portal using **HubSpot CRM API v3**.

---

## 📋 Overview

The HubSpot Deals ETL integrates with HubSpot’s CRM **Objects API** to page through deals and retrieve the required deal properties.

### ✅ **Required Endpoint (Essential)**
| **API Endpoint** | **Purpose** | **Version** | **Required Scopes** | **Usage** |
|---|---|---:|---|---|
| `/crm/v3/objects/deals` | List/paginate deals with selected properties | v3 | `crm.objects.deals.read` | **Required** |

### 🔧 **Optional Endpoints (Recommended for Robustness)**
| **API Endpoint** | **Purpose** | **Version** | **Required Scopes** | **Usage** |
|---|---|---:|---|---|
| `/crm/v3/properties/deals` | Retrieve all deal properties (default + custom) | v3 | `crm.objects.deals.read` | Optional |
| `/crm/v3/objects/deals/{dealId}` | Fetch a single deal by ID | v3 | `crm.objects.deals.read` | Optional |
| `/crm/v3/objects/deals/search` | Filtered extraction (by pipeline, stage, updated time, etc.) | v3 | `crm.objects.deals.read` | Optional |

### 🎯 **Recommendation**
Start with **`GET /crm/v3/objects/deals`** (plus **`GET /crm/v3/properties/deals`** once at startup to discover available properties). This keeps extraction simple, efficient, and resilient to portal-specific customization.

---

## 🔐 Authentication Requirements

### **Private App Access Token (Recommended for single-portal integrations)**

All requests to HubSpot APIs use the standard bearer token header:

```http
Authorization: Bearer <HUBSPOT_PRIVATE_APP_ACCESS_TOKEN>
Content-Type: application/json
```

### **Required Scopes**
- **`crm.objects.deals.read`**: Read access to deal records.

---

## 🌐 HubSpot API Endpoints

### 🎯 **PRIMARY ENDPOINT (Required for Basic Deals Extraction)**

### 1. **List Deals** - `/crm/v3/objects/deals` ✅ **REQUIRED**

**Purpose**: Retrieve a paginated list of deals with selected properties.

**Method**: `GET`

**Base URL**: `https://api.hubapi.com`

**URL**: `https://api.hubapi.com/crm/v3/objects/deals`

**Query Parameters**:
- **`limit`**: Maximum number of deals per page.
- **`after`**: Cursor token for the next page (from `paging.next.after`).
- **`properties`**: Comma-separated list of deal properties to return (unknown/invalid names are ignored).
- **`archived`**: `true|false` (default `false`) to include only archived results.

> Note: HubSpot also supports `associations` and `propertiesWithHistory` on this endpoint, but they are **not required** for this ETL unless you explicitly need them.

**Request Example**:
```http
GET https://api.hubapi.com/crm/v3/objects/deals?limit=100&archived=false&properties=dealname,amount,closedate,pipeline,dealstage,hs_object_id,hs_lastmodifieddate
Authorization: Bearer <HUBSPOT_PRIVATE_APP_ACCESS_TOKEN>
Content-Type: application/json
```

**Response Structure**:

```json
{
  "results": [
    {
      "id": "123456789",
      "properties": {
        "dealname": "ACME - Renewal 2026",
        "amount": "1500.00",
        "closedate": "2026-03-20T00:00:00.000Z",
        "pipeline": "default",
        "dealstage": "contractsent",
        "hs_object_id": "123456789",
        "hs_lastmodifieddate": "2026-03-22T10:12:11.123Z"
      },
      "createdAt": "2026-01-10T08:12:30.000Z",
      "updatedAt": "2026-03-22T10:12:11.123Z",
      "archived": false
    }
  ],
  "paging": {
    "next": {
      "after": "123456790",
      "link": "?after=123456790"
    }
  }
}
```

**Pagination rule**:
- If `paging.next.after` exists, call the same endpoint again with `after=<paging.next.after>`.
- Stop when the response omits `paging.next`.

---

## 🧾 Deal Properties (How to list “all available properties”)

HubSpot portals can have **custom deal properties**, so the only reliable way to list *all* available deal properties for a given account is to query the properties endpoint (instead of hardcoding a static list).

**Best-practice interpretation of “list all available deal properties”**:
- **Per portal (most accurate)**: call `GET /crm/v3/properties/deals` and treat the response as the source of truth (includes default + custom properties).
- **From HubSpot docs (reference only)**: HubSpot maintains documentation about default deal properties, but it may not reflect customizations in your portal. Use it for general understanding, not for extraction logic.

### 2. **List Deal Properties** - `/crm/v3/properties/deals` 🔧 **OPTIONAL (Recommended)**

**Purpose**: Retrieve all deal property definitions for the portal (default + custom), including types and labels.

**Method**: `GET`

**URL**: `https://api.hubapi.com/crm/v3/properties/deals`

**Request Example**:
```http
GET https://api.hubapi.com/crm/v3/properties/deals
Authorization: Bearer <HUBSPOT_PRIVATE_APP_ACCESS_TOKEN>
Content-Type: application/json
```

**Response (shape)**
- The response is an array of property definitions. Each entry includes fields like `name`, `label`, `type`, `fieldType`, and flags such as `hasUniqueValue`.

**Best practice for ETL**
- **Fetch once at startup** and cache the results (property definitions change infrequently).
- Use the returned `name` values to construct the `properties` parameter for the deals list endpoint.
- For very wide schemas, prefer selecting a curated subset of properties rather than requesting everything.

### Common “core” deal properties (returned by default on list)
HubSpot indicates the following deal properties are returned by default when listing deals:
- `dealname`
- `amount`
- `closedate`
- `pipeline`
- `dealstage`

---

## 📊 Data Extraction Flow

### 🎯 **SIMPLE FLOW (Recommended)**
Use **one endpoint** and cursor-based pagination.

```python
import requests

BASE_URL = "https://api.hubapi.com"

def extract_all_deals(access_token: str, properties: list[str], limit: int = 100, archived: bool = False):
    headers = {
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/json",
    }

    after = None
    all_results = []

    while True:
        params = {
            "limit": limit,
            "archived": str(archived).lower(),
            "properties": ",".join(properties),
        }
        if after is not None:
            params["after"] = after

        resp = requests.get(f"{BASE_URL}/crm/v3/objects/deals", headers=headers, params=params, timeout=60)
        resp.raise_for_status()

        data = resp.json()
        all_results.extend(data.get("results", []))

        paging = data.get("paging", {})
        next_ = paging.get("next")
        if not next_ or "after" not in next_:
            break

        after = next_["after"]

    return all_results
```

---

## ⚡ Performance Considerations

### **Rate limiting**
HubSpot enforces burst (per 10 seconds) and daily limits depending on portal tier and distribution type. For **privately distributed apps / legacy private apps**, the docs provide typical limits such as:
- **Per 10 seconds (burst)**: 100 (Free/Starter) or 190 (Professional/Enterprise) requests per app
- **Per day**: shared across apps within the same HubSpot account

HubSpot also returns rate limit headers on many endpoints, for example:
- `X-HubSpot-RateLimit-Interval-Milliseconds`
- `X-HubSpot-RateLimit-Max`
- `X-HubSpot-RateLimit-Remaining`
- `X-HubSpot-RateLimit-Daily`
- `X-HubSpot-RateLimit-Daily-Remaining`

### **Best practices**
- **Use the highest safe `limit`** to reduce total API calls.
- **Throttle** to stay below the 10-second rolling limit.
- **Cache** reference data like property definitions.
- **Avoid high concurrency** unless you have explicit throughput requirements.

---

## 🚨 Error Handling (Recommended behavior)

HubSpot errors are JSON and intended to be human readable. Treat all fields as optional when parsing.

### Sample error response (generic)
```json
{
  "status": "error",
  "message": "This will be a human readable message with details about the error.",
  "errors": [
    {
      "message": "discount was not a valid number",
      "code": "INVALID_INTEGER",
      "context": {
        "propertyName": ["discount"]
      }
    }
  ],
  "category": "VALIDATION_ERROR",
  "correlationId": "a43683b0-5717-4ceb-80b4-104d02915d8c"
}
```

### Common status codes to handle
- **`400 Bad Request`**: invalid formatting/values (e.g., bad property name in certain APIs).
- **`401 Unauthorized`**: invalid/expired token.
- **`403 Forbidden`**: missing scopes (e.g., missing `crm.objects.deals.read`).
- **`414 URI Too Long`**: too many query parameters / too long URL (reduce number of properties per request).
- **`423 Locked`**: high-volume sync locking; pause at least 2 seconds before retrying.
- **`429 Too Many Requests`**: rate limited; **respect `Retry-After`** header before retrying.
- **`5xx` / timeouts (502/503/504/524/etc.)**: transient HubSpot load; pause then retry with backoff.
- **`477 Migration in Progress`**: portal migration; retry after the `Retry-After` delay.

### Sample `429` rate limit response (shape)
```json
{
  "status": "error",
  "message": "You have reached your daily limit.",
  "errorType": "RATE_LIMIT",
  "correlationId": "c033cdaa-2c40-4a64-ae48-b4cec88dad24",
  "policyName": "DAILY",
  "requestId": "3d3e35b7-0dae-4b9f-a6e3-9c230cbcf8dd"
}
```

### Retry strategy (best practice)
- For **`429` and transient `5xx`**: exponential backoff with jitter, and always honor `Retry-After` if present.
- Set a hard cap on retries and log the HubSpot `correlationId` when present for debugging/support.

---

## 🧪 Testing API Integration (quick checks)

### Test listing deals
```bash
curl -X GET "https://api.hubapi.com/crm/v3/objects/deals?limit=5&archived=false&properties=dealname,amount,closedate,pipeline,dealstage" ^
  -H "Authorization: Bearer <HUBSPOT_PRIVATE_APP_ACCESS_TOKEN>" ^
  -H "Content-Type: application/json"
```

### Test listing deal properties
```bash
curl -X GET "https://api.hubapi.com/crm/v3/properties/deals" ^
  -H "Authorization: Bearer <HUBSPOT_PRIVATE_APP_ACCESS_TOKEN>" ^
  -H "Content-Type: application/json"
```

---

## 📞 Support Resources

- **Deals API (v3)**: `https://developers.hubspot.com/docs/api-reference/crm-deals-v3/guide`
- **List deals endpoint (OpenAPI reference)**: `https://developers.hubspot.com/docs/api-reference/crm-deals-v3/basic/get-crm-v3-objects-0-3`
- **Properties API (v3)**: `https://developers.hubspot.com/docs/api-reference/crm-properties-v3/guide`
- **API usage limits**: `https://developers.hubspot.com/docs/api/usage-details`
- **Error handling**: `https://developers.hubspot.com/docs/api-reference/error-handling`