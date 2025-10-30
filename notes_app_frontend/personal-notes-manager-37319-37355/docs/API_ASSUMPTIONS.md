# API Assumptions for Notes Frontend Integration

## Overview

This document describes the expected REST and WebSocket interfaces that the Astro Notes frontend uses. It is intended for backend implementers who need to provide a compatible API or for integrators configuring reverse proxies and CORS.

All examples assume JSON requests and responses unless otherwise noted.

## Base URL and CORS

- The frontend resolves its API base URL from environment variables in this order:
  1) PUBLIC_API_BASE
  2) PUBLIC_BACKEND_URL
  3) window.location.origin (browser fallback)

- Ensure your backend is reachable at one of the configured base URLs and that CORS is enabled to allow requests from the frontend’s origin.

Minimum CORS headers recommended:
- Access-Control-Allow-Origin: <your-frontend-origin> (or * during local dev)
- Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
- Access-Control-Allow-Headers: Content-Type, Accept
- Access-Control-Allow-Credentials: (optional) If you require session cookies and the frontend uses credentials: "include"

For local development, the Astro dev server at port 3000 sets Access-Control-Allow-Origin: * for responses it serves. Your backend must still separately allow cross-origin requests from the frontend.

## Authentication

The current frontend calls fetch with credentials: "include" to permit cookie-based sessions if present. If your backend requires authenticated access, you can:
- Issue a session cookie on the same domain or a domain permitted by CORS with credentials.
- Or adapt the API to accept token-based headers and adjust the frontend accordingly.

Out of the box, the frontend does not send authorization headers; it relies on cookies if present.

## Resources and Types

### Note

A note is expected to match the following shape (src/lib/types.ts):

```
{
  "id": "string",
  "title": "string",
  "content": "string",
  "createdAt": "ISO8601 timestamp string",
  "updatedAt": "ISO8601 timestamp string",
  "favorite": true|false,        // optional, defaults to false
  "tags": [ { "id": "string", "name": "string", "color": "string?" } ] | [ "tagId", ... ] // optional
}
```

### Paginated<T> (List Responses)

```
{
  "items": [ ... ],
  "total": 123,
  "page": 1,
  "pageSize": 12
}
```

### Error shape (APIError)

The frontend normalizes errors and looks for the following fields when the backend returns an error body:

```
{
  "message": "Human readable error message",
  "code": "Machine readable string code (optional)",
  "...": "Any other details"
}
```

If the backend returns a different structure, ensure at least a top-level "message" string is included. Otherwise, the frontend will fallback to statusText or a generic "Request failed".

## Endpoints

The API client is defined in src/lib/api.ts. All endpoints are relative to the resolved base URL.

- GET /notes
  - Purpose: List notes with pagination and optional filters.
  - Query params (all optional):
    - page: number (default 1)
    - pageSize: number (default determined by backend; frontend often uses 12)
    - search: string (search within title/content)
    - favorite: boolean (true to filter only favorites)
    - tag: string (filter by tag id or name)
  - Response: Paginated<Note>

- GET /notes/:id
  - Purpose: Retrieve a single note by id.
  - Response: Note
  - Status Codes:
    - 200 on success
    - 404 if not found (include a JSON error with message)

- POST /notes
  - Purpose: Create a new note.
  - Request body: Partial<Pick<Note, "title" | "content" | "favorite" | "tags">>
    - Example:
      ```
      {
        "title": "My note",
        "content": "Hello world",
        "favorite": false,
        "tags": ["work","ideas"] // or tag objects with id/name if you prefer
      }
      ```
  - Response: Note (the created resource with server-assigned id and timestamps)
  - Status Codes:
    - 201 on success
    - 400 on validation errors (return a JSON object with message and optional code/details)

- PUT /notes/:id
  - Purpose: Replace an existing note.
  - Request body: Same fields as POST /notes (partial is acceptable; backend may treat as full replace)
  - Response: Note (the updated resource)
  - Status Codes:
    - 200 on success
    - 404 if not found
    - 400 on validation errors

- PATCH /notes/:id
  - Purpose: Update select fields on an existing note (e.g., favorite toggle, title/content edits).
  - Request body: Partial<Pick<Note, "title" | "content" | "favorite" | "tags">>
  - Response: Note (the updated resource)
  - Status Codes:
    - 200 on success
    - 404 if not found
    - 400 on validation errors

- DELETE /notes/:id
  - Purpose: Delete a note by id.
  - Response: Either 204 No Content or a minimal JSON body indicating success.
  - Status Codes:
    - 204 or 200 on success
    - 404 if not found

## Response Conventions

- Content-Type: application/json for JSON responses.
- For 204 responses, no body is expected and the frontend will treat it as success.
- For errors (4xx/5xx), respond with JSON body including at least a "message" field. Optionally include "code" for machine-readable classification and any extra "details".

## Timeouts and Retries

- The frontend applies a default request timeout of 10 seconds.
- The client retries certain transient failures:
  - Network-level errors (not explicit aborts)
  - HTTP 502/503/504 responses
  - Up to 2 retries with simple backoff.

Backends should return stable error codes and messages. If your backend intentionally streams or long-polls, consider increasing timeouts and adjusting the frontend accordingly.

## WebSocket Integration (Optional)

If PUBLIC_WS_URL is configured, the frontend will establish a WebSocket connection in the browser. It expects messages in this shape:

```
{
  "type": "notes.created" | "notes.updated" | "notes.deleted",
  "payload": { "id": "string", ... } // optional fields
}
```

Behavior:
- On any notes.* message, the frontend invalidates its cached list and fetches a fresh page in the background using the last query params.
- If a specific note is currently selected and the event type is notes.updated or notes.created, the frontend will re-fetch that note.
- If notes.deleted payload contains the id of the currently selected note, the selection is cleared.

WebSocket CORS and cross-origin notes:
- Ensure your WS endpoint permits connections from the frontend origin (CORS for WS is handshake-based).
- Use ws:// for local dev or wss:// for production with TLS.

## Example Error Payloads

Validation error (400):
```
{
  "code": "VALIDATION_ERROR",
  "message": "Title is required.",
  "details": { "field": "title" }
}
```

Not found (404):
```
{
  "code": "NOT_FOUND",
  "message": "Note not found."
}
```

Server error (500):
```
{
  "code": "INTERNAL_ERROR",
  "message": "An unexpected error occurred."
}
```

## Example CORS Config (Illustrative)

Express.js snippet:
```js
import cors from "cors";
app.use(cors({
  origin: ["http://localhost:3000", "https://your-frontend.example.com"],
  methods: ["GET","POST","PUT","PATCH","DELETE","OPTIONS"],
  allowedHeaders: ["Content-Type","Accept"],
  credentials: true
}));
```

Nginx snippet:
```
add_header Access-Control-Allow-Origin $http_origin always;
add_header Access-Control-Allow-Methods "GET, POST, PUT, PATCH, DELETE, OPTIONS" always;
add_header Access-Control-Allow-Headers "Content-Type, Accept" always;
add_header Access-Control-Allow-Credentials "true" always;

if ($request_method = OPTIONS) {
  return 204;
}
```

## Versioning and Compatibility

- The frontend assumes stable endpoint paths as listed above.
- Additive fields in responses should not break the UI.
- If you change the URL structure or required fields, you will need to update src/lib/api.ts and potentially the store and components.

## References

- API client implementation: notes_app_frontend/src/lib/api.ts
- Types used by the UI: notes_app_frontend/src/lib/types.ts
- Environment resolution: notes_app_frontend/src/lib/env.ts
- WebSocket client: notes_app_frontend/src/lib/ws.ts
