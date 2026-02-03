# Authenticating to the GenIA-L API as an External Consumer

This guide provides detailed instructions for authenticating with the GenIA-L API. It explains the OIDC flow to generate an access token that can be used in the `Authorization` header of API requests, including support for multi-tenant access.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Authentication Flow Overview](#authentication-flow-overview)
   - [Basic Flow (Single Tenant)](#basic-flow)
   - [Multi-Tenant Flow](#multi-tenant-flow)
4. [Scope Requirements](#scope-requirements)
5. [Retrieving an Access Token](#retrieving-an-access-token)
   - [OIDC Overview](#oidc-overview)
   - [Token Endpoint](#token-endpoint)
   - [Required Parameters](#required-parameters)
   - [Example Request](#example-request)
   - [Response](#response)
6. [Making API Requests](#making-api-requests)
   - [Endpoint Structure](#endpoint-structure)
   - [Basic API Access (Single Tenant)](#basic-api-access)
   - [Multi-Tenant API Access](#multi-tenant-api-access)
7. [Setting Up Multi Tenant Connection](#setting-up-tenant-connection)
   - [Check Connection Status](#check-connection-status)
   - [Get Management URL](#get-management-url)
   - [Complete the Connection](#complete-the-connection)
8. [Access Levels](#access-levels)
9. [Token Management](#token-management)
   - [Token Expiration](#token-expiration)
   - [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)
11. [Quick Reference](#quick-reference)
12. [Further Reading](#further-reading)

---

<a name="introduction"></a>

## Introduction

The GenIA-L API uses OpenID Connect (OIDC) and OAuth 2.0 to authenticate external consumers. This guide explains how to retrieve an access token using the Client Credentials Grant Type and how to use it to authenticate API requests.

The API supports two authentication modes:

- **Basic API Access**: General Single Tenant API access
- **Multi-Tenant Access**: Multi Tenant API access

---

<a name="prerequisites"></a>

## Prerequisites

Before you start, you'll need:

1. **OAuth Client Credentials** (`client_id` and `client_secret`) issued during registration
2. **API Version** you want to access (e.g., `v1`, `v2`, `v3`, `v4`, `v5`)

If you don't have credentials yet, please contact your account manager.

---

<a name="authentication-flow-overview"></a>

## Authentication Flow Overview

<a name="basic-flow"></a>

### Basic Flow (Single Tenant)

```
1. Get OAuth Token (with scopes: openid sdu-genial-api)
   ↓
2. Make API Request with Authorization + X-API-Tenant-Id headers
```

<a name="multi-tenant-flow"></a>

### Multi-Tenant Flow

```
1. Get OAuth Token (with scopes: openid sdu-genial-api sdu-genial-api-multitenant)
   ↓
2. Make API Request with Authorization + X-API-Tenant-Id headers
   │
   ├─ No Connection Set Up → TRIAL plan (strict rate limits)
   │
   └─ Connection Set Up → BASIC plan (standard rate limits)
```

> **Note:** You can start making tenant requests immediately with just the `X-API-Tenant-Id` header. Without an active tenant connection, you'll have **TRIAL** access with strict rate limiting. To get **BASIC** access with higher limits, set up a tenant connection (see [Setting Up Tenant Connection](#setting-up-tenant-connection)).

---

<a name="scope-requirements"></a>

## Scope Requirements

The scopes you request determine what access you'll have:

| Scope                        | Purpose                  | Required For                           |
| ---------------------------- | ------------------------ | -------------------------------------- |
| `openid`                     | Base OIDC authentication | All requests                           |
| `sdu-genial-api`             | GenIA-L API access       | All requests                           |
| `sdu-genial-api-multitenant` | Multi-tenant-specific features | Requests for multiple tenants |

> **Important:** Include `sdu-genial-api-multitenant` in your token request if you plan to use multi tenant features. This scope must be present in the initial token - you cannot add it to an existing token.

---

<a name="retrieving-an-access-token"></a>

## Retrieving an Access Token

<a name="oidc-overview"></a>

### OIDC Overview

OpenID Connect (OIDC) is a protocol built on top of OAuth 2.0 that provides secure authentication and authorization. For a detailed understanding of the Client Credentials Grant Type used in this flow, refer to the [Ping Identity Developer Guide](https://docs.pingidentity.com/developer-resources/oauth_20_developer_guide/client-credentials-grant-type.html).

<a name="token-endpoint"></a>

### Token Endpoint

The endpoint for retrieving an access token is:

```
POST https://login.sdu.nl/as/token.oauth2
```

<a name="required-parameters"></a>

### Required Parameters

To retrieve an access token, send a POST request to the token endpoint with the following parameters in the body, encoded as `application/x-www-form-urlencoded`:

| Parameter       | Description                                                        | Required | Example Value                                                            |
| --------------- | ------------------------------------------------------------------ | -------- | ------------------------------------------------------------------------ |
| `client_id`     | The client ID issued during registration.                          | Yes      | `your-client-id`                                                         |
| `client_secret` | The client secret issued during registration.                      | Yes      | `your-client-secret`                                                     |
| `grant_type`    | The grant type for the request.                                    | Yes      | `client_credentials`                                                     |
| `scope`         | The scope(s) requested for the token (space-separated if multiple) | Yes      | `openid sdu-genial-api` optional with `sdu-genial-api-multitenant` scope |

<a name="example-request"></a>

### Example Request

#### For Basic API Access Only

```bash
curl -X POST https://login.sdu.nl/as/token.oauth2 \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "grant_type=client_credentials" \
  -d "scope=openid sdu-genial-api"
```

#### For Multi-Tenant Access

```bash
curl -X POST https://login.sdu.nl/as/token.oauth2 \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "grant_type=client_credentials" \
  -d "scope=openid sdu-genial-api sdu-genial-api-multitenant"
```

<a name="response"></a>

### Response

A successful response returns a JSON object containing the access token:

```json
{
  "access_token": "eyJraWQiOi...",
  "token_type": "Bearer",
  "expires_in": 7199
}
```

| Field          | Description                                             |
| -------------- | ------------------------------------------------------- |
| `access_token` | The token used to authenticate API requests             |
| `token_type`   | The type of token (always `Bearer`)                     |
| `expires_in`   | The token's lifetime in seconds (approximately 2 hours) |

Save the `access_token` - you'll use this for all API requests.

---

<a name="making-api-requests"></a>

## Making API Requests

<a name="endpoint-structure"></a>

### Endpoint Structure

All GenIA-L API requests follow this structure:

```plaintext
https://genial-api.sdu.nl/{VERSION}/{ENDPOINT}
```

- **`{VERSION}`**: API version (e.g., `v1`, `v2`, `v3`, `v4`, `v5`)
- **`{ENDPOINT}`**: Endpoint name (e.g., `step` for v1, `message` for v2+)

<a name="basic-api-access"></a>

### Basic API Access (Single Tenant)

For basic API access:

```bash
curl -X POST https://genial-api.sdu.nl/v5/message \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "X-API-Tenant-Id: YOUR CLIENT_ID" \
  -H "Content-Type: application/json" \
  -d '{ your request payload }'
```

This provides **default access** - usage plan assignment happens manually during onboarding.

> **Note:** For detailed information about request payloads and response structure, refer to the [Response Parsing Documentation](response_parsing.md).

<a name="multi-tenant-api-access"></a>

### Multi-Tenant API Access

To access the API with multiple Tenants, provide unique identifiers for your tenants to the `X-API-Tenant-Id` header. The TenantId needs to be an ID unique to each of your tenants:

```bash
curl -X POST https://genial-api.sdu.nl/v4/message \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "X-API-Tenant-Id: your-unique-tenant-id" \
  -H "Content-Type: application/json" \
  -d '{ your request payload }'
```

**Access Levels:**

- **Without active tenant connection:** Works immediately with **TRIAL** plan (strict rate limits)
- **With active tenant connection:** Upgrade to **BASIC** plan (standard rate limits) - see [Setting Up Tenant Connection](#setting-up-tenant-connection)

---

<a name="setting-up-tenant-connection"></a>

## Setting Up Tenant Connection

You can use mulit tenant features immediately with just the unique tenant identifier in the `X-API-Tenant-Id` header, but you'll be on the **TRIAL** plan with strict rate limits. To get **BASIC** plan access with higher limits, set up a tenant connection.

> **Prerequisites:** Your access token must include the `sdu-genial-api-multitenant` scope. If not, request a new token with the correct scopes (see [Retrieving an Access Token](#retrieving-an-access-token)).

<a name="check-connection-status"></a>

### Check Connection Status

Before setting up a connection, check if one already exists:

```bash
curl -X GET https://api-gateway-authentication-service.prod.sduoneplatform.nl/tenant-connection/status \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "X-API-Tenant-Id: your-tenant-id"
```

**Response:**

```json
{
  "connected": true
}
```

- `connected: true` → You're all set! Proceed to use the API with tenant access.
- `connected: false` → Continue to get the management URL.

<a name="get-management-url"></a>

### Get Management URL

If not connected, request a management URL to set up the connection:

```bash
curl -X GET https://api-gateway-authentication-service.prod.sduoneplatform.nl/tenant-connection/management-url \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "X-API-Tenant-Id: your-tenant-id"
```

**Response:**

```json
{
  "url": "https://oidc.ro2.nl/interaction/connect-tenant/cd2dec55-6138-4513-8103-640426bca752"
}
```

<a name="complete-the-connection"></a>

### Complete the Connection

1. **Open the management URL** in your browser (from the response above)
2. **Authenticate** when prompted
3. **Authorize the tenant connection** - follow the on-screen instructions
4. **Verify** the connection is complete:

```bash
curl -X GET https://api-gateway-authentication-service.prod.sduoneplatform.nl/tenant-connection/status \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "X-API-Tenant-Id: your-tenant-id"
```

You should now see `"connected": true`. The changes can take up to 5 minutes to propagate.

---

<a name="access-levels"></a>

## Access Levels

Your access level is determined automatically based on your connection status:

| Access Level | When                                       | Rate Limiting                       | What You Can Access                        |
| ------------ | ------------------------------------------ | ----------------------------------- | ------------------------------------------ |
| **DEFAULT**  | Using `X-API-Tenant-Id` as single tenant (tenant id === client_id)                      | Manual assignment during onboarding | Access with quota defined in your contract |
| **TRIAL**    | Using `X-API-Tenant-Id` without connection | Strict limits                       | Access with limited trial quota            |
| **BASIC**    | Using `X-API-Tenant-Id` with connection    | Standard limits                     | Access with basic quota                    |

---

<a name="token-management"></a>

## Token Management

<a name="token-expiration"></a>

### Token Expiration

Access tokens expire after approximately **2 hours** (7199 seconds). When you receive a `401 Unauthorized`:

1. Request a new token from the OIDC provider
2. Update your application with the new token
3. Retry your API request

<a name="best-practices"></a>

### Best Practices

- **Cache tokens** until they expire (check `expires_in` in token response)
- **Refresh proactively** before expiration if possible
- **Store securely** - never expose tokens in client-side code or logs

---

<a name="quick-reference"></a>

## Quick Reference

### Get Token (Basic Access)

```bash
curl -X POST https://login.sdu.nl/as/token.oauth2 \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id={YOUR_CLIENT_ID}" \
  -d "client_secret={YOUR_CLIENT_SECRET}" \
  -d "grant_type=client_credentials" \
  -d "scope=openid sdu-genial-api"
```

### Get Token (Multi-Tenant Access)

```bash
curl -X POST https://login.sdu.nl/as/token.oauth2 \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id={YOUR_CLIENT_ID}" \
  -d "client_secret={YOUR_CLIENT_SECRET}" \
  -d "grant_type=client_credentials" \
  -d "scope=openid sdu-genial-api sdu-genial-api-multitenant"
```

### Check Connection Status

```bash
curl -X GET https://api-gateway-authentication-service.prod.sduoneplatform.nl/tenant-connection/status \
  -H "Authorization: Bearer {TOKEN}" \
  -H "X-API-Tenant-Id: {TENANT_ID}"
```

### Get Management URL

```bash
curl -X GET https://api-gateway-authentication-service.prod.sduoneplatform.nl/tenant-connection/management-url \
  -H "Authorization: Bearer {TOKEN}" \
  -H "X-API-Tenant-Id: {TENANT_ID}"
```

### Make API Request

```bash
curl -X POST https://genial-api.sdu.nl/v5/message \
  -H "Authorization: Bearer {TOKEN}" \
  -H "X-API-Tenant-Id: {TENANT_ID}" \
  -H "Content-Type: application/json" \
  -d '{ your request payload }'
```

> **Note:** `X-API-Tenant-Id` header is required, also for single tenant mode - In Single Tenant mode the Tenant ID needs to equal the Client ID. In Multi Tenant mode the TenantId needs to be an ID unique to each of your tenants.

---

<a name="further-reading"></a>

## Further Reading

- [Ping Identity OAuth 2.0 Developer Guide](https://docs.pingidentity.com/developer-resources/oauth_20_developer_guide/client-credentials-grant-type.html)
- [OpenID Configuration](https://federate.prod.ping.awssdu.nl/.well-known/openid-configuration)
- [Response Parsing Documentation](response_parsing.md) - Details about request payloads and response structure for the `/message` endpoint

---
