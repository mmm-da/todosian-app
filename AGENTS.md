# Working with the Todosian task-note fork as an AI agent

This fork is an Android task app backed by Markdown notes in an Obsidian vault (or any folder synced to the phone). One eligible Markdown file is one task, its parent folder is its list, and the app operates with zero network permissions.

## Project Structure

The current code still contains the original line-based implementation while the migration in `docs/ROADMAP.md` is in progress. Keep one `:app` module and move code toward this structure only as each phase needs it:

```
Todosian/
├── app/                         # Main application module
│   ├── src/main/
│   │   ├── AndroidManifest.xml
│   │   ├── java/com/isotjs/todosian/
│   │   │   ├── data/
│   │   │   │   ├── model/        # Task note, summary and list models
│   │   │   │   ├── storage/      # SAF operations and task-note codec
│   │   │   │   ├── index/        # Room index and reconciliation
│   │   │   │   ├── sync/         # Pending writes and workers
│   │   │   │   └── settings/
│   │   │   ├── datetime/        # Offline RU/EN date-time parser
│   │   │   ├── notifications/   # Due reminders (AlarmManager)
│   │   │   ├── ui/
│   │   │   │   ├── tasklist/
│   │   │   │   ├── editor/
│   │   │   │   ├── components/
│   │   │   │   ├── home/
│   │   │   │   ├── onboarding/
│   │   │   │   ├── settings/
│   │   │   │   ├── theme/
│   │   │   │   └── widget/
│   │   └── res/                  # Android resources
│   └── build.gradle.kts
├── docs/                        # Storage contract and implementation roadmap
├── fastlane/                    # F-Droid metadata (descriptions, changelogs, screenshots)
├── gradle/                      # Gradle wrapper and version catalog
├── build.gradle.kts
└── settings.gradle.kts
```

## Core Architecture Principles

1. **Files are the committed source of truth**: Markdown files contain committed task data. A persistent local index is allowed as a rebuildable projection, and durable drafts/pending writes preserve local user work until it reaches SAF.
2. **Privacy-first**: No network permissions, no analytics, no telemetry. The app only accesses the folder selected by the user via Storage Access Framework (SAF).
3. **Document integrity**: Patch only the explicitly changed filename, owned frontmatter fields, or body. Preserve untouched content, unknown frontmatter, whitespace, encoding, and line endings.
4. **Obsidian compatibility**: Store task metadata in flat YAML frontmatter. The task title is the filename, its comment is the entire Markdown body, and task tags come only from inline tags in that body.
5. **Fast local UI**: Screens and widgets read indexed projections. File reconciliation and write-back run off the UI thread and survive process death.
6. **Small architecture**: Keep one application module and use `java.time`, coroutines, SAF, Room, WorkManager, and AlarmManager directly. Do not add layers or dependencies without a current caller.

Note: the app works with any folder you pick via SAF. Many users point it at an Obsidian vault or a synced subfolder, but Obsidian itself is not required.

## Rules for working on the project

1. Always create a new branch for feature work.
2. Branch naming conventions: `fix/...`, `feat/...`, `ref/...`, `docs/...`, `chore/...`.
3. Keep branch descriptions concise and specific.
4. Keep your branch up to date with `main` to minimize conflicts.
5. Commit message format: `type(scope): short description` (scope optional).
6. All user-facing strings go in `app/src/main/res/values/strings.xml`.
7. Do not edit non-English translations unless you are fluent in that language.
8. Follow Kotlin + Android best practices.

## AI-only guidelines

1. You are NOT allowed to use the following commands:
   - You are not to commit, push, or merge any changes to any branch.
   - You should absolutely NOT use any commands that would modify the git history, do force pushes (except for rebases on your own branch), or delete branches without explicit instructions from a human.
2. Always follow the guidelines and instructions provided by human contributors.
3. Ensure the absolutely highest code quality in all contributions, including proper formatting, clear variable naming, and comprehensive comments where necessary.
4. Comments should be added only for complex logic or non-obvious code. Avoid redundant comments that simply restate what the code does.
5. Prioritize performance, battery efficiency, and maintainability in all code contributions. Always consider the impact of your changes on the overall user experience and app performance.
6. If you have any doubts ask a human contributor. Never make assumptions about the requirements or implementation details without clarification.
7. If you do not test your changes using the instructions in the next section, you will be faced with reprimands from human contributors and may be asked to redo your work. Always ensure that you test your changes thoroughly before asking for a final review.
8. You are absolutely **not allowed to bump the version** of the app in ANY way. Version bumps are only done by the core development team after manual review.

## Building and testing your changes

1. After making changes to the code, you should build the app to ensure that there are no compilation errors. Use the following commands from the root directory of the project:

```bash
./gradlew :app:assembleDebug
```

2. If the build is not successful, review the error messages, fix the issues in your code, and try building again.

3. Run unit tests (especially important for Markdown parsing changes):

```bash
./gradlew :app:testDebugUnitTest
```

4. Run linter to ensure code quality:

```bash
./gradlew :app:lint
```

5. Once the build is successful, you can test your changes on an emulator or a physical device. Install the generated APK located at `app/build/outputs/apk/debug/` and ask a human for help testing the specific features you worked on.

## Task Document Contract

Follow the full contract and migration sequence in `docs/ROADMAP.md`.

1. One eligible Markdown file represents one task.
2. The task title is the filename without `.md`; changing the title renames the file.
3. The task comment is the entire Markdown body after frontmatter. Do not create managed comment sections or headings.
4. Extract task tags only from inline tags in the body. Ignore frontmatter `tags` for task selection and display, but preserve them.
5. A note is a task when its body contains at least one configured task tag.
6. Store task ID, status, priority, dates, and recurrence as flat, app-owned frontmatter fields.
7. Do not store a separate task title in frontmatter or infer it from a Markdown heading.

### Document Integrity

1. Preserve untouched body and frontmatter regions character-for-character, including unknown fields, comments, whitespace, encoding, and line endings.
2. A metadata edit patches only owned frontmatter fields. A comment edit replaces only the body. A title edit only renames the document.
3. Do not regenerate a document from its indexed summary.
4. Read and write UTF-8. Detect and preserve existing line endings.
5. Re-read the target document before applying a pending write so external edits are not silently overwritten.
6. Merge independent changes; report conflicting changes to the same filename, field, or body.
7. Do not assume SAF providers support atomic replacement or keep the same URI after rename/move. Verify writes and retain enough pending state to retry safely.

### Tags

1. Extract tags with their source ranges so tag edits do not rebuild the body.
2. Ignore tag-like text inside fenced code blocks and inline code.
3. Compare tags case-insensitively and support Obsidian nested tags containing `/`.
4. Recompute tags, widget preview, and task eligibility after a body edit.
5. Removing the last configured task tag removes the note from task views; it does not delete the file.

### Testing Markdown Logic

When modifying document parsing or writing logic:

1. Add unit tests in `app/src/test/java/`.
2. Cover empty and malformed frontmatter, an empty body, Unicode, long lines, CRLF/LF, inline and fenced code, nested tags, and unknown YAML fields.
3. Verify untouched regions character-for-character.
4. Verify metadata writes preserve the body and body writes preserve unrelated frontmatter.

## Local Index and Pending Writes

1. Screens, notifications, and widgets read the local index instead of scanning SAF on each render.
2. Committed index rows are rebuildable from Markdown. Drafts and pending writes are not and must survive reconciliation or index rebuilds.
3. Persist a desired-state command and its optimistic projection update in one Room transaction before scheduling work.
4. Commands must be idempotent. Use `SetTaskStatus(done)` instead of `ToggleTask`.
5. Resume pending work after process death, including failure between the Room transaction and WorkManager enqueue.
6. Do not let completion of an older write overwrite a newer local version.
7. Content observers and provider timestamps are refresh hints, not correctness guarantees. Reconcile on startup, manual refresh, and background work.

## Widgets and Note Editor

1. Widget rendering reads indexed summaries and performs no direct SAF reads.
2. Widget actions submit durable desired-state commands through the same write path as the app.
3. Open a dedicated note editor activity from widgets using the stable task ID, not a filename, line number, or mutable URI.
4. The main app and editor activity share the same editor UI and state holder; do not duplicate editing logic.
5. The editor treats the filename as title and the complete Markdown body as one comment.

## Natural-Language Date and Time

1. Parsing is offline and deterministic; do not add network or ML dependencies.
2. Use `java.time` and pass `Clock`, `ZoneId`, and language explicitly.
3. Support only the RU/EN grammar documented in `docs/ROADMAP.md`; extend it from tests for real phrases.
4. Return the matched source range and distinguish date-only from date-time values.
5. Preview the parsed result and require user confirmation before changing task metadata or removing words from input.

## Storage Access Framework (SAF) Testing

When modifying file access logic:

1. Test with different folder locations:
   - Internal storage
   - SD card
   - Cloud storage providers (Google Drive, Dropbox mounted via SAF)
2. Test permission persistence across app restarts
3. Test graceful handling of:
   - Folder deleted externally
   - Permissions revoked
   - Files modified by external apps (Obsidian, text editors)
4. Test concurrent access scenarios (e.g., Syncthing syncing while app is open)

## Categories, File Selection, and Naming

1. Every folder under the selected root is a list; a task belongs to its immediate parent folder.
2. Only `.md` files whose body contains a configured task tag are indexed as tasks.
3. Files whose name contains `sync-conflict` are ignored.
4. The filename without `.md` is the task title. Sanitize invalid filename characters and reject collisions before rename/create.
5. Keep task identity stable when a provider changes the document URI after rename or move.

## F-Droid Publishing

When preparing for a release:

1. **Do NOT bump version numbers** - this is done by core maintainers only
2. Update changelogs in `fastlane/metadata/android/en-US/changelogs/<versionCode>.txt`
3. Ensure no proprietary dependencies are introduced
4. Verify that no network permissions are added (check `AndroidManifest.xml`)
5. Test that the app builds reproducibly:
   ```bash
   ./gradlew clean
   ./gradlew :app:assembleRelease
   ```
6. Metadata updates (screenshots, descriptions) go to `fastlane/metadata/android/`

## Notifications & Due Reminders

1. The app uses `AlarmManager.setAndAllowWhileIdle()` to schedule a single daily reminder at a user-configurable time (default 19:00). A `BroadcastReceiver` (`DueReminderReceiver`) handles the alarm and `BOOT_COMPLETED` to re-arm the alarm after reboot.
2. During migration the notifier still scans files. Its target implementation queries the local index for due-soon tasks and posts a single notification per day (deduplicated by signature).
3. Reminder enablement and time are independent settings; do not couple reminders to legacy Tasks plugin support.
4. Android 13+ requires `POST_NOTIFICATIONS` to show reminders; `RECEIVE_BOOT_COMPLETED` is required to re-arm the alarm after reboot.
5. No network permission is used.
6. Do not switch reminders back to WorkManager periodic work: periodic work is inexact and does not honor a fixed time of day, which caused reminders to not fire in the evening.

## Resource Management

### String Resources

All user-facing strings must be in `app/src/main/res/values/strings.xml`:

```xml
<string name="example_key">Example text</string>
```

Access in code:

```kotlin
// In Compose
Text(text = stringResource(R.string.example_key))

// In non-Compose code
context.getString(R.string.example_key)
```

### Resource Access in Compose

When accessing resources in Jetpack Compose, **do not** use `LocalContext.current` to query resource values (e.g., `context.getString()`, `context.getColor()`, `context.resources.getDrawable()`). Changes to the Configuration object will not invalidate `LocalContext.current` reads, leading to stale values when the system configuration changes (e.g., locale, theme, orientation).

**Always use Compose resource APIs instead:**

- `stringResource()` for strings
- `colorResource()` for colors
- `painterResource()` for drawables/images

If you need direct `Resources` access, use `LocalResources.current` instead of `LocalContext.current.resources`.

If you need to resolve strings outside of a composable (for example, inside a callback), prefer passing resource IDs through events and resolving them in the UI layer, or ensure the lookup is configuration-safe.

This prevents the lint error:

```
Issue id: LocalContextGetResourceValueCall
```

## Settings & Preferences

If you need to add a new preference:

1. Define it in `data/settings/`.
2. Add UI in the settings screen and a user-facing string resource.
3. Ensure it persists correctly and consider migration from the existing preference structure.
4. Task-tag settings contain one or more normalized body tags; first-version matching succeeds when any configured tag is present.
5. Reminder enablement is separate from tag parsing and legacy Tasks plugin settings.
6. Reconcile the index after task-tag or selected-root changes without deleting drafts or pending writes.

## Common Pitfalls

1. **Do not load entire files into memory**: For large Markdown files, use streaming or line-by-line processing.

Current implementation note:
The current SAF reader uses `readLines()` and loads the whole document into memory. Keep an eye on very large Markdown files when changing parsing or I/O behavior.

2. **Do not assume file structure**: Users may have arbitrary nested folders and file naming conventions.
3. **Do not treat the index as proof that files are unchanged**: Re-read before write and reconcile external changes.
4. **Do not block the UI thread**: File I/O and reconciliation must run on background dispatchers.
5. **Do not introduce dependencies** that require network permissions, analytics, crash reporting, or closed-source libraries.
6. **Do not add architecture for future features**: Keep one module and add a boundary only for a current caller or testability need.

## Testing Checklist

Before submitting your work for review:

- [ ] Code builds successfully (`./gradlew :app:assembleDebug`)
- [ ] Unit tests pass (`./gradlew :app:testDebugUnitTest`)
- [ ] Linter passes with no errors (`./gradlew :app:lint`)
- [ ] Tested on emulator or physical device
- [ ] Verified that metadata changes preserve the body and unknown frontmatter
- [ ] Verified that body changes preserve unrelated frontmatter
- [ ] Verified pending writes recover after process death and retries are idempotent
- [ ] Verified widget rendering and actions do not scan SAF
- [ ] Verified frontmatter tags do not affect task selection
- [ ] Verified that the app works without network permissions
- [ ] Verified that the change doesn't break Obsidian compatibility
- [ ] Added/updated unit tests for any Markdown parsing changes
- [ ] Updated strings.xml if any user-facing text was added
- [ ] Followed Kotlin code style and Android best practices

## Build Notes (Current)

1. Module: `:app`
2. `minSdk`: 30
3. `targetSdk`: 37
4. Compose UI (Material 3)
5. Java source/target compatibility is set in Gradle (see `app/build.gradle.kts`).

## Questions?

If you have any questions or need clarification, ask a human contributor before proceeding. It's better to ask than to make incorrect assumptions.
