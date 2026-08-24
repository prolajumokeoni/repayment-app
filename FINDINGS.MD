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
- **My fix:** Stored the interval handle in `pollInterval` so it can be cancelled. Added `onUnmounted` — a Vue lifecycle hook that runs when the component is removed from the screen (user navigates away). Inside it, `clearInterval(pollInterval)` stops the timer and `window.removeEventListener('resize', handleResize)` removes the resize listener. Whatever is started in `onMounted` must be cleaned up in `onUnmounted`, otherwise it keeps running in the background consuming memory even after the user has left the page. I confirmed the issue by watching the network tab in Chrome DevTools and seeing `/api/loans/:id/schedule` firing continuously for all 25 loans without any user interaction.

---


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