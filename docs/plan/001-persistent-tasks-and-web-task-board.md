---
plan_id: 001-persistent-tasks-and-web-task-board
spec: 001-persistent-tasks-and-web-task-board
feature: Tasks kept in a system Postgres database, and a web task board to list and add them
gitops_app: ika100-demo/taskboard8-gitops
status: draft
repos:
  - id: taskboard8-api
    shape: service-python
    summary: Keep tasks in Postgres so they survive restarts and are shared by every instance
    acs: [AC-001.1, AC-001.2, AC-001.3]
    criteria:
      - "Given three tasks created through `POST /tasks` against a database, when the service is stopped and a new service process is started against the same database (`DATABASE_URL`), then `GET /tasks` answers `200` with exactly those three tasks, each with the same `id`, `title` and `due_date`, in the order they were created (system AC-001.1)"
      - "Given two service instances connected to the same database, when a task is created through `POST /tasks` on one of them, then every later `GET /tasks` on either instance contains that task, and tasks created alternately on both instances are listed by both in the one order they were created (system AC-001.2)"
      - "Given an empty database, when a client sends `GET /tasks`, then the answer is `200` with `[]`; and every rule of the unchanged task contract (`201` body with exactly `id`, `title`, `due_date`; trimmed titles; `422` for an invalid title or due date with nothing stored) holds with tasks kept in the database (system AC-001.1)"
      - "Given the database is unreachable (stopped, wrong host, or `DATABASE_URL` unset), when a client sends `GET /tasks` or a valid `POST /tasks`, then the answer is `503` within 5 seconds and no task is created; an invalid `POST /tasks` still answers `422` (system AC-001.3)"
      - "Given the database is unreachable, when `/ready` and `/health` are called, then `/ready` answers `503` within 1 second and `/health` answers `200`; the process keeps running and does not exit, also when the database is already unreachable at start-up (system AC-001.3)"
      - "Given the database became unreachable after tasks were created, when it is reachable again, then within 30 seconds and without restarting the service `/ready` answers `200` and `GET /tasks` answers `200` with the tasks that existed before the outage (system AC-001.3)"
    arguments: |
      Storage only changes: the HTTP contract of spec 001-create-and-list-tasks (docs/specs/001-create-and-list-tasks/design.md)
      stays exactly as it is, and its acceptance tests stay green, now against Postgres.
      Config: `DATABASE_URL` (CloudNativePG URL with scheme `postgresql://`, user, password, host, port 5432 and database; PG* are injected too). If your driver
      needs `postgresql+psycopg://`, rewrite the scheme in code, never in gitops. Missing `DATABASE_URL` = database unreachable.
      Schema: ensured idempotently on the first successful connection (not only at start-up), guarded by a Postgres advisory
      lock so two replicas starting together do not race. Order by a database-generated monotonically increasing column
      (identity/bigserial), not by a timestamp. `id` stays an opaque uuid4 string.
      Tests run against a real Postgres that `devbox run test` starts itself (devbox `postgresql` package with a throwaway
      data dir per session, or an equivalent) so they pass locally and in CI with no extra service; each test starts from an
      empty table. "Restart" and "two instances" are tested with separate `create_app()` instances on one database.
      Smoke: keep the existing ones; a read-only `GET /tasks` -> 200 smoke must remain for staging/prod.
    depends_on: []
    done: false
  - id: taskboard8-web
    shape: web-nextjs
    summary: Task board at the root URL that lists all tasks and adds new ones
    acs: [AC-001.4, AC-001.5, AC-001.6, AC-001.7, AC-001.8, AC-001.9]
    criteria:
      - "Given the Task API at `TASK_API_URL` returns tasks from `GET /tasks`, when a person opens `/`, then the page shows one entry per task in the order the API returned them (oldest first), each with its title and its `due_date` shown exactly as the `YYYY-MM-DD` string the API sent, or `No due date` when it is `null`; the list is fetched on every request, never cached or prerendered (system AC-001.4)"
      - "Given `GET /tasks` returns `[]`, when a person opens `/`, then the page shows `No tasks yet.` and no list, together with the form to add a task (system AC-001.5)"
      - "Given the board is open, when a person enters the title `Buy milk`, optionally picks a due date, and submits, then the web app sends `POST /tasks` to the Task API with the trimmed title and the due date as `YYYY-MM-DD` or `null`, and without a page reload the task the API returned appears as the last entry with its title and due date (or `No due date`), both form fields are emptied, and after a reload the task is still listed (system AC-001.6)"
      - "Given the board is open, when a person submits a title that is empty or only whitespace, then no request reaches the Task API, the page shows `Enter a title.` next to the title field and the form keeps what was entered; with a title longer than 200 characters after trimming it shows `The title can be at most 200 characters.` instead (system AC-001.7)"
      - "Given the Task API is unreachable, answers a `5xx` status, does not answer within 5 seconds, or `TASK_API_URL` is unset, when a person opens `/`, then the page answers `200` and shows `Tasks could not be loaded. Please try again later.` instead of the list, with the form still shown (system AC-001.8)"
      - "Given the Task API is unreachable, slow (over 5 seconds) or answers anything but `201` to `POST /tasks`, when a person submits a valid task, then the page shows `The task could not be saved. Please try again.`, the list is unchanged and the form keeps what was entered (system AC-001.8)"
      - "Given a task whose title is `<script>alert(1)</script> & <b>bold</b>`, when a person opens the board or has just added that task, then the title is shown literally as those characters, no script runs and no text is bold (system AC-001.9)"
    arguments: |
      The browser never calls the Task API: the server renders the list and proxies the create (see the contract, A9).
      `/api/health` and `/api/ready` must NOT depend on the Task API (AC-001.8 needs the page to load while it is down).
      Show `due_date` as the string the API sent; never parse it into a Date (time-zone shifts). Render titles as React
      text only; no dangerouslySetInnerHTML anywhere.
      Tests: unit tests with a mocked fetch; Playwright e2e against a stub Task API (a small HTTP server implementing
      `GET/POST /tasks` of the contract, switchable to down / 500 / slow) at `TASK_API_URL`.
      Smoke: read-only `@smoke` "root page loads 200 and shows a list or `No tasks yet.`"; `@smoke @writes` adds
      `smoke-<uuid>` and sees it after reload (dev only).
      Document `TASK_API_URL` in docs/env-vars.md (server-only; never `NEXT_PUBLIC_`).
    depends_on: []
    done: false
  - id: taskboard8-gitops
    shape: gitops-app
    summary: Declare the Postgres database, give it to the Task API and point the web app at the Task API
    acs: [AC-001.1, AC-001.2, AC-001.3, AC-001.4, AC-001.6]
    gitops:
      - {resource: postgres}
      - {uses: postgres, component: taskboard8-api}
      - {env: {TASK_API_URL: 'http://taskboard8-api'}, component: taskboard8-web}
    depends_on: []
    done: false
gitops_pin: []
---

## Decomposition

Three entries, one parallel wave, no `depends_on`: no shared library is needed (the web app talks to the API over
HTTP with the contract below, which is the unchanged contract of taskboard8-api spec 001), and every component
degrades safely when another part arrives later (the API without a database answers `503` and stays alive; the web
app without `TASK_API_URL` or with the API down shows its error message). So the API, the web app and the gitops
change build in parallel and land together in the system ship's compose PR.

## taskboard8-api — Keep tasks in Postgres so they survive restarts and are shared by every instance

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2, AC-001.3 · **Depends on:** []

Today `TaskStore` keeps tasks in a list in process memory (`app.state.task_store`). It is replaced by a store that
reads and writes the `tasks` table of the database named by `DATABASE_URL`; routes, models and validation stay as
they are (spec A1, non-goal: no contract change). The repo owns persistence (AC-001.1), sharing between instances
(AC-001.2, which falls out of one shared database; tested with two app instances on one database, the cluster
replica count is not part of this spec) and the outage behaviour (AC-001.3): `503` on task endpoints, `/ready`
reflecting the database, `/health` independent of it, and reconnecting by itself (a pool that checks connections
before use or a fresh connection per request, with the timeouts of the contract) so that recovery takes well under 30
seconds without a restart. Start-up must not fail when the database is unreachable; otherwise the pod would crash-loop
instead of reporting not-ready (spec A4). Tasks held in memory at deploy time are not migrated (A3).

## taskboard8-web — Task board at the root URL that lists all tasks and adds new ones

**Shape:** `web-nextjs` · **Criteria:** AC-001.4 to AC-001.9 · **Depends on:** []

The template's starter page at `/` is replaced by the board (A5). **How the web reaches the API (A9): server-side
only.** The page is a dynamic Server Component that calls `GET {TASK_API_URL}/tasks` on each request; the form is a
client component that posts to the web app's own route `POST /api/tasks`, which validates and forwards to
`POST {TASK_API_URL}/tasks`, then appends the returned task to the list without a reload (A8). Reasons: the Task API
URL is a cluster-internal Service name the browser cannot resolve, no CORS is needed in the API, and every failure
mode (down, `5xx`, timeout) is handled in one place on the server. The repo owns the list and empty state
(AC-001.4, AC-001.5), the create flow (AC-001.6), client-side title validation with the exact messages (AC-001.7, A7),
the two error messages (AC-001.8) and literal rendering of titles (AC-001.9).

## Contract

### Task API (taskboard8-api), unchanged from taskboard8-api spec 001

Base URL inside the environment: `http://taskboard8-api` (Service port 80 -> container 8080), given to the web app as
`TASK_API_URL` (no trailing slash). All bodies `application/json`; no authentication.

- `GET /tasks` -> `200`, a JSON array of tasks, oldest first (creation order), `[]` when none. A task is exactly
  `{"id": string, "title": string, "due_date": "YYYY-MM-DD" | null}`; `id` is an opaque non-empty string (do not parse),
  `title` is already trimmed, 1 to 200 characters.
- `POST /tasks` with `{"title": string, "due_date": "YYYY-MM-DD" | null}` (`due_date` may be omitted) -> `201` with the
  created task (same shape). The API trims `title`; the trimmed value must be 1 to 200 characters. Unknown fields are
  ignored.
  - `422` invalid body (FastAPI's standard `{"detail": [...]}`); nothing is stored.
  - `503` the database is unreachable (only for a valid body; validation runs first, so an invalid body still gets
    `422`); nothing is stored.
- `GET /ready` -> `200` when a `SELECT 1` on the database succeeds within 500 ms, else `503`. `GET /health` -> `200`
  whenever the process runs, never touches the database.
- Persistence: tasks are kept in the system's Postgres database (`DATABASE_URL`); order is the order of creation
  across all instances.

### Web app internal route (taskboard8-web, used only by its own browser code)

- `POST /api/tasks` with `{"title": string, "due_date": "YYYY-MM-DD" | null}`. The route trims the title and rejects
  an empty or over-200-character title itself with `422` (the browser has already checked; this guards direct calls)
  without calling the Task API; otherwise it forwards to the Task API's `POST /tasks`.
  - `201` with the Task API's task, unchanged.
  - `502` `{"error": "task_api_unavailable"}` for anything else from the Task API (unreachable, timeout, `5xx`, `422`
    or any unlisted status); the page then shows `The task could not be saved. Please try again.`
- Title length is counted in Unicode code points after `String.prototype.trim()` (Python counts the same way); the
  Task API's rule is authoritative.
- Due date input: an `<input type="date">`; empty means `null`. Display strings are exactly those of the spec:
  `No due date`, `No tasks yet.`, `Enter a title.`, `The title can be at most 200 characters.`,
  `Tasks could not be loaded. Please try again later.`, `The task could not be saved. Please try again.`

### Errors

- Task API error format: `422` is FastAPI's `{"detail": [ ... ]}`; `503` has a JSON body `{"detail": "database
  unavailable"}`, but callers and tests assert only the status, never the body (spec A4).
- The web app treats the Task API's answers as follows. `GET /tasks`: only `200` with a JSON array is success;
  anything else (connection error, timeout, any non-`200` status, a body that is not an array) shows
  `Tasks could not be loaded. Please try again later.` with page status `200`. `POST /tasks`: only `201` is success;
  everything else, `422` included, is "could not be saved" and the form keeps its input.
- The web app's own route answers only `201`, `422` or `502` as listed above; its browser code treats any other
  status or a network error like `502`.
- No retries anywhere: a person retries by reloading or submitting again (no live refresh, A8).

### Timeouts

- Web server -> Task API, `GET /tasks` and `POST /tasks`: 5 seconds total each (`AbortSignal.timeout(5000)`); on
  expiry the call counts as unavailable (messages above). The page is rendered on each request, so a slow API delays
  the page by at most 5 seconds before it shows the load error.
- Browser -> web `POST /api/tasks`: 10 seconds; on expiry the page shows the save error and keeps the form input.
- Task API -> Postgres: connect timeout 2 seconds, statement timeout 3 seconds, so a task request answers `503`
  within 5 seconds when the database is down or hanging; `/ready`'s check is bounded to 500 ms (the kubelet probe
  times out after 1 second). Broken connections are dropped and a new one is opened on the next request, so after
  the database is back the API serves again on its next request and `/ready` turns `200` at the next 5-second probe.
- When the Task API's only pod is not ready, its Service has no endpoints: the web app sees a connection error and
  shows its error messages; that is the intended behaviour.

## gitops-app PR

Operations, in this order (run by `/system:ship` with `cplat resource add` / `cplat component set`):

1. `{resource: postgres}` declares the system database (A2: in-cluster CloudNativePG, credentials generated by the
   platform). It is the system's first operator-backed resource, so `operators/root.yaml` must be applied once per
   cluster (`/system:ship` does it on the local `cluster-up` cluster; on any other cluster a person runs
   `KUBE_CONTEXT=<ctx> devbox run bootstrap`).
2. `{uses: postgres, component: taskboard8-api}` injects `DATABASE_URL` and `PG*`; the database is rendered in every
   environment where the API runs (today: dev).
3. `{env: {TASK_API_URL: 'http://taskboard8-api'}, component: taskboard8-web}` wires the web app to the API Service.

Both components are already exposed (`taskboard8-api`, `taskboard8-web` hosts) and need no exposure change. No
backup setting (non-goal).

**Pins:** none planned (`gitops_pin: []`). Both components run only in `dev`, which tracks `main` and is landed by the
system ship's compose PR (spec 144). Staging is a later `/instance:promote` of both components together; promoting
the API to staging also renders the database there through its `uses`.

**Verify after Argo reconciles dev:** the CloudNativePG cluster is healthy; `taskboard8-api` pods are ready
(`/ready` 200, i.e. the database is reachable); `http://taskboard8-api.taskboard8-dev.localhost:8088/tasks` answers
`200`; `http://taskboard8-web.taskboard8-dev.localhost:8088/` shows the board; a task added there is still listed
after the API deployment is restarted (`kubectl rollout restart` is a human step, or let the next pin roll it).
