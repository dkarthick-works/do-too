# PRD — do-too

A calendar-centric, reminder-heavy to-do application for iOS and Android.

## 1. Overview

do-too is a mobile to-do app whose core differentiators are **calendar-centric browsing**
(the app opens on today's tasks, not an undifferentiated list) and **heavy, reliable
reminders/notifications** (multiple reminders per task, snooze, actionable notifications,
alarm-style delivery).

- **Platforms**: iOS + Android, single codebase (**Flutter**).
- **Users**: single-user in v1. Data model is designed so multi-user accounts, task
  assignment, and sharing can be added later without migration pain.
- **Data**: local-first. All data in on-device SQLite in v1. Cloud sync to a self-hosted
  Postgres database comes later, behind an abstract sync/repository layer — v1 code has no
  backend coupling.

## 2. Screens (v1)

### 2.1 Home / Calendar list
- App launches on **Today** (Day view).
- Toggle between **Day** and rolling **3-day** view. (Week/Month deferred to v2.)
- Each day section shows, in order:
  1. **Overdue** rolled-over one-off tasks, marked red (Today only — see §4.3).
  2. Tasks sorted by time; untimed tasks grouped at the top.
  3. Collapsed **Completed** section at the bottom.
- Each task row: completion checkbox, title, time (if set), subtask progress indicator
  (e.g. 2/5).
- Tap a task → detail/edit screen.

### 2.2 Create / Edit task
- **Title** (required), **description**.
- **Date** (required — undated tasks are not allowed), **time** (optional; untimed tasks
  group at top of the day).
- **Subtasks**: add / check / reorder / delete.
- **Recurrence**: frequency (daily / weekly / monthly / custom RRULE), interval, end
  condition (forever / until date / N times).
- **Reminders**: notification toggle. ON creates one reminder **at due time** by default;
  preset chips add more (5 min / 30 min / 1 hr / 1 day before) plus custom offset/time.
  Multiple reminders per task.
- **Delete**: one-off deletes directly; recurring prompts a dialog — **this / this and
  future / all occurrences**.

## 3. Task model (v1 — deliberately minimal)

Title, description, date, time, subtasks, reminders, recurrence. **No** priority, tags,
lists/projects, or attachments in v1.

## 4. Behavior rules

### 4.1 Recurrence
- Full RRULE support: daily / weekly / monthly / custom; end = forever, until date, or count.
- **Per-occurrence completion**: each occurrence of a recurring task has its own completed
  state.
- **Editing a recurring task applies to this + future occurrences only.** The edit splits the
  series: the old task's rule is capped with UNTIL, and a new task (linked by `series_id`)
  carries the new fields/rule forward. Past occurrences keep their old values.
- **Deleting a recurring task asks scope**: this occurrence (skip flag) / this and future
  (UNTIL cap) / all (soft-delete series).

### 4.2 Subtasks
- Plain checklist: title + checkbox + position. No dates or reminders on subtasks.
- Parent task is completed **manually** — completing all subtasks does not auto-complete it.

### 4.3 Overdue
- **One-off tasks auto-roll over to Today** when incomplete, displayed in red. This is a
  display-time rule — the original date is preserved in the database.
- **Recurring occurrences never roll over.** A missed occurrence stays in the past as missed.

### 4.4 Completed
- Completed tasks move to a collapsed "Completed" section at the bottom of their day.

### 4.5 Timezone
- **Floating local time.** A 9:00 AM task fires at 9:00 AM in whatever timezone the user is
  in. All datetimes are stored as local wall-clock values, not UTC instants.

## 5. Notifications (core feature)

### 5.1 Behavior
- Multiple reminders per task; each is an offset from due time (0 = at due time) or an
  absolute time; individually enable/disable.
- Notification **actions**:
  - **Done** — marks the occurrence complete and cancels its remaining reminders.
  - **Snooze** — opens a **picker** (e.g. 10 min / 1 hr / tonight / tomorrow) and
    reschedules that single fire. No per-task snooze default.
- Optional **full-screen alarm** style for critical reminders.
- Task create/edit cancels and reschedules all enabled reminders for the scheduling window.

### 5.2 OS constraints
- **Android 12+**: request `SCHEDULE_EXACT_ALARM`; use exact alarms. Full-screen intent
  permission for alarm-style. Guide users to disable battery optimization (Xiaomi / Samsung /
  Huawei aggressively kill background alarms).
- **iOS**: local notifications only via `UNUserNotificationCenter`; **64
  pending-notification cap** → schedule a rolling window of the next N occurrences and
  re-top-up on every app open and task edit. No dynamic background rescheduling.

## 6. Data model (SQLite, v1)

- `tasks`: id (UUID), title, description, due_date, due_time (nullable = untimed),
  recurrence_rule (nullable RRULE string incl. UNTIL/COUNT), series_id (nullable — links
  split-off series segments), completed (one-offs only), created_at, updated_at, deleted_at
  (soft delete), owner_id (nullable — future multi-user).
- `subtasks`: id, task_id (FK), title, completed, position.
- `occurrences`: id, task_id (FK), occurrence_date, completed, skipped (single-occurrence
  delete) — materialized per-instance state for recurring tasks.
- `reminders`: id, task_id (FK), offset_minutes (0 = at due time) or absolute fire_at,
  enabled.
- All datetimes stored as local wall-clock (floating).
- All tables carry `updated_at` + `deleted_at` → ready for future Postgres last-write-wins
  sync.

## 7. Tech choices

| Concern | Choice |
|---|---|
| Framework | Flutter |
| State | Riverpod |
| Routing | go_router |
| Local DB | Drift (typed SQLite, migrations) |
| Notifications | `flutter_local_notifications` + `awesome_notifications` + `timezone` |
| Recurrence | `rrule` package |

Proposed structure (feature-first):

```
lib/
  main.dart
  core/            # drift db, notifications service, recurrence util, rollover rule
  features/
    tasks/         # model, repository, providers, create/edit screen
    calendar/      # day + 3-day views, home screen
  shared/          # widgets, theme
```

## 8. Build phases (post-PRD approval)

1. Scaffold Flutter app; Riverpod + go_router + Drift schema/migrations.
2. Task CRUD + subtasks.
3. Calendar home: Day + 3-day, completion, overdue rollover rule, collapsed Completed section.
4. Recurrence: RRULE UI, occurrence expansion, per-occurrence completion, series-split edit,
   this/future/all delete.
5. Notifications: windowed schedule/cancel, exact alarms, permissions, Done/Snooze-picker
   actions, full-screen alarm option.
6. Polish: empty states, timezone floating correctness, OEM battery guidance screen.

## 9. Deferred to v2+

- Week and Month calendar views.
- Multi-user accounts, task assignment, sharing.
- Cloud sync to self-hosted Postgres (built behind the repository interface from day 1).
- Priority, tags, lists/projects, attachments.
- Settings screen (snooze presets config, day-start hour, hide-completed toggle).

## 10. Acceptance criteria (v1)

- One-off task with near-term reminder fires on time; Done and Snooze-picker actions work
  from the notification.
- Daily recurring task renders distinct occurrences across Day/3-day; occurrences complete
  independently; editing splits the series (past unchanged); delete honors
  this/future/all.
- Incomplete one-off from yesterday appears on Today in red; missed recurring occurrence
  stays in the past.
- Android: exact-alarm permission prompt appears; reminders survive on a physical OEM
  device with battery optimization guidance applied.
- iOS: notification permission prompt appears; rolling window tops up past the 64-pending
  cap.
- `flutter test` passes: recurrence expansion, rollover rule, series split.
