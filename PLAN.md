# PRD & Build Plan — do-too (reminder-heavy to-do app)

## Context
Greenfield mobile to-do app. Empty repo at `/Users/deeka/Developer/web/do-too`. Core
differentiator vs generic to-do apps = **calendar-centric browsing** + **heavy, reliable
reminders/notifications**. This plan defines the v1 PRD and the initial build so a future
session can scaffold and implement. Decisions below were confirmed with the user
interactively.

## Locked decisions
- **Stack**: Flutter (single codebase iOS + Android; best notification-plugin maturity).
- **Users**: solo/single-user for v1, but data model built so multi-user + sharing bolts on
  later (nullable `owner_id` / `assignee_id`, stable UUIDs, soft-delete).
- **Data**: local-first. SQLite on device now; cloud sync (user's own Postgres) added later
  via an abstract sync layer — no backend coupling in v1 code.
- **Views v1**: Day (default launch view = today) + rolling 3-day. Week/Month deferred to v2.
- **Task model**: MINIMAL — title, description, date, time, subtasks, reminders, recurrence.
  No priority/tags/lists/attachments in v1.
- **Recurrence**: full (daily / weekly / monthly / custom RRULE-style); each occurrence has
  its own completion state.
- **Reminders**: rich — multiple per task, snooze, mark-done from notification, optional
  full-screen alarm.

## Deliverable of this plan
On approval, create **`PRD.md`** in project root capturing everything below, then scaffold the
Flutter project. (Plan mode currently blocks writes outside this plan file.)

## Screens (v1)
1. **Home / Calendar list** — launches on Today (Day view). Toggle Day ⇄ 3-day. Shows tasks
   for the selected day(s), each with completion checkbox, time, subtask progress. Tap task →
   detail/edit.
2. **Create / Edit task** — title, description, date, time, subtasks (add/check/delete),
   recurrence rule, reminders (add multiple, each = offset or absolute time; toggle
   notification on/off), snooze default.

## Data model (SQLite, v1)
- `tasks`: id (UUID), title, description, due_date, due_time (nullable), recurrence_rule
  (nullable, RRULE string), completed (for non-recurring), created_at, updated_at,
  deleted_at (soft delete), owner_id (nullable — future multi-user).
- `subtasks`: id, task_id (FK), title, completed, position.
- `occurrences` (for recurring tasks): id, task_id (FK), occurrence_date, completed —
  materialized per-instance completion so recurring tasks track done-per-day.
- `reminders`: id, task_id (FK), fire_at / offset, enabled, snooze_minutes.
- All tables carry `updated_at` + `deleted_at` to make future Postgres sync + conflict
  resolution (last-write-wins) straightforward.

## Recommended Flutter libraries
- **State**: Riverpod.
- **Local DB**: Drift (typed SQLite; migrations; strong local-first + future-sync fit).
- **Notifications**: `flutter_local_notifications` (scheduled local reminders, timezone) +
  `awesome_notifications` (rich actions: snooze/done buttons, full-screen intent). `timezone`
  package for correct cross-zone scheduling.
- **Recurrence**: `rrule` package to expand recurrence into occurrences.
- **Routing**: go_router.

## Notification design (the hard part — OS constraints)
- Android 12+: request `SCHEDULE_EXACT_ALARM`; use exact alarms for precise reminders.
  Full-screen intent permission for alarm-style reminders. Warn users about battery
  optimization (Xiaomi/Samsung/Huawei kill background alarms).
- iOS: `UNUserNotificationCenter` local notifications only; no dynamic background
  rescheduling — schedule all reminder instances up front. Request notification permission on
  first reminder creation.
- On task create/edit: (re)schedule all enabled reminders for upcoming occurrences (rolling
  window, e.g. next N occurrences) since iOS caps pending notifications (64).
- Notification actions: "Done" (marks occurrence complete + cancels siblings), "Snooze"
  (reschedules by snooze_minutes).

## Proposed structure (feature-first)
```
lib/
  main.dart
  core/            # db (drift), notifications service, recurrence util, timezone init
  features/
    tasks/         # model, repository, providers, create/edit screen
    calendar/      # day + 3-day views, home screen
  shared/          # widgets, theme
```

## Build phases
1. Scaffold Flutter app, wire Riverpod + go_router + Drift schema + migrations.
2. Task CRUD + subtasks (repository + create/edit screen).
3. Calendar home: Day view (today) + 3-day toggle, completion.
4. Recurrence: RRULE input UI + occurrence expansion + per-occurrence completion.
5. Notifications service: schedule/cancel, exact alarms, permissions, actions (snooze/done),
   full-screen alarm option.
6. Polish: empty states, edit/delete flows, timezone correctness.

## Deferred to v2+
Week/Month calendar views; multi-user accounts + task assignment/sharing; cloud sync to
user's Postgres (build behind the abstract repository/sync interface from day 1); priority,
tags, lists, attachments.

## Verification
- `flutter run` on iOS simulator + Android emulator.
- Create one-off task with reminder → confirm notification fires at set time (use a near-term
  time); test Snooze + Done actions.
- Create recurring task (daily) → confirm distinct occurrences render across Day/3-day and
  complete independently.
- Android: verify exact-alarm permission prompt; test on a physical OEM device for battery-kill
  behavior. iOS: verify permission prompt + scheduled-instance limit handling.
- `flutter test` for repository/recurrence-expansion unit tests.
