# Task-note fork roadmap

Status: agreed planning baseline. No implementation or migration from the original
Todosian storage format exists yet.

## Product scope

The fork changes Todosian around five features:

1. One Markdown file is one task; its immediate parent folder is its list.
2. Task metadata is stored in Obsidian-compatible YAML frontmatter instead of the
   Obsidian Tasks plugin suffix format.
3. A dedicated activity creates and edits notes opened from a widget.
4. An offline RU/EN parser extracts dates and times from natural-language text.
5. A persistent local index serves the app and widgets without reading SAF files
   on each interaction, then writes queued changes back to Markdown.

There is no import from the original "one file = one list" format in the first
version.

## Agreed document model

For `Work/Prepare report.md`:

```markdown
---
task-id: "48fb9fee-ff7d-4b90-aedc-9e607564aa7c"
task-status: todo
task-priority: high
task-due-at: 2026-10-12T09:00:00
task-created: 2026-10-09T10:00:00
---

Collect the results and check the totals with the team.

#task #work/report
```

The data sources are deliberately unambiguous:

| Task data | Source |
| --- | --- |
| Title | Filename without `.md` |
| List | Immediate parent folder |
| Comment | Entire Markdown body after frontmatter |
| Tags | Inline tags in the Markdown body only |
| ID, status, priority and dates | Frontmatter |

Rules:

- Do not add `task-title` or a managed Markdown heading. Editing the title renames
  the file.
- Do not add comment sections or timestamped comment headings. The whole body is
  one editable comment.
- Frontmatter `tags` may exist and must be preserved, but it is ignored for task
  selection and display.
- Every folder below the selected root is a separate list. The selected root may
  also contain tasks.
- A Markdown file is an indexed task when its body contains at least one of the
  configured task tags. Tag comparison is case-insensitive.
- Removing all matching body tags removes the note from task views without
  deleting the file.
- `task-id` is a UUID and remains stable across rename and move operations. The
  index assigns a local UUID when it is missing and writes it into the note on
  explicit save, not during a read-only scan.
- `task-due-date` stores a date-only deadline. `task-due-at` stores a local date
  and time. They are mutually exclusive so a date-only task is not silently
  converted to midnight.

`shortcut: the first version matches any configured tag and has no boolean tag
query language; add one only if real filtering use cases require it.`

`shortcut: task-owned frontmatter uses flat scalar fields and preserves all other
frontmatter as opaque text; add a full YAML model only if task metadata needs
nested values.`

## Local index and write-back

Markdown remains the canonical store for committed data. A Room database is a
rebuildable projection plus durable local work that has not yet reached SAF.

The minimum persisted data is:

- task ID, document URI/path, filename, parent-list identity;
- status, priority, dates and recurrence needed by task views;
- body, extracted body tags and widget preview;
- document revision and synchronization state;
- pending write commands and editor drafts.

The widget reads only indexed summary fields. It must not scan or parse Markdown.
The editor may use the indexed body immediately and reconcile it with SAF in the
background.

### Write path

1. In one Room transaction, update the projection and insert a pending command.
2. Refresh affected app state and widgets from the projection.
3. Enqueue persistent write work.
4. Re-read the target document and compare its revision with the command base.
5. Patch only the requested filename, frontmatter fields, or body.
6. Mark that local version as committed; do not acknowledge newer edits by
   mistake.

Commands are desired-state operations and must be safe to retry:

- `CreateTask`
- `SetTaskStatus`
- `UpdateTaskMetadata`
- `UpdateTaskBody`
- `RenameTask`
- `MoveTask`
- `DeleteTask`

Use `SetTaskStatus(done)` rather than `ToggleTask`, because a retry must not flip
the state twice.

Pending commands and drafts are not rebuildable and must survive index rebuilds.
On process start, schedule work whenever pending commands exist; this covers a
process death between the Room transaction and WorkManager enqueue call.

Before each write, read the current document. Merge changes to independent
regions, for example an external body edit and a local status edit. If both sides
changed the same metadata field, filename, or body from the same base revision,
surface a conflict rather than silently overwriting either side.

SAF providers do not guarantee atomic replacement or stable URIs after rename.
The storage layer must verify writes, retain recovery information for a pending
operation, and update indexed document references after rename or move.

## Natural-language date and time

The parser is deterministic, offline Kotlin code. It receives `Clock`, `ZoneId`
and language explicitly so unit tests do not depend on the current time or device
locale.

First-version RU/EN grammar:

- `сегодня`, `завтра`, `послезавтра`;
- `today`, `tomorrow`, `day after tomorrow`;
- weekday names and `следующий/next <weekday>`;
- `через/in N минут/minutes`, `часов/hours`, `дней/days`, `недель/weeks`;
- date plus time, for example `завтра в 18:30` and `tomorrow at 6:30 pm`;
- unambiguous numeric dates and dates with month names.

The result contains the source range, date-only or date-time value, and an
ambiguity flag. The UI previews the interpretation and applies it only after user
confirmation. Removing the matched words from the title or body is a separate
explicit action.

`shortcut: the first parser is rule-based and handles only documented phrases;
extend its grammar from failing user examples instead of adding ML or a third-party
NLP engine.`

## Dedicated editor activity

`NoteEditorActivity` is the direct target of widget intents. It supports create
and edit modes and shares one `NoteEditorScreen` and ViewModel with navigation
from the main app.

The editor exposes only the task title, metadata and one Markdown body field.
Saving performs the corresponding durable commands. A title edit is a rename; a
body edit replaces the comment and causes tags and preview to be re-indexed.

Each widget item passes a stable task ID, not a line number, title or mutable URI.
Distinct task intents must remain distinct after process recreation.

## Implementation roadmap

### Phase 0: freeze storage rules

- [ ] Finalize frontmatter keys, enum values and invalid-value behavior.
- [ ] Define exact inline-tag syntax and Markdown contexts ignored by extraction.
- [ ] Define revision, conflict and recovery behavior.
- [ ] Turn the date/time examples above into an executable grammar test table.

Exit condition: sample notes cover valid, malformed and externally edited files.

### Phase 1: task-note codec and SAF operations

- [ ] Replace line-based task identity with `TaskNote` and `TaskList` models.
- [ ] Read the filename, frontmatter and body without normalizing untouched text.
- [ ] Extract body tags with source ranges, excluding fenced and inline code.
- [ ] Patch owned frontmatter fields while preserving unknown fields and body.
- [ ] Create, rename, move and delete individual Markdown notes and folders.

Exit condition: metadata edits preserve the body byte-for-byte, body edits
preserve unrelated frontmatter, and renames preserve file content.

### Phase 2: index, reconciliation and pending writes

- [ ] Add the Room schema for lists, task summaries/bodies, tags, drafts and
  pending commands.
- [ ] Recursively index the selected root and reconcile external changes.
- [ ] Implement optimistic local transactions and idempotent write-back.
- [ ] Resume pending operations after process death and expose failures/conflicts.
- [ ] Refresh widgets whenever indexed widget data changes.

Exit condition: create/edit/complete survives process termination and eventually
reaches Markdown without losing external changes.

### Phase 3: migrate application screens

- [ ] Show the recursive folder/list hierarchy from the index.
- [ ] Read list, Daily Focus and search data from indexed queries.
- [ ] Replace line-level add/edit/toggle/delete/move operations with task commands.
- [ ] Add task-tag configuration and independent reminder settings.
- [ ] Remove Tasks plugin UI and obsolete line-position/subtask assumptions.

Exit condition: normal app use no longer depends on `MarkdownParser` todo lines.

### Phase 4: editor activity and widgets

- [ ] Add `NoteEditorActivity` and register it in the manifest.
- [ ] Open create/edit directly from both widgets and the main app.
- [ ] Render widget rows exclusively from indexed summaries.
- [ ] Change widget completion to `SetTaskStatus` through the pending-write path.
- [ ] Make manual refresh enqueue reconciliation rather than read SAF in Glance.

Exit condition: widget render and task actions perform no direct SAF reads, and
the correct task opens after application process recreation.

### Phase 5: RU/EN date-time parser

- [ ] Implement tokenization, RU rules, EN rules and temporal resolution using
  `java.time`.
- [ ] Integrate preview/confirmation into title and body input.
- [ ] Persist accepted values to frontmatter through `UpdateTaskMetadata`.
- [ ] Test month/year boundaries, locale ambiguity and daylight-saving changes.

Exit condition: the documented grammar passes with a fixed clock and zone.

### Phase 6: reminders and hardening

- [ ] Query due tasks from the index while retaining `AlarmManager` for the fixed
  daily reminder time.
- [ ] Test revoked permissions, deleted/moved folders, SD cards and SAF providers.
- [ ] Test concurrent Obsidian/Syncthing edits and repeated widget actions.
- [ ] Measure initial indexing and widget latency on a large test vault.
- [ ] Update README and user-facing documentation after behavior is implemented.

Exit condition: build, unit tests, lint and device tests cover the complete write
and recovery path.

## Target project structure

Keep one `:app` module until build time or test isolation proves another module is
needed:

```text
app/src/main/java/com/isotjs/todosian/
├── data/
│   ├── model/          # TaskNote, TaskSummary, TaskList
│   ├── storage/        # SAF document operations and task-note codec
│   ├── index/          # Room entities, DAOs and reconciliation
│   ├── sync/           # Pending command processor and workers
│   └── settings/
├── datetime/           # Small RU/EN rule-based parser
├── notifications/
└── ui/
    ├── home/
    ├── tasklist/
    ├── editor/         # NoteEditorActivity and shared editor UI
    ├── dailyfocus/
    ├── widget/
    ├── settings/
    ├── components/
    └── theme/
```

Do not create separate domain/use-case modules or wrappers in advance. Extract a
new boundary only when two real callers need it or Android dependencies prevent a
plain unit test.

## Required verification

At every implementation phase:

```bash
./gradlew :app:assembleDebug
./gradlew :app:testDebugUnitTest
./gradlew :app:lint
```

Add instrumentation tests for Room migrations, SAF integration, activity intents
and widget actions. Critical regression cases are:

- metadata write preserves body, unknown frontmatter, whitespace and line endings;
- body write preserves unrelated frontmatter;
- frontmatter tags never make a note a task;
- rename/move preserves `task-id` and pending commands;
- pending work is safe after a crash or duplicate execution;
- external and local independent changes merge, while same-region changes conflict;
- widget rendering and status changes do not read SAF;
- RU/EN parsing uses a fixed clock and handles ambiguous input explicitly.

## Explicitly out of scope for the first version

- migration from original Todosian todo-line files;
- Obsidian Tasks plugin emoji metadata;
- network sync, accounts, telemetry or analytics;
- a general YAML editor or full Markdown AST;
- ML-based language parsing;
- boolean tag query expressions;
- splitting the application into multiple Gradle modules.
