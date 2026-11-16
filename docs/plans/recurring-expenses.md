# Recurring expenses: posted when you sign in

Status: planned 2026-11-16, decisions confirmed the same day, not started. Allocated-expense only; allocated-transaction recurring builds on this later (see "Later").

Each phase below is a stop point. Finish it, summarise, and wait for the go-ahead before the next one.

## Where things stand

- Cashflow already has recurring expense templates (`recurring_expenses`, `recurring_expense_allocations`, `recurring_expense_runs`), CRUD screens and a Recurring page. The template is defined and edited only in Cashflow, which is what we want.
- Posting is the part we're replacing. `expense:process-recurring` runs **daily from the scheduler** with a long-lived `API_SERVICE_TOKEN`. It posts **one** occurrence per template per run, so a two-month gap would take two days of scheduler runs to clear. It also has to be trusted with a service token that can write anywhere.
- The `runs` row is only written **after** every allocation has posted. `CreateExpense` POSTs once per allocation and isn't atomic, so a failure part-way through a split followed by a retry would **duplicate** the items that already went through.
- "Disable" today is a Pause/Resume toggle (`active`) plus a hard delete that also deletes the run history.
- Auth: the bearer token lives in an encrypted cookie (`cte_bearer`); there's no local users table. `Guard::attempt()` has the token at sign-in. API tokens don't expire (`costs-to-expect-api/config/sanctum.php:50`, `expiration => null`), and sign-out only forgets the cookies, so a token stays valid until it's revoked at the API.
- Queue: `jobs`, `job_batches` and `failed_jobs` tables already exist. `.env.example` has `QUEUE_CONNECTION=sync` and `docker-compose.yml` has no worker.
- Recurring is reachable from every resource type today, including transaction ones, where posting would 422 (no `transaction_type`).

## Behaviour we are building

- The **definition lives only in Cashflow**: name, description, currency, amount, optional category/subcategory, percentage allocations across resources, day of month, start date, optional end date. Monthly only.
- **Nothing posts on a schedule.** Posting happens in background jobs that are queued for a signed-in user and **use that user's current bearer token**. The service token and the scheduled command go away.
- **Catch-up**: when you sign in after a gap, every occurrence that fell due while you were away is posted, each with `effective_date` set to **its own due date**. Example: phone insurance, 5.99, on the 5th, no sign-in for two months: the next sign-in posts the missed 5th of each month plus today's 5th if it has arrived, oldest first.
- **When it's queued** for a user:
  - on sign-in, and
  - on the first authenticated page load of each day (tracked in the session, so no extra queries on normal requests), because the remember-me cookie lasts 30 days and a user may not do a fresh sign-in for weeks, and
  - straight after creating or restoring a template that has something due (see Phase 2).
- **Disabled = soft-deleted.** The Recurring page lists what's active. Disabled ones are in a separate "Disabled" list, and can be restored from there. There is no Pause/Resume.
- A job only touches templates of resource types **the signed-in user is permitted to** (per the API, `permittedResourceTypes()`), because that is all their token can write to.

## Decisions (confirmed 2026-11-16)

1. **Trigger**: sign-in **and** the first authenticated request each day.
2. **Disable model**: soft-delete only. The `active` flag and Pause/Resume are removed. Existing paused rows become soft-deleted (so they stay out of the active list and are restorable).
3. **Restore**: skips to the next future occurrence from today. Nothing is back-posted for the time it was disabled.
4. **Past start date on create**: backfills. The first run is the first occurrence on or after `starts_on`, so created on 16 Nov with a 5 Sept start, Sept/Oct/Nov are posted by the catch-up path.
5. Edits apply to occurrences that haven't posted yet. Already-posted items are API items and aren't touched.

Defaults I've chosen that you can change: the backfill limit is **24 months** (a start date further back than that fails validation, so a typo of 2016 can't post 120 expenses), and a single job posts at most **24 occurrences** per template, leaving the rest for the next trigger.

## Design

### The token never leaves the job payload

- The jobs implement `ShouldBeEncrypted`, so the bearer token is encrypted in the `jobs` row (and in `failed_jobs`) with `APP_KEY`, not stored in plaintext.
- The token is only held in a job's constructor argument. It is never written to a model, the cache, the session store or a log, and exception reports must not include the payload.
- Housekeeping prunes `failed_jobs` and `job_batches` after a day or two.

### Jobs

- `PostDueRecurringExpenses(userId, bearer)`, the coordinator. `ShouldBeUnique` by user id (a few minutes), so two tabs or two triggers don't double up. It fetches the user's permitted resource types, finds templates due (`next_run_date <= today`, expense resource types only), and dispatches a `Bus::batch` of one job per template. A batch is used so a revoked token can cancel the rest, and so `allowFailures` keeps one bad template from blocking the others.
- `PostRecurringExpense(recurringExpenseId, bearer)`, one per template, `Batchable`, with `WithoutOverlapping` keyed by template id (two users on a shared resource type can both trigger). It loops oldest due occurrence first while `next_run_date <= min(today, ends_on)`: post one occurrence, advance `next_run_date`, repeat. It stops at the first failure for that template so occurrences always post in order.
- A timeout shorter than the database queue's `retry_after` (90s in `config/queue.php`), so a slow job can't be picked up a second time while it's still running.

### Exactly-once posting

The unit of work is **one occurrence** (`recurring_expense_id` + `run_date`, already unique in the schema).

1. **Claim** the run row first (`firstOrCreate` on the unique key), before any POST. A completed run is skipped.
2. For each allocation without an `api_item_id`: POST the item, then **save the returned id immediately**.
3. Assign the category/subcategory if the resource type has categories on and the item isn't marked categorised. Categorisation failures **fail the run** here (unlike a manual expense, nobody is watching this one), and a retry only redoes the categorisation.
4. When every allocation has an id and is categorised, set `completed_at` and advance `next_run_date`.

Per-allocation state goes in a small child table (`recurring_expense_run_items`: `run_id`, `resource_id`, `api_item_id`, `categorised_at`, unique on run + resource). It replaces the `created_item_ids` JSON, and it matches the `api_item_id` retry pattern already planned for pending income in `allocated-transaction.md`.

### Failure handling (retry is always safe because of the above)

| Outcome | What happens |
|---|---|
| 401/403 from the API | Token no longer valid. Cancel the batch, record nothing against any template (it isn't the template's fault). The next trigger retries. |
| 429 | `release()` with backoff. The API's limit is ~60 a minute and a catch-up with splits and categories makes several requests per occurrence (POST per allocation, plus 1 or 2 for the category). Posts can't be pooled (GET/HEAD only), so they go one at a time. |
| Connection error / 5xx | `release()` with backoff, a few tries, then give up until the next trigger. |
| 422 / 404 (data problem: deleted resource, deleted category, bad amount) | Store a short human-readable `last_error` and `last_failed_at` on the template, stop that template, carry on with the others. Shown on the Recurring page until the next successful run, and cleared on success. |

### Dispatcher

One `QueueDueRecurringExpenses` action, called from three places: `SignIn`, a middleware on the authenticated route group (session flag for "already checked today"), and the create/restore actions. It does a cheap local `exists` check for anything due before dispatching, so it never queues empty jobs. `Guard` needs to expose the token and user id after a successful `attempt()`, since the cookie it queues isn't readable in the same request.

## Phases

### Phase 1: Posting engine and triggers (replaces the command)
- Migration: `recurring_expense_runs.completed_at`, the `recurring_expense_run_items` table, `recurring_expenses.last_error` / `last_failed_at`. Backfill existing runs (`completed_at = created_at`, items from `created_item_ids`), then drop the JSON column.
- Extract the item payload and category-assignment code out of `CreateExpense` into shared helpers, so manual and recurring posting can't drift. The manual path's behaviour doesn't change, and `allocated-transaction.md` Phase 3 wants the same extraction, so whichever lands first does it.
- `PostRecurringRun` action (the exactly-once unit above), the two jobs, the dispatcher, `Guard` token accessors, the `SignIn` hook and the daily middleware.
- Remove `ProcessRecurringExpenses`, the `Schedule::command` line, `app.api.service_token`, `API_SERVICE_TOKEN` in `.env.example` and its README row. Delete `ProcessRecurringExpensesCategoriesTest` and replace it (below).
- Pause/Resume and hard delete are untouched in this phase, so the engine keeps filtering on `active` until Phase 2.
- Tests (`Http::fake`, `FakesTheApi`): a two-month gap posts the missed dates oldest first and leaves `next_run_date` in the future; re-running the job posts nothing new; a split that fails on the second allocation resumes with only the missing POST; a category failure retries only the category; 401 cancels and leaves templates clean; 422 records `last_error` and doesn't block another template; 429 releases; a resource type the user isn't permitted for is skipped; the daily trigger fires once per day per session and not at all when nothing is due; the jobs implement `ShouldBeEncrypted`.
- Result: due recurring expenses, including every missed month, post in the background with the signed-in user's token. The UI is unchanged.

### Phase 2: Disable/restore, backfill, expense-only
- Migration: `deleted_at`, then map `active = false` rows to soft-deleted and drop `active`. `SoftDeletes` on `RecurringExpense`; the engine's `active` filter becomes the default soft-delete scope. Ended templates (`next_run_date` past `ends_on`) stay in the active list and show as ended.
- `DisableRecurringExpense` (replaces `DeleteRecurringExpense`) and `RestoreRecurringExpense` (restored `next_run_date` = first occurrence from today). Remove `ToggleRecurringExpense` and the `recurring.toggle` route. Add `recurring.restore` with `->withTrashed()`. Edit/update on a disabled template 404s.
- `NextRunDate` takes an explicit `$from`: create passes `starts_on` (backfill), update (when the day or start date changed) and restore pass `max(today, starts_on)`. Validation: `starts_on` no more than 24 months back on create.
- Creating (and restoring) with something due dispatches immediately with the request's own token, with a flash such as "3 earlier months will be added shortly".
- Expense-only: `ResourceType::tracksExpenses()`, a middleware on the recurring routes (404 for transaction resource types), and the Recurring links hidden for them (nav in `layouts/app.blade.php`, the dashboard and resource-page buttons). `allocated-transaction.md` Phase 1 also hides these links, so whichever lands first does it.
- Tests: soft delete/restore, the paused-to-disabled migration, restore skips missed months, backfill creates the right occurrences, the 24-month limit, transaction resource types 404.
- Result: you can disable and restore, past start dates backfill, and transaction resource types don't see the feature.

### Phase 3: Recurring section
- Active list: name, split, amount, and a status line: next run date; "Due, posting shortly" while `next_run_date <= today`; "Couldn't post: <last_error>" (with "will retry next time you're signed in"); "Ended <date>". Show when it last posted (latest completed run).
- "Disabled" list below, a `<details>`, with Restore.
- Hero/empty-state copy says they're added automatically when you sign in.
- Run `bin/css` after the Blade changes. Update `PooledFormPagesTest` / page tests for the new list.
- Result: you can see at a glance what's active, what's due, what failed and why, and get disabled ones back.

### Phase 4: Hardening and docs
- Scheduler entries for `queue:prune-failed` and `queue:prune-batches`. This is the scheduler's only remaining job and it's housekeeping, not posting.
- Edge-case tests: day 31 into February and leap years, `ends_on` falling inside a gap, editing the amount or allocations while a run is pending, two users on a shared resource type, API down mid-split, the 24-occurrence cap continuing on the next trigger, the app's timezone (UTC) "today".
- README: replace the recurring paragraphs, the `API_SERVICE_TOKEN` row, and the testing sentence that mentions recurring date math. Document the queue worker.
- Full `composer test` run and Pint.
- Result: ready to deploy.

## Deployment prerequisites (you)

- Production needs `QUEUE_CONNECTION=database` and a running worker (a Forge daemon: `php artisan queue:work`, with `queue:restart` in the deploy script). Without a worker the jobs queue and never run.
- `.env.example` stays on `sync` so local dev posts inline at sign-in (simple to test, but sign-in will wait for the posts). If you'd rather mirror prod locally, run `docker compose exec app php artisan queue:work`, there's no worker container today.
- Remove `API_SERVICE_TOKEN` from the server's `.env` after Phase 1 is deployed.

## Risks

- **Totals lag the sign-in.** The dashboard loads before the background posts finish, so a first load after a long gap may be missing the catch-up until a refresh. The "Due, posting shortly" state in Phase 3 is the visible hint.
- **API rate limit.** A long gap with several split, categorised templates can hit 60 a minute. Backoff and resume-safe retries cover it, but a user may see recurring items trickle in over several minutes.
- **A permanently failing template blocks only itself**, and stays blocked until you edit it (for example a deleted resource in its split). That's deliberate, to keep occurrences in order, and the failure is shown on the Recurring page.
- **No user signs in, nothing posts.** That's the design. Months can be skipped for a long absence beyond the 24-occurrence cap, but not lost, because they're picked up on the following triggers.
- A user on a shared resource type will post for the other user's templates, using their own token. That is intended: the API permission check is the gate, not `resource_types.user_id` (which is just whoever first linked it).

## Later

- **Allocated-transaction recurring**: add `transaction_type` to the template and to the posting payload, and lift the expense-only gate. The jobs, claiming, per-allocation tracking and failure handling stay as they are.
- Other frequencies (weekly, yearly), a "delete permanently" on the Disabled list, and emailing on repeated failure are out of scope.
