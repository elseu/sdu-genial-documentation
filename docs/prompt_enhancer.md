# Prompt Enhancement API

This guide provides detailed instructions for using the GenIA-L Prompt Enhancement API. It explains how to send prompt enhancement requests and receive intelligent suggestions to improve your prompts, with support for maintaining conversation history to avoid duplicate suggestions.

---

## Table of Contents

1. [Introduction](#introduction)
2. [API Endpoint](#api-endpoint)
3. [Request Structure](#request-structure)
   - [Request Payload](#request-payload)
   - [Example Request](#example-request)
4. [Response Structure](#response-structure)
   - [Response Format](#response-format)
   - [Example Response](#example-response)
5. [Use Cases](#use-cases)
6. [Authentication](#authentication)
7. [Rate Limiting](#rate-limiting)

---

<a name="introduction"></a>

## Introduction

The Prompt Enhancement API provides intelligent suggestions to improve your prompts. Unlike streaming endpoints, this API returns a simple JSON response with prompt suggestions. You can maintain a history of previous suggestions to ensure the model provides fresh, non-repetitive recommendations. The API does _NOT_ return a new prompt, it returns a suggestion you could implement to further specify your prompt.

**Key Features:**

- **Non-streaming**: Returns complete JSON response
- **History-aware**: Avoids duplicate suggestions by tracking previous recommendations
- **Flexible integration**: Perfect for autocomplete, validation, and iterative improvement workflows

---

<a name="api-endpoint"></a>

## API Endpoint

The endpoint for prompt enhancement is:

```plaintext
POST https://genial-api.sdu.nl/{VERSION}/enhance-prompt
```

- **`{VERSION}`**: Replace with the API version (e.g. `v4`, not available in prior versions).

### Example Request

```http
POST https://genial-api.sdu.nl/v4/enhance-prompt
Authorization: Bearer eyJraWQiOi...
Content-Type: application/json
```

---

<a name="request-structure"></a>

## Request Structure

<a name="request-payload"></a>

### Request Payload

The API expects a JSON payload with the following structure:

| Field     | Type   | Description                              | Required |
| --------- | ------ | ---------------------------------------- | -------- |
| `prompt`  | string | The prompt you want to enhance           | Yes      |
| `history` | array  | Array of previous suggestions (optional) | No       |

**History Array Structure:**

Each item in the `history` array should contain:

| Field        | Type   | Description                             | Required |
| ------------ | ------ | --------------------------------------- | -------- |
| `suggestion` | string | The previous suggestion text            | Yes      |
| `prompt`     | string | The original prompt for that suggestion | Yes      |

<a name="example-request"></a>

### Example Request

```json
{
  "prompt": "wat is de vpb voor een schildersbedrijf?",
  "history": [
    {
      "suggestion": "Vermeld of je vraag betrekking heeft op de vennootschapsbelasting (VPB) voor een specifieke situatie of algemeen, en voeg eventueel de context of het type onderneming toe.",
      "prompt": "wat is de vpb?"
    }
  ]
}
```

**Explanation:**

- The **prompt** field contains the current prompt you want to enhance
- The **history** array contains previous suggestions to avoid repetition
- Each history item includes both the suggestion and the original prompt it was based on

---

<a name="response-structure"></a>

## Response Structure

<a name="response-format"></a>

### Response Format

The API returns a JSON object with the following structure:

| Field        | Type   | Description                       |
| ------------ | ------ | --------------------------------- |
| `suggestion` | string | The enhanced prompt suggestion    |
| `prompt`     | string | The original prompt (echoed back) |

<a name="example-response"></a>

### Example Response

```json
{
  "suggestion": "Vermeld of je vraag betrekking heeft op de vennootschapsbelasting (VPB) voor een specifieke situatie of algemeen, en voeg eventueel de context of het type onderneming toe.",
  "prompt": "wat is de vpb?"
}
```

---

<a name="use-cases"></a>

## Use Cases

The Prompt Enhancement API can be integrated into various workflows to improve user experience and prompt quality.

### Autocomplete Implementation

Build an autocomplete experience that suggests prompt improvements as users type.

### Prompt Validation Workflow

Integrate into your AI workflow to validate and improve prompts before processing.

### Iterative Prompt Improvement

Create a sophisticated prompt improvement system with multiple iterations.

---

<a name="authentication"></a>

## Authentication

The Prompt Enhancement API uses the same authentication mechanism as other GenIA-L API endpoints. Include the Bearer token in the Authorization header:

```http
Authorization: Bearer {access_token}
```

For detailed authentication instructions, refer to the [Authentication Guide](./authentication.md).

---

<a name="rate-limiting"></a>

## Rate Limiting

The Prompt Enhancement API enforces the same rate-limiting policies as other GenIA-L API endpoints. These limits ensure fair usage and system stability. Please contact your technical consultant for more information about your current usage plan.

---

## Best Practices

1. **History Management**: Keep your suggestion history manageable (10-20 items max) to avoid large payloads
2. **Debouncing**: Implement debouncing for autocomplete features to avoid excessive API calls
3. **Error Handling**: Always implement proper error handling for network failures
4. **Caching**: Consider caching suggestions for similar prompts to reduce API calls
5. **User Feedback**: Allow users to accept/reject suggestions to improve your history tracking

---

## Conclusion

The Prompt Enhancement API provides a powerful way to improve prompt quality through intelligent suggestions. By maintaining conversation history and implementing appropriate workflows, you can create sophisticated prompt improvement experiences that enhance user interactions with AI systems.

For more information about other GenIA-L API endpoints, refer to the [Response Parsing Guide](./response_parsing.md) and [Authentication Guide](./authentication.md).
