# VitalSync API Reference

> **VitalSync Internal Documentation** | Version 2.1 | Last Updated: June 2026
>
> Complete API reference for the VitalSync personalized healthcare supplement recommendation platform. All endpoints are RESTful, accept and return JSON, and require authentication unless noted otherwise.

---

## Table of Contents

- [Overview](#overview)
  - [Base URL](#base-url)
  - [Authentication](#authentication-overview)
  - [Common Headers](#common-headers)
  - [Error Response Format](#error-response-format)
  - [Rate Limiting](#rate-limiting)
- [Authentication Endpoints](#authentication-endpoints)
  - [POST /api/auth/register](#post-apiauthregister)
  - [POST /api/auth/login](#post-apiauthlogin)
  - [POST /api/auth/refresh](#post-apiauthrefresh)
- [WHOOP Integration](#whoop-integration)
  - [GET /api/whoop/auth](#get-apiwhooopauth)
  - [GET /api/whoop/callback](#get-apiwhooopcallback)
  - [GET /api/whoop/data](#get-apiwhooopdata)
  - [GET /api/whoop/signals](#get-apiwhooopsignals)
  - [GET /api/whoop/status](#get-apiwhooopstatus)
- [Biomarkers](#biomarkers)
  - [POST /api/biomarkers/report](#post-apibiomarkersreport)
  - [GET /api/biomarkers/report/:id](#get-apibiomarkersreportid)
  - [GET /api/biomarkers/reports](#get-apibiomarkersreports)
  - [GET /api/biomarkers/classify](#get-apibiomarkersclassify)
- [Recommendations](#recommendations)
  - [POST /api/recommendations/generate](#post-apirecommendationsgenerate)
  - [GET /api/recommendations/latest](#get-apirecommendationslatest)
  - [GET /api/recommendations/history](#get-apirecommendationshistory)
- [User Profile](#user-profile)
  - [GET /api/user/profile](#get-apiuserprofile)
  - [PUT /api/user/profile](#put-apiuserprofile)
  - [POST /api/user/medications](#post-apiusermedications)

---

## Overview

### Base URL

| Environment | Base URL |
|---|---|
| **Production** | `https://api.vitalsync.io/v1` |
| **Staging** | `https://api-staging.vitalsync.io/v1` |
| **Development** | `http://localhost:3000/api` |

All endpoint paths in this document are relative to the base URL. For local development, replace the base accordingly.

### Authentication Overview

VitalSync uses **JWT-based authentication** with short-lived access tokens and long-lived refresh tokens:

| Token Type | Lifetime | Storage | Purpose |
|---|---|---|---|
| **Access Token** | 15 minutes | Client memory (never localStorage) | Authenticates API requests |
| **Refresh Token** | 7 days | HTTP-only secure cookie | Obtains new access tokens |

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Auth Service
    participant API as API Server

    C->>A: POST /api/auth/login (credentials)
    A-->>C: { accessToken, refreshToken (cookie) }
    C->>API: GET /api/user/profile (Bearer accessToken)
    API-->>C: 200 OK (profile data)
    Note over C: Access token expires (15 min)
    C->>A: POST /api/auth/refresh (refreshToken cookie)
    A-->>C: { new accessToken, new refreshToken (cookie) }
```

### Common Headers

| Header | Value | Required |
|---|---|---|
| `Content-Type` | `application/json` | Yes (for POST/PUT) |
| `Authorization` | `Bearer <access_token>` | Yes (unless noted) |
| `Accept` | `application/json` | Recommended |
| `X-Request-ID` | UUID v4 | Optional (for tracing) |
| `X-Client-Version` | `2.1.0` | Optional (for analytics) |

### Error Response Format

All error responses follow a consistent structure:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable error description",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ],
    "requestId": "req_7f8a9b2c-1234-5678-9abc-def012345678"
  }
}
```

#### Standard Error Codes

| HTTP Status | Error Code | Description |
|---|---|---|
| `400` | `VALIDATION_ERROR` | Request body validation failed |
| `401` | `UNAUTHORIZED` | Missing or invalid access token |
| `403` | `FORBIDDEN` | Authenticated but insufficient permissions |
| `404` | `NOT_FOUND` | Requested resource does not exist |
| `409` | `CONFLICT` | Resource already exists (e.g., duplicate registration) |
| `422` | `UNPROCESSABLE_ENTITY` | Request understood but semantically invalid |
| `429` | `RATE_LIMITED` | Too many requests — retry after `Retry-After` header |
| `500` | `INTERNAL_ERROR` | Unexpected server error |
| `503` | `SERVICE_UNAVAILABLE` | Downstream service (WHOOP, etc.) temporarily unavailable |

### Rate Limiting

| Endpoint Category | Rate Limit | Window |
|---|---|---|
| Authentication | 10 requests | per minute per IP |
| WHOOP Integration | 30 requests | per minute per user |
| Biomarkers | 60 requests | per minute per user |
| Recommendations | 20 requests | per minute per user |
| User Profile | 60 requests | per minute per user |

Rate limit headers are included in every response:

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 57
X-RateLimit-Reset: 1717747200
```

---

## Authentication Endpoints

### POST /api/auth/register

Creates a new user account.

| Property | Value |
|---|---|
| **Method** | `POST` |
| **URL** | `/api/auth/register` |
| **Auth Required** | ❌ No |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "email": "user@example.com",
  "password": "SecureP@ssw0rd!",
  "firstName": "Jane",
  "lastName": "Doe",
  "dateOfBirth": "1990-05-15",
  "sex": "female",
  "agreedToTerms": true,
  "agreedToPrivacyPolicy": true,
  "disclaimerAcknowledged": true
}
```

| Field | Type | Required | Validation |
|---|---|---|---|
| `email` | string | ✅ | Valid email format; unique |
| `password` | string | ✅ | Min 8 chars, 1 uppercase, 1 lowercase, 1 number, 1 special char |
| `firstName` | string | ✅ | 1–50 chars |
| `lastName` | string | ✅ | 1–50 chars |
| `dateOfBirth` | string (ISO 8601) | ✅ | Must be ≥18 years old |
| `sex` | enum | ✅ | `"male"`, `"female"`, `"other"` |
| `agreedToTerms` | boolean | ✅ | Must be `true` |
| `agreedToPrivacyPolicy` | boolean | ✅ | Must be `true` |
| `disclaimerAcknowledged` | boolean | ✅ | Must be `true` |

#### Response — `201 Created`

```json
{
  "success": true,
  "data": {
    "user": {
      "id": "usr_a1b2c3d4e5f6",
      "email": "user@example.com",
      "firstName": "Jane",
      "lastName": "Doe",
      "createdAt": "2026-06-07T11:00:00Z"
    },
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 900
  }
}
```

The `refreshToken` is set as an HTTP-only secure cookie (not in the response body).

#### Status Codes

| Status | Description |
|---|---|
| `201` | Account created successfully |
| `400` | Validation error (missing/invalid fields) |
| `409` | Email already registered |
| `422` | Terms/disclaimer not acknowledged or user under 18 |

#### Example

```bash
curl -X POST https://api.vitalsync.io/v1/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "SecureP@ssw0rd!",
    "firstName": "Jane",
    "lastName": "Doe",
    "dateOfBirth": "1990-05-15",
    "sex": "female",
    "agreedToTerms": true,
    "agreedToPrivacyPolicy": true,
    "disclaimerAcknowledged": true
  }'
```

---

### POST /api/auth/login

Authenticates a user and returns access/refresh tokens.

| Property | Value |
|---|---|
| **Method** | `POST` |
| **URL** | `/api/auth/login` |
| **Auth Required** | ❌ No |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "email": "user@example.com",
  "password": "SecureP@ssw0rd!"
}
```

| Field | Type | Required | Validation |
|---|---|---|---|
| `email` | string | ✅ | Valid email format |
| `password` | string | ✅ | Non-empty |

#### Response — `200 OK`

```json
{
  "success": true,
  "data": {
    "user": {
      "id": "usr_a1b2c3d4e5f6",
      "email": "user@example.com",
      "firstName": "Jane",
      "lastName": "Doe",
      "profileComplete": true,
      "whoopConnected": true,
      "lastLogin": "2026-06-07T11:00:00Z"
    },
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 900
  }
}
```

#### Status Codes

| Status | Description |
|---|---|
| `200` | Login successful |
| `400` | Missing email or password |
| `401` | Invalid credentials |
| `429` | Too many login attempts (account temporarily locked for 15 min) |

#### Example

```bash
curl -X POST https://api.vitalsync.io/v1/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "SecureP@ssw0rd!"
  }'
```

---

### POST /api/auth/refresh

Exchanges a valid refresh token for a new access token and refresh token pair.

| Property | Value |
|---|---|
| **Method** | `POST` |
| **URL** | `/api/auth/refresh` |
| **Auth Required** | ❌ No (uses refresh token cookie) |
| **Content-Type** | `application/json` |

#### Request Body

No request body required. The refresh token is read from the HTTP-only `refreshToken` cookie.

#### Response — `200 OK`

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 900
  }
}
```

A new `refreshToken` cookie is also set (token rotation for security).

#### Status Codes

| Status | Description |
|---|---|
| `200` | Token refreshed successfully |
| `401` | Refresh token missing, expired, or revoked |
| `403` | Refresh token reuse detected (potential token theft — all sessions invalidated) |

#### Example

```bash
curl -X POST https://api.vitalsync.io/v1/api/auth/refresh \
  -H "Content-Type: application/json" \
  --cookie "refreshToken=eyJhbGciOiJIUzI1NiIs..."
```

> [!WARNING]
> **Refresh token rotation is enforced.** Each refresh token can only be used once. If a previously-used refresh token is presented, this indicates potential token theft — all active sessions for the user are immediately invalidated, and the user must re-authenticate.

---

## WHOOP Integration

VitalSync integrates with the WHOOP platform via OAuth 2.0 to retrieve physiological data for wellness recommendations.

```mermaid
sequenceDiagram
    participant U as User
    participant VS as VitalSync
    participant W as WHOOP API

    U->>VS: GET /api/whoop/auth
    VS-->>U: Redirect to WHOOP OAuth consent
    U->>W: Authorize VitalSync
    W-->>U: Redirect to callback with auth code
    U->>VS: GET /api/whoop/callback?code=xxx
    VS->>W: Exchange code for tokens
    W-->>VS: Access + refresh tokens
    VS-->>U: 200 OK (WHOOP connected)
    U->>VS: GET /api/whoop/data
    VS->>W: Fetch recovery, sleep, strain data
    W-->>VS: Physiological data
    VS-->>U: Processed WHOOP data
```

### GET /api/whoop/auth

Initiates the WHOOP OAuth 2.0 authorization flow by redirecting the user to WHOOP's consent screen.

| Property | Value |
|---|---|
| **Method** | `GET` |
| **URL** | `/api/whoop/auth` |
| **Auth Required** | ✅ Yes |

#### Query Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `redirect` | string | ❌ | Post-authorization redirect URL (default: `/dashboard`) |

#### Response — `302 Found`

Redirects to WHOOP OAuth consent page:

```
HTTP/1.1 302 Found
Location: https://api.prod.whoop.com/oauth/oauth2/auth
  ?client_id=vitalsync_prod_xxxxx
  &redirect_uri=https://api.vitalsync.io/v1/api/whoop/callback
  &response_type=code
  &scope=read:recovery read:sleep read:cycles
  &state=eyJhbGciOiJIUzI1NiIs...
```

#### Status Codes

| Status | Description |
|---|---|
| `302` | Redirect to WHOOP OAuth consent |
| `401` | User not authenticated |

#### Example

```bash
curl -X GET https://api.vitalsync.io/v1/api/whoop/auth \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -L  # Follow redirects
```

---

### GET /api/whoop/callback

Handles the OAuth callback from WHOOP after user authorization. Exchanges the authorization code for access and refresh tokens.

| Property | Value |
|---|---|
| **Method** | `GET` |
| **URL** | `/api/whoop/callback` |
| **Auth Required** | ❌ No (state parameter contains auth context) |

#### Query Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `code` | string | ✅ | Authorization code from WHOOP |
| `state` | string | ✅ | Anti-CSRF state token (JWT containing user ID and redirect URL) |

#### Response — `302 Found`

Redirects to the post-authorization page (dashboard or custom redirect):

```
HTTP/1.1 302 Found
Location: https://app.vitalsync.io/dashboard?whoop=connected
```

On error:

```
HTTP/1.1 302 Found
Location: https://app.vitalsync.io/settings?whoop=error&reason=consent_denied
```

#### Status Codes

| Status | Description |
|---|---|
| `302` | Successfully connected — redirect to dashboard |
| `400` | Invalid or missing authorization code |
| `403` | State parameter validation failed (CSRF protection) |
| `503` | WHOOP API unavailable |

#### Example

```bash
# This endpoint is called by WHOOP's redirect — not typically called manually.
# The URL will look like:
# https://api.vitalsync.io/v1/api/whoop/callback?code=abc123&state=eyJhbGci...
```

---

### GET /api/whoop/data

Fetches the latest WHOOP physiological data for the authenticated user.

| Property | Value |
|---|---|
| **Method** | `GET` |
| **URL** | `/api/whoop/data` |
| **Auth Required** | ✅ Yes |

#### Query Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `days` | integer | ❌ | `7` | Number of days of data to retrieve (1–90) |
| `metrics` | string | ❌ | `all` | Comma-separated list: `recovery`, `sleep`, `strain`, `workouts` |

#### Response — `200 OK`

```json
{
  "success": true,
  "data": {
    "userId": "usr_a1b2c3d4e5f6",
    "whoopUserId": "whp_98765",
    "syncedAt": "2026-06-07T10:30:00Z",
    "period": {
      "start": "2026-05-31T00:00:00Z",
      "end": "2026-06-07T00:00:00Z"
    },
    "recovery": [
      {
        "date": "2026-06-07",
        "recoveryScore": 72,
        "restingHeartRate": 54,
        "hrvRmssd": 68.5,
        "spo2": 97.2,
        "skinTemp": 33.1,
        "respiratoryRate": 15.2
      },
      {
        "date": "2026-06-06",
        "recoveryScore": 85,
        "restingHeartRate": 51,
        "hrvRmssd": 78.3,
        "spo2": 98.0,
        "skinTemp": 32.9,
        "respiratoryRate": 14.8
      }
    ],
    "sleep": [
      {
        "date": "2026-06-07",
        "sleepPerformance": 78,
        "totalSleepDuration": 25920,
        "remSleepDuration": 6480,
        "deepSleepDuration": 5400,
        "lightSleepDuration": 12960,
        "awakeDuration": 1080,
        "sleepEfficiency": 0.89,
        "sleepConsistency": 72,
        "respiratoryRate": 15.2
      }
    ],
    "strain": [
      {
        "date": "2026-06-07",
        "dayStrain": 12.4,
        "averageHeartRate": 78,
        "maxHeartRate": 165,
        "caloriesBurned": 2350
      }
    ]
  }
}
```

> [!NOTE]
> Sleep duration values are in **seconds**. Convert to hours/minutes for display (e.g., 25920 seconds = 7 hours 12 minutes).

#### Status Codes

| Status | Description |
|---|---|
| `200` | Data retrieved successfully |
| `401` | User not authenticated |
| `403` | WHOOP not connected — direct user to `/api/whoop/auth` |
| `404` | No WHOOP data available for the requested period |
| `503` | WHOOP API temporarily unavailable |

#### Example

```bash
curl -X GET "https://api.vitalsync.io/v1/api/whoop/data?days=7&metrics=recovery,sleep" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

---

### GET /api/whoop/signals

Returns analyzed health signals derived from WHOOP data — processed and interpreted for use by the recommendation engine.

| Property | Value |
|---|---|
| **Method** | `GET` |
| **URL** | `/api/whoop/signals` |
| **Auth Required** | ✅ Yes |

#### Query Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `days` | integer | ❌ | `14` | Analysis window (7–90 days) |

#### Response — `200 OK`

```json
{
  "success": true,
  "data": {
    "userId": "usr_a1b2c3d4e5f6",
    "analysisWindow": {
      "start": "2026-05-24T00:00:00Z",
      "end": "2026-06-07T00:00:00Z",
      "daysAnalyzed": 14
    },
    "signals": {
      "sleepQuality": {
        "score": 0.68,
        "trend": "declining",
        "trendDelta": -0.08,
        "classification": "below_optimal",
        "insights": [
          "Average sleep efficiency 82% (target: >85%)",
          "Deep sleep averaging 58 min/night (target: >90 min)",
          "Sleep consistency score 64% — variable bedtime detected"
        ]
      },
      "recoveryStatus": {
        "score": 0.74,
        "trend": "stable",
        "trendDelta": -0.02,
        "classification": "adequate",
        "insights": [
          "Average recovery score 71% over 14 days",
          "3 days with recovery <50% (yellow/red zone)"
        ]
      },
      "stressIndicators": {
        "score": 0.55,
        "trend": "increasing",
        "trendDelta": 0.12,
        "classification": "elevated",
        "insights": [
          "HRV trending downward (-12% over 14 days)",
          "Resting heart rate trending upward (+3 bpm over 14 days)",
          "Elevated strain-to-recovery ratio detected"
        ]
      },
      "exerciseLoad": {
        "score": 0.82,
        "trend": "stable",
        "classification": "optimal",
        "insights": [
          "Average daily strain 12.4 (moderate-high activity level)",
          "Good strain distribution across the week"
        ]
      }
    },
    "supplementRelevance": {
      "magnesium": {
        "relevance": "high",
        "reason": "Declining sleep quality and elevated stress indicators suggest magnesium may support relaxation and sleep"
      },
      "omega3": {
        "relevance": "moderate",
        "reason": "Elevated inflammatory signals (high strain, slow recovery) may benefit from omega-3 support"
      },
      "vitaminD": {
        "relevance": "moderate",
        "reason": "Recovery patterns consistent with potential vitamin D optimization opportunity"
      }
    },
    "disclaimer": "Wellness insights only. Not medical advice. Consult your healthcare provider."
  }
}
```

#### Status Codes

| Status | Description |
|---|---|
| `200` | Signals analyzed successfully |
| `401` | User not authenticated |
| `403` | WHOOP not connected |
| `404` | Insufficient WHOOP data (need ≥7 days) |
| `503` | Analysis service temporarily unavailable |

#### Example

```bash
curl -X GET "https://api.vitalsync.io/v1/api/whoop/signals?days=14" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

---

### GET /api/whoop/status

Checks the current WHOOP connection status for the authenticated user.

| Property | Value |
|---|---|
| **Method** | `GET` |
| **URL** | `/api/whoop/status` |
| **Auth Required** | ✅ Yes |

#### Response — `200 OK`

```json
{
  "success": true,
  "data": {
    "connected": true,
    "whoopUserId": "whp_98765",
    "connectedAt": "2026-05-15T08:30:00Z",
    "lastSync": "2026-06-07T10:30:00Z",
    "tokenStatus": "valid",
    "scopes": ["read:recovery", "read:sleep", "read:cycles"],
    "dataAvailable": {
      "recovery": true,
      "sleep": true,
      "strain": true,
      "workouts": true
    }
  }
}
```

**When disconnected:**

```json
{
  "success": true,
  "data": {
    "connected": false,
    "whoopUserId": null,
    "connectedAt": null,
    "lastSync": null,
    "tokenStatus": "none",
    "scopes": [],
    "dataAvailable": {
      "recovery": false,
      "sleep": false,
      "strain": false,
      "workouts": false
    }
  }
}
```

#### Status Codes

| Status | Description |
|---|---|
| `200` | Status retrieved (connected or disconnected) |
| `401` | User not authenticated |

#### Example

```bash
curl -X GET https://api.vitalsync.io/v1/api/whoop/status \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

---

## Biomarkers

### POST /api/biomarkers/report

Submits a new blood panel / lab report with biomarker values.

| Property | Value |
|---|---|
| **Method** | `POST` |
| **URL** | `/api/biomarkers/report` |
| **Auth Required** | ✅ Yes |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "reportDate": "2026-06-01",
  "labProvider": "Quest Diagnostics",
  "biomarkers": [
    {
      "name": "vitamin_d_25oh",
      "value": 28.5,
      "unit": "ng/mL"
    },
    {
      "name": "vitamin_b12",
      "value": 310,
      "unit": "pg/mL"
    },
    {
      "name": "ferritin",
      "value": 22,
      "unit": "ng/mL"
    },
    {
      "name": "magnesium_rbc",
      "value": 4.8,
      "unit": "mg/dL"
    },
    {
      "name": "zinc_serum",
      "value": 72,
      "unit": "mcg/dL"
    },
    {
      "name": "tsh",
      "value": 2.1,
      "unit": "mIU/L"
    },
    {
      "name": "hba1c",
      "value": 5.4,
      "unit": "%"
    },
    {
      "name": "crp_hs",
      "value": 1.8,
      "unit": "mg/L"
    },
    {
      "name": "homocysteine",
      "value": 9.2,
      "unit": "umol/L"
    },
    {
      "name": "omega3_index",
      "value": 5.1,
      "unit": "%"
    }
  ],
  "notes": "Fasting blood draw. Morning sample."
}
```

| Field | Type | Required | Validation |
|---|---|---|---|
| `reportDate` | string (ISO 8601 date) | ✅ | Must not be in the future |
| `labProvider` | string | ❌ | Free text, max 200 chars |
| `biomarkers` | array | ✅ | At least 1 biomarker required |
| `biomarkers[].name` | string | ✅ | Must be a recognized biomarker identifier (see biomarker catalog) |
| `biomarkers[].value` | number | ✅ | Positive number |
| `biomarkers[].unit` | string | ✅ | Must match expected unit for the biomarker |
| `notes` | string | ❌ | Free text, max 500 chars |

#### Supported Biomarker Identifiers

| Identifier | Biomarker | Expected Unit |
|---|---|---|
| `vitamin_d_25oh` | 25-Hydroxyvitamin D | ng/mL |
| `vitamin_b12` | Vitamin B12 (cobalamin) | pg/mL |
| `ferritin` | Ferritin | ng/mL |
| `iron_serum` | Serum Iron | mcg/dL |
| `tibc` | Total Iron Binding Capacity | mcg/dL |
| `magnesium_rbc` | RBC Magnesium | mg/dL |
| `magnesium_serum` | Serum Magnesium | mg/dL |
| `zinc_serum` | Serum Zinc | mcg/dL |
| `selenium_serum` | Serum Selenium | mcg/L |
| `folate_serum` | Serum Folate | ng/mL |
| `tsh` | Thyroid Stimulating Hormone | mIU/L |
| `hba1c` | Hemoglobin A1c | % |
| `crp_hs` | High-sensitivity C-Reactive Protein | mg/L |
| `homocysteine` | Homocysteine | umol/L |
| `omega3_index` | Omega-3 Index | % |
| `vitamin_a_retinol` | Retinol (Vitamin A) | mcg/dL |
| `calcium_serum` | Serum Calcium | mg/dL |
| `potassium_serum` | Serum Potassium | mEq/L |
| `sodium_serum` | Serum Sodium | mEq/L |
| `cortisol_am` | Morning Cortisol | mcg/dL |

#### Response — `201 Created`

```json
{
  "success": true,
  "data": {
    "reportId": "rpt_f1e2d3c4b5a6",
    "reportDate": "2026-06-01",
    "biomarkersProcessed": 10,
    "classifications": [
      {
        "name": "vitamin_d_25oh",
        "value": 28.5,
        "unit": "ng/mL",
        "zone": "suboptimal",
        "referenceRange": {
          "deficient": { "max": 20 },
          "suboptimal": { "min": 20, "max": 40 },
          "optimal": { "min": 40, "max": 80 },
          "elevated": { "min": 80 }
        },
        "insight": "Your vitamin D level is in the suboptimal zone. Consider supplementation to reach the optimal range of 40-80 ng/mL."
      }
    ],
    "createdAt": "2026-06-07T11:00:00Z",
    "disclaimer": "Biomarker classifications are for wellness purposes only. Not a clinical diagnosis. Consult your healthcare provider."
  }
}
```

#### Status Codes

| Status | Description |
|---|---|
| `201` | Report submitted and processed |
| `400` | Validation error (invalid biomarker name, unit, or value) |
| `401` | User not authenticated |
| `422` | Unrecognized biomarker identifier or unit mismatch |

#### Example

```bash
curl -X POST https://api.vitalsync.io/v1/api/biomarkers/report \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "reportDate": "2026-06-01",
    "labProvider": "Quest Diagnostics",
    "biomarkers": [
      { "name": "vitamin_d_25oh", "value": 28.5, "unit": "ng/mL" },
      { "name": "ferritin", "value": 22, "unit": "ng/mL" }
    ]
  }'
```

---

### GET /api/biomarkers/report/:id

Retrieves a specific biomarker report by ID.

| Property | Value |
|---|---|
| **Method** | `GET` |
| **URL** | `/api/biomarkers/report/:id` |
| **Auth Required** | ✅ Yes |

#### Path Parameters

| Parameter | Type | Description |
|---|---|---|
| `id` | string | Report ID (e.g., `rpt_f1e2d3c4b5a6`) |

#### Response — `200 OK`

```json
{
  "success": true,
  "data": {
    "reportId": "rpt_f1e2d3c4b5a6",
    "reportDate": "2026-06-01",
    "labProvider": "Quest Diagnostics",
    "biomarkers": [
      {
        "name": "vitamin_d_25oh",
        "displayName": "Vitamin D (25-OH)",
        "value": 28.5,
        "unit": "ng/mL",
        "zone": "suboptimal",
        "referenceRange": {
          "deficient": { "max": 20 },
          "suboptimal": { "min": 20, "max": 40 },
          "optimal": { "min": 40, "max": 80 },
          "elevated": { "min": 80 }
        },
        "previousValue": 24.0,
        "previousDate": "2026-03-01",
        "trend": "improving"
      },
      {
        "name": "ferritin",
        "displayName": "Ferritin",
        "value": 22,
        "unit": "ng/mL",
        "zone": "suboptimal",
        "referenceRange": {
          "deficient": { "max": 15 },
          "suboptimal": { "min": 15, "max": 40 },
          "optimal": { "min": 40, "max": 200 },
          "elevated": { "min": 200 }
        },
        "previousValue": null,
        "previousDate": null,
        "trend": null
      }
    ],
    "notes": "Fasting blood draw. Morning sample.",
    "createdAt": "2026-06-07T11:00:00Z",
    "disclaimer": "Biomarker classifications are for wellness purposes only. Not a clinical diagnosis."
  }
}
```

#### Status Codes

| Status | Description |
|---|---|
| `200` | Report retrieved successfully |
| `401` | User not authenticated |
| `403` | Report belongs to another user |
| `404` | Report not found |

#### Example

```bash
curl -X GET https://api.vitalsync.io/v1/api/biomarkers/report/rpt_f1e2d3c4b5a6 \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

---

### GET /api/biomarkers/reports

Lists all biomarker reports for the authenticated user.

| Property | Value |
|---|---|
| **Method** | `GET` |
| **URL** | `/api/biomarkers/reports` |
| **Auth Required** | ✅ Yes |

#### Query Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `page` | integer | ❌ | `1` | Page number |
| `limit` | integer | ❌ | `10` | Results per page (max 50) |
| `sortBy` | string | ❌ | `reportDate` | Sort field: `reportDate`, `createdAt` |
| `sortOrder` | string | ❌ | `desc` | `asc` or `desc` |

#### Response — `200 OK`

```json
{
  "success": true,
  "data": {
    "reports": [
      {
        "reportId": "rpt_f1e2d3c4b5a6",
        "reportDate": "2026-06-01",
        "labProvider": "Quest Diagnostics",
        "biomarkerCount": 10,
        "summary": {
          "optimal": 4,
          "suboptimal": 4,
          "deficient": 1,
          "elevated": 1
        },
        "createdAt": "2026-06-07T11:00:00Z"
      },
      {
        "reportId": "rpt_a9b8c7d6e5f4",
        "reportDate": "2026-03-01",
        "labProvider": "LabCorp",
        "biomarkerCount": 8,
        "summary": {
          "optimal": 3,
          "suboptimal": 3,
          "deficient": 2,
          "elevated": 0
        },
        "createdAt": "2026-03-05T14:30:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 10,
      "totalReports": 2,
      "totalPages": 1
    }
  }
}
```

#### Status Codes

| Status | Description |
|---|---|
| `200` | Reports listed successfully |
| `401` | User not authenticated |

#### Example

```bash
curl -X GET "https://api.vitalsync.io/v1/api/biomarkers/reports?page=1&limit=10" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

---

### GET /api/biomarkers/classify

Classifies a set of biomarker values into zones (deficient, suboptimal, optimal, elevated) without persisting a report. Useful for real-time UI feedback during data entry.

| Property | Value |
|---|---|
| **Method** | `GET` |
| **URL** | `/api/biomarkers/classify` |
| **Auth Required** | ✅ Yes |

#### Query Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `biomarker` | string | ✅ | Biomarker identifier (e.g., `vitamin_d_25oh`) |
| `value` | number | ✅ | The value to classify |
| `unit` | string | ✅ | Unit of measurement |
| `sex` | string | ❌ | `male` or `female` (overrides profile; used for sex-specific ranges) |
| `age` | integer | ❌ | Age in years (overrides profile; used for age-specific ranges) |

#### Response — `200 OK`

```json
{
  "success": true,
  "data": {
    "biomarker": "vitamin_d_25oh",
    "value": 28.5,
    "unit": "ng/mL",
    "zone": "suboptimal",
    "zoneColor": "#F59E0B",
    "referenceRange": {
      "deficient": { "max": 20 },
      "suboptimal": { "min": 20, "max": 40 },
      "optimal": { "min": 40, "max": 80 },
      "elevated": { "min": 80 }
    },
    "interpretation": "Your vitamin D level is below the optimal range. Supplementation with vitamin D3 may support healthy levels.",
    "disclaimer": "For wellness reference only. Not a clinical diagnosis."
  }
}
```

#### Status Codes

| Status | Description |
|---|---|
| `200` | Classification returned |
| `400` | Missing required parameters |
| `401` | User not authenticated |
| `422` | Unrecognized biomarker or unit mismatch |

#### Example

```bash
curl -X GET "https://api.vitalsync.io/v1/api/biomarkers/classify?biomarker=vitamin_d_25oh&value=28.5&unit=ng/mL" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

---

## Recommendations

### POST /api/recommendations/generate

Generates a personalized supplement recommendation protocol based on the user's profile, biomarkers, WHOOP data, and medication list. This is the core endpoint of VitalSync.

| Property | Value |
|---|---|
| **Method** | `POST` |
| **URL** | `/api/recommendations/generate` |
| **Auth Required** | ✅ Yes |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "biomarkerReportId": "rpt_f1e2d3c4b5a6",
  "includeWhoopData": true,
  "whoopAnalysisDays": 14,
  "healthGoals": ["energy", "sleep", "immunity"],
  "dietaryPreferences": ["vegetarian"],
  "budgetTier": "moderate",
  "maxSupplements": 8
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `biomarkerReportId` | string | ❌ | Specific report to base recommendations on (defaults to most recent) |
| `includeWhoopData` | boolean | ❌ | Include WHOOP signals in recommendation logic (default: `true` if connected) |
| `whoopAnalysisDays` | integer | ❌ | WHOOP analysis window in days (default: `14`, range: 7–90) |
| `healthGoals` | array of strings | ❌ | Priority goals: `energy`, `sleep`, `immunity`, `stress`, `fitness`, `cognition`, `skin`, `longevity` |
| `dietaryPreferences` | array of strings | ❌ | `vegetarian`, `vegan`, `pescatarian`, `halal`, `kosher`, `glutenFree`, `dairyFree` |
| `budgetTier` | string | ❌ | `budget`, `moderate`, `premium` (affects product recommendations) |
| `maxSupplements` | integer | ❌ | Maximum number of supplements to recommend (default: `10`, range: 1–15) |

#### Response — `200 OK`

```json
{
  "success": true,
  "data": {
    "recommendationId": "rec_1a2b3c4d5e6f",
    "generatedAt": "2026-06-07T11:00:00Z",
    "protocol": {
      "name": "Personalized Wellness Protocol",
      "version": "2.1",
      "totalSupplements": 6,
      "estimatedMonthlyCost": {
        "budget": 45.00,
        "moderate": 78.00,
        "premium": 135.00
      }
    },
    "supplements": [
      {
        "rank": 1,
        "nutrient": "Vitamin D3",
        "form": "cholecalciferol",
        "dosage": "4000 IU",
        "frequency": "daily",
        "timing": "With breakfast (fat-containing meal)",
        "duration": "Ongoing — retest in 3 months",
        "confidenceScore": 0.91,
        "evidenceLabel": "Strong Evidence",
        "rationale": "Your reported vitamin D level (28.5 ng/mL) is below the optimal range of 40-80 ng/mL. Supplementation supports bone health, immune function, and mood.",
        "sources": {
          "biomarkers": ["vitamin_d_25oh: 28.5 ng/mL (suboptimal)"],
          "whoop": ["Recovery score trending below optimal"],
          "healthGoals": ["immunity", "energy"]
        },
        "interactions": [],
        "claim": "Vitamin D supports calcium absorption and helps maintain strong bones and healthy immune function.",
        "disclaimer": "These statements have not been evaluated by the FDA. This product is not intended to diagnose, treat, cure, or prevent any disease."
      },
      {
        "rank": 2,
        "nutrient": "Magnesium Glycinate",
        "form": "bisglycinate chelate",
        "dosage": "300 mg elemental",
        "frequency": "daily",
        "timing": "30 minutes before bed",
        "duration": "Ongoing",
        "confidenceScore": 0.82,
        "evidenceLabel": "Strong Evidence",
        "rationale": "WHOOP data shows declining sleep quality and elevated stress indicators. Magnesium supports relaxation, sleep quality, and muscle recovery.",
        "sources": {
          "biomarkers": ["magnesium_rbc: 4.8 mg/dL (suboptimal)"],
          "whoop": ["Sleep quality declining", "Stress indicators elevated"],
          "healthGoals": ["sleep", "stress"]
        },
        "interactions": [],
        "claim": "Magnesium supports healthy muscle and nerve function, energy production, and restful sleep.",
        "disclaimer": "These statements have not been evaluated by the FDA. This product is not intended to diagnose, treat, cure, or prevent any disease."
      }
    ],
    "interactions": {
      "detected": 0,
      "warnings": [],
      "contraindications": []
    },
    "timingSchedule": {
      "morning": [
        { "time": "With breakfast", "supplements": ["Vitamin D3 4000 IU", "Omega-3 1g EPA+DHA"] }
      ],
      "afternoon": [
        { "time": "With lunch", "supplements": ["Iron Bisglycinate 18mg + Vitamin C 250mg"] }
      ],
      "evening": [
        { "time": "30 min before bed", "supplements": ["Magnesium Glycinate 300mg"] }
      ]
    },
    "retestRecommendation": {
      "suggestedDate": "2026-09-07",
      "biomarkersToRetest": ["vitamin_d_25oh", "ferritin", "magnesium_rbc"],
      "reason": "Retest in 3 months to evaluate supplementation effectiveness and adjust protocol."
    },
    "disclaimer": "For educational purposes only. Not medical advice. These statements have not been evaluated by the FDA. This is not intended to diagnose, treat, cure, or prevent any disease. Consult your healthcare provider before starting any supplement."
  }
}
```

#### Status Codes

| Status | Description |
|---|---|
| `200` | Recommendations generated successfully |
| `400` | Invalid request parameters |
| `401` | User not authenticated |
| `404` | Specified biomarker report not found |
| `422` | Insufficient data to generate recommendations (need at least profile + 1 biomarker or WHOOP data) |
| `503` | Recommendation engine temporarily unavailable |

#### Example

```bash
curl -X POST https://api.vitalsync.io/v1/api/recommendations/generate \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "biomarkerReportId": "rpt_f1e2d3c4b5a6",
    "includeWhoopData": true,
    "healthGoals": ["energy", "sleep", "immunity"],
    "maxSupplements": 8
  }'
```

---

### GET /api/recommendations/latest

Retrieves the most recently generated recommendation protocol for the authenticated user.

| Property | Value |
|---|---|
| **Method** | `GET` |
| **URL** | `/api/recommendations/latest` |
| **Auth Required** | ✅ Yes |

#### Response — `200 OK`

Returns the same structure as `POST /api/recommendations/generate` response, representing the most recent protocol.

#### Status Codes

| Status | Description |
|---|---|
| `200` | Latest recommendations retrieved |
| `401` | User not authenticated |
| `404` | No recommendations generated yet |

#### Example

```bash
curl -X GET https://api.vitalsync.io/v1/api/recommendations/latest \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

---

### GET /api/recommendations/history

Retrieves historical recommendation protocols, allowing users to track how their protocol has evolved over time.

| Property | Value |
|---|---|
| **Method** | `GET` |
| **URL** | `/api/recommendations/history` |
| **Auth Required** | ✅ Yes |

#### Query Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `page` | integer | ❌ | `1` | Page number |
| `limit` | integer | ❌ | `10` | Results per page (max 50) |
| `from` | string (ISO 8601) | ❌ | — | Start date filter |
| `to` | string (ISO 8601) | ❌ | — | End date filter |

#### Response — `200 OK`

```json
{
  "success": true,
  "data": {
    "recommendations": [
      {
        "recommendationId": "rec_1a2b3c4d5e6f",
        "generatedAt": "2026-06-07T11:00:00Z",
        "supplementCount": 6,
        "topSupplements": ["Vitamin D3", "Magnesium Glycinate", "Omega-3"],
        "dataSourcesUsed": {
          "biomarkers": true,
          "whoop": true,
          "medications": true
        },
        "interactionsDetected": 0,
        "averageConfidenceScore": 0.78
      },
      {
        "recommendationId": "rec_7g8h9i0j1k2l",
        "generatedAt": "2026-03-10T09:15:00Z",
        "supplementCount": 5,
        "topSupplements": ["Vitamin D3", "Iron Bisglycinate", "B-Complex"],
        "dataSourcesUsed": {
          "biomarkers": true,
          "whoop": false,
          "medications": true
        },
        "interactionsDetected": 1,
        "averageConfidenceScore": 0.72
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 10,
      "totalRecommendations": 2,
      "totalPages": 1
    }
  }
}
```

#### Status Codes

| Status | Description |
|---|---|
| `200` | History retrieved |
| `401` | User not authenticated |

#### Example

```bash
curl -X GET "https://api.vitalsync.io/v1/api/recommendations/history?page=1&limit=10" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

---

## User Profile

### GET /api/user/profile

Retrieves the authenticated user's profile, including health information and preferences.

| Property | Value |
|---|---|
| **Method** | `GET` |
| **URL** | `/api/user/profile` |
| **Auth Required** | ✅ Yes |

#### Response — `200 OK`

```json
{
  "success": true,
  "data": {
    "id": "usr_a1b2c3d4e5f6",
    "email": "user@example.com",
    "firstName": "Jane",
    "lastName": "Doe",
    "dateOfBirth": "1990-05-15",
    "age": 36,
    "sex": "female",
    "height": {
      "value": 165,
      "unit": "cm"
    },
    "weight": {
      "value": 62,
      "unit": "kg"
    },
    "healthGoals": ["energy", "sleep", "immunity"],
    "dietaryPreferences": ["vegetarian"],
    "allergies": ["shellfish"],
    "medicalConditions": ["hypothyroidism"],
    "medications": [
      {
        "name": "levothyroxine",
        "rxcui": "10582",
        "dosage": "75 mcg",
        "frequency": "daily",
        "addedAt": "2026-05-01T00:00:00Z"
      }
    ],
    "lifestyle": {
      "activityLevel": "moderate",
      "smokingStatus": "never",
      "alcoholConsumption": "occasional",
      "sleepGoalHours": 8
    },
    "integrations": {
      "whoop": {
        "connected": true,
        "connectedAt": "2026-05-15T08:30:00Z"
      }
    },
    "preferences": {
      "budgetTier": "moderate",
      "maxSupplements": 8,
      "preferredForms": ["capsule", "softgel"],
      "notificationPreferences": {
        "supplementReminders": true,
        "retestReminders": true,
        "weeklyInsights": true
      }
    },
    "profileComplete": true,
    "createdAt": "2026-04-20T10:00:00Z",
    "updatedAt": "2026-06-05T14:30:00Z"
  }
}
```

#### Status Codes

| Status | Description |
|---|---|
| `200` | Profile retrieved |
| `401` | User not authenticated |

#### Example

```bash
curl -X GET https://api.vitalsync.io/v1/api/user/profile \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

---

### PUT /api/user/profile

Updates the authenticated user's profile. Supports partial updates — only include fields that are being changed.

| Property | Value |
|---|---|
| **Method** | `PUT` |
| **URL** | `/api/user/profile` |
| **Auth Required** | ✅ Yes |
| **Content-Type** | `application/json` |

#### Request Body (Partial Update)

```json
{
  "height": {
    "value": 165,
    "unit": "cm"
  },
  "weight": {
    "value": 61,
    "unit": "kg"
  },
  "healthGoals": ["energy", "sleep", "immunity", "stress"],
  "dietaryPreferences": ["vegetarian", "glutenFree"],
  "allergies": ["shellfish", "soy"],
  "medicalConditions": ["hypothyroidism"],
  "lifestyle": {
    "activityLevel": "high",
    "smokingStatus": "never",
    "alcoholConsumption": "occasional",
    "sleepGoalHours": 8
  },
  "preferences": {
    "budgetTier": "premium",
    "maxSupplements": 10,
    "preferredForms": ["capsule", "softgel", "liquid"]
  }
}
```

| Field | Type | Required | Validation |
|---|---|---|---|
| `firstName` | string | ❌ | 1–50 chars |
| `lastName` | string | ❌ | 1–50 chars |
| `dateOfBirth` | string (ISO 8601) | ❌ | Must be ≥18 years old |
| `sex` | enum | ❌ | `"male"`, `"female"`, `"other"` |
| `height` | object | ❌ | `value` (number) + `unit` (`"cm"` or `"in"`) |
| `weight` | object | ❌ | `value` (number) + `unit` (`"kg"` or `"lb"`) |
| `healthGoals` | array of strings | ❌ | Valid goals: `energy`, `sleep`, `immunity`, `stress`, `fitness`, `cognition`, `skin`, `longevity` |
| `dietaryPreferences` | array of strings | ❌ | Valid: `vegetarian`, `vegan`, `pescatarian`, `halal`, `kosher`, `glutenFree`, `dairyFree` |
| `allergies` | array of strings | ❌ | Free text entries |
| `medicalConditions` | array of strings | ❌ | Free text entries (used for contraindication screening) |
| `lifestyle` | object | ❌ | See lifestyle fields below |
| `lifestyle.activityLevel` | enum | ❌ | `sedentary`, `light`, `moderate`, `high`, `athlete` |
| `lifestyle.smokingStatus` | enum | ❌ | `never`, `former`, `current` |
| `lifestyle.alcoholConsumption` | enum | ❌ | `none`, `occasional`, `moderate`, `heavy` |
| `lifestyle.sleepGoalHours` | number | ❌ | 4–12 |
| `preferences` | object | ❌ | See preferences fields above |

#### Response — `200 OK`

```json
{
  "success": true,
  "data": {
    "message": "Profile updated successfully",
    "updatedFields": ["weight", "healthGoals", "dietaryPreferences", "allergies", "lifestyle", "preferences"],
    "profileComplete": true,
    "updatedAt": "2026-06-07T11:05:00Z",
    "recommendationImpact": {
      "recalculationNeeded": true,
      "reason": "Health goals and dietary preferences changed — recommendations may differ"
    }
  }
}
```

> [!TIP]
> When the response includes `"recalculationNeeded": true`, the client should prompt the user to regenerate their recommendations via `POST /api/recommendations/generate` to reflect their updated profile.

#### Status Codes

| Status | Description |
|---|---|
| `200` | Profile updated successfully |
| `400` | Validation error (invalid field values) |
| `401` | User not authenticated |
| `422` | Semantically invalid (e.g., conflicting dietary preferences) |

#### Example

```bash
curl -X PUT https://api.vitalsync.io/v1/api/user/profile \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "healthGoals": ["energy", "sleep", "immunity", "stress"],
    "weight": { "value": 61, "unit": "kg" }
  }'
```

---

### POST /api/user/medications

Updates the user's current medication list. This is a **replace** operation — the provided list replaces the entire existing medication list.

| Property | Value |
|---|---|
| **Method** | `POST` |
| **URL** | `/api/user/medications` |
| **Auth Required** | ✅ Yes |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "medications": [
    {
      "name": "levothyroxine",
      "dosage": "75 mcg",
      "frequency": "daily",
      "prescribedFor": "hypothyroidism"
    },
    {
      "name": "atorvastatin",
      "dosage": "20 mg",
      "frequency": "daily",
      "prescribedFor": "cholesterol management"
    },
    {
      "name": "omeprazole",
      "dosage": "20 mg",
      "frequency": "daily",
      "prescribedFor": "acid reflux"
    }
  ]
}
```

| Field | Type | Required | Validation |
|---|---|---|---|
| `medications` | array | ✅ | Can be empty array `[]` to clear all medications |
| `medications[].name` | string | ✅ | Medication name (free text — normalized to RxNorm server-side) |
| `medications[].dosage` | string | ❌ | Dosage with unit (e.g., "20 mg", "75 mcg") |
| `medications[].frequency` | string | ❌ | `daily`, `twice_daily`, `weekly`, `as_needed`, `other` |
| `medications[].prescribedFor` | string | ❌ | Reason for medication (free text) |

#### Response — `200 OK`

```json
{
  "success": true,
  "data": {
    "medications": [
      {
        "name": "levothyroxine",
        "normalizedName": "Levothyroxine Sodium",
        "rxcui": "10582",
        "class": "thyroid_hormone",
        "dosage": "75 mcg",
        "frequency": "daily",
        "prescribedFor": "hypothyroidism",
        "addedAt": "2026-06-07T11:10:00Z"
      },
      {
        "name": "atorvastatin",
        "normalizedName": "Atorvastatin Calcium",
        "rxcui": "83367",
        "class": "statin",
        "dosage": "20 mg",
        "frequency": "daily",
        "prescribedFor": "cholesterol management",
        "addedAt": "2026-06-07T11:10:00Z"
      },
      {
        "name": "omeprazole",
        "normalizedName": "Omeprazole",
        "rxcui": "7646",
        "class": "proton_pump_inhibitor",
        "dosage": "20 mg",
        "frequency": "daily",
        "prescribedFor": "acid reflux",
        "addedAt": "2026-06-07T11:10:00Z"
      }
    ],
    "interactionPreview": {
      "totalInteractions": 4,
      "high": 1,
      "moderate": 3,
      "low": 0,
      "details": [
        {
          "severity": "HIGH",
          "medication": "levothyroxine",
          "nutrient": "Calcium, Iron, Magnesium",
          "summary": "These minerals bind levothyroxine and reduce absorption by up to 64%. A 4-hour separation is required.",
          "interactionId": "INT-LEVO-CAFEMG-001"
        },
        {
          "severity": "MODERATE",
          "medication": "atorvastatin",
          "nutrient": "CoQ10",
          "summary": "Statins may deplete CoQ10 via shared mevalonate pathway. Supplementation recommended.",
          "interactionId": "INT-STAT-COQ10-001"
        },
        {
          "severity": "MODERATE",
          "medication": "omeprazole",
          "nutrient": "Magnesium, B12, Calcium, Iron",
          "summary": "Long-term PPI use impairs absorption of multiple nutrients. Specific supplement forms recommended.",
          "interactionId": "INT-PPI-MULTI-001"
        },
        {
          "severity": "MODERATE",
          "medication": "omeprazole + levothyroxine",
          "nutrient": "Levothyroxine absorption",
          "summary": "PPIs may reduce levothyroxine absorption. Monitor TSH if recently started PPI.",
          "interactionId": "INT-PPI-LEVO-001"
        }
      ]
    },
    "recommendationImpact": {
      "recalculationNeeded": true,
      "reason": "Medication list changed — drug-nutrient interactions require recommendation recalculation"
    },
    "updatedAt": "2026-06-07T11:10:00Z"
  }
}
```

> [!IMPORTANT]
> The `interactionPreview` in the response gives an immediate preview of detected drug-nutrient interactions. The full interaction analysis is applied when recommendations are generated via `POST /api/recommendations/generate`. See [DRUG_INTERACTIONS.md](./DRUG_INTERACTIONS.md) for the complete interaction database.

#### Status Codes

| Status | Description |
|---|---|
| `200` | Medication list updated |
| `400` | Validation error (empty medication name, etc.) |
| `401` | User not authenticated |
| `422` | Medication name could not be normalized (unknown drug — stored as-is with warning) |

#### Example

```bash
curl -X POST https://api.vitalsync.io/v1/api/user/medications \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "medications": [
      {
        "name": "levothyroxine",
        "dosage": "75 mcg",
        "frequency": "daily",
        "prescribedFor": "hypothyroidism"
      },
      {
        "name": "atorvastatin",
        "dosage": "20 mg",
        "frequency": "daily",
        "prescribedFor": "cholesterol management"
      }
    ]
  }'
```

---

## Appendix: API Changelog

| Version | Date | Changes |
|---|---|---|
| **2.1** | June 2026 | Added `/api/whoop/signals` endpoint. Enhanced `/api/recommendations/generate` with WHOOP signal integration. Added `interactionPreview` to medication update response. |
| **2.0** | March 2026 | Major: WHOOP integration endpoints. Biomarker classification zones. Recommendation confidence scores. JWT refresh token rotation. |
| **1.5** | January 2026 | Added `/api/biomarkers/classify` endpoint. Pagination support for reports and history. Rate limiting implementation. |
| **1.0** | November 2025 | Initial API release. Authentication, biomarker reports, basic recommendations, user profile management. |

---

> [!NOTE]
> This API reference is auto-validated against the production OpenAPI specification. All example request/response payloads are tested in CI. For the machine-readable OpenAPI spec, see `/docs/openapi.yaml`. For SDK documentation, see `/docs/SDK_GUIDE.md`.
