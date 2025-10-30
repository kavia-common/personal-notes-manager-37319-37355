# Personal Notes Manager — Frontend (Astro)

## Overview

This repository contains the frontend for a web-based Personal Notes application built with Astro. It provides a modern, accessible interface to create, edit, browse, search, favorite, and delete notes. The frontend communicates with a backend REST API and can optionally consume WebSocket updates for live refresh behavior.

If you are integrating this UI with your own backend, please review docs/API_ASSUMPTIONS.md for the expected endpoints and payload shapes that this frontend uses.

## Prerequisites

- Node.js 18+ (LTS recommended)
- npm 9+ (or compatible package manager)
- A running backend that exposes the REST API described in docs/API_ASSUMPTIONS.md, or use mock endpoints during development.

## Getting Started

1) Install dependencies

```sh
cd notes_app_frontend
npm install
```

2) Configure environment variables

Create a .env file at notes_app_frontend/.env or export variables from your environment. Only variables prefixed with PUBLIC_ are read by the frontend.

The application recognizes the following variables (all optional unless noted otherwise):

- PUBLIC_API_BASE
  - Preferred base URL for REST API requests. If set, it takes highest precedence for API calls.
  - Example: https://api.example.com

- PUBLIC_BACKEND_URL
  - Fallback base URL if PUBLIC_API_BASE is not set.
  - Example: http://localhost:8080

- PUBLIC_WS_URL
  - Optional WebSocket endpoint for live updates. When provided, the UI will listen for notes.* messages and auto-refresh views.
  - Example: wss://api.example.com/ws or ws://localhost:8080/ws

- PUBLIC_FRONTEND_URL
  - Optional canonical URL for the frontend itself (not required).

- PUBLIC_NODE_ENV
  - Optional environment name (development, production, etc.).

- PUBLIC_NEXT_TELEMETRY_DISABLED
  - Optional, not required by Astro; kept for environment parity if used in your ecosystem.

- PUBLIC_ENABLE_SOURCE_MAPS
  - If set, can influence build tooling to include source maps.

- PUBLIC_PORT
  - Not required by Astro dev server here; the server port is configured in astro.config.mjs (default 3000). Leave unset unless you customize the server config.

- PUBLIC_TRUST_PROXY
  - Optional flag for proxies; not used directly by this frontend.

- PUBLIC_LOG_LEVEL
  - If set to "debug", enables verbose logging for WebSocket events in the browser console.

- PUBLIC_HEALTHCHECK_PATH
  - Optional; not used directly by the frontend UI.

- PUBLIC_FEATURE_FLAGS
  - Optional JSON/string for feature flags you may introduce in your fork.

- PUBLIC_EXPERIMENTS_ENABLED
  - Optional flag to toggle experimental UI features you may add.

Resolution for API base URL:
- src/lib/env.ts chooses the API base in this order: PUBLIC_API_BASE -> PUBLIC_BACKEND_URL -> window.location.origin.
- Trailing slashes are stripped automatically.

Example .env (notes_app_frontend/.env):

```
PUBLIC_API_BASE=http://localhost:8080
PUBLIC_WS_URL=ws://localhost:8080/ws
PUBLIC_LOG_LEVEL=debug
```

3) Run the development server

```sh
npm run dev
```

The Astro dev server uses the configuration in notes_app_frontend/astro.config.mjs:
- Host: 0.0.0.0
- Port: 3000
- Access-Control-Allow-Origin: * (to simplify local integration)

Visit:
- http://localhost:3000

4) Build for production

```sh
npm run build
```

5) Preview the built site

```sh
npm run preview
```

## Routes and Features

- / — Welcome page with quick navigation to notes.
- /notes — Notes index page with:
  - List or grid view toggle (via view=list|grid)
  - Favorites filter (via filter=favorites)
  - Search box with debounced query updates (?q=...)
  - Pagination controls
  - Favorite toggle and delete actions on each note

- /notes/new — Create a new note using the NoteEditor.

- /notes/:id — View a single note with:
  - Favorite toggle
  - Edit navigation
  - Delete with confirmation

- /notes/:id?edit=1 — Edit an existing note inline using the NoteEditor.

Accessibility and performance highlights:
- Focus management on content load and dialog usage
- Reduced layout shift with skeleton loaders
- Optimistic UI updates for favorite toggles and deletions
- Batched and cached list/detail data via a lightweight store (src/lib/store.ts)
- Abortable in-flight requests on navigation or new queries

## Optional WebSocket Live Updates

If PUBLIC_WS_URL is defined, the frontend will connect to the WebSocket endpoint on load (browser only). Incoming messages of the form:

```
{ "type": "notes.created" | "notes.updated" | "notes.deleted", "payload": { ... } }
```

will trigger a window "notesUpdated" event and cause the store to refresh list data and, when relevant, reload the currently selected note. If the WebSocket is unreachable or closes, the client retries with a capped backoff. The app continues to work without WebSockets; they are entirely optional.

See src/lib/ws.ts for the client behavior.

## Development Notes

- The API client: src/lib/api.ts centralizes REST calls with:
  - Base URL resolution
  - JSON handling
  - Basic retry for transient 5xx and network issues
  - Abortable requests with timeouts
  - Normalized error structure (APIError)

- The store: src/lib/store.ts handles list/detail state, caching, optimistic updates, and subscribable UI updates. It also wires live updates when the WebSocket is enabled.

- Styling follows an "Ocean Professional" theme with accessible focus states and minimal motion.

## Linting

From notes_app_frontend:

```sh
npm run lint
```

This uses eslint with TypeScript support as configured in eslint.config.mjs.

## Deployment

A production deployment should serve the built static assets from notes_app_frontend/dist behind your preferred static server or CDN. Ensure CORS on your backend allows requests from your frontend origin as described in docs/API_ASSUMPTIONS.md.

## Troubleshooting

- 404/Network Errors: Confirm PUBLIC_API_BASE or PUBLIC_BACKEND_URL is reachable and CORS allows your frontend origin.
- CORS: Verify Access-Control-Allow-Origin includes your frontend’s origin and that credentials mode matches your backend requirements.
- WebSocket fails: Ensure PUBLIC_WS_URL points to a ws:// or wss:// endpoint and that your server accepts cross-origin WS connections when necessary.

## References

- Astro: https://docs.astro.build
- API Expectations: docs/API_ASSUMPTIONS.md
