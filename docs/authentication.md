# Authenticating to the GenIA-L API as an External Consumer

This guide provides detailed instructions for authenticating with the GenIA-L API. It explains the OIDC flow to generate an access token that can be used in the `Authorization` header of API requests.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Retrieving an Access Token](#retrieving-an-access-token)
   - [OIDC Overview](#oidc-overview)
   - [Token Endpoint](#token-endpoint)
   - [Required Parameters](#required-parameters)
   - [Example Request](#example-request)
   - [Response](#response)
3. [Using the Access Token](#using-the-access-token)
   - [Authorization Header](#authorization-header)
   - [Example Request](#example-auth-header)
4. [Further Reading](#further-reading)

---

<a name="introduction"></a>

## Introduction

The GenIA-L API uses OpenID Connect (OIDC) and OAuth 2.0 to authenticate external consumers. This guide explains how to retrieve an access token using the Client Credentials Grant Type and how to use it to authenticate API requests.

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
POST https://federate.acc.ping.awssdu.nl/as/token.oauth2
```

<a name="required-parameters"></a>

### Required Parameters

To retrieve an access token, send a POST request to the token endpoint with the following parameters in the body, encoded as `application/x-www-form-urlencoded`:

| Parameter       | Description                                   | Required | Example Value          |
| --------------- | --------------------------------------------- | -------- | ---------------------- |
| `client_id`     | The client ID issued during registration.     | Yes      | `{your-client-id}`     |
| `client_secret` | The client secret issued during registration. | Yes      | `{your-client-secret}` |
| `grant_type`    | The grant type for the request.               | Yes      | `client_credentials`   |
| `scope`         | The scope(s) requested for the token.         | Yes      | `openid sdu-gen-ai`    |

<a name="example-request"></a>

### Example Request

```http
POST https://federate.acc.ping.awssdu.nl/as/token.oauth2
Content-Type: application/x-www-form-urlencoded

client_id=your-client-id
&client_secret=your-client-secret
&grant_type=client_credentials
&scope=openid sdu-gen-ai
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

- `access_token`: The token used to authenticate API requests.
- `token_type`: The type of token (always `Bearer`).
- `expires_in`: The token's lifetime in seconds.

---

<a name="using-the-access-token"></a>

## Using the Access Token

<a name="authorization-header"></a>

### Authorization Header

To authenticate a request to the GenIA-L API, include the access token in the `Authorization` header:

```http
Authorization: Bearer {access_token}
```

<a name="example-auth-header"></a>

### Example Request: Authenticated API Call

Below is an example of how to make an authenticated API request to the `/step` endpoint. For detailed information about the payload and response structure, refer to the `RESPONSE_PARSING` documentation.

#### Endpoint Structure

```plaintext
https://secure.{VERSION}.genial-ext.{ENVIRONMENT}.sduoneplatform.nl/step
```

- **`{VERSION}`**: Replace with the API version (e.g., `v1`, `v2`, or `v3`).
- **`{ENVIRONMENT}`**: Replace with the environment (`acc` for acceptance, `prod` for production).

#### Example Request

```http
POST https://secure.v1.genial-ext.acc.sduoneplatform.nl/step
Authorization: Bearer eyJraWQiOi...
```

#### Notes:

- **Authorization**: The `Authorization` header must include a valid Bearer token.
- **Version and Environment**: Ensure the correct API version and environment are specified based on your deployment context.
- **Payload and Response**: See `RESPONSE_PARSING` for comprehensive details about constructing the payload and interpreting the response.

---

<a name="further-reading"></a>

## Further Reading

- [Ping Identity OAuth 2.0 Developer Guide](https://docs.pingidentity.com/developer-resources/oauth_20_developer_guide/client-credentials-grant-type.html)
- [OpenID Configuration](https://federate.prod.ping.awssdu.nl/.well-known/openid-configuration)

---
