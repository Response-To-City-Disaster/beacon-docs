# beacon-dashboard

Emergency Response Team (ERT) web dashboard for the Beacon emergency-response platform. A React 18 + TypeScript single-page application that gives ERT members, dispatchers, and administrators a unified interface for managing city-scale emergencies in real time — from first report through dispatch coordination, evacuation management, and incident closure.

---

## Overview

When an emergency is reported, the people responsible for coordinating the response need a clear, fast, always-up-to-date view of what is happening. The Beacon dashboard is that view. It provides:

- A live incident feed with severity-coded cards updated every 30 seconds
- Detailed incident management: approve reports, escalate severity, dispatch responders, create evacuation zones
- A dispatch coordination panel with real-time status tracking via Server-Sent Events
- Evacuation zone management with advisory broadcast to citizens
- A communication centre backed by WebSocket team channels
- Operational analytics and performance metrics for incident response times
- User management for SysAdmins: ERT member onboarding, ground-staff roster
- An AI support bot integration for monitoring and escalating citizen distress conversations

The dashboard is the primary tool for ERT members and administrators. It is not used by citizens (who use the mobile app) or ground staff (who primarily use the mobile app with a focused responder view).

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Framework | React 18 + TypeScript 5 | Component model and type safety |
| Build | Vite 5 | Fast HMR dev server, optimized production builds |
| Styling | Tailwind CSS 3 | Utility-first styling with a consistent dark theme |
| UI Primitives | Radix UI (shadcn/ui) | Accessible, unstyled Radix components wrapped in Tailwind |
| Icons | Lucide React | Consistent icon set |
| Charts | Recharts | Analytics and metrics visualizations |
| Maps | TomTom Maps SDK | Incident location mapping |
| HTTP | Custom `request` utility (fetch-based) | Token injection, refresh, base URL management |
| WebSocket | Native browser WebSocket API | Team channel real-time messaging |
| Auth | JWT with auto-refresh | Via beacon-iam |
| Serving | nginx (Docker) | Static file serving and API proxying |
| Deployment | GKE | Kubernetes deployment |

---

## Architecture

The dashboard follows Atomic Design component hierarchy and a strict container/presenter separation:

```
Pages (data fetching, composes organisms)
  └── Organisms (complex sections — filters, tables, panels — local UI state only)
       └── Molecules (cards, rows, form groups — no API calls)
            └── Atoms (badges, icons, status dots — pure display)
```

**Key principle:** Pages fetch their own data. Navigation passes only IDs — never objects. A page must be independently loadable from a direct URL without depending on data passed through navigation state. This means refreshing any page, or sharing a URL, always works correctly.

### State Management

State is managed with React's built-in primitives (useState, useReducer, useContext) combined with custom hooks. There is no global state library. Each page owns its own data via hooks like `useIncidents()` or `useIncidentById(id)`. Shared auth state lives in `authService` (a module singleton, not React context), accessed via `useAuth()`.

### API Layer

All calls to Beacon backend services go through the `src/utils/request.ts` utility, which handles:
- JWT injection via `authService.authenticatedFetch()`
- 401 → token refresh → retry (transparent to callers)
- Base URL resolution (proxied in dev, direct in production)
- Consistent error throwing

Service files (`src/services/<domain>.api.ts`) export plain async functions that call `request`. Components never call `fetch()` directly for Beacon APIs.

### Real-Time Updates

Two mechanisms are used for real-time data:
- **Polling**: The incident list and most dashboard panels poll their respective endpoints every 30 seconds using `setInterval` in `useEffect` hooks. All polling intervals are cleaned up on component unmount.
- **WebSocket**: The communication centre uses the native WebSocket API for team channel messaging. The connection is managed in `useCommunication()` and reconnects automatically on disconnect.

---

## Features

### Incident Management

The core workflow: a citizen reports an incident through the mobile app, it appears in the ERT dashboard's incident feed as a `reported` record. An ERT member reviews it, approves the report (transitioning it to `active`), and the dispatch workflow begins.

From the incident detail view, ERT members can:
- Approve or reject the initial report
- Escalate the severity level
- Add structured notes and updates to the incident timeline
- Create dispatch assignments for ground staff
- Create evacuation zones with geographic boundaries
- Close the incident when the situation is resolved

The incident detail page shows a TomTom map with the incident pin and any active dispatch routes. The severity badge colour system matches the mobile app: critical (red), high (orange), medium (yellow), low (blue).

### Dispatch Coordination

The dispatch panel shows all active dispatches and their real-time status via Server-Sent Events from beacon-core. When a dispatch is created, the assigned ground-staff member receives a push notification; when they accept or reject it, the dashboard updates immediately without polling.

ERT members can view the dispatch timeline (created → pending acceptance → accepted → in-progress → completed) and take action if a dispatch is not acknowledged within the timeout window.

### Evacuation Management

Admins and ERT members can create evacuation zones linked to an active incident. A zone has a name, geographic boundary (drawn on the TomTom map), status (advisory / mandatory), and a message. The advisory broadcast button sends a push notification to all citizens currently located within the zone boundary.

### Communication Centre

A real-time team channel system backed by beacon-communication-platform's WebSocket API. ERT members and ground staff can coordinate through persistent channels scoped to incidents or standing teams. The dashboard shows all channels the user is a participant of, with live message updates, read receipts, and typing indicators.

### Analytics

The analytics section provides ERT commanders with performance metrics: average incident response time by type and severity, dispatch acceptance rate, incidents by geographic area, and active-incident timeline. Charts are rendered with Recharts. Data is served from beacon-core's metrics and analytics endpoints.

### User Management *(Admin only)*

SysAdmins can manage the platform user roster from the dashboard: view all users, review and approve ERT membership applications, deactivate accounts, and reset passwords (via Keycloak admin API calls through beacon-iam).

---

## Project Structure

```
src/
├── components/
│   ├── atoms/             # SeverityBadge, StatusDot, LoadingSpinner, EmptyState, TimeAgo
│   ├── molecules/         # IncidentCard, UserRow, SearchInput, StatCard
│   ├── organisms/         # IncidentTable, DispatchPanel, EvacuationPanel, ChannelList
│   ├── pages/             # ERTDashboardPage, AllIncidentsPage, IncidentDetailsPage,
│   │                      # CommunicationCenterPage, AnalyticsPage, UserManagementPage
│   └── ui/                # shadcn/ui primitives — DO NOT MODIFY
├── hooks/
│   ├── useIncidents.ts    # Incidents list with 30s polling
│   ├── useIncidentById.ts # Single incident by ID
│   ├── useAuth.ts         # Auth state helpers
│   └── useWeather.ts      # Weather widget data
├── services/
│   ├── auth.ts            # JWT management, token refresh
│   ├── core.api.ts        # beacon-core endpoints
│   ├── api.ts             # beacon-iam + communication endpoints (aggregator)
│   └── supportBot.ts      # beacon-support-bot endpoints
├── types/
│   ├── core.types.ts      # Incident, Dispatch, Evacuation, Vault
│   └── chat.types.ts      # Message, Channel, WebSocket payloads
├── utils/
│   ├── request.ts         # HTTP utility with auth and refresh
│   ├── date.utils.ts      # formatTimeAgo, formatTimestamp
│   └── incident.utils.ts  # getColorFromSeverity, getSeverityLabel
└── routes.tsx             # App routing and navigation state
```

---

## Local Development

### Prerequisites

| Tool | Version |
|---|---|
| Node.js | 18+ |
| npm | 9+ |

### Setup

```bash
# 1. Install dependencies
npm install

# 2. Configure the backend proxy
# Copy the example env file and set the backend URL
cp .env.example .env.local
# Edit .env.local:
#   VITE_API_BASE_URL=http://localhost:8080   (or the deployed service URL)

# 3. Start the dev server
npm run dev
```

The Vite dev server starts on `http://localhost:5173` with hot module replacement enabled. All `/api/*` requests are proxied to the configured backend URL, so CORS is not an issue during development.

### Available Scripts

| Command | Description |
|---|---|
| `npm run dev` | Start development server with HMR |
| `npm run build` | Production build (TypeScript compile + Vite bundle) |
| `npm run build:check` | TypeScript type check + build — run before committing |
| `npm run lint` | ESLint with TypeScript rules |
| `npm run preview` | Preview the production build locally |
| `npm run format` | Prettier code formatting |

Always run `npm run build:check` before pushing — it catches type errors that `npm run dev` silently ignores.

---

## Docker

The production image is a two-stage build: Vite builds the static assets in a Node.js container, then nginx serves them from an Alpine-based image.

```bash
# Build the image
docker build -t beacon-dashboard:latest .

# Run locally (maps port 80 to 3000)
docker run -p 3000:80 beacon-dashboard:latest
```

The `nginx.conf` file configures:
- Single-page app routing (`try_files $uri $uri/ /index.html`) so deep links work correctly
- Gzip compression for JS and CSS assets
- Cache-control headers: HTML files are not cached (so deploys take effect immediately), static assets get long-lived cache headers

---

## Deployment (GKE)

```bash
# Build and push to Container Registry
make gke-build-push

# Deploy to Kubernetes
make gke-deploy

# Check status
make gke-pods

# View logs
make gke-logs
```

The Kubernetes manifests are in `k8s/`. The deployment uses a RollingUpdate strategy so new versions are deployed without downtime. The nginx container serves on port 80; the Kubernetes Service exposes it on the cluster-internal hostname `beacon-dashboard`.

---

## Design System

The dashboard uses a layered dark theme consistent with the mobile app's colour system:

| Layer | Tailwind Class | Hex | Usage |
|---|---|---|---|
| Deepest | `bg-gray-950` | `#0A0A0A` | Page / screen background |
| Surface | `bg-gray-900` | `#171717` | Cards, panels, sidebars |
| Elevated | `bg-gray-800` | `#262626` | Inputs, dropdowns, hover states |
| Border | `border-gray-800` | `#262626` | All borders |
| Muted text | `text-gray-400` | `#A3A3A3` | Labels, metadata, placeholders |
| Body text | `text-white` | `#FFFFFF` | Primary content |
| Accent | `text-blue-400` | `#60A5FA` | Actions, active states |

Severity colours follow the same P0/P1/P2/P3 system as the mobile app: `red-400` (critical), `orange-400` (high), `yellow-400` (medium), `blue-400` (low). Never introduce new colour tokens — use the established Tailwind scale.

---

## Contributing

Before writing any code, read `CLAUDE.md` in the repository root. It contains the mandatory engineering guidelines for this codebase including component hierarchy rules, naming conventions, API layer rules, hook patterns, accessibility requirements, and a complete PR checklist. Code that violates these guidelines will not be merged.
