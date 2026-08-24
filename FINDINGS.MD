# FINDINGS

## Bugs I found and fixed

### 1. XSS via `v-html` on server-supplied description
- **Severity:** Critical
- **Who it hurts and how:** Every operations user with the page open, and every system they are authenticated into. `<td v-html="loan.description">` renders raw HTML from the API. The seed data on ln_05 already contains a working XSS payload (`<img src=x onerror="document.title='xss'">`). In production a compromised loan record could execute arbitrary JavaScript in an authenticated session — exfiltrating tokens, impersonating actions, pivoting laterally.
- **Cause:** `v-html` renders the value as HTML instead of text.
- **My fix:** Replace `v-html` with `{{ loan.description }}`. Vue escapes the value; the HTML is displayed as literal text.


### 2. Memory leak — `setInterval` and `addEventListener` never cleaned up
- **Severity:** High
- **Who it hurts and how:** Ops users who leave the repayments screen open all day. Every component mount added a new interval and a new resize listener without removing the old ones. By afternoon, hundreds of stacked intervals were all firing `loadLoans` in parallel, each making network requests and triggering Vue re-renders — which is exactly the sluggishness and fan noise ops reported.
- **Cause:** `setInterval(loadLoans, 5000)` discarded its return handle so `clearInterval` could never be called. `window.addEventListener('resize', handleResize)` had no corresponding `removeEventListener`.
- **My fix:** Removed `setInterval` entirely and replaced it with a manual Refresh button. Auto-polling every 5 seconds on a page ops leaves open all day was the root cause — there was no reason to keep hitting the server without the user asking for it. Also removed `handleResize`, `isNarrow`, and the resize `addEventListener` entirely — `isNarrow` was never referenced anywhere in the template so the listener was pure dead code with nothing to clean up. With both the interval and the resize listener gone, `onUnmounted` was no longer needed and was also removed. I confirmed the issue by watching the network tab in Chrome DevTools and seeing requests firing continuously without any user interaction.

---

### 3. Wrong customer shown in modal (index-based row identity)
- **Severity:** High
- **Who it hurts and how:** Any ops user drilling into a loan record — they may be reading and acting on the wrong customer's data. Ops confirmed this happened more often after searching, which is exactly when it would be worst.
- **Cause:** `expandedRow` stored the loop index `i`. After a search or filter change, `loans.value` was replaced with a new array and the saved index mapped to a completely different loan.
- **My fix:** Renamed `expandedRow` to `selectedLoanId` and stored `loan.id` instead of the array index. Renamed `expandedLoan` computed to `selectedLoan` and renamed `expandRow` to `openModal` — names now describe what they actually do. `:key` on the row changed from `i` to `loan.id`. If the selected loan is filtered out the modal closes rather than showing random data. Also moved the `label` computation into `loadLoans` so loans carry their label on the original reactive object — this eliminated a redundant `.find()` call since `openModal` now receives the original directly and can set `loan.schedule` without needing to look up the source object again. `visibleLoans` simplified to just `loans.value`.

---

### 4. Outstanding total: wrong value and wrong format
- **Severity:** Medium
- **Who it hurts and how:** Finance and ops — the headline figure they use to track portfolio exposure was both incorrect and formatted differently from every other amount on screen.
- **Cause (value):** `totalOutstanding` summed `loans.value`, which is the currently filtered list. Filtering by status or searching silently dropped loans from the total. The component already fetches `/api/customers/:id/summary` whose `outstanding_kobo` is the authoritative server-side figure.
- **Cause (format):** `total.toFixed(2)` returns a string. `formatNaira(v)` calls `v.toLocaleString()` — on a `String`, `toLocaleString()` is inherited from `Object` and returns the string unchanged with no thousands separators. Table rows passed a number so they formatted correctly; the header total did not.
- **My fix:** `totalOutstanding` returns `(summary.value?.outstanding_kobo ?? 0) / 100` — a number from the server's authoritative integer. `formatNaira` now receives a number in both places.

---

---

### 5. `overdueCount` includes paid loans with past due dates
- **Severity:** Low
- **Who it hurts and how:** Ops leads reading the headline overdue count — it was always inflated by paid loans whose due dates happened to be in the past.
- **Cause:** `loans.value.filter(l => l.due_date < new Date().toISOString())` counts any loan with a past due date regardless of status. Two paid loans in the dataset have past due dates so the count was always 2 too high.
- **My fix:** Filter by `l.status === 'overdue'` — the authoritative field set by the backend.

---