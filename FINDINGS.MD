# FINDINGS

## Bugs I found and fixed

### 1. XSS via `v-html` on server-supplied description
- **Severity:** Critical
- **Who it hurts and how:** Every ops user with the page open, and every system they are authenticated into. `<td v-html="loan.description">` renders raw HTML from the API. The seed data on ln_05 already contains a working XSS payload (`<img src=x onerror="document.title='xss'">`). In production a compromised loan record could execute arbitrary JavaScript in an authenticated session — exfiltrating tokens, impersonating actions, pivoting laterally.
- **Cause:** `v-html` renders the value as HTML instead of text.
- **My fix:** Replace `v-html` with `{{ loan.description }}`. Vue escapes the value; the HTML is displayed as literal text.

---

### 2. Wrong customer shown in modal (index-based row identity)
- **Severity:** High
- **Who it hurts and how:** Any ops user drilling into a loan record — they may be reading and acting on the wrong customer's data. Ops confirmed this happened more often after searching, which is exactly when it would be worst.
- **Cause:** `expandedRow` stored the loop index `i`. After a search or filter change, `loans.value` was replaced with a new array and the saved index mapped to a completely different loan.
- **My fix:** Replaced `expandedRow` with a `selectedLoan` ref that stores the loan object directly. Row click now calls `openModal(loan)` which sets `selectedLoan.value = loan` — the actual reactive object, not an index. `:key` changed from `i` to `loan.id`. Modal closes on refresh since `loadLoans` resets `selectedLoan` to null when the loan list is replaced.

---

### 3. Outstanding total: wrong value and wrong format
- **Severity:** High
- **Who it hurts and how:** Finance and ops — the headline figure used to track portfolio exposure was both incorrect and formatted differently from every other amount on screen. Finance flagged the correct figure as ₦3,511,152.18 while the screen showed something else entirely. Decisions about portfolio exposure were being made on wrong data.
- **Cause (value):** `totalOutstanding` summed `loans.value`, the currently filtered list. Searching or filtering silently dropped loans from the total. The component already fetches `/api/customers/:id/summary` whose `outstanding_kobo` is the authoritative server-side figure.
- **Cause (format):** `total.toFixed(2)` returns a string. `formatNaira(v)` calls `v.toLocaleString()` — on a `String`, `toLocaleString()` is inherited from `Object` and returns the string unchanged with no thousands separators.
- **My fix:** `totalOutstanding` returns `(summary.value?.outstanding_kobo ?? 0) / 100` — a number from the server's authoritative integer. Also destructured `refresh: refreshSummary` from `useFetch` and call it inside `loadLoans` so the total stays in sync after every refresh and payment.

---

### 4. Memory leak and endless schedule fetching
- **Severity:** High
- **Who it hurts and how:** Ops users who leave the repayments screen open all day. Stacked intervals fired `loadLoans` continuously, each triggering 25 serial schedule requests — this was the sluggishness and fan noise ops reported. Even making the schedule fetch lazy would not have been enough on its own — the interval was still hammering the loans endpoint every 5 seconds all day, which is what kept the fan running and caused the overheating.
- **Cause:** `setInterval(loadLoans, 5000)` was called inside `onMounted` but the handle was never saved, so `clearInterval` could never be called. The interval ran forever. On top of that, schedule was fetched serially for all 25 loans inside every `loadLoans` call.
- **My fix:** Removed `setInterval` entirely and replaced with a manual Refresh button. Removed `handleResize` and `isNarrow` — dead code never used in the template. Schedule is now fetched lazily in `openModal` only when the user clicks a row, cached so it never re-fetches the same loan. I confirmed by watching the Chrome DevTools network tab and seeing requests firing continuously without any user interaction.

---

### 5. `overdueCount` includes paid loans with past due dates
- **Severity:** Low
- **Who it hurts and how:** Ops leads reading the headline overdue count — it was always inflated by paid loans whose due dates happened to be in the past.
- **Cause:** `loans.value.filter(l => l.due_date < new Date().toISOString())` counts any loan with a past due date regardless of status. Two paid loans in the dataset have past due dates so the count was always 2 too high.
- **My fix:** Filter by `l.status === 'overdue'` — the authoritative field set by the backend.

---

### 6. Silent failure in `pay()` — no user feedback on 502
- **Severity:** Low
- **Who it hurts and how:** Ops users who click Pay and assume it succeeded when it did not. The repay endpoint fails ~25% of the time. With no error handling the button appeared to do nothing.
- **Cause:** No `try/catch` around `$fetch`. The `loan.status = 'paid'` line after the `await` was never reached on failure.
- **My fix:** Wrapped in `try/catch`. On error, `alert` tells the user to retry.

---

## Things I noticed but deliberately did not fix

### `rates` fetched but never used
- **Why I left it:** `const { data: rates } = await useFetch('/api/rates')` fetches `{ base_rate, penalty_rate, effective_from }` and is never referenced anywhere. This looks like dead code from an unfinished feature. Silently removing it could break a feature branch that depends on it.

### Search API responding slower for shorter terms
- **Why I left it:** The README explicitly marks this as intentional — a two-letter search matches more rows so it scans more. This is realistic backend behaviour. I made no attempt to change the API's response time.

### `/api/loans/:id/schedule` takes ~150ms per call
- **Why I left it:** The README marks this as intentional — it reflects the real cost of a per-row lookup. I did not try to speed up the API. Instead I fixed the frontend to fetch schedule lazily on row click rather than for all 25 loans on every load, which makes the latency acceptable.

### `/api/loans/:id/repay` fails ~25% of the time with a 502
- **Why I left it:** The README marks this as intentional — it exists so that failure handling is observable. The 502 itself is not a bug. What was a bug was the frontend silently swallowing the error. I fixed the frontend error handling only.

### Money returned as integer kobo in `amount_kobo`
- **Why I left it:** The README explains this is how the real API works — 100 kobo = ₦1. I did not change the API format. I made sure all amount calculations in the frontend correctly divide by 100 before displaying.

### `handleResize` references `isNarrow` before its `const` declaration
- **Why I left it:** This looks like a temporal dead zone error. In practice it is safe — `handleResize` is a function declaration (hoisted) and is only ever called from the resize listener which fires after `onMounted`, by which point every `const` is initialised. Both were removed as dead code since `isNarrow` was never used in the template. Marking the ordering as a bug would have been wrong.

### No way to filter loans by status in the UI
- **Why I left it:** The `filters.status` reactive property is wired up and the API already supports a `status` param — the plumbing is there. A status dropdown next to the Refresh button would let ops narrow to `active`, `overdue`, or `paid` loans without searching. I did not add it because the README rules say to keep the same columns and rendered information. This would be the first thing I'd add with a green light from the team.

---

## Where I used AI

I used Claude Code (claude-sonnet-4-6) to implement fixes once I had identified the problems. I also used it to help explain root causes and think through solutions.

My primary debugging was done in Chrome DevTools network tab. I spotted that `/api/loans/:id/schedule` was firing for all 25 loans continuously without any user interaction. I also saw that `loans?status=all&search=` was firing repeatedly at intervals even when I was not doing anything — that was `setInterval` calling `loadLoans` every 5 seconds — and again on every keystroke because `watch(filters)` had no debounce. I identified the root causes and proposed the direction of each fix.

One place where the AI suggestion was wrong: it proposed fetching all schedules on first load using `Promise.all` with a cache check inside `loadLoans`. This caused a silent infinite reactive loop — mutating `loan.schedule` on a reactive object re-triggered `watch(filters)` which called `loadLoans` again. I caught this from the network tab and changed the approach to lazy loading on row click of each schedule instead.
