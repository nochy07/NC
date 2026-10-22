# UniPlan – n8n setup guide

Two importable workflows (targeting n8n 1.x, recent versions – Google Sheets node v4.5, Switch/If v3.2/v2.2, Execute Workflow v1.1):

| File | Workflow name | Purpose |
|---|---|---|
| `uniplan-main-workflow.json` | UniPlan - Main Bot | Telegram Trigger → Edit Fields → Switch → validation (Code + If) → sub-workflow → Telegram replies |
| `uniplan-timetable-subworkflow.json` | UniPlan - Timetable Operations | Reusable sub-workflow: Sheets read/append/update, conflict detection, `/next` logic, activity logging |

> **Testing status:** the JSON files were checked for validity and internal consistency (unique node names, every connection target exists), and the Code-node logic was run locally against mock data (validation, conflicts, sorting, `/next` across days, cross-user isolation). They have **not** been imported into a live n8n instance or run against real Telegram/Google APIs. Do the smoke test at the bottom after setup.

## 1. Google Sheet
Create one spreadsheet with two tabs. Row 1 must contain exactly these headers:

- **Classes**: `class_id | user_id | class_code | class_description | day | start_time | end_time | room | teacher | credits`
- **ActivityLog**: `timestamp | user_id | action | status | details`

**Upgrading from the earlier version?** The old `class_name` column is gone. Easiest: create a fresh *Classes* tab with the headers above. To keep data, rename `class_name` → `class_description`, add `class_code`, `teacher` and `credits` columns, then re-import both workflows (the Sheets nodes' column lists changed).

Format all columns of both tabs as **Plain text** (Format → Number → Plain text) so IDs and `10:00` times are never reinterpreted. Copy the spreadsheet ID from its URL (`/d/<ID>/edit`).

## 2. Credentials (create in n8n, then select on nodes)
- **Telegram API**: create a bot with @BotFather (`/newbot`), paste the token into a Telegram credential. Optionally register the command menu with `/setcommands`.
- **Google Sheets OAuth2** (or service account shared with the sheet as Editor).
- **OpenAI API** (only for `/import`): used by the *Call Vision Model* HTTP Request node (predefined credential type *OpenAI*).

Placeholders to replace after import: `REPLACE_WITH_CREDENTIAL_ID`, `REPLACE_WITH_SPREADSHEET_ID` (every Google Sheets node, listed below), and `REPLACE_WITH_SUBWORKFLOW_ID`.

*Call Vision Model* needs the OpenAI credential; *Get Screenshot* and the other Telegram nodes need the Telegram credential.

Nodes needing the spreadsheet ID + Sheets credential: *Log Rejected Command* (main); *Get User Classes*, *Append Class*, *Update Class*, *Mark Class Dropped*, *Log Activity* (sub-workflow).

## 3. Import and link (order matters)
1. Import **UniPlan - Timetable Operations**, configure its Sheets nodes, **save**, and note its workflow ID (the number in its URL).
2. Import **UniPlan - Main Bot**, set Telegram credentials on all Telegram nodes, set the sheet on *Log Rejected Command*, and in *Run Timetable Operation* choose the sub-workflow (or paste its ID).
3. For `/import`: open *Prepare Vision Request* and set `const MODEL = '...'` to a vision-capable chat model available to your OpenAI account (it deliberately ships as a placeholder and the command fails with a clear error until you set it).
4. Activate **Main Bot** only (the sub-workflow does not need to be active). Telegram webhooks require an HTTPS public URL (`WEBHOOK_URL` env var when self-hosting). Only one active workflow can use a given bot token as a trigger.

## 4. Timezone for `/next`
Open the sub-workflow → *Process Operation* code node → edit the first line:
```js
const TIMEZONE = 'Europe/Paris';   // any IANA name, e.g. 'America/New_York'
```
This is the single authoritative setting for `/next` and is independent of server/instance timezone. (ActivityLog timestamps are always UTC ISO strings.) The reply states which timezone was used.

## 5. Behaviour summary
- Commands are matched exactly (case-insensitive, `/cmd@BotName` supported); anything else gets a helpful error. No AI/LLM nodes are used.
- `/add Code | Description | Day | Start | End | Room | Teacher | Credits`. **Required for planning:** a code *or* a description (or both), Day, Start, End. **Optional:** Room, Teacher, Credits – omit trailing fields, leave a slot empty (`| |`), or write `TBD`/`?`/`-`. Example with unknowns: `/add COMM 355 | Private Equity | Tue | 8:30AM | 10:00AM | | | 3`. After adding, the bot lists what is still unknown and how to fill it in with `/edit`.
- Times accept 24-hour (`14:30`) or 12-hour (`8:30AM`) and are stored as `HH:MM`. Credits must be a number such as `3` or `1.5`.
- `/add`, `/edit` and `/drop` are parsed and validated in the main workflow (missing required fields, bad day, bad times, end ≤ start, bad credits, bad ID format); failures are logged and answered without touching *Classes*.
- The sub-workflow reads only rows where `user_id` equals the requester (Sheets lookup filter **and** a second strict check in code). `/drop` can only match IDs inside that filtered set; another user's ID gives the same "not found" reply.
- Conflict check: same user, same day, `newStart < existingEnd && newEnd > existingStart` (back-to-back classes are allowed). Conflicts return a message and write nothing.
- `/edit ClassID | Field | NewValue` changes one field (`code`, `description`, `day`, `start`, `end`, `room`, `teacher`, `credits`). Optional fields can be cleared with `-`; a class must always keep a code or description. The change is validated like `/add` (valid day/time, end later than start) and re-checked for overlaps against the user's *other* classes; on any failure nothing is written. It only works on the requester's own class IDs.
- `/today`, `/tomorrow` and `/day Monday` (Mon–Sun accepted) list one day's classes sorted by time. "Today"/"tomorrow" use the `TIMEZONE` constant in *Process Operation*, same as `/next`.
- **`/import`** – send a screenshot (as a photo or image file) with `/import` as the caption. Flow: Telegram file download → *Prepare Vision Request* (base64 image + fixed extraction prompt) → OpenAI chat completion → *Parse Vision Response* → sub-workflow. The model is only used to read the image; it never chooses commands. Everything it returns is re-validated in code (day, times, end > start, lengths, credits), then each meeting goes through the same overlap check as `/add`, one row per day/time meeting. Rules applied: session type, section numbers and meeting-date ranges are ignored; `RM 101` → `RM101`; repeated teacher/room text is de-duplicated; units like `3.00` → `3`; `COMM 324 Investments & Portfolio Mgmt.` is split into code and description. Missing room/teacher/credits are left blank and reported. Meetings that clash with existing classes (including re-importing the same screenshot) are skipped and listed.
  Optional caption overrides: `/import COMM 355 | Private Equity | 3` – used when the image contains exactly one class (useful when the screenshot shows no course code or units).
- `/schedule` sorts by day then start time; `/next` handles "later today", "tomorrow", later days and wrapping to next week, and shows a class currently in progress.
- ActivityLog stores only: UTC timestamp, user ID, action, status and a short detail (e.g. `class_id=…`, `overlaps=…`, a validation reason). Message text and class names are never logged.

## 6. Reliability design
- **Retries**: Retry On Fail (3 tries, 2 s apart) on all Sheets and Telegram nodes. Both *Append Class* and *Update Class* match on `class_id`, so retried writes are idempotent.
- **No duplicate writes**: the class ID is generated *once* in code, and *Append Class* uses **Append or Update matching `class_id`**, so a retried write updates the same row instead of adding another.
- **`/drop` = tombstone update**, not row deletion. Deleting by row index is unsafe with retries and concurrent users (rows shift, so a retry could delete a different row). Instead the row is overwritten (`user_id = __dropped__`, other fields `-`), which is idempotent and wipes the private data. Periodically filter `user_id = __dropped__` in the sheet and delete those rows manually if you want a tidy sheet.
- **Logging is best-effort**: log nodes use *Continue* on error so a logging failure never blocks the user's reply. Because the ActivityLog has no unique key, a retried log append can, rarely, produce a duplicate log row.
- **User-facing failures**: if the sub-workflow fails (e.g. Sheets outage), the user receives a generic "something went wrong" message, then a *Stop and Error* node marks the execution as failed so it shows up in n8n's Executions list. There is no separate error workflow; check the Executions list for failures.

## 7. `/import` privacy and accuracy
- The screenshot is sent to OpenAI for analysis. Tell users, and avoid screenshots containing other personal data (the bot's `/help` mentions that an AI model reads the image).
- Text inside an image could try to instruct the model; the output is treated purely as data and validated, so the worst case is wrong class rows for that same user.
- Vision models can misread small text. The reply lists every imported row with its ID so it can be corrected with `/edit` or `/drop`.
- Limits: one image per command (if you send several photos at once only the one with the caption is processed), 8 MB max, up to 10 classes / 20 meetings per import, 100 classes per user in total.
- `/import` is a single opt-in command; all other commands remain fully deterministic.

## 8. Known limitations
- Term date ranges (e.g. "08/09/2026 – 08/12/2026" in a university portal) are not stored; every class repeats weekly indefinitely.
- `/start` and `/help` replies are not logged (no data operation).
- Limit of 100 classes per user (constant `MAX_CLASSES_PER_USER` in *Process Operation*); `/schedule` output is truncated at ~3,900 characters (Telegram's 4,096 limit).
- Two simultaneous `/add` commands from the same user could both pass the conflict check; Google Sheets has no transactions.
- Google Sheets API quotas apply (reads scan the Classes tab filtered by user; fine for a class-sized dataset, not for thousands of users).
- If your n8n version is older and a node rejects imported parameters, re-select the operation in that node's UI; the parameter mapping is straightforward.

## 9. Smoke test
1. `/start`, `/help` → text replies.
2. `/add Databases | Monday | 10:00 | 12:00 | B204` → ✅ with ID; row appears in *Classes*.
3. Repeat it → conflict message, no new row.
4. `/add X | Funday | 10:00 | 12:00 | R`, `/add X | Monday | 12:00 | 10:00 | R`, `/add` → validation errors.
5. `/add COMM 355 | Private Equity | Tue | 8:30AM | 10:00AM` (no room/teacher/credits), `/edit <ID> | teacher | Libitz,Greg`, `/edit <ID> | teacher | -` (confirm the cell is actually cleared in the sheet), `/edit <ID> | room | B305`, `/edit` with an overlapping time (should be rejected), `/today`, `/tomorrow`, `/day Fri`, `/schedule`, `/next`, `/drop <ID>`, `/drop <ID>` again (not found), `/foo` (unknown).
6. Check *ActivityLog* rows. Use a second Telegram account to confirm it cannot see or drop the first account's classes.
7. `/import`: send a timetable screenshot with caption `/import` → expect one row per meeting; send `/import` with no image → usage message; re-send the same screenshot → "Nothing was imported" (overlaps).
8. Temporarily break the sheet ID in the sub-workflow to confirm the failure notice and the failed execution in the Executions list.
