# FINDINGS

## Bugs I found and fixed

### 1. XSS via `v-html` on server-supplied description
- **Severity:** Critical
- **Who it hurts and how:** Every operations user with the page open, and every system they are authenticated into. `<td v-html="loan.description">` renders raw HTML from the API. The seed data on ln_05 already contains a working XSS payload (`<img src=x onerror="document.title='xss'">`). In production a compromised loan record could execute arbitrary JavaScript in an authenticated session — exfiltrating tokens, impersonating actions, pivoting laterally.
- **Cause:** `v-html` renders the value as HTML instead of text.
- **My fix:** Replace `v-html` with `{{ loan.description }}`. Vue escapes the value; the HTML is displayed as literal text.