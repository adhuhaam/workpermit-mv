# Xpat Mobile API — Reference Library

> **Source:** Official **Xpat MV** mobile app (`mobile-xpat.egov.mv`)  
> **Use:** Hand this file to any AI agent or developer building a lookup app.  
> **Swagger:** https://mobile-xpat.egov.mv/swagger/v1/swagger.json

---

## Authentication

| Item | Value |
|------|--------|
| **Base URL** | `https://mobile-xpat.egov.mv/api/v1` |
| **Header name** | `ApiKey` |
| **Header value** | Your API key (same key embedded in the Xpat MV app) |
| **Not used** | `Authorization: Bearer ...` |

```http
ApiKey: d110e2a8-5adc-4f7b-90a0-701b4fedf476
```

Store the key **server-side only**. Never expose it in mobile/web client bundles or public repos.

---

## Rules

1. **Both** `WorkPermitNumber` and `PassportNumber` are **required** on every lookup.
2. Sending only one parameter returns **400** with an error message.
3. There is **no** bulk/list endpoint — one permit + passport pair per request.
4. Query parameter names are **case-sensitive** as shown below.

---

## Endpoints

### 1. Work permit details (JSON)

| | |
|--|--|
| **Method** | `GET` |
| **Path** | `/WorkPermit` |
| **Full URL** | `https://mobile-xpat.egov.mv/api/v1/WorkPermit` |

**Query parameters**

| Name | Required | Example |
|------|----------|---------|
| `WorkPermitNumber` | Yes | `WP00595305` |
| `PassportNumber` | Yes | `V7255877` |

**Example request (curl)**

```bash
curl -s -G "https://mobile-xpat.egov.mv/api/v1/WorkPermit" \
  -H "ApiKey: YOUR_API_KEY" \
  --data-urlencode "WorkPermitNumber=WP00595305" \
  --data-urlencode "PassportNumber=V7255877"
```

**Success:** `200` — JSON object (see [Response fields](#response-fields-workpermit) below).

**Error:** `400` / `404` etc. — JSON with `errors` array:

```json
{
  "errors": ["You are required to provide passport number and work permit number."]
}
```

---

### 2. Employee photo (image)

| | |
|--|--|
| **Method** | `GET` |
| **Path** | `/WorkPermit/GetImage` |
| **Full URL** | `https://mobile-xpat.egov.mv/api/v1/WorkPermit/GetImage` |

**Query parameters**

| Name | Required | Source |
|------|----------|--------|
| `PhotoId` | Yes | From `photoUrl` in WorkPermit JSON |
| `ServiceId` | Yes | From `photoUrl` in WorkPermit JSON |

The `photoUrl` field in the WorkPermit response is a URL containing `photoId` and `serviceId` query params (case may vary: `photoId` / `PhotoId`, `serviceId` / `ServiceId`).

**Parse example (JavaScript)**

```javascript
function parsePhotoIds(photoUrl) {
  if (!photoUrl) return null;
  const url = new URL(photoUrl);
  const photoId = url.searchParams.get("photoId") ?? url.searchParams.get("PhotoId");
  const serviceId = url.searchParams.get("serviceId") ?? url.searchParams.get("ServiceId");
  if (photoId && serviceId) return { photoId, serviceId };
  return null;
}
```

**Example request (curl)**

```bash
curl -s -G "https://mobile-xpat.egov.mv/api/v1/WorkPermit/GetImage" \
  -H "ApiKey: YOUR_API_KEY" \
  --data-urlencode "PhotoId=PHOTO_ID_FROM_JSON" \
  --data-urlencode "ServiceId=SERVICE_ID_FROM_JSON" \
  -o employee.jpg
```

**Success:** `200` — image bytes (`image/jpeg` or similar).

---

### 3. Work permit card (PNG)

| | |
|--|--|
| **Method** | `GET` |
| **Path** | `/WorkPermitCard/GetWorkPermitCard` |
| **Full URL** | `https://mobile-xpat.egov.mv/api/v1/WorkPermitCard/GetWorkPermitCard` |

**Query parameters**

| Name | Required | Example |
|------|----------|---------|
| `WorkPermitNumber` | Yes | `WP00595305` |
| `PassportNumber` | Yes | `V7255877` |

**Example request (curl)**

```bash
curl -s -G "https://mobile-xpat.egov.mv/api/v1/WorkPermitCard/GetWorkPermitCard" \
  -H "ApiKey: YOUR_API_KEY" \
  --data-urlencode "WorkPermitNumber=WP00595305" \
  --data-urlencode "PassportNumber=V7255877" \
  -o permit-card.png
```

**Success:** `200` — PNG image of the official permit card.

---

## Response fields (WorkPermit)

Typical JSON field names returned by `GET /WorkPermit`:

| Field | Type | Description |
|-------|------|-------------|
| `workPermitNumber` | string | e.g. `WP00595305` |
| `passportNumber` | string | e.g. `V7255877` |
| `fullName` | string | Employee full name |
| `firstName` | string \| null | |
| `middleName` | string \| null | |
| `lastName` | string \| null | |
| `gender` | string \| null | e.g. `Male` |
| `dateOfBirth` | string \| null | ISO date |
| `nationality` | string \| null | e.g. `Indian` |
| `isoAlpha3CountryCode` | string \| null | e.g. `IND` |
| `contactNumber` | string \| null | Employee contact |
| `occupationName` | string \| null | e.g. `Mason, Construction` |
| `isValid` | string \| null | Validity label e.g. `Valid` / `Cancelled` |
| `workPermitStateName` | string \| null | Permit state |
| `workPermitIssuedDate` | string \| null | ISO date |
| `workPermitExpiry` | string \| null | ISO date |
| `employerName` | string \| null | |
| `employerNumber` | string \| null | Registration no. e.g. `C05442019` |
| `employerContactNumber` | string \| null | |
| `photoUrl` | string \| null | URL with photoId + serviceId for GetImage |
| `verifyUrl` | string \| null | Official eGov verification link |

Fields may be `null` if not returned for a record.

---

## TypeScript types (copy-paste)

```typescript
export interface WorkPermitRecord {
  workPermitNumber: string;
  workPermitStateName: string | null;
  occupationName: string | null;
  isValid: string | null;
  fullName: string | null;
  firstName: string | null;
  middleName: string | null;
  lastName: string | null;
  gender: string | null;
  dateOfBirth: string | null;
  passportNumber: string | null;
  isoAlpha3CountryCode: string | null;
  nationality: string | null;
  contactNumber: string | null;
  photoUrl: string | null;
  verifyUrl: string | null;
  workPermitIssuedDate: string | null;
  workPermitExpiry: string | null;
  employerName: string | null;
  employerNumber: string | null;
  employerContactNumber: string | null;
}

export interface ApiErrorResponse {
  errors: string[];
}
```

---

## URL builders (copy-paste)

```typescript
const XPAT_BASE = "https://mobile-xpat.egov.mv/api/v1";

function headers(apiKey: string): HeadersInit {
  return { ApiKey: apiKey };
}

function workPermitUrl(wp: string, passport: string): string {
  const u = new URL(`${XPAT_BASE}/WorkPermit`);
  u.searchParams.set("WorkPermitNumber", wp.trim());
  u.searchParams.set("PassportNumber", passport.trim());
  return u.toString();
}

function cardUrl(wp: string, passport: string): string {
  const u = new URL(`${XPAT_BASE}/WorkPermitCard/GetWorkPermitCard`);
  u.searchParams.set("WorkPermitNumber", wp.trim());
  u.searchParams.set("PassportNumber", passport.trim());
  return u.toString();
}

function imageUrl(photoId: string, serviceId: string): string {
  const u = new URL(`${XPAT_BASE}/WorkPermit/GetImage`);
  u.searchParams.set("PhotoId", photoId);
  u.searchParams.set("ServiceId", serviceId);
  return u.toString();
}
```

---

## Minimal lookup flow

```
1. GET /WorkPermit?WorkPermitNumber=WP...&PassportNumber=...
   → JSON record

2. Parse record.photoUrl → photoId, serviceId
   → GET /WorkPermit/GetImage?PhotoId=...&ServiceId=...
   → employee photo (optional)

3. GET /WorkPermitCard/GetWorkPermitCard?WorkPermitNumber=WP...&PassportNumber=...
   → permit card PNG
```

---

## Test data (known working)

| Work permit | Passport |
|-------------|----------|
| `WP00595305` | `V7255877` |

---

## What this API does NOT provide

- Search by name only
- Search by work permit only (without passport)
- Employee list / employer bulk export
- Write/update operations (read-only)

---

## Disclaimer

Unofficial documentation derived from the public mobile API. Use only for permits you are authorized to view. The API key is tied to the official app; obtain/use it responsibly.
