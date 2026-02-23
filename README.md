# Shiptivitas To-Do App

A full-stack kanban board web application engineered for freight shipping logistics management. Built on a **decoupled client-server architecture** with a React single-page application frontend and a RESTful Express.js API backed by SQLite, the system enables shippers to visually track and manage shipping requests across workflow stages via real-time drag-and-drop interactions.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [System Design](#system-design)
- [Tech Stack](#tech-stack)
- [Core Features](#core-features)
- [Drag-and-Drop Engine](#drag-and-drop-engine)
- [Backend API](#backend-api)
- [Database Schema](#database-schema)
- [Priority Reordering Algorithm](#priority-reordering-algorithm)
- [Component Architecture](#component-architecture)
- [Getting Started](#getting-started)
- [API Reference](#api-reference)

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                         CLIENT (React SPA)                       │
│                                                                  │
│   ┌──────────┐    ┌───────────────────────────────────────────┐  │
│   │Navigation│    │              Board (Stateful)              │  │
│   │          │    │  ┌──────────┐ ┌──────────┐ ┌──────────┐   │  │
│   │  Home    │    │  │ Swimlane │ │ Swimlane │ │ Swimlane │   │  │
│   │  Board   │    │  │ Backlog  │ │InProgress│ │ Complete │   │  │
│   │          │    │  │┌────────┐│ │┌────────┐│ │┌────────┐│   │  │
│   └──────────┘    │  ││  Card  ││ ││  Card  ││ ││  Card  ││   │  │
│                   │  │└────────┘│ │└────────┘│ │└────────┘│   │  │
│                   │  └──────────┘ └──────────┘ └──────────┘   │  │
│                   │         ▲  Dragula Drop Containers  ▲      │  │
│                   └─────────┼───────────────────────────┼──────┘  │
│                             │     DOM Reconciliation    │        │
└─────────────────────────────┼───────────────────────────┼────────┘
                              │        HTTP / JSON        │
┌─────────────────────────────┼───────────────────────────┼────────┐
│                         SERVER (Express.js)                       │
│                                                                  │
│   ┌──────────────────────────────────────────────────────────┐   │
│   │                  RESTful API v1 Layer                     │   │
│   │  GET /api/v1/clients       GET /api/v1/clients/:id       │   │
│   │  PUT /api/v1/clients/:id   (with priority reordering)    │   │
│   └──────────────────────────┬───────────────────────────────┘   │
│                              │                                   │
│   ┌──────────────────────────▼───────────────────────────────┐   │
│   │              SQLite (better-sqlite3)                      │   │
│   │              Persistent Storage Layer                     │   │
│   └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

The system follows a **separation-of-concerns** model: the React SPA handles all presentation logic, drag-and-drop interaction, and client-side state management, while the Express API encapsulates business logic including input validation, status transition enforcement, and a priority reordering algorithm that maintains consistent ordering invariants across swimlanes.

---

## System Design

### Design Principles

- **Decoupled Client-Server**: The frontend and backend operate as independent services communicating via a versioned REST API (`/api/v1/`), enabling independent deployment and horizontal scaling.
- **Optimistic UI Updates**: The drag-and-drop engine immediately reflects state changes in the UI by reverting Dragula's DOM mutations and delegating re-rendering entirely to React's virtual DOM reconciliation, avoiding DOM state divergence.
- **Idempotent Reordering**: The backend priority algorithm computes absolute positions from relative insertions, ensuring that repeated identical requests produce the same state (no priority drift).
- **Graceful Degradation**: The frontend initializes with a self-contained client dataset, allowing the UI to remain fully interactive even if the backend is unavailable.

### Data Flow

1. User drags a `Card` element from a source `Swimlane` to a target `Swimlane`
2. Dragula fires a `drop` event with references to `(el, target, source, sibling)`
3. `Board.updateClient()` immediately calls `drake.cancel(true)` to revert DOM mutations
4. Target swimlane is resolved via React ref identity comparison
5. The client object is cloned with the updated status and spliced into the correct position relative to the sibling element
6. `setState()` triggers React's reconciliation, re-rendering only the affected swimlanes

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Frontend** | React 16.7 | Component-based UI with class lifecycle methods |
| **Drag-and-Drop** | Dragula 3.7 | Cross-container drag-and-drop with DOM mutation events |
| **Styling** | Bootstrap 4.2 + Custom CSS | Responsive grid layout with status-coded card theming |
| **Backend** | Express 4.16 | RESTful API server with JSON middleware |
| **Database** | SQLite via better-sqlite3 5.4 | Synchronous, zero-config embedded relational database |
| **Transpilation** | Babel 7 (@babel/preset-env) | ES6+ module syntax and modern JS feature support |
| **Dev Server** | babel-watch | File-watching transpiler with automatic server restart |
| **Build Tool** | Create React App (react-scripts 2.1) | Webpack-based build pipeline with HMR |

---

## Core Features

### Kanban Workflow Board
Three-column kanban board representing the shipping request lifecycle:
- **Backlog** -- Queued requests awaiting processing (grey-coded)
- **In Progress** -- Active shipments under fulfillment (blue-coded)
- **Complete** -- Fulfilled and closed requests (green-coded)

### Real-Time Drag-and-Drop
Cards can be freely dragged between any swimlane. The system handles both **cross-swimlane status transitions** (e.g., Backlog to In Progress) and **intra-swimlane reordering** (priority adjustment within the same column), with immediate visual feedback.

### Tab-Based Navigation
Bootstrap nav-tab interface with conditional rendering between the Home landing view and the Shipping Requests board, managed through React component state.

### Animated Landing Page
Home tab features a CSS cubic-bezier animated element (`transition: all 1.5s cubic-bezier(0,1,.98,0)`) that demonstrates smooth physics-based easing on hover interaction.

---

## Drag-and-Drop Engine

The drag-and-drop system is built on **Dragula** with a custom React integration layer that solves the fundamental tension between imperative DOM manipulation libraries and React's declarative rendering model.

### The DOM Reconciliation Problem

Dragula operates by directly mutating the DOM (moving child nodes between containers). This conflicts with React's virtual DOM, which expects to be the sole authority over DOM structure. Allowing both systems to mutate the DOM simultaneously causes **state divergence** -- the virtual DOM tree and the actual DOM tree fall out of sync, leading to rendering artifacts and lost state.

### Solution: Cancel-and-Reconcile Pattern

```
Dragula fires drop event
        │
        ▼
drake.cancel(true)          ← Revert ALL DOM mutations Dragula made
        │
        ▼
Resolve target swimlane     ← Compare event target against React refs
        │
        ▼
Compute new client list     ← Clone, re-status, and splice into position
        │
        ▼
setState({ clients })       ← React re-renders from authoritative state
```

The `Board` component intercepts Dragula's `drop` event, **immediately reverts all DOM changes** via `drake.cancel(true)`, computes the new state purely in JavaScript, and triggers a React re-render. This ensures React remains the single source of truth for DOM state, while Dragula handles only the drag interaction UX (visual feedback, hit detection, container boundaries).

### Implementation Details

- **Ref-Based Container Registration**: Three `React.createRef()` instances are created in the constructor and passed to Swimlane components. Dragula is initialized in `componentDidMount` with these ref targets as its container array.
- **Sibling-Relative Positioning**: The drop handler uses the `sibling` parameter (the element the card was dropped before) to compute insertion index via `Array.findIndex()`, preserving the user's intended position.
- **Lifecycle Cleanup**: The Dragula instance is destroyed in `componentWillUnmount` to prevent memory leaks and orphaned event listeners.

---

## Backend API

The Express server exposes a versioned RESTful API (`/api/v1/`) with input validation, parameterized SQL queries, and a priority reordering engine.

### Connection Management

A single `better-sqlite3` connection is maintained for the application lifetime (connection pooling is unnecessary for SQLite's serialized access model). The connection is registered for graceful shutdown on both `SIGTERM` and `SIGINT` signals, preventing database corruption on process termination.

### Input Validation Layer

All endpoints validate inputs before database access:
- **ID validation**: Checks for `NaN` coercion and verifies existence via a `SELECT ... LIMIT 1` lookup
- **Status validation**: Enforces the enum constraint `{'backlog' | 'in-progress' | 'complete'}`
- **Priority validation**: Ensures positive integer values

Each validator returns a structured error response with both a short `message` (suitable for UI display) and a `long_message` (suitable for developer debugging).

---

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS clients (
    id          INTEGER PRIMARY KEY,    -- Auto-incrementing unique identifier
    name        TEXT    NOT NULL,        -- Client / company name
    description TEXT,                   -- Shipping request description
    status      TEXT,                   -- Workflow state: 'backlog' | 'in-progress' | 'complete'
    priority    INTEGER                 -- Ordering within status group (1 = highest)
);
```

The `priority` column encodes **intra-swimlane ordering**: within each status group, clients are sorted by ascending priority (1 appears at the top). This enables the API to return clients in their correct visual order and supports the reordering algorithm's insert-and-renumber logic.

The schema is initialized from `clients.sql`, which seeds 20 client records distributed across all three status groups (11 backlog, 5 in-progress, 4 complete).

---

## Priority Reordering Algorithm

The PUT endpoint implements a three-case reordering algorithm that maintains the **ordering invariant**: no two clients within the same status group share a priority value, and priorities form a contiguous sequence `[1, 2, ..., n]`.

### Case Analysis

| Case | Condition | Behavior |
|------|-----------|----------|
| **No-op** | `oldStatus == newStatus && oldPriority == priority` | No database writes |
| **Intra-swimlane reorder** | `oldStatus == newStatus && oldPriority != priority` | Reorder within the same status group |
| **Cross-swimlane move** | `oldStatus != newStatus` | Remove from old group, insert into new group, renumber both |

### Float-Based Insertion Technique

For intra-swimlane reordering, the algorithm uses a **fractional intermediate priority** (`priority - 0.5`) to position the moved client between two existing entries. After assignment, all clients in the affected group are sorted by priority and remapped to contiguous integers:

```
Before:  [A:1, B:2, C:3, D:4]     Move D to position 2
Step 1:  D.priority = 2 - 0.5 = 1.5
Step 2:  Sort → [A:1, D:1.5, B:2, C:3]
Step 3:  Renumber → [A:1, D:2, B:3, C:4]
```

This avoids the complexity of gap-based or linked-list reordering schemes while guaranteeing correct insertion semantics in `O(n log n)` time (dominated by the sort).

For cross-swimlane moves, the algorithm partitions the full client list into three groups (old status, new status, and all others), independently renumbers the old and new groups, and merges the results before performing a batch update.

---

## Component Architecture

```
App (Class Component)
 ├── Navigation (Class Component)
 │     └── Bootstrap nav-tabs with active state binding
 │
 └── [Conditional Render]
       ├── HomeTab (Functional Component)
       │     └── Animated landing with CSS cubic-bezier transition
       │
       └── Board (Class Component) ← Stateful container
              ├── Swimlane "Backlog" (Class Component) ← Presentational
              │     └── Card[] (Class Component) ← Status-coded display
              ├── Swimlane "In Progress"
              │     └── Card[]
              └── Swimlane "Complete"
                    └── Card[]
```

### Pattern: Container / Presentational Separation

- **`Board`** acts as the **container component**: it owns the client state, initializes Dragula, handles drop events, and computes state transitions. No sibling or child component mutates board state directly.
- **`Swimlane`** and **`Card`** are **presentational components**: they receive data via props and render UI without side effects. `Swimlane` exposes a `dragulaRef` prop to allow the parent to attach Dragula containers without breaking encapsulation.

### State Architecture

```
App.state
 └── selectedTab: 'home' | 'shipping-requests'

Board.state
 └── clients
       ├── backlog:    Client[]
       ├── inProgress: Client[]
       └── complete:   Client[]
```

State is partitioned by component responsibility. `App` manages navigation state; `Board` manages domain state. No global state store is used -- the component tree is shallow enough that prop drilling is sufficient, avoiding unnecessary abstraction overhead.

---

## Getting Started

### Prerequisites

- **Node.js** >= 11.10.0
- **npm** >= 6.7.0

### Installation

```bash
# Clone the repository
git clone https://github.com/taivu1998/Shiptivitas-To-Do-App.git
cd Shiptivitas-To-Do-App

# Install backend dependencies
cd backend
npm install

# Install frontend dependencies
cd ../app
npm install
```

### Running the Application

Start both servers in separate terminal sessions:

```bash
# Terminal 1 -- Backend API (port 3001)
cd backend
npm start

# Terminal 2 -- Frontend Dev Server (port 3000)
cd app
npm start
```

The React dev server will open `http://localhost:3000` automatically. Navigate to the **Shipping Requests** tab to interact with the kanban board.

### Running Tests

```bash
cd app
npm test
```

---

## API Reference

### `GET /api/v1/clients`

Retrieve all clients, optionally filtered by workflow status.

| Parameter | Type | In | Required | Description |
|-----------|------|-----|----------|-------------|
| `status` | string | query | No | Filter: `backlog`, `in-progress`, or `complete` |

**Response `200`**:
```json
[
  {
    "id": 1,
    "name": "Stark, White and Abbott",
    "description": "Cloned Optimal Architecture",
    "status": "in-progress",
    "priority": 1
  }
]
```

### `GET /api/v1/clients/:id`

Retrieve a single client by ID.

| Parameter | Type | In | Required | Description |
|-----------|------|-----|----------|-------------|
| `id` | integer | path | Yes | Client ID |

**Response `200`**: Single client object.

**Response `400`**: `{ "message": "Invalid id provided.", "long_message": "Cannot find client with that id." }`

### `PUT /api/v1/clients/:id`

Update a client's status and/or priority. Triggers the reordering algorithm across affected swimlanes.

| Parameter | Type | In | Required | Description |
|-----------|------|-----|----------|-------------|
| `id` | integer | path | Yes | Client ID |
| `status` | string | body | No | New status: `backlog`, `in-progress`, or `complete` |
| `priority` | integer | body | No | Target position within the status group (1 = top) |

**Response `200`**: Full array of all clients with recalculated priorities.

**Response `400`**: Validation error with `message` and `long_message` fields.

---

## License

MIT
