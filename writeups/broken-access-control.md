
# Broken Access Control

**Category summary:** Broken Access Control occurs when an application fails to properly restrict what authenticated users are allowed to do, letting them view or modify data and functionality that should be off-limits to them. Across this category, 9 of 12 challenges were fully solved; the remaining 3 are documented below with the technical reasons they could not be completed in this environment.

---

## 1. Admin Section

**Difficulty:** ⭐⭐

**Steps:** Navigated directly to `http://127.0.0.1:42000/#/administration` — a route that exists and functions fully but is never linked anywhere in the site's UI.

**Why it worked:** No authentication/authorization check exists on this route; it relies purely on "security through obscurity" (hoping nobody finds the URL).

**Impact:** Exposes the full list of registered users and admin controls to anyone who guesses or discovers the URL.

**Fix:** Enforce server-side role checks on every route regardless of whether it's linked in the UI.

## 2. Five-Star Feedback

**Difficulty:** ⭐⭐

**Steps:** From the Admin Section, located Customer Feedback and deleted the 5-star entry via the trash icon.

**Why it worked:** No proper authorization on the feedback-deletion endpoint beyond reaching the hidden admin page.

**Impact:** Anyone who finds the admin page can delete legitimate customer reviews, manipulating the store's public reputation.

**Fix:** Enforce role-based authorization server-side on the delete-feedback endpoint.

## 3. View Basket
**Difficulty:** ⭐⭐
**Steps:** Logged in as a regular user, opened own basket (ID 6), used DevTools "Copy as fetch" to capture the authenticated request, then changed the basket ID in the URL from `6` to `1` and replayed it.
**Why it worked:** The `/rest/basket/:id` endpoint checks that a valid token was presented, but never checks whether the basket ID in the URL actually belongs to that token's owner — a classic IDOR (Insecure Direct Object Reference).
**Impact:** Any logged-in user can view any other user's basket contents.
**Fix:** Server must verify the basket's owner ID matches the authenticated user's ID before returning data.

## 4. Manipulate Basket
**Difficulty:** ⭐⭐⭐
**Steps:** Captured the POST request used to add an item to the basket, then used HTTP Parameter Pollution — sending the `BasketId` field twice in the same JSON body (own ID first, target ID second): `{"ProductId":1,"BasketId":"6","quantity":1,"BasketId":"1"}`.
**Why it worked:** The server's ownership-validation step reads the first `BasketId` value and approves it, but the actual database write uses the last `BasketId` value — a mismatch between what's validated and what's executed.
**Impact:** An attacker can add unwanted or malicious items into another user's basket.
**Fix:** Reject requests containing duplicate JSON keys, and ensure validation and execution always read the same value.

## 5. Forged Feedback
**Difficulty:** ⭐⭐⭐
**Steps:** Captured a normal feedback submission, then edited the `UserId` field in the request body from the logged-in user's own ID to a different user's ID before sending.
**Payload:** `{"UserId":1,"captchaId":1,"captcha":"12","comment":"FORGEDgood","rating":2}`
**Why it worked:** The server trusts the client-supplied `UserId` field instead of deriving the author's identity from the authenticated token.
**Impact:** Anyone can post fake feedback attributed to any other user, including impersonating admins.
**Fix:** Never accept identity fields from client input; always derive the acting user's ID from their verified token server-side.

## 6. Forged Review
**Difficulty:** ⭐⭐⭐
**Steps:** Same technique as Forged Feedback — captured a review submission and edited the `author` field in the request body to a different user's email.
**Payload:** `{"message":"FORGED review","author":"admin@juice-sh.op"}`
**Why it worked:** The `author` field is plain client-supplied text, not derived from the server-verified token.
**Impact:** Anyone can post or edit reviews under another user's identity.
**Fix:** Derive the review author from the authenticated session token, never from client input.

## 7. Product Tampering
**Difficulty:** ⭐⭐⭐
**Steps:** Logged in as admin, queried `/rest/products/search?q=o-saft` to find the product's database ID (9), then sent a PUT request to `/api/Products/9` with a modified description field.
**Payload:** `{"description":"<a href=\"https://owasp.slack.com\" target=\"_blank\">More...</a>"}`
**Why it worked:** The product update endpoint accepts admin-authenticated changes without sanitizing or validating the HTML/URL content being injected.
**Impact:** Demonstrates stored content tampering — an attacker with elevated access could inject malicious links or stored XSS into content shown to every site visitor.
**Fix:** Sanitize/validate HTML content before storing it in product fields.

## 8. Web3 Sandbox
**Difficulty:** ⭐
**Steps:** Navigated directly to `http://127.0.0.1:42000/#/web3-sandbox` — another unlinked, undiscoverable-by-UI route.
**Why it worked:** Same security-through-obscurity issue as Admin Section — a fully functional smart contract dev tool left exposed with no authentication.
**Impact:** Exposes internal developer tooling to any visitor.
**Fix:** Remove unused/internal tools from production deployments, or protect them with authentication.

## 9. Easter Egg (+ bonus: Poison Null Byte)
**Difficulty:** ⭐⭐⭐⭐
**Steps:** Browsed the exposed `/ftp` directory, attempted to open `eastere.gg` directly (blocked: "403 Only .md and .pdf files are allowed"), then bypassed the filter using a Poison Null Byte: `http://127.0.0.1:42000/ftp/eastere.gg%2500.md`.
**Why it worked:** The file-extension security filter reads the full filename and approves it based on the trailing `.md`, but the underlying file-system code stops reading at the null byte (`%00`), actually opening `eastere.gg` — a mismatch between what's validated and what's executed, the same root pattern seen throughout this category.
**Impact:** Allows bypassing file-type restrictions to access any file in a restricted directory, regardless of its actual extension.
**Fix:** Validate file extensions using the fully-resolved file path after all string processing, not before.

---

## Not completed (documented with technical justification)

### AI Debugging — Blocked (requires LLM configuration)
This challenge, along with three other AI-tagged challenges introduced in a recent Juice Shop version, requires a configured LLM/AI backend (e.g., an OpenAI API key) connected to the chatbot feature. Without it, the chatbot only returns scripted fallback responses, and the official companion guide's solution for this specific challenge is itself unpublished/blank. Not solvable without additional environment setup beyond the scope of this assignment.

### CSRF — Attempted, blocked by modern browser security defaults
**Vulnerability (confirmed, not disputed):** The `/profile` endpoint accepts username changes via a simple POST request with no CSRF token, relying solely on a cookie for authentication — a textbook CSRF flaw.
**Why the live exploit could not be demonstrated:** Modern browsers (Firefox 97+/Chrome 80+) now require cookies marked `SameSite=None` to also carry the `Secure` flag before they will be attached to cross-site requests. Since Juice Shop runs on plain HTTP, its session cookie cannot be marked `Secure`, so current browsers silently refuse to send it on the cross-origin request this attack depends on. This exploit worked as designed on Firefox 96 and older, per the challenge's own documentation, but is now blocked by browser security standards that postdate the challenge's design.
**Fix (still valid regardless):** Implement CSRF tokens on all state-changing requests.

### SSRF — Deferred (depends on Injection-category prerequisite)
This 6-star challenge requires first completing a server-side command execution exploit from the Injection category ("infect the server with malware via arbitrary command execution"), which falls outside this category's scope. To be revisited once Injection challenges are completed.
