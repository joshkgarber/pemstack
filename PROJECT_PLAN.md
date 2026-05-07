# Enablement Content Repository — Project Plan

## 1. Overview
Build a foundation web app for a **Platform Enablement Manager** to record and manage enablement content discovered during the **needs assessment** and **discovery** phases of projects.

**Tech Stack**
- **Backend:** FastAPI + SQLAlchemy (async) + SQLite + Uvicorn
- **Frontend:** Vite + Lit (web components) + Vanilla CSS
- **Data Format:** JSON over REST

**Goal for this phase:** Full CRUD for all schema entities via a clean, component-based UI.

---

## 2. Project Structure (Monorepo)

```
enablement-repo/
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py              # FastAPI app, CORS, routing
│   │   ├── database.py          # SQLAlchemy engine, session, base
│   │   ├── models.py            # ORM tables
│   │   ├── schemas.py           # Pydantic request/response models
│   │   └── api/
│   │       ├── __init__.py
│   │       ├── use_cases.py     # /use-cases endpoints
│   │       ├── personas.py      # /personas endpoints
│   │       ├── goals.py         # /goals endpoints
│   │       ├── actions.py       # /actions endpoints
│   │       └── adoption_metrics.py  # /adoption-metrics endpoints
│   ├── requirements.txt
│   └── run.py                   # `python run.py` entry point
├── frontend/
│   ├── index.html
│   ├── package.json
│   ├── vite.config.js
│   ├── src/
│   │   ├── main.js              # Vite entry, mounts <app-root>
│   │   ├── styles.css           # Global styles, CSS custom properties
│   │   ├── api-client.js        # Thin fetch wrapper over backend REST
│   │   ├── router.js            # Simple hash-based client router
│   │   ├── components/
│   │   │   ├── app-root.js      # Top-level shell, routing outlet
│   │   │   ├── nav-bar.js       # Navigation between sections
│   │   │   ├── use-case-list.js # List view + create button
│   │   │   ├── use-case-detail.js # Show + edit a use case
│   │   │   ├── use-case-form.js   # Create/edit use case meta
│   │   │   ├── persona-list.js
│   │   │   ├── persona-form.js
│   │   │   ├── goal-list.js
│   │   │   ├── goal-form.js
│   │   │   ├── action-list.js
│   │   │   ├── action-form.js
│   │   │   ├── adoption-metric-list.js
│   │   │   ├── adoption-metric-form.js
│   │   │   └── shared/
│   │   │       ├── data-table.js   # Reusable sortable table
│   │   │       ├── confirm-dialog.js
│   │   │       └── toast-alert.js
│   │   └── views/
│   │       ├── dashboard-view.js
│   │       ├── use-case-view.js
│   │       └── editor-view.js      # Generic entity editor
│   └── public/
│       └── (empty)
├── enablement_content_repo_schema.md   # Source schema
└── PROJECT_PLAN.md                   # This file
```

---

## 3. Database Schema (SQLite + SQLAlchemy)

Normalized relational design mapped from the Pydantic schema.

### Tables

| Table | Key Columns | Relationships |
|-------|-------------|---------------|
| **use_cases** | `id` (PK, UUID/string), `name`, `description`, `created_at`, `updated_at` | 1:N personas, goals, actions |
| **personas** | `id`, `use_case_id` (FK), `name`, `description` | M:N goals via `persona_goals` |
| **goals** | `id`, `use_case_id` (FK), `name`, `description` | 1:N success_criteria, adoption_metrics; M:N personas |
| **success_criteria** | `id`, `goal_id` (FK), `text` | Belongs to goal |
| **actions** | `id`, `use_case_id` (FK), `text`, `image_url` | Belongs to use case |
| **adoption_metrics** | `id`, `goal_id` (FK), `name`, `description` | Belongs to goal |
| **persona_goals** | `persona_id`, `goal_id` | Association table |

### Design Notes
- `id` fields use UUID strings (generated client-side or server-side; server-side is simpler for MVP).
- `success_criteria` is broken out into its own table (1:N) to allow future editing of individual criteria.
- `persona_goals` association captures which goals are relevant to which persona within a use case.
- SQLite will store UUIDs as `TEXT`. JSON support available if needed, but we stay normalized for relational integrity.

---

## 4. Backend API Design

### Base URL
`http://localhost:8000/api/v1`

### Endpoints

| Resource | Method | Path | Description |
|----------|--------|------|-------------|
| Use Cases | GET/POST | `/use-cases` | List all; create new |
| Use Case | GET/PUT/DELETE | `/use-cases/{id}` | Read, update, delete (cascade) |
| Use Case nested | GET | `/use-cases/{id}/full` | Fetch with nested personas, goals, actions |
| Personas | GET/POST | `/personas` | List (optionally filter by `?use_case_id=`); create |
| Persona | GET/PUT/DELETE | `/personas/{id}` | CRUD |
| Goals | GET/POST | `/goals` | List (filter by `?use_case_id=`); create |
| Goal | GET/PUT/DELETE | `/goals/{id}` | CRUD |
| Actions | GET/POST | `/actions` | List (filter by `?use_case_id=`); create |
| Action | GET/PUT/DELETE | `/actions/{id}` | CRUD |
| Adoption Metrics | GET/POST | `/adoption-metrics` | List (filter by `?goal_id=`); create |
| Adoption Metric | GET/PUT/DELETE | `/adoption-metrics/{id}` | CRUD |
| Success Criteria | GET/POST | `/success-criteria` | List (filter by `?goal_id=`); create |
| Success Criterion | GET/PUT/DELETE | `/success-criteria/{id}` | CRUD |

### Response Shape
All JSON responses wrap the Pydantic model directly. List endpoints return `List[Schema]`. Errors use standard FastAPI HTTPException with detail messages.

### CORS
Allow `localhost:5173` (Vite dev server) for local development.

---

## 5. Frontend Architecture

### Build & Dev
- **Vite** handles bundling, dev server, and HMR.
- Entry: `index.html` → `src/main.js`.
- Output: static files served by any web server (or `python -m http.server` for quick test).

### Component Design (Lit)
Each major entity gets a **list** component and a **form** component.

| Component | Responsibility |
|-----------|--------------|
| `<app-root>` | Layout shell, holds router state, renders current view |
| `<nav-bar>` | Links to Dashboard, Use Cases, Personas, Goals, Actions |
| `<dashboard-view>` | Summary stats (counts), recent use cases |
| `<use-case-list>` | Table of use cases, click to open, delete with confirm |
| `<use-case-form>` | Inputs for name/description; emits save event |
| `<use-case-detail>` | Displays a use case and embeds nested lists for personas, goals, actions |
| `<entity-list>` / `<entity-form>` | Generic patterns applied to personas, goals, actions, metrics |

### State Management
- **No external state library.** Each component fetches its own data via `api-client.js`.
- For shared updates (e.g., deleting a use case should refresh the list), use simple DOM events (`CustomEvent`) or re-fetch on `connectedCallback`.
- Router stores only the current path hash; components read URL params to load IDs.

### Routing (Hash-based)
| Route | View |
|-------|------|
| `#/` | Dashboard |
| `#/use-cases` | UseCase list |
| `#/use-cases/new` | Create use case |
| `#/use-cases/{id}` | UseCase detail |
| `#/use-cases/{id}/edit` | Edit use case |
| `#/personas` | Persona list (all) |
| `#/goals` | Goal list (all) |
| `#/actions` | Action list (all) |

### API Client (`api-client.js`)
Thin wrapper around `fetch`:
```js
export const api = {
  get: (path) => fetch(`/api/v1${path}`).then(r => r.json()),
  post: (path, body) => fetch(`/api/v1${path}`, { method: 'POST', headers: {'Content-Type':'application/json'}, body: JSON.stringify(body) }),
  put: (path, body) => fetch(`/api/v1${path}`, { method: 'PUT', headers: {'Content-Type':'application/json'}, body: JSON.stringify(body) }),
  del: (path) => fetch(`/api/v1${path}`, { method: 'DELETE' }),
};
```
Vite dev proxy configured to forward `/api/v1` to `http://localhost:8000/api/v1`.

---

## 6. UI/UX Foundation

### Styling Approach
- **Vanilla CSS** with CSS custom properties (variables) for theming.
- Layout: Flexbox + CSS Grid. No heavy UI framework to keep bundle small.
- Responsive by default (stack columns on narrow screens).

### CSS Custom Properties (example)
```css
:root {
  --color-primary: #2563eb;
  --color-surface: #ffffff;
  --color-text: #111827;
  --radius: 6px;
  --shadow: 0 1px 3px rgba(0,0,0,0.1);
}
```

### Shared Components
- **`<data-table>`:** Accepts columns and rows as properties; renders sortable table with action buttons.
- **`<confirm-dialog>`:** Modal overlay for destructive actions (delete).
- **`<toast-alert>`:** Brief success/error messages stacked top-right.

---

## 7. Development Workflow

### Running the App Locally

1. **Backend**
   ```bash
   cd backend
   python -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   python run.py
   # → http://localhost:8000
   ```

2. **Frontend**
   ```bash
   cd frontend
   npm install
   npm run dev
   # → http://localhost:5173
   ```

### Build for Deployment
```bash
cd frontend
npm run build
# Static files output to frontend/dist/
```

FastAPI can optionally serve `frontend/dist` via `StaticFiles` for a single-process deployment, or use a reverse proxy.

---

## 8. Phase 1 Deliverables (This Engagement)

1. **Backend foundation**
   - [ ] SQLite database with all tables and relationships.
   - [ ] FastAPI app with full REST CRUD for use-cases, personas, goals, actions, adoption-metrics, success-criteria.
   - [ ] Nested read endpoint `/use-cases/{id}/full` to fetch complete use-case graph.
   - [ ] CORS enabled for local dev.

2. **Frontend foundation**
   - [ ] Vite project scaffolded with Lit dependency.
   - [ ] `api-client.js` communicating with backend.
   - [ ] Hash router and `<app-root>` shell.
   - [ ] Dashboard view with counts.
   - [ ] UseCase list, create, detail, edit views.
   - [ ] Persona list and form (linked to a use case).
   - [ ] Goal list and form (linked to a use case; includes inline success criteria and adoption metrics).
   - [ ] Action list and form (linked to a use case).
   - [ ] Reusable `<data-table>` and `<confirm-dialog>`.

3. **Integration**
   - [ ] Frontend proxy configured so `npm run dev` talks to backend seamlessly.
   - [ ] End-to-end manual test: create a use case → add personas → add goals with criteria/metrics → add actions.

---

## 9. Future Phases (Out of Scope for Now)

- Import/export (JSON/CSV) of enablement content.
- Full text search across all content.
- Content repurposing / templating engine.
- User authentication and multi-tenancy.
- Image upload for action screenshots (replace `image_url` with file storage).
- Audit log and versioning of content changes.

---

## 10. Schema-to-Code Mapping Quick Reference

| Schema Model | DB Table(s) | API Resource | Frontend Component(s) |
|--------------|-------------|--------------|----------------------|
| `UseCase` | `use_cases` | `/use-cases` | `use-case-list`, `use-case-form`, `use-case-detail` |
| `Persona` | `personas` + `persona_goals` | `/personas` | `persona-list`, `persona-form` |
| `Goal` | `goals` | `/goals` | `goal-list`, `goal-form` |
| `Action` | `actions` | `/actions` | `action-list`, `action-form` |
| `AdMet` | `adoption_metrics` | `/adoption-metrics` | `adoption-metric-list`, `adoption-metric-form` |
| `success_criteria` (field) | `success_criteria` | `/success-criteria` | Inline in `goal-form` |
