---
spec_id: 001-persistent-tasks-and-web-task-board
title: Persistent tasks and web task board
status: done
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks and web task board

## Problem

The taskboard8 Task API keeps tasks in its own memory, so every restart or new rollout wipes the list, and two running instances would each show a different list. Nobody can rely on a task still being there tomorrow. On top of that, the only way to see or record a task is to call the API by hand: the web app still shows the template's starter page. This spec makes tasks durable in a Postgres database that the system declares, and turns the web app into a task board where people can see all tasks and add new ones.

## Stories

As a person using the task board, I want the tasks I recorded to still be there after the system restarts or is redeployed, so that I can trust the list.
**Repos:** taskboard8-api, taskboard8-gitops

As a person using the task board, I want to open the web app and see every task with its due date, so that I know what needs doing without calling the API.
**Repos:** taskboard8-web, taskboard8-api, taskboard8-gitops

As a person using the task board, I want to add a task with a title and an optional due date in the web app, so that I can record work without any tool besides my browser.
**Repos:** taskboard8-web, taskboard8-api, taskboard8-gitops

## Acceptance criteria

- **AC-001.1** Given three tasks created through `POST /tasks` of the Task API, when every instance of the Task API is restarted, then `GET /tasks` returns `200` with exactly those three tasks, each with the same `id`, `title` and `due_date` as before, in the order they were created.
- **AC-001.2** Given the Task API running with two instances, when a task is created through one request, then every later `GET /tasks`, whichever instance answers it, contains that task.
- **AC-001.3** Given the database is unreachable, when a client sends `GET /tasks` or a valid `POST /tasks`, then the response is `503`, the Task API's `/ready` answers `503` while `/health` still answers `200`, and no task is created; when the database is reachable again, then within 30 seconds and without a restart of the Task API, `GET /tasks` answers `200` with the tasks that existed before the outage.
- **AC-001.4** Given tasks exist, when a person opens the web app's root URL, then the page shows one entry per task, oldest first, each with its title and its due date as `YYYY-MM-DD`, or the text `No due date` when the task has none, matching what `GET /tasks` of the Task API returns at that moment.
- **AC-001.5** Given no task exists, when a person opens the web app's root URL, then instead of a list the page shows the text `No tasks yet.` together with the form to add a task.
- **AC-001.6** Given the task board is open, when a person enters the title `Buy milk`, optionally picks a due date, and submits the form, then without reloading the page the new task appears as the last entry of the list with that title and due date (or `No due date`), the form fields are emptied, and the task is also returned by `GET /tasks` of the Task API and is still listed after the page is reloaded.
- **AC-001.7** Given the task board is open, when a person submits the form with a title that is empty, only whitespace, or longer than 200 characters after trimming, then no task is created, the form keeps what the person entered, and the page shows `Enter a title.` (empty or whitespace) or `The title can be at most 200 characters.` (too long) next to the title field.
- **AC-001.8** Given the Task API is unreachable or answers with a `5xx` status, when a person opens the web app's root URL, then the page still loads with status `200` and shows `Tasks could not be loaded. Please try again later.` instead of the list; and when a person submits a valid task in that state, then the page shows `The task could not be saved. Please try again.` and the form keeps what the person entered.
- **AC-001.9** Given a task whose title is `<script>alert(1)</script> & <b>bold</b>`, when a person opens the task board, then the title is shown literally as those characters, no script runs and no text is rendered bold.

## Non-goals

- Editing, completing or deleting tasks, in the API or in the web app.
- Users, login, ownership or per-user task lists; authentication or authorization of the web app or the API (consistent with taskboard8-api spec 001: who can reach the system is decided by its exposure in each environment).
- Pagination, filtering, sorting options or search on the board.
- Live updates: a task created by someone else appears on an open board only after the page is reloaded.
- Database backups, point-in-time restore and a backup schedule (the system's `backup:` setting); a later spec can add them.
- Carrying over tasks held in the Task API's memory at the moment the new version is deployed.
- A limit on the number of stored tasks.
- Changing the Task API's request and response contract (fields, validation rules, status codes of taskboard8-api spec 001).

## Assumptions

- **A1** The Task API's contract from taskboard8-api spec 001 stays exactly as it is (`POST /tasks`, `GET /tasks`, fields `id`, `title`, `due_date`, `422` on invalid input, trimmed titles, oldest first); only where tasks are kept changes (other option: extend the contract, e.g. a `created_at` field; affects AC-001.1, AC-001.4, AC-001.6).
- **A2** Tasks are stored in one Postgres database declared as a system resource and running inside the cluster in each environment the Task API runs in, reachable only by the system's components, with credentials generated and injected by the platform (other option: an external managed database; affects AC-001.1, AC-001.2, AC-001.3).
- **A3** Tasks in the Task API's memory when the new version is deployed are not carried over; this matches today's behaviour, where every rollout already empties the list (other option: a one-off copy, which is impossible for memory that ends with the old process; affects AC-001.1).
- **A4** While the database is unreachable the Task API answers `503` and reports itself not ready, but stays alive so it is not restarted in a loop; tests assert only the status, not the error body (other option: `500`, or failing liveness too; affects AC-001.3).
- **A5** The task board lives at the web app's root URL and replaces the template's starter page (other option: a separate `/tasks` page; affects AC-001.4 to AC-001.9).
- **A6** The board lists tasks oldest first, the same order as `GET /tasks`, and shows due dates as `YYYY-MM-DD` (other option: newest first, or a localized date format; affects AC-001.4, AC-001.6).
- **A7** The web app checks the title (required, at most 200 characters after trimming) before sending, with the messages in AC-001.7, in English; the due date is entered with a date picker and may be left empty (other option: rely only on the API's `422` and show a generic error; affects AC-001.7).
- **A8** The board has no live refresh; after a successful create it shows the new task without a page reload, other people's changes appear on reload (other option: polling every few seconds; affects AC-001.6).
- **A9** The web app reaches the Task API through the system's internal wiring; how (server-side or from the browser) is left to the plan (other option: fix it here; affects AC-001.4, AC-001.6, AC-001.8).

## Open questions

## Changelog

- 2026-11-09 created
- 2026-11-09 approved
- 2026-11-10 building
- 2026-11-10 done
