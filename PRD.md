# Product Requirements Document — do-too

A calendar-centric, reminder-heavy to-do application for iOS and Android.

## 1. Overview

do-too is a mobile to-do application built around two core differentiators:

1. **Calendar-centric browsing.** The app opens directly onto today's tasks rather than an
   undifferentiated list, and the primary way of navigating tasks is by day.
2. **Heavy, reliable reminders and notifications.** Tasks support multiple reminders,
   snoozing, actionable notification buttons, and alarm-style delivery — and the app is
   engineered around the platform constraints that normally make mobile reminders flaky.

**Platforms.** iOS and Android from a single codebase, built with **Flutter**.

**Users.** Version 1 is single-user. However, the data model is designed so that multi-user
accounts, task assignment, and sharing can be added later without painful migrations.

**Data.** The app is local-first. In version 1, all data lives in an on-device SQLite
database. Cloud synchronization to a self-hosted Postgres database will come later, behind
an abstract sync/repository layer — version 1 code has no backend coupling of any kind.

## 2. Screens (v1)

### 2.1 Home / Calendar list

- The app launches on **Today** (the Day view).
- The user can toggle between the **Day** view and a rolling **3-day** view. Week and Month
  views are deferred to version 2.
- Each day section is laid out in the following order:
  1. **Overdue** one-off tasks that have rolled over, highlighted in red (shown on Today
     only — see §4.3 for the rollover rule).
  2. The day's tasks sorted by time, with untimed tasks grouped at the top.
  3. A collapsed **Completed** section at the bottom.
- Each task row shows a completion checkbox, the title, the time (if one is set), and a
  subtask progress indicator (for example, "2/5").
- Tapping a task opens the detail/edit screen.

### 2.2 Create / Edit task

- **Title** (required) and **description**.
- **Date** (required — undated tasks are not allowed) and **time** (optional; untimed tasks
  are grouped at the top of their day).
- **Subtasks**: the user can add, check off, reorder, and delete them.
- **Recurrence**: frequency (daily, weekly, monthly, or a custom RRULE), an interval, and an
  end condition (forever, until a date, or after N occurrences).
- **Reminders**: a notification toggle. Turning it on creates one reminder **at the due
  time** by default; preset chips add more (5 minutes, 30 minutes, 1 hour, or 1 day before),
  plus a custom offset or absolute time. A task can have any number of reminders.
- **Delete**: a one-off task deletes directly. Deleting a recurring task prompts a dialog
  asking for scope — **this occurrence / this and future occurrences / all occurrences**.

## 3. Task model (v1 — deliberately minimal)

A task consists of a title, description, date, time, subtasks, reminders, and a recurrence
rule. Version 1 intentionally excludes priority levels, tags, lists/projects, and
attachments.

## 4. Behavior rules

### 4.1 Recurrence

- Full RRULE support: daily, weekly, monthly, and custom rules, ending either never, at a
  date, or after a count of occurrences.
- **Per-occurrence completion.** Each occurrence of a recurring task carries its own
  completed state, so a daily task can be done on Monday and missed on Tuesday.
- **Edits apply to this and future occurrences only.** An edit splits the series: the
  existing task's rule is capped with an UNTIL date, and a new task (linked via `series_id`)
  carries the updated fields and rule forward. The new segment copies the subtasks (reset to
  unchecked) and reminders from the original; per-occurrence state stays with the old
  segment. Past occurrences keep their original values.
- **Deletes ask for scope.** Deleting a recurring task prompts: this occurrence (recorded as
  a skip), this and future (UNTIL cap), or all occurrences (soft-delete of the series).

### 4.2 Subtasks

- Subtasks are a plain checklist: a title, a checkbox, and a position. They do not have
  their own dates or reminders.
- The parent task is completed **manually** — checking off every subtask does not
  auto-complete the parent.

### 4.3 Overdue

- **One-off tasks automatically roll over to Today** when left incomplete, and are shown in
  red. This is a display-time rule only — the task's original date is preserved in the
  database. Rolling over does **not** re-fire past reminders: reminders fire once at their
  scheduled time, and snoozing is the mechanism for re-nagging.
- **Recurring occurrences never roll over.** A missed occurrence stays in the past, marked
  as missed. Only one-off tasks roll forward.

### 4.4 Completed tasks

- When a task is completed, it moves into a collapsed "Completed" section at the bottom of
  its day.

### 4.5 Timezone

- Times are **floating local time**. A task set for 9:00 AM fires at 9:00 AM in whatever
  timezone the user happens to be in. All datetimes are stored as local wall-clock values,
  never as UTC instants.

## 5. Notifications (core feature)

### 5.1 Behavior

- A task can have multiple reminders. Each is either an offset from the due time (0 meaning
  "at due time") or an absolute time, and each can be enabled or disabled individually.
- Notifications carry two **actions**:
  - **Done** — marks the occurrence complete and cancels its remaining reminders. This is a
    true background action: it completes directly from the notification without launching
    the app.
  - **Snooze** — notification action buttons cannot show an inline picker, so tapping
    Snooze opens the app onto a **snooze bottom sheet** (10 minutes, 1 hour, tonight,
    tomorrow, or a custom time) and reschedules that single firing. In v1, **tonight means
    8:00 PM today** and **tomorrow means 9:00 AM the next day** — these are hardcoded until
    the v2 settings screen makes them configurable. There is no per-task snooze default;
    the user chooses each time.
- An optional **full-screen alarm** style is available for critical reminders.
- Creating or editing a task cancels and reschedules all of its enabled reminders within the
  scheduling window.

### 5.2 Platform constraints

- **Android 12+.** The app requests the `SCHEDULE_EXACT_ALARM` permission and uses exact
  alarms for reminders, plus the full-screen intent permission for alarm-style delivery. The
  app guides users to disable battery optimization, since OEM skins (Xiaomi, Samsung,
  Huawei) aggressively kill background alarms.
- **iOS.** Only local notifications are available, via `UNUserNotificationCenter`, and iOS
  caps pending notifications at **64**. The app therefore schedules a rolling window of the
  **soonest ~50 reminder firings across all tasks** (soonest-first, leaving headroom for
  snoozes and immediate notifications) and tops the window up on every app open and every
  task edit. There is no dynamic background rescheduling on iOS.

## 6. Data model (SQLite, v1)

- **`tasks`** — id (UUID), title, description, due_date, due_time (nullable = untimed; for
  recurring tasks, due_date and due_time act as the RRULE's DTSTART),
  recurrence_rule (nullable RRULE string, including UNTIL/COUNT), series_id (nullable; links
  the segments of a split series), completed (used for one-offs only), created_at,
  updated_at, deleted_at (soft delete), owner_id (nullable; reserved for future multi-user).
- **`subtasks`** — id, task_id (FK), title, completed, position.
- **`occurrences`** — id, task_id (FK), occurrence_date, completed, skipped (set when a
  single occurrence is deleted). This table materializes per-instance state for recurring
  tasks, and rows are created **lazily** — only when an occurrence is completed, skipped,
  or snoozed. Display and reminder scheduling expand the RRULE on the fly; an absent row
  simply means the default state (incomplete). This keeps the table from growing
  unboundedly for never-ending rules.
- **`reminders`** — id, task_id (FK), offset_minutes (0 = at due time) or an absolute
  fire_at, enabled.

Two cross-cutting conventions:

- All datetimes are stored as local wall-clock values (floating), per the timezone rule in
  §4.5.
- Every table carries `updated_at` and `deleted_at`, so a future last-write-wins sync to
  Postgres can be layered on without schema changes.

## 7. Technology choices

| Concern | Choice |
|---|---|
| Framework | Flutter |
| Design system | Material 3 (seed color `#4F46E5` Deep Indigo, light + dark themes) |
| State management | Riverpod |
| Routing | go_router |
| Local database | Drift (typed SQLite with migrations) |
| Notifications | `flutter_local_notifications` + `timezone` |
| Recurrence | `rrule` package |

A single notification plugin is used deliberately: `flutter_local_notifications` covers
everything the PRD needs — exact alarms (`exactAllowWhileIdle`), full-screen intents, and
notification action buttons — and running a second plugin alongside it (such as
`awesome_notifications`) causes conflicts, since each registers its own Android receivers
and iOS delegate.

**Design system notes.** The UI is Material 3 throughout, themed from seed color `#4F46E5`
(Deep Indigo) via `ColorScheme.fromSeed`, with light and dark modes. Deep Indigo was chosen
for its calm, focused feel and its strong visual separation from the error-red used for
overdue styling. Small adaptive touches are used where platform feel matters most:
`showAdaptiveDialog` for the recurring-delete scope dialog, adaptive switches, and iOS-style
date/time pickers on iOS. Component mapping for the PRD's key elements:

- Reminder preset chips (5m / 30m / 1h / 1d) → Material chips.
- Day ⇄ 3-day toggle → `SegmentedButton`.
- Collapsed Completed section → `ExpansionTile`.
- New-task entry → FAB.
- Overdue rollover styling → the theme `ColorScheme` error color.

Proposed project structure (feature-first):

```
lib/
  main.dart
  core/            # drift db, notifications service, recurrence util, rollover rule
  features/
    tasks/         # model, repository, providers, create/edit screen
    calendar/      # day + 3-day views, home screen
  shared/          # widgets, theme
```

## 8. Build phases (after PRD approval)

1. Scaffold the Flutter app; wire up Riverpod, go_router, and the Drift schema with
   migrations.
2. Task CRUD and subtasks.
3. Calendar home: Day and 3-day views, completion, the overdue rollover rule, and the
   collapsed Completed section.
4. Recurrence: RRULE input UI, occurrence expansion, per-occurrence completion, series-split
   editing, and this/future/all deletion.
5. Notifications: windowed scheduling and cancellation, exact alarms, permission flows, the
   background Done action, the Snooze action with its in-app snooze sheet, and the
   full-screen alarm option.
6. Polish: empty states, floating-timezone correctness, and an OEM battery-optimization
   guidance screen.

## 9. Deferred to v2+

- Week and Month calendar views.
- Multi-user accounts, task assignment, and sharing.
- Cloud sync to self-hosted Postgres (kept behind the repository interface from day one).
- Priority, tags, lists/projects, and attachments.
- A settings screen (snooze preset configuration, day-start hour, hide-completed toggle).

## 10. Acceptance criteria (v1)

- A one-off task with a near-term reminder fires on time; **Done** completes the task
  directly from the notification without opening the app, and **Snooze** opens the in-app
  snooze sheet and reschedules the firing to the chosen time.
- A daily recurring task renders distinct occurrences across the Day and 3-day views;
  occurrences complete independently; editing splits the series while leaving the past
  unchanged; deletion honors the this/future/all choice.
- An incomplete one-off task from yesterday appears on Today in red; a missed recurring
  occurrence stays in the past.
- Android: the exact-alarm permission prompt appears, and reminders survive on a physical
  OEM device once the battery-optimization guidance has been applied.
- iOS: the notification permission prompt appears, and the rolling window tops up correctly
  past the 64-pending-notification cap.
- `flutter test` passes unit tests covering recurrence expansion, the rollover rule, and
  series splitting.
