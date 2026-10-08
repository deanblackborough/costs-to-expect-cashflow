# Allocated transaction: income and expense

Status: planned 2026-10-08, decisions confirmed the same day, not started. This is the next task.

Each phase below is a stop point. Finish it, summarise, and wait for the go-ahead before the next one.

## Where things stand

- The API already has the `allocated-transaction` item type. `transaction_type` (`expense` | `income`) is **required** on create and update. It also has `publish_after` and `actualised_total` (total x percentage). Amounts are always positive. Summaries accept a `transaction_type` filter, but without it they add income and expense together.
- Cashflow lets you create a transaction resource type, but every form, action and total assumes `allocated-expense`. No `transaction_type` is sent, so adding an item gets a 422 from the API today. Nothing in the app knows about income.

## Behaviour we are building

- **Expense**: as today, straight to the API with `transaction_type=expense`.
- **Income** is a pipeline: `draft` -> `invoiced` -> `received`.
  - `draft` and `invoiced` live **only in Cashflow's database**. They never reach the API.
  - `late` is derived, not stored: status is `invoiced` and the expected pay date has passed.
  - Marking an income **received** posts it to the API as `transaction_type=income`. The received date becomes `effective_date`. From then on it counts in the main totals.
- **Income fields**: name, description, currency, expected pay date, optional category/subcategory, percentage allocations across resources (same as expenses), plus pricing in one of three modes:
  - `hourly`: rate x hours,
  - `daily`: rate x days, or
  - `fixed`: the user enters the total directly (e.g. an invoiced amount).
  
  The total is always stored, so sums never need to know the mode. Hours and days can be fractional (7.5 hours, half a day).
- **Pending income** is counted but kept as a separate figure: totals and counts for draft, invoiced and late. It is never added into the main totals.
- **Totals** for a transaction resource type show income, expenses and net (income - expenses). There is no "gross" figure.

## Decisions (confirmed 2026-10-08)

1. "Approved" and "marked received" are the same step.
2. Pending figures are **not** filtered by reporting period. They show what is outstanding now, so last tax year's late invoice still shows.
3. The main totals are income, expenses, then net. Net is worked out per currency, never across currencies.
4. A received income row stays in the local database (read-only) once pushed. That makes retry after a partial failure safe, and it keeps the rate/quantity/invoice dates, which the API has nowhere to store. Edits after receipt go through the normal API item edit.
5. Three pricing modes from the start: hourly (rate x hours), daily (rate x days) and fixed total. Fixed total is its own mode, not an override of the other two.
6. Recurring expenses will be dealt with later. They are still WIP and don't send `transaction_type`, so hide the Recurring entry points for transaction resource types and leave that code alone.
7. **Receiving** lets the user change the amount and the date. The dialog defaults to the invoiced total and today. The amount entered is what goes to the API. There are no partial payments: one income is received once.
8. **Expected pay date** is picked directly. It is required when an income is marked invoiced and optional on a draft. No payment-terms calculation.
9. **VAT/tax** is not handled yet and will be added later. Amounts are plain figures. Keep the table easy to extend (no assumptions baked into `total`).

## API changes (repo: costs-to-expect-api)

These are additive and don't break the existing item types. Only Phase 4 needs them, so they can run in parallel with Phases 1-3. Start them early, because they need deploying.

- **A. Summary grouped by transaction type.** Add a `transaction_types=true` summary parameter, on both the resource summary and the resource-type summary. It returns one row per currency and transaction type, `{currency, transaction_type, count, subtotal}`. Without it, Phase 4 needs two requests (income, expense) per window per scope, which doubles the pool sizes in `PeriodTotals`. The fallback is workable but costs more requests.
- **B. Verify and fix the date-window summaries for uncategorised items.** `AllocatedTransaction\Models\Summary::filteredSummary()` (and the allocated-expense twin) inner-joins `item_category` and `item_sub_category`. Read from the code, any `filter=effective_date:...` summary would drop items with no category or subcategory. Cashflow's period totals all use that filter, and categories can be turned off per resource type. Phase 0 starts with a failing API test to prove or disprove this. If it holds, switch to left joins whenever no category/subcategory filter is given. It may also be what the "empty filtered summary on a new resource" note in `Uri::itemsSummary()` is about.
- No API change is needed for the pending income itself. It stays local by design.

## Phases

### Phase 0: API groundwork (costs-to-expect-api)
- Failing test for B, then the fix.
- Parameter A on both summary endpoints, with the transformer and OPTIONS/docs updated, and tests.
- Check `transaction_type` can be changed through PATCH and that `include-unpublished` isn't needed.
- Deliver as an API PR. Cashflow doesn't change.

### Phase 1: Transaction resource types work for expenses (cashflow only)
- `ResourceType::isTransactional()` helper.
- Expense create/update send `transaction_type=expense` for transaction resource types. Edit keeps the existing type.
- Item lists show an income/expense badge. Hide Recurring for transaction resource types.
- Tests via `FakesTheApi`, with a transaction-typed resource type.
- Result: a transaction resource type is usable for expenses, with no new storage.

### Phase 2: Local pending income (cashflow only, nothing pushed yet)
- Migrations and models, mirroring the `RecurringExpense` and `RecurringExpenseAllocation` pattern:
  - `pending_incomes`: `resource_type_id`, name, description, `currency_id`, `pricing_type` (hourly|daily|fixed, a string column), `rate` and `quantity` (both nullable, so null for fixed; quantity means hours or days depending on the type), `total`, `expected_on`, `status` (draft|invoiced|received), `invoiced_on`, `received_on`, `received_total`, `category_id`, `subcategory_id`.
  - `pending_income_allocations`: `resource_id`, percentage, `sort_order`, nullable `api_item_id`. This is for Phase 3's retry.
- `isLate()` / `scopeLate()` derived from status and `expected_on`.
- Pricing: total = rate x quantity for hourly and daily, rounded half-up to 2dp, done in one place in an action class rather than in the controller. For fixed, the entered total is used as is.
- Validation: `expected_on` required to move to invoiced, optional on a draft.
- Routes, controllers and views under `resource-types/{resourceType}/income`: pipeline list grouped by state, create, edit, delete, "mark invoiced". Gated by a middleware like `EnsureCategoriesEnabled` so expense resource types can't reach them.
- Reuse the expense form components (split, details, amount). Add a pricing mode switch (hourly / daily / fixed) that shows rate + hours, rate + days, or just a total.
- Result: income can be drafted, invoiced and shown as late, entirely locally.

### Phase 3: Mark received -> API
- `MarkIncomeReceived` action. The user supplies the received amount and date, defaulting to the invoiced total and today (decision 7). Store them as `received_total` and `received_on`. For each allocation without an `api_item_id`: POST the item (`transaction_type=income`, `total` = received amount, `effective_date` = received date, with the allocation's `percentage`), save the returned id straight away, then assign the category. The row flips to `received` only once every allocation has an id. The retry must reuse the stored received amount and date, not read the form again.
- A retry after a partial failure resumes instead of duplicating. This matters because `CreateExpense` shows these POSTs aren't atomic.
- Extract the category-assignment helper out of `CreateExpense` so both share it.
- For hourly and daily income, add a line to the pushed description (e.g. `12.5h @ 40.00` or `3d @ 400.00`), since the API can't store the rate or quantity. If the received amount differs from the invoiced total, the line also says what was invoiced.
- Optional: "add income already received" on the create form, which skips the pending stage and posts straight through.
- Result: received income appears in the API and the existing item lists.

### Phase 4: Totals
- Pending totals use the invoiced `total`. Once received, the API's figure (the received amount) is what counts, so the item moves from pending to income and may change value by whatever the user adjusted.
- Extend `PeriodTotals` so a transaction resource type's entries carry `income`, `expense` and `net` per currency. Use API change A, or two requests per window if A hasn't landed. Keep the current `totals` shape for expense resource types.
- New `PendingIncomeTotals` service, local DB only so it adds no API requests. Per currency: draft, invoiced and late totals and counts. Per resource, it splits `total x percentage / 100` using the same rounding the API stores. For example, 33% of 100.01 must match what MySQL stores in `decimal(13,2)`.
- Dashboard hero and resource page: show income / expenses / net, and a separate "Pending income" figure with draft, invoiced and late counts, linking to the pipeline.
- Decide what the share bar means for transaction resource types, since `PeriodTotals::share()` assumes positive expense totals. Likely base it on income, or hide it.
- Run `bin/css` after the Blade changes.

### Phase 5: Hardening and copy
- Edge cases: multiple currencies, an allocation pointing at a deleted resource, the API down at the moment of receipt, the received date earlier than the invoiced date, a received amount of zero or less.
- README features and landing page copy ("money in and money out").
- Full test run.

## Risks

- The share bar and the headline figure in `PeriodTotals` assume expense-only data, so Phase 4 touches shared code. Keep the expense path's output identical, with tests pinning it.
- The API's rate limit is a consideration for Phase 4 if API change A isn't ready (see `config/app/api.php` on pool concurrency). The limit is being raised for specific accounts.
- Money maths currently uses floats with `number_format` at the edges. The rate x hours product and the allocation split should round explicitly, not rely on float formatting.
