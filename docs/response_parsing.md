# Handling and Parsing the GenIA-L API Response

This guide provides detailed instructions for interacting with the GenIA-L API that returns a stream of text. It covers the request payload structure, explains the response format, and provides parsing logic with examples, including a TypeScript code example. The goal is to help you correctly parse the streamed data and extract meaningful information, regardless of the programming language or framework you are using.

## Table of Contents

1. [API Request Structure](#api-request)
   - [Request Endpoint](#request-endpoint)
   - [Request Payload](#request-payload)
   - [Request Rate Limiting](#rate-limiting)
   - [Example Request](#example-request)
2. [Overview of the API Response Format](#overview)
3. [Parsing the Response](#parsing)
   - [Start and End Tokens](#tokens)
   - [Sections of the Response](#sections)
   - [Content Part Types](#content-types)
   - [Content Part Structure](#content-structure)
   - [Content Part Markdown](#content-markdown)
   - [Metadata Structure](#metadata-structure)
4. [Implementing the Parser](#implementation)
   - [Parsing Logic](#logic)
   - [State Management](#state)
5. [TypeScript Example Implementation](#typescript)
6. [Conclusion](#conclusion)

---

<a name="api-request"></a>

## 1. API Request Structure

Before parsing the response, it's essential to understand how to structure the API request.

<a name="request-endpoint"></a>

### Request Endpoint

```plaintext
https://genial-api.sdu.nl/{VERSION}/{ENDPOINT}
```

- **`{VERSION}`**: Replace with the API version (e.g., `v1`, `v2`, or `v3`).
- **`{ENDPOINT}`**: Replace with the endpoint belonging to the version (`step` for v1, `message` for v2+).

#### Example Request

```http
POST https://genial-api.sdu.nl/v3/message
Authorization: Bearer eyJraWQiOi...
```

<a name="rate-limiting"></a>

### Request Rate Limiting

To prevent server overload, the API endpoints enforce strict rate-limiting policies based on AWS Usage plans. These limits ensure fair usage and system stability. Please contact your technical consultant for more information about your current usage plan.

<a name="request-payload"></a>

### Request Payload

Besides the required Authorization header, the API expects a JSON payload with the following structure:

- **feature** (`string`): The feature you are accessing (e.g., `"chat"`).
- **messages** (`array`): An array of message objects representing the conversation history.

Each message object in the **messages** array has the following structure:

- **role** (`string`): The role of the message sender (`"user"` or `"assistant"`).
- **content** (`array`): An array of content parts.
- **metadata** (`object`, optional): Additional metadata for the message.

Each content part in the **content** array is an object with the following properties:

- **type** (`string`): The type of content part (e.g., `"question"`, `"answer"`, `"system"`, `"suggestion"`).
- **value** (`string`): The actual content value.

<a name="example-request"></a>

### Example Request

```json
{
  "feature": "chat",
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "question",
          "value": "What is the capital of France?"
        }
      ]
    },
    {
      "role": "assistant",
      "content": [
        {
          "type": "answer",
          "value": "The capital of France is Paris."
        }
      ],
      "metadata": {...}
    },
    {
      "role": "user",
      "content": [
        {
          "type": "question",
          "value": "How many people live in this city?"
        }
      ]
    }
  ]
}
```

**Explanation**:

- The **feature** is set to `"chat"`.
- The **messages** array contains the conversation history.
  - The first message is from the user, containing a question.
  - The second message is from the assistant, containing an answer and all the metadata.
  - The third message is from the user, containing a followup question.

---

<a name="overview"></a>

## 2. Overview of the API Response Format

The API returns a streamed response containing different sections, each marked by specific start and end tokens. The response includes metadata and content parts that need to be parsed separately.

### Example Response

```plaintext
[#START_OF_CONTENT_PART_1<ANSWER>#]>
The capital of France is Paris.
[#END_OF_CONTENT_PART_1<ANSWER>#]
[#START_OF_METADATA#]
{
  "references": [],
  "output": {
    "query_analyser": {
      "query": "What is the capital of France?",
      "keywords": "capital, France",
      "law_ids": []
    },
    "workflow_selector": null
  }
}
[#END_OF_METADATA#]
```

### Example ERROR response

```plaintext
[#START_OF_CONTENT_PART_1<ANSWER>#]>
The capital of France is Paris.
[#END_OF_CONTENT_PART_1<ANSWER>#]
[#START_OF_ERROR#]
{
  "description": "SduGenAIError: ComponentError: SparseRetrieverError: retrieved no results for query wat is loonheffing?.",
  "user_description": "We konden geen relevante documenten vinden voor je vraag.",
  "code": 112
}
[#END_OF_ERROR#]
```

The response is a mix of structured data (JSON) and unstructured text, divided into sections demarcated by tokens.

---

<a name="parsing"></a>

## 3. Parsing the Response

To work with the API response, you need to parse the streamed data by identifying the different sections and extracting the relevant information for your application.

<a name="tokens"></a>

### Start and End Tokens

Each section in the response is marked by start and end tokens in the following format:

- **Start token**: `[#START_OF_SECTION_NAME#]`
- **End token**: `[#END_OF_SECTION_NAME#]`

For example:

- **Start of Metadata**: `[#START_OF_METADATA#]`
- **End of Metadata**: `[#END_OF_METADATA#]`

<a name="sections"></a>

### Sections of the Response

The response can contain the following sections:

1. **Content Part**: Contains content as Markdown.
2. **Metadata**: Contains additional metadata in JSON format. Always sent as 1 complete JSON chunk.
3. **Error**: Contains error information if any. Always sent as 1 complete JSON chunk.

**Content Parts** can have different types, specified in the start and end tokens:

- `CONTENT_PART_n<TYPE>` where `n` is the part id(x), and `TYPE` is the content type (e.g., `ANSWER`, `QUESTION`).

For example:

- **Start of Content Part 1 of type ANSWER**: `[#START_OF_CONTENT_PART_1<ANSWER>#]`
- **End of Content Part 1 of type ANSWER**: `[#END_OF_CONTENT_PART_1<ANSWER>#]`

<a name="content-types"></a>

### Content Part Types

Content parts can be of various types:

- **ANSWER**: An answer from the assistant.
- **QUESTION**: A question from the user.
- **SUGGESTION**: A suggestion for the user.
- **JSON**: A message with a JSON body.

<a name="content-structure"></a>

### Content Part Structure

Each content part can be extracted to the following properties useful to the application:

- **type** (`string`): The type of content part (e.g., `"answer"`, `"question"`, `"suggestion"`, `"system"`).
- **value** (`string`): The textual content in Markdown.
- **part_id** (`string`): The identifier of the content part.
- **status** (`string`, optional): The status of the content part (`"streaming"`, `"done"`, or `"error"`).

**Content Part Types and Their Expected Structure**:

- **ANSWER**:
  - **type**: `"answer"`
  - **value**: The assistant's response text in Markdown.
- **QUESTION**:
  - **type**: `"question"`
  - **value**: The user's question text in Markdown.
- **SUGGESTION**:
  - **type**: `"suggestion"`
  - **value**: The assistant's suggested text in Markdown.
- **JSON**:
  - **type**: `"json"`
  - **value**: The complete JSON payload containing component data
  - **component**: The component identifier (e.g., "reasoning")
  - **props**: Component-specific data extracted from the `value` field

<a name="content-markdown"></a>

### Content Part Markdown

It's important to know that the GenIA-L API responds with Markdown in the Content Parts. This means that you need a Markdown parser to render the response as it is intended. The Markdown also consists of Custom Markdown Components. As of writing we return 1 Custom Markdown Component for the Footnotes.

- **Footnotes**:
  Footnotes are returned as `<InlineReference ...props>{number}</InlineReference>` with the same props on it as the [normal references in the Metadata](#metadata-structure). If you do not want to support these footnotes you will need to configure your Markdown parser to exclude specific nodes, all HTML, or manually strip the response of the open and closing tags.

<a name="json-content-part"></a>

### Content Part JSON

A JSON content part is a structured response that contains information that can be used to render different UI elements. Unlike regular content parts that contain text, JSON content parts provide structured data that applications can use to build interactive components.

In V4 we introduced JSON content parts as a new content type. These structured responses allow the API to provide rich, interactive data that can be rendered as specialized UI components. The first type of JSON content part we've implemented is the **Reasoning** component, which exposes the AI's thinking process.

#### JSON Content Part Structure

All JSON content parts follow this general structure:

```typescript
{
  "type": "json",
  "component": string,        // The component type identifier
  "value": object,           // Component-specific data
  "part_id": string,         // Optional part identifier
  "status": string           // Optional status (streaming, done, error)
}
```

#### JSON Content Part Streaming Behavior

**Important**: JSON content parts are streamed as **complete chunks** and are **overwritten** with new chunks that have the same `PART_N` ID. This means:

- **Not deltas**: New chunks are not incremental updates - they are complete new versions of the state
- **Overwrite behavior**: Each new chunk with the same part ID completely replaces the previous version
- **Final state**: You should always use the final JSON content part of a given `PART_N` to get the complete state

**Example streaming sequence:**

```plaintext
[#START_OF_CONTENT_PART_1<JSON>#]
{
  "type":"json",
  "component":"reasoning",
  "value": {
    "title":"Aan het nadenken"
  }
}
[#END_OF_CONTENT_PART_1<JSON>#]

[#START_OF_CONTENT_PART_1<JSON>#]
{
  "type":"json",
  "component":"reasoning",
  "value": {
    "title":"Klaar met nadenken",
    "is_reasoning":false,
    "steps":[
      {
        "type":"thought",
        "label":"Vraag geformuleerd",
        "content":"Wat is de vennootschapsbelasting (vpb)?"
      }
    ]
  }
}
[#END_OF_CONTENT_PART_1<JSON>#]
```

In this example, the second chunk completely overwrites the first one. Your application should:

1. **Replace** the previous state with the new complete state
2. **Not merge** or diff the content
3. **Use the final version** when processing the complete response

#### Available JSON Components

Currently, the following JSON components are available:

1. **Reasoning Component** (`component: "reasoning"`) - Exposes AI thinking process
2. _More components will be added in future versions_

#### Reasoning Component

The reasoning component is the first JSON content part type. This reasoning payload allows us to expose the thinking of the GenIA-L API and let consumers render this reasoning in their application. The content that used to be in the SYSTEM messages has been moved to this reasoning component:

```
[#START_OF_CONTENT_PART_1<JSON>#]
{
  "type":"json",
  "component":"reasoning",
  "value":
    {
      "title":"Klaar met nadenken",
      "is_reasoning":false,
      "steps":[
        {
          "type":"thought",
          "label":"Vraag geformuleerd ",
          "content":"Wat is de vennootschapsbelasting (vpb)?"
        }
      ]
    }
  }
[#END_OF_CONTENT_PART_1<JSON>#]
```

### Reasoning Component

The reasoning component is a structured JSON payload that exposes the AI's thinking process and decision-making steps. This component allows applications to render the AI's reasoning in a user-friendly way, providing transparency into how the AI arrived at its conclusions.

#### Structure

The reasoning component follows this structure:

```typescript
{
  "type": "json",
  "component": "reasoning",
  "value": {
    "title": string,
    "is_reasoning": boolean,
    "steps": ReasoningStep[]
  }
}
```

#### Properties

- **`title`** (`string`): A descriptive title for the reasoning process (e.g., "Klaar met nadenken", "Thinking complete").
- **`is_reasoning`** (`boolean`, optional): Indicates whether the AI is currently in a reasoning state. When `false`, it means the reasoning process is complete.
- **`steps`** (`array`): An array of reasoning steps that show the AI's thought process.

#### Reasoning Steps

Each step in the `steps` array can be one of two types:

##### Thought Steps

Thought steps represent the AI's internal reasoning and analysis:

```typescript
{
  "type": "thought",
  "label": string,        // Optional descriptive label
  "content": string       // The actual thought content
}
```

**Example:**

```json
{
  "type": "thought",
  "label": "Vraag geformuleerd",
  "content": "Wat is de vennootschapsbelasting (vpb)?"
}
```

##### Action Steps

Action steps represent specific actions or decisions the AI is taking:

```typescript
{
  "type": "action",
  "label": string,        // Optional descriptive label
  "content": ActionContentItem[]
}
```

Where `ActionContentItem` has the structure:

```typescript
{
  "label": string         // Description of the action item
}
```

**Example:**

```json
{
  "type": "action",
  "label": "Searching for information",
  "is_reasoning": true,
  "content": [
    {
      "label": "Querying legal database"
    },
    {
      "label": "Analyzing tax regulations"
    }
  ]
}
```

#### Complete Example

Here's a complete example of a reasoning component:

```json
{
  "type": "json",
  "component": "reasoning",
  "value": {
    "title": "Klaar met nadenken",
    "is_reasoning": false,
    "steps": [
      {
        "type": "thought",
        "label": "Vraag geformuleerd",
        "content": "Wat is de vennootschapsbelasting (vpb)?"
      },
      {
        "type": "action",
        "label": "Zoeken naar informatie",
        "content": [
          {
            "label": "Zoeken in juridische database"
          },
          {
            "label": "Analyseren van belastingregelgeving"
          }
        ]
      },
      {
        "type": "thought",
        "label": "Analyse voltooid",
        "content": "Ik heb relevante informatie gevonden over vennootschapsbelasting en kan nu een uitgebreid antwoord geven."
      }
    ]
  }
}
```

#### Implementation Guidelines

When implementing support for the reasoning component:

1. **Parse the JSON structure** carefully, ensuring all required fields are present.
2. **Handle the `is_reasoning` flag** to show appropriate UI states (e.g., loading indicators when `true`).
3. **Render steps sequentially** to show the progression of the AI's thinking.
4. **Differentiate between step types** using appropriate visual indicators (e.g., icons for thoughts vs. actions).
5. **Support optional labels** to provide context for each step.
6. **Handle streaming updates** as the reasoning process evolves.

#### UI Considerations

- **Progressive disclosure**: Consider showing a summary initially with the option to expand detailed reasoning.
- **Visual hierarchy**: Use different styling for thought steps vs. action steps.
- **Loading states**: Show appropriate indicators when `is_reasoning` is `true`.
- **Accessibility**: Ensure reasoning steps are accessible to screen readers and other assistive technologies.

#### Implementing Support for JSON Components

When implementing support for JSON content parts:

1. **Check the component type**: Use the `component` field to determine which component you're dealing with
2. **Parse component-specific data**: Access the `props` field for component-specific data
3. **Handle unknown components gracefully**: Provide fallback behavior for components you don't recognize
4. **Support streaming updates**: Handle status changes as components stream in
5. **Handle overwrite behavior**: Always replace previous state with new complete state (don't merge)

#### Handling Multiple JSON Chunks with Same Part ID

When processing a complete response, you may encounter multiple JSON content parts with the same part ID. You need to:

1. **Extract the final version**: Use the last JSON content part for each part ID
2. **Update your regex**: Ensure your parsing logic handles multiple chunks with the same ID
3. **State management**: Always overwrite your local state with the complete new object

**Regex for extracting final JSON content parts:**

```typescript
// This regex will match the json contentparts
const finalJsonParts = response.match(/\[#START_OF_CONTENT_PART_(\d+)<JSON>#\][\s\S]*?\[#END_OF_CONTENT_PART_\1<JSON>#\]/g);

// Or for a more comprehensive approach, extract all and keep the last one per part ID
const allJsonParts = response.match(/\[#START_OF_CONTENT_PART_(\d+)<JSON>#\][\s\S]*?\[#END_OF_CONTENT_PART_\1<JSON>#\]/g);
const partMap = new Map();

allJsonParts?.forEach((part) => {
  const partId = part.match(/\[#START_OF_CONTENT_PART_(\d+)<JSON>#\]/)?.[1];
  if (partId) {
    partMap.set(partId, part);
  }
});

// partMap now contains only the final version of each part
```

#### Future JSON Components

The JSON content part system is designed to be extensible. When new components are introduced, they will follow the same structure with a unique `component` identifier and component-specific `props`.

#### Migration from V3

In V3, some reasoning information was included in SYSTEM messages. In V4, this has been moved to the dedicated reasoning component for better structure and parsing. When migrating from V3:

- Look for JSON content parts with `component: "reasoning"`
- Parse the structured `value` object instead of extracting from SYSTEM messages
- Update your UI to handle the new structured format
- Implement a component router to handle different JSON component types

<a name="metadata-structure"></a>

### Metadata Structure

The **metadata** section provides additional information about the response. It consists of:

- **references** (`array`): A list of references used in the response.
- **output** (`object`, optional): Additional output information.
- **smart_actions** (`array`, optional): A list of proposed followup actions.

**Reference Structure**:

Each reference in the **references** array has the following properties:

- **chunk_id** (`string`): Identifier of the content chunk.
- **chunk_txt** (`string`, optional): Text content of the chunk.
- **chunk_url** (`string`, optional): URL to access the chunk.
- **link_attributes** (`object`, optional): Additional attributes for the link.
- **title** (`string`): Title of the reference.
- **source** (`string`, optional): Source identifier.
- **publication_identifier** (`string`, optional): Identifier of the publication.
- **publication_title** (`string`, optional): Title of the publication.
- **publisher_identifier** (`string`, optional): Identifier of the publisher.
- **validity** (`object`, optional): Validity information.

**Validity Structure**:

The **validity** object indicates whether the reference is valid and provides a rationale:

- **is_valid** (`boolean`): Indicates if the reference is valid.
- **rationale** (`string`, optional): Explanation of the validity.

**Output Structure**:

The **output** object can contain:

- **query_analyser** (`object`, optional): Information about the query analysis.
- **workflow_selector** (`object`, optional): Information about the selected workflow.

**Query Analyser Structure**:

- **query** (`string`, optional): The original query.
- **question** (`string`, optional): The reformulated question.
- **keywords** (`string`, optional): Extracted keywords.
- **law_ids** (`array`, optional): List of law identifiers.

**Workflow Selector Structure**:

- **option** (`string`): Selected workflow option (e.g., `"KEYWORDS_SEARCH"`, `"QUESTION_SEARCH"`, `"CHAT_TASK"`, `"SUMMARIZE_ECLI"`).
- **matched_pattern** (`string`, optional): Pattern that was matched.

**Smart Action Structure**:

Each smart action object in the **smart_actions** array has the following properties:

- **action** (`enum`): The type of action (e.g., `SmartActionType.Default`, `SmartActionType.Translate`).
- **prompt** (`string`): The prompt associated with the action.
- **label** (`string`): The user-facing label for the action.
- **icon** (`enum`): The icon associated with the action (e.g., `SmartActionIcon.Scale`, `SmartActionIcon.Languages`).

---

<a name="implementation"></a>

## 4. Implementing the Parser

To parse the response, you need to:

1. Read the streamed data incrementally.
2. Identify the start and end tokens to determine the current section.
3. Extract the content between the tokens.
4. Parse JSON content when applicable.
5. Handle content parts differently based on their type.

<a name="logic"></a>

### Parsing Logic

Here is the high-level parsing logic:

1. **Initialize** an empty buffer and set the current section to `null`.
2. **As data is streamed**:
   - **Append** it to the buffer.
   - **Check for a start token** in the buffer.
     - If found, **update the current section** based on the token.
     - Remove the token from the buffer.
   - **If the current section is set**:
     - **Check for the corresponding end token** in the buffer.
       - If found, **extract the data** between the start and end tokens.
       - **Process the data** based on the current section.
       - **Reset the current section** to `null`.
       - Remove the token from the buffer.
   - **If the current section is `CONTENT_PART`**:
     - **Process the content incrementally** (especially important for streaming text).
     - **For JSON content parts**: Handle overwrite behavior - new chunks with the same part ID replace previous versions.
3. **Continue** until the end of the stream.

**Special consideration for JSON content parts**: When processing a complete response, you may encounter multiple JSON content parts with the same part ID. Always use the final version of each part ID, as earlier versions are overwritten by later ones.

<a name="state"></a>

### State Management

You need to maintain the following state:

- **Buffer**: Holds the accumulated data from the stream.
- **Current Section**: Indicates which section is currently being processed.
- **Content Part Index**: If in a content part, keeps track of the part id(x).
- **Content Part Type**: The type of content part being processed.

---

<a name="typescript"></a>

## 5. TypeScript Example Implementation

Below is an example of how to implement the parser in TypeScript.

### Parsing in TypeScript

**Note**: This example assumes you are working in a browser or Node.js environment that supports streams and TypeScript.

#### Parsing Logic Implementation

```typescript
/* eslint-disable @typescript-eslint/naming-convention */
/* eslint-disable @typescript-eslint/no-unnecessary-condition */

interface ContentPart {
  type: string;
  value: string | object;
  part_id: string | null;
  status?: string;
  component?: string;
  props?: object;
}

enum ContentPartStatus {
  Streaming = "streaming",
  Done = "done",
  Error = "error",
}

enum ContentPartType {
  Answer = "answer",
  Question = "question",
  Suggestion = "suggestion",
  System = "system",
  Json = "json",
}

enum StreamSection {
  Metadata = "metadata",
  Error = "error",
  ContentPart = "content_part",
}

type ChunkData =
  | { key: StreamSection.Metadata; value: any }
  | { key: StreamSection.ContentPart; value: ContentPart }
  | { key: StreamSection.Error; value: any };

type ChunkCallback = (chunk: ChunkData) => void;

interface ParseState {
  buffer: string;
  currentSection: StreamSection | null;
  contentBuffer: string;
  contentPartIndex: string | null;
  contentPartType: string | null;
}

/**
 * Reads from a ReadableStream and processes the chunks according to the provided chunk handler.
 * It tracks the parsing state internally and routes chunks of data to their corresponding section.
 *
 * @param reader - The stream reader for reading the data.
 * @param chunkHandler - A callback function that gets called with the parsed chunks,
 * allowing the caller to process the data.
 */
export async function readAndParseStream(reader: ReadableStreamDefaultReader<Uint8Array>, chunkHandler: ChunkCallback) {
  const state: ParseState = {
    buffer: "",
    currentSection: null,
    contentBuffer: "",
    contentPartIndex: null,
    contentPartType: null,
  };

  await readStream(reader, (chunk) => {
    parseChunk(chunk, state, chunkHandler);
  });
}

/**
 * Reads from a given stream and decodes the data as UTF-8 text. For each chunk read, the `onChunk` callback is invoked.
 *
 * @param reader - The stream reader for reading the data.
 * @param onChunk - A callback function invoked with each decoded chunk of data.
 */
async function readStream(reader: ReadableStreamDefaultReader<Uint8Array>, onChunk: (chunk: string) => void): Promise<void> {
  const decoder = new TextDecoder();
  while (true) {
    const { done, value } = await reader.read();
    if (done) {
      break;
    }
    const decodedValue = decoder.decode(value, { stream: true });
    onChunk(decodedValue);
  }
}

/**
 * Parses a chunk of data from the stream and updates the internal parsing state.
 *
 * @param chunk - The incoming chunk of data from the stream.
 * @param state - The internal parsing state, tracking buffers and current section.
 * @param onParsedChunk - Callback to process the parsed chunk.
 */
function parseChunk(chunk: string, state: ParseState, onParsedChunk: ChunkCallback): void {
  state.buffer += chunk;

  while (true) {
    if (!state.currentSection) {
      const startTokenMatch = state.buffer.match(/\[#START_OF_(CONTENT|METADATA|ERROR|CONTENT_PART_\d+(?:<[^>]+>)?)#\]/);
      if (startTokenMatch) {
        const newState = handleStartToken(state.buffer, startTokenMatch);
        Object.assign(state, newState);

        // Manually add a chunk for the content part to indicate streaming has started
        if (state.currentSection === StreamSection.ContentPart) {
          onParsedChunk({
            key: StreamSection.ContentPart,
            value: {
              type: state.contentPartType as ContentPartType,
              value: "",
              status: ContentPartStatus.Streaming,
              part_id: state.contentPartIndex,
            },
          });
        }
        continue;
      }
    }

    const endTokenMatch = state.buffer.match(/\[#END_OF_(CONTENT|METADATA|ERROR|CONTENT_PART_\d+(?:<[^>]+>)?)#\]/);

    if (endTokenMatch) {
      const data = state.buffer.slice(0, endTokenMatch.index);

      if (state.currentSection === StreamSection.ContentPart) {
        // Make sure we process any remaining content before the end token
        if (data) {
          processContentPart(data, state, onParsedChunk);
        }

        // Manually add a chunk for the content part to indicate streaming has ended
        onParsedChunk({
          key: StreamSection.ContentPart,
          value: {
            type: state.contentPartType as ContentPartType,
            value: "",
            status: ContentPartStatus.Done,
            part_id: state.contentPartIndex,
          },
        });
      } else {
        handleEndToken(data, state, onParsedChunk);
      }

      state.buffer = state.buffer.slice(endTokenMatch.index! + endTokenMatch[0].length);
      state.currentSection = null;
      state.contentPartIndex = null;
      state.contentPartType = null;
      continue;
    }

    if (state.currentSection === StreamSection.ContentPart) {
      processContentPart(state.buffer, state, onParsedChunk);
      state.buffer = "";
    }

    break;
  }
}

/**
 * Handles the start of a new section based on the token matched in the stream buffer.
 *
 * @param buffer - The current stream buffer.
 * @param startTokenMatch - The regex match object for the start token.
 *
 * @returns The updated state with the new section and buffer.
 */
function handleStartToken(buffer: string, startTokenMatch: RegExpMatchArray): Partial<ParseState> {
  const section = startTokenMatch[1].toLowerCase();
  const newSection = section.startsWith("content_part") ? StreamSection.ContentPart : (section as StreamSection);
  const newContentPartIndex = section.startsWith("content_part") ? section.split("_")[2].split("<")[0] : null;
  const newContentPartType = section.startsWith("content_part") ? section.match(/<([^>]+)>/)?.[1].toLowerCase() || null : null;
  const newBuffer = buffer.slice(startTokenMatch.index! + startTokenMatch[0].length);
  return {
    buffer: newBuffer,
    currentSection: newSection,
    contentPartIndex: newContentPartIndex,
    contentPartType: newContentPartType,
  };
}

/**
 * Handles the end of a section by processing the data between start and end tokens.
 *
 * @param data - The data between the start and end tokens.
 * @param state - The current parsing state.
 * @param onChunk - The callback to handle the parsed data chunk.
 */
function handleEndToken(data: string, state: ParseState, onChunk: ChunkCallback) {
  if (!data) return;

  switch (state.currentSection) {
    case StreamSection.Metadata:
    case StreamSection.Error:
      try {
        const parsed = JSON.parse(data.trim());
        onChunk({ key: state.currentSection, value: parsed });
      } catch (error) {
        console.error("Error parsing JSON:", error);
      }
      break;
    case StreamSection.ContentPart:
      processContentPart(data, state, onChunk);
      break;
    default:
      break;
  }
}

/**
 * Processes a content part by handling the content buffer and updating the parsing state.
 *
 * @param content - The content to be processed.
 * @param state - The current parsing state.
 * @param onChunk - The callback to handle the parsed content part.
 */
function processContentPart(content: string, state: ParseState, onChunk: ChunkCallback) {
  const contentType = (state.contentPartType as ContentPartType) || ContentPartType.Question;

  // Handle JSON content parts specially
  // Note: JSON content parts with the same part_id overwrite previous versions
  if (contentType === ContentPartType.Json) {
    try {
      const parsedJson = JSON.parse(content.trim());
      onChunk({
        key: StreamSection.ContentPart,
        value: {
          type: contentType,
          value: parsedJson,
          part_id: state.contentPartIndex,
          component: parsedJson.component,
          props: parsedJson.value,
        },
      });
    } catch (error) {
      // If JSON parsing fails, treat as regular content
      onChunk({
        key: StreamSection.ContentPart,
        value: {
          type: contentType,
          value: content,
          part_id: state.contentPartIndex,
        },
      });
    }
  } else {
    onChunk({
      key: StreamSection.ContentPart,
      value: {
        type: contentType,
        value: content,
        part_id: state.contentPartIndex,
      },
    });
  }
}
```

#### Usage Example

```typescript
// Assume 'responseStream' is a ReadableStream<Uint8Array> from the API
const reader = responseStream.getReader();
const chunks: ChunkData[] = [];

// Track JSON content parts by part_id to handle overwrites
const jsonContentParts = new Map<string, any>();

await readAndParseStream(reader, (chunk) => {
  chunks.push(chunk);

  // Handle different types of chunks
  if (chunk.key === "content_part") {
    const contentPart = chunk.value;

    // Handle JSON content parts
    if (contentPart.type === "json") {
      const partId = contentPart.part_id;
      const component = contentPart.component;
      const props = contentPart.props;

      // Store/overwrite the JSON content part for this part_id
      jsonContentParts.set(partId, { component, props });

      switch (component) {
        case "reasoning":
          console.log("Reasoning component updated:", props);

          // Render reasoning steps
          props.steps.forEach((step, index) => {
            if (step.type === "thought") {
              console.log(`Thought ${index + 1}: ${step.content}`);
            } else if (step.type === "action") {
              console.log(`Action ${index + 1}: ${step.label}`);
              step.content.forEach((item, itemIndex) => {
                console.log(`  - ${item.label}`);
              });
            }
          });
          break;

        // Future components can be handled here
        // case "other_component":
        //   handleOtherComponent(props);
        //   break;

        default:
          console.log("Unknown JSON component:", component, props);
          break;
      }
    } else {
      // Handle other content parts
      console.log("Content part:", contentPart);
    }
  } else {
    // Handle metadata or error chunks
    console.log("Parsed chunk:", chunk);
  }
});

// After streaming is complete, jsonContentParts contains the final state
// of each JSON content part (latest version for each part_id)
console.log("Final JSON content parts:", jsonContentParts);

// After parsing, 'chunks' will contain all the parsed data:
[
  {
    key: "content_part",
    value: {
      type: "answer",
      value: "",
      part_id: "1",
      status: "streaming",
    },
  },
  {
    key: "content_part",
    value: {
      type: "answer",
      value: "The ",
      part_id: "1",
    },
  },
  {
    key: "content_part",
    value: {
      type: "answer",
      value: "capital ",
      part_id: "1",
    },
  },
  {
    key: "content_part",
    value: {
      type: "answer",
      value: "of ",
      part_id: "1",
    },
  },
  {
    key: "content_part",
    value: {
      type: "answer",
      value: "France ",
      part_id: "1",
    },
  },
  {
    key: "content_part",
    value: {
      type: "answer",
      value: "is ",
      part_id: "1",
    },
  },
  {
    key: "content_part",
    value: {
      type: "answer",
      value: "Paris.",
      part_id: "1",
    },
  },
  {
    key: "content_part",
    value: {
      type: "answer",
      value: "",
      part_id: "1",
      status: "done",
    },
  },
  {
    key: "content_part",
    value: {
      type: "json",
      value: {
        type: "json",
        component: "reasoning",
        value: {
          title: "Klaar met nadenken",
          is_reasoning: false,
          steps: [
            {
              type: "thought",
              label: "Vraag geformuleerd",
              content: "Wat is de vennootschapsbelasting (vpb)?",
            },
          ],
        },
      },
      part_id: "2",
      component: "reasoning",
      props: {
        title: "Klaar met nadenken",
        is_reasoning: false,
        steps: [
          {
            type: "thought",
            label: "Vraag geformuleerd",
            content: "Wat is de vennootschapsbelasting (vpb)?",
          },
        ],
      },
    },
  },
  {
    key: "metadata",
    value: {
      references: [
        {
          chunk_id: "1",
          title: "Title",
          chunk_txt: "Chunk text",
          chunk_url: "https://example.com",
          link_attributes: {},
          source: "Source",
          publication_identifier: "Publication",
          publisher_identifier: "Publisher",
        },
      ],
      output: {},
    },
  },
];
```

---

<a name="conclusion"></a>

## 6. Conclusion

By following the above instructions, you can parse the API response effectively. The key steps involve:

- **Reading the streamed data incrementally**.
- **Identifying start and end tokens** to determine the current section.
- **Extracting and processing the content** between the tokens based on the section type.
- **Handling JSON content carefully**, especially when parsing incomplete JSON objects in a stream.

When implementing the parser, ensure that you handle edge cases, such as:

- Incomplete tokens due to stream chunking.
- Potential errors during JSON parsing.

**Testing your implementation thoroughly** is crucial to ensure reliability.

Feel free to adapt the parsing logic and the TypeScript example code to suit your specific needs and the programming language or framework you are using.

---

**Note**: The above guide is designed to be language-agnostic and focuses on the structure and parsing logic of the API response. The included TypeScript code example provides a concrete implementation that you can use as a reference.
