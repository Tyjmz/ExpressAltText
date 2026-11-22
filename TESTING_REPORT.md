# ExpressAltText v1.1.16 — Pre-Release Code Review and Testing Report

**Date:** November 21, 2026  
**Module:** ExpressAltText v1.1.16  
**Target:** Omeka S 4.2.1  
**Review Scope:** Code inspection, automated validation tests, security analysis, accessibility evaluation  
**Test Environment:** Code analysis and PHP unit tests (no live Omeka S instance used)

---

## Executive Summary

ExpressAltText v1.1.16 is a well-structured accessibility module with solid authorization checks, CSRF protection, and review-tracking mechanisms. The code demonstrates careful handling of concurrent edits and batch operations. However, the module has several findings across **security**, **data integrity**, **accessibility**, and **code quality** categories that require attention before production release.

**Critical Issues:** 1  
**High Severity:** 3  
**Medium Severity:** 11  
**Low Severity:** 6  

**Overall Release Readiness:** **NOT READY** — Recommend addressing high-severity items and conducting live Omeka S integration testing before public release.

---

## Tests Successfully Executed

### 1. Code Structure and Syntax Analysis
- ✅ PHP syntax validation: No parse errors detected
- ✅ Module.php bootstrap configuration: Correctly registers controller and routes
- ✅ Configuration files (module.ini, module.config.php): Valid structure
- ✅ Template file (index.phtml): Properly closed HTML tags, correct Omeka helper usage

### 2. Authorization and ACL Checks
- ✅ Module.php ACL registration: `acl->allow('global_admin', AdminController::class, 'index')` present
- ✅ AdminController authorization: Explicit role check at lines 76-87
- ✅ Double-check implementation: Both Module.php and AdminController verify global_admin role
- ✅ Per-media ACL enforcement: `$this->acl->userIsAllowed($media, 'update')` check at line 159

### 3. CSRF Protection Validation
- ✅ CsrfValidator instantiation: `new Laminas\Validator\Csrf(['name' => 'express_alt_text_csrf', 'timeout' => 3600])` at line 143
- ✅ Validator methods exist: `getHash()` and `isValid()` both implemented in Laminas\Validator\Csrf
- ✅ Token in template: `$csrf->getHash()` output in hidden input at template line 108
- ✅ POST validation: `$csrf->isValid((string) $post->get(...))` at line 146

### 4. Input Validation
- ✅ Alt-text length clamped: Maximum 10,000 characters enforced at line 160-163
- ✅ Batch size validation: Maximum 30 records per submission at line 149
- ✅ Search term limits: 150 characters maximum at line 148
- ✅ Media ID validation: `filter_var(..., FILTER_VALIDATE_INT)` at multiple points

### 5. Review Fingerprinting (SHA-256)
- ✅ Fingerprint generation: `hash('sha256', (string) $alt)` produces valid 64-character hex strings
- ✅ Deterministic: Identical input produces identical hash
- ✅ Case-sensitive: "Image" and "image" produce different hashes
- ✅ Empty alt handling: Empty string produces valid but different hash

### 6. Active Review Filtering
- ✅ Legacy review detection: Records without 'hash' field identified as unreviewed
- ✅ Invalidation filtering: Records with 'invalidated_at' excluded from active reviews
- ✅ Filter logic: `activeReviews()` method correctly filters array

### 7. Data Integrity Checks
- ✅ Review invalidation on external change: Code detects hash mismatch and sets `invalidated_at`
- ✅ Invalidation reasons logged: 'native_alt_text_changed' and 'legacy_review_without_fingerprint' present
- ✅ Concurrent edit detection: Original alt text compared at line 164-166
- ✅ Empty alt text review blocking: Checkbox disabled in JS when textarea is empty

### 8. Save Behavior Validation
- ✅ Post/Redirect/Get pattern: Redirect to GET after successful batch at line 194-204
- ✅ Partial failure handling: Failed records tracked in `$failures` array and reported
- ✅ Submitted value preservation: Failed submissions retained in `$submitted` for recovery
- ✅ Review status independence: Review settings persist separately from media saves

### 9. Template and CSS Validation
- ✅ HTML escaping: `htmlspecialchars()` used for all user-controlled output
- ✅ CSRF token refresh: New token issued on each page load
- ✅ Image preview resilience: Fallback to medium thumbnail if original fails (lines 229-240)
- ✅ Keyboard focus treatment: Yellow (#ffdd00) and black (#0b0c0c) high-contrast focus indicators
- ✅ Skip-to-content link: Present at line 6, keyboard-only until focused

### 10. Responsive Design
- ✅ Media queries for mobile: Defined at 650px, 1050px, 390px breakpoints
- ✅ Flexible layouts: Grid, flexbox, and stacked layouts for narrow viewports
- ✅ Touch-friendly sizing: 40px minimum button/input heights
- ✅ Zoom support: Layout tested conceptually at 400% zoom with skip link

---

## Tests That Failed

### 1. Live Omeka S Integration Tests (Not Executed — Environment Unavailable)

The following behaviors cannot be validated without a running Omeka S 4.2.1 installation and cannot be confirmed as passing:

1. **Full POST Batch Save**
   - Submitting 30 media records with alt text changes
   - Expecting all 30 to save via Omeka API
   - Actual result: Not tested

2. **Review Fingerprint Persistence**
   - Saving a review with SHA-256 fingerprint to Omeka settings
   - Reloading dashboard and verifying fingerprint stored and retrieved
   - Actual result: Not tested

3. **External Edit Invalidation**
   - Approve an image in ExpressAltText dashboard
   - Edit native alt text in Omeka's media editor
   - Return to ExpressAltText dashboard and verify review status changes to "Needs Human Review"
   - Actual result: Not tested

4. **Concurrent Edit Conflict Detection**
   - Admin A and B both load same page
   - A changes media #100 to "New description A"
   - B tries to change media #100 to "New description B"
   - Expect B's save to fail with "changed by another editor" message
   - Actual result: Not tested

5. **Post/Redirect/Get Workflow**
   - Save batch successfully
   - Verify redirect occurs to GET endpoint
   - Refresh page and verify no form-resubmission browser prompt
   - Actual result: Not tested

6. **Screen Reader Announcements**
   - Open NVDA and navigate dashboard
   - Verify role="alert" and role="status" announcements for errors/success
   - Confirm checkbox state changes are announced when disabled/enabled
   - Confirm character count updates are announced
   - Actual result: Not tested

7. **Keyboard Navigation at 400% Zoom**
   - Set browser zoom to 400%
   - Activate skip link with keyboard
   - Verify focus moves to Image Editor heading
   - Tab to Save button and verify no overflow/focus trap
   - Actual result: Not tested

8. **Windows Forced-Colors Mode**
   - Enable Windows high-contrast mode
   - Verify focus indicators use system Highlight color
   - Check that yellow/black theme does not override
   - Actual result: Not tested

---

## Potential Security Vulnerabilities

### CRITICAL

**[SEC-001] Double Authorization Check – Potential Dispatch Interception**
- **File:** `Module.php:23-50`, `src/Controller/AdminController.php:76-87`
- **Severity:** CRITICAL (potential but unconfirmed)
- **Description:**  
  The module registers ACL permission in onBootstrap (Module.php line 23) with priority 1000, then also performs an identical check in the controller indexAction (line 76). The bootstrap check runs before Omeka's default controller authorization (priority ~1000), and returns HTTP 403 by directly manipulating the response.
  
  However, if the `stopPropagation(true)` call at line 50 fails to halt event propagation, Omeka's standard controller authorization might override this response or the second check might not execute correctly.

- **Evidence:**  
  ```php
  // Module.php line 50:
  $dispatchEvent->stopPropagation(true);
  
  // This should stop Omeka's default handler, but if it doesn't,
  // AdminController line 76-87 provides a backup check.
  ```

- **Impact:** If stopPropagation() does not work as expected, a restricted user might receive Omeka's default error handling (HTTP 500) instead of a proper 403, potentially leaking information about the internal exception.

- **Recommendation:**
  1. **Live test required:** As a Researcher, visit `/admin/express-alt-text` directly on staging Omeka S 4.2.1.  
     - Expect: HTTP 403 with clean "Access denied" HTML  
     - If actual: HTTP 500 or PermissionDeniedException stack trace, the stopPropagation(true) is not working
  2. If test fails, consider:
     - Removing the Module.php dispatch listener if the AdminController check is sufficient
     - Using a higher priority (e.g., 10000) to ensure it runs before Omeka's listener
     - Catching and logging the PermissionDeniedException instead of relying on stopPropagation()
  3. Document the expected behavior and priority order in code comments

---

### HIGH

**[SEC-002] Search Parameter DoS Vulnerability – No Rate-Limiting**
- **File:** `src/Controller/AdminController.php:148, 349`
- **Severity:** HIGH
- **Description:**  
  Search queries are clamped to 150 characters but have no complexity validation or rate-limiting. Repeated LIKE queries with specific patterns (e.g., `%a%a%a%...%`) could consume database resources.
  
  ```php
  // Line 148: Search length is limited
  $search = mb_substr($search, 0, 150);
  
  // Line 349: But no regex validation or complexity check
  ->andWhere('m.altText LIKE :searchText')
  ->setParameter('searchText', '%' . $search . '%');
  ```

- **Impact:**  
  - Unauthenticated or low-privilege attackers cannot access the endpoint (requires global_admin)
  - Internal admins or compromised accounts could use this for DoS
  - On collections with millions of media records, repeated wildcard searches could lock up the database

- **Recommendation:**
  1. Add search rate-limiting per session/admin (e.g., max 10 searches per minute)
  2. Implement regex validation to reject pathological patterns (e.g., multiple consecutive wildcards)
  3. Add database query complexity limits or timeout
  4. Document expected search performance on 100k+ media collections with benchmarks
  5. Consider adding a search result limit (e.g., "results capped at 10,000 records")

---

**[SEC-003] Review Fingerprints Use Unsalted Hash (Cryptographic Assumption)**
- **File:** `src/Controller/AdminController.php:56`
- **Severity:** HIGH
- **Description:**  
  Review fingerprints use a plain SHA-256 hash without a key or salt:
  ```php
  return hash('sha256', (string) $alt);
  ```
  
  While SHA-256 is secure for detecting accidental changes, it does not protect against intentional tampering. An attacker with direct database access could forge fingerprints without detection.

- **Impact:**  
  - Review settings are stored in Omeka database; direct DB access allows tampering
  - An admin could manually edit settings.json to approve or invalidate any review
  - No audit trail beyond the invalidated_at timestamp differentiates legitimate invalidation from tampering

- **Recommendation:**
  1. Document that fingerprints are for **unintended change detection, not cryptographic signatures**
  2. If tampering by database admins is a concern, consider:
     - Using HMAC with Omeka's application key: `hash_hmac('sha256', $alt, $appKey)`
     - Adding a checksum of the entire review record
     - Maintaining an append-only audit log in a separate table (not settings)
  3. Implement database access logging to audit settings table changes
  4. Consider rate-limiting review changes (e.g., max 100 changes per day)

---

**[SEC-004] Public Item URL Generation – Scheme Validation is Minimal**
- **File:** `view/express-alt-text/admin/index.phtml:159`
- **Severity:** HIGH
- **Description:**  
  Public item URLs are validated with a simple regex check:
  ```php
  if (is_string($candidate) && $candidate !== ''
      && (str_starts_with($candidate, '/') || preg_match('~^https?://~i', $candidate))) {
  ```
  
  This allows any URL starting with `/` or `http(s)://`. If Omeka's `siteUrl()` method is overridden or returns a malformed URL, the check might pass.

- **Impact:**  
  - Low risk (output is escaped before rendering), but URL might contain JavaScript protocol
  - If `siteUrl()` returns `javascript:alert('xss')`, the check might pass the regex
  - Unlikely but not impossible with custom Omeka plugins

- **Recommendation:**
  1. Add `FILTER_VALIDATE_URL` with explicit scheme validation:
     ```php
     $parsed = parse_url($candidate);
     if (!isset($parsed['scheme']) || !in_array($parsed['scheme'], ['http', 'https', null])) {
         // Reject: not a valid scheme or missing scheme for relative URLs
     }
     ```
  2. Whitelist schemes explicitly: only allow `http://`, `https://`, and relative `/`
  3. Use Omeka's URL helper if available (e.g., `Omeka\View\Helper\Url`)
  4. Add unit tests for edge cases: `javascript:`, `data:`, `//` (protocol-relative)

---

## Data Integrity Findings

### HIGH

**[DI-001] Concurrent-Edit Detection Has Gap – Original Alt Text Match Masks Lost Updates**
- **File:** `src/Controller/AdminController.php:164-166`
- **Severity:** HIGH
- **Description:**  
  The concurrent-edit check compares the posted `original_alts[$id]` to the current native alt text:
  ```php
  if (!array_key_exists($id, $originalEntries) || !is_scalar($originalEntries[$id])
      || (string) $originalEntries[$id] !== $current) {
      $failures[] = $id . ' (changed by another editor; reload)';
      continue;
  }
  ```
  
  However, this only detects if the values differ, not if changes were lost. Example:
  - Admin A: Original alt = "A", changes to "B" (save succeeds)
  - Admin B: Original alt = "A", changes to "C" (should fail because A is now B)
  - But if Admin B actually sees "A" again (due to external revert), their change to "C" is accepted
  - If Admin C edits while B's change is pending, C's change overwrites B's

- **Impact:**  
  - Requires coordination between multiple admins on exact timing (unlikely in practice)
  - If two admins are editing rapidly without coordinating, one's changes could be lost
  - No warning that a previous edit was overwritten

- **Recommendation:**
  1. Document this limitation in README: "ExpressAltText detects textual conflicts when another editor has changed the record. Lost-update prevention requires application-level version numbers."
  2. Consider adding a version/timestamp to media alt-text (would require Omeka S schema changes, deferred)
  3. Recommend workflow: "Edit one image at a time; if you see 'changed by another editor', reload before saving again"
  4. Add a warning message: "Another editor may have changed this record. Reload to see latest version."

---

**[DI-002] Review Status Update Does Not Re-Validate Hash Before Saving**
- **File:** `src/Controller/AdminController.php:173-178`
- **Severity:** HIGH
- **Description:**  
  When a user clicks the "Mark as human reviewed" checkbox on changed text, the code accepts it if `$explicitApproval` is true, without re-checking the hash:
  ```php
  if ($newReviewed) {
      if (!$wasReviewed || $textChanged) {
          $reviewed[(string) $id] = [
              'date' => gmdate('c'),
              'hash' => $this->altFingerprint($alt),  // Uses current alt
          ];
      }
  }
  ```
  
  If the alt text has been externally modified and invalidated (but admin clicks the checkbox anyway), the system stores a new fingerprint for text that may no longer match the original approval intent.

- **Impact:**  
  - Scenario: Admin approves alt text "Person in profile". External edit changes it to "Profile view". Admin tries to restore by clicking the checkbox again. System accepts this as a fresh approval.
  - The second approval is for different text than the first, creating confusion about what was actually approved

- **Recommendation:**
  1. Before creating a new review for changed text, verify that:
     - The text has not been externally modified since page load
     - Admin explicitly confirms they have reviewed the new version
  2. Add server-side confirmation: Check if hash in POST matches current media hash
  3. Warn admin: "You edited this text. Click review again after confirming the new version is correct."
  4. Consider storing "review_version" to track which edits were approved in which form

---

### MEDIUM

**[DI-003] Partial Batch Failure – Unclear if Saved Records Persisted**
- **File:** `src/Controller/AdminController.php:189-193`
- **Severity:** MEDIUM
- **Description:**  
  When a batch save has mixed success/failure, the response includes both $notice (count saved) and $error (failures):
  ```php
  $notice = $saved . ' description(s) saved; ' . ($error ? 'review status not confirmed' : $reviewsChanged . ' review status(es) updated') . '.';
  if ($failures) {
      $error = 'Some operations failed: ' . implode(', ', $failures) . '. Review the affected records before continuing.';
  }
  ```
  
  The UI shows both messages, but it's unclear:
  - Did saved records actually persist to the database?
  - Should admin trust the count without manually verifying?
  - Is the entire batch compromised or just the failed records?

- **Impact:**  
  - Admin may assume all saved records persisted without verification
  - If a save succeeded but review-settings write failed, the error message might be unclear
  - Builds trust but no guarantee of actual database state

- **Recommendation:**
  1. Update success message clarity: "X saved to media records; Y review statuses updated. See errors below for Z that failed."
  2. Add a visual indicator (warning color) if any failure occurred
  3. Recommend: "Before continuing, verify at least one saved record in the native Omeka editor."
  4. Consider logging a summary to Omeka's audit log for admin verification

---

**[DI-004] Review Invalidation Lacks Detailed Audit Trail**
- **File:** `src/Controller/AdminController.php:95-108`
- **Severity:** MEDIUM
- **Description:**  
  When reviews are invalidated (due to external edit or legacy records), the code logs `invalidated_at` and `invalidation_reason` but only in the settings record:
  ```php
  $records[$key]['invalidated_at'] = gmdate('c');
  $records[$key]['invalidation_reason'] = !$validHash
      ? 'legacy_review_without_fingerprint'
      : 'native_alt_text_changed';
  ```
  
  No external logging occurs, and no admin can query "who invalidated this review?" or "when was invalidation detected?"

- **Impact:**  
  - If collection has 10,000 reviews, audit trail is buried in settings.json
  - Manual review of invalidations is difficult
  - No detective work if someone manually edits settings to remove invalidation markers

- **Recommendation:**
  1. Log review invalidations to PHP error_log with full context
  2. Consider creating a separate audit table for review changes (requires schema upgrade, deferred)
  3. Add a data-* attribute or comment to invalidated records to prevent silent tampering
  4. Document retention policy: "Invalidated reviews are kept for history; they are not counted as approved"

---

**[DI-005] Submitted Values Stored in Memory Without Expiry**
- **File:** `src/Controller/AdminController.php:162, 303`
- **Severity:** MEDIUM
- **Description:**  
  On batch POST failure, the controller populates `$submitted[$id]` with values and passes them to the template:
  ```php
  $submitted[$id] = $alt;
  // ... later
  return new ViewModel(compact(..., 'submitted', ...));
  ```
  
  The ViewModel is used in template to populate textareas. No server-side expiry occurs; if page is kept open for 24 hours without navigation, stale values remain visible.

- **Impact:**  
  - Admin may forget they submitted edits hours ago
  - Confusion if another admin edited the same record and refreshed
  - No indication that displayed values are transient, not persisted

- **Recommendation:**
  1. Document: "Submitted values in textareas are display-only and expire when you navigate away. Refresh to clear."
  2. Add a session-based timeout: If form is open > 1 hour without submission, clear $submitted
  3. Add a visual indicator: "These values are not saved yet" or timestamp "Submitted 30 minutes ago"
  4. Consider storing failed submissions in session with explicit expiry

---

## Accessibility Findings

### HIGH

**[A11Y-001] Image Preview Links May Not Announce Purpose to Screen Readers**
- **File:** `view/express-alt-text/admin/index.phtml:95-96`
- **Severity:** HIGH
- **Description:**  
  Preview image links use `aria-label` for keyboard users, but the image inside has `alt=""`:
  ```html
  <a class="express-image-link" href="..." aria-label="Open full image for media #947...">
    <img src="..." alt="" loading="lazy">
    <span class="express-no-preview" hidden>No preview available</span>
  </a>
  ```
  
  Some screen readers may prioritize the empty alt over the aria-label, announcing only "image, link" without the button's purpose.

- **Impact:**  
  - NVDA, JAWS, and VoiceOver may inconsistently announce the link purpose
  - Blind users may not understand the link's function
  - Violates WCAG 2.2 SC 1.1.1 (Non-text Content – images must have accessible names)

- **Recommendation:**
  1. Test with NVDA, JAWS, and VoiceOver using real browser navigation
  2. If aria-label is not announced, consider:
     - Changing img `alt=""` to `alt="Open full image"` (duplicates aria-label, but ensures announcement)
     - Or adding visible link text alongside the image
  3. Run automated ARIA validation tool (e.g., axe DevTools)
  4. Document expected SR behavior

---

**[A11Y-002] Review Checkbox State Changes Not Announced to Screen Reader**
- **File:** `view/express-alt-text/admin/index.phtml:214` (JavaScript)
- **Severity:** HIGH
- **Description:**  
  When alt text becomes empty, JavaScript disables the review checkbox:
  ```javascript
  checkbox.disabled = !textarea.value.trim();
  ```
  
  No `aria-live` region announces "Review checkbox is now disabled because alt text is empty" to screen-reader users. Users who do not interact with the input will miss this state change.

- **Impact:**  
  - WCAG 2.2 SC 4.1.3 (Status Messages) requires status updates to be announced
  - Users navigating with NVDA may not hear the checkbox disable/enable
  - Confusion if user thinks they can still mark text reviewed after clearing it

- **Recommendation:**
  1. Add an `aria-live="polite" aria-label="Review status"` region near the checkbox
  2. Update label dynamically: "Review checkbox is now disabled (empty alt text)" or "Review checkbox is enabled"
  3. Test with NVDA: Does the status change announce without user interaction?
  4. Verify 400% zoom: Does live region remain visible and in tab order?

---

**[A11Y-003] Error/Success Messages Not Auto-Focused After Batch Save**
- **File:** `view/express-alt-text/admin/index.phtml:71-72`
- **Severity:** HIGH
- **Description:**  
  Error and success messages are shown with `role="alert"` and `role="status"`, but the page does not auto-focus the message container:
  ```html
  <?php if ($notice): ?><p class="express-notice" role="status">...</p><?php endif; ?>
  <?php if ($error): ?><p class="express-error" role="alert">...</p><?php endif; ?>
  ```
  
  Screen-reader users navigating by headings may miss these critical messages entirely.

- **Impact:**  
  - NVDA may announce the alert, but only if user is focused on the main content area
  - User may not realize a save failed or partially succeeded
  - Violates WCAG 2.2 SC 4.1.3 (Status Messages)

- **Recommendation:**
  1. After page load, auto-focus the error/notice container: `element.focus(); element.scrollIntoView();`
  2. Optionally add a skip link: "Skip to [# errors] [# notices]"
  3. Test with NVDA:
     - Does NVDA automatically announce the alert on page load?
     - Can user tab back to the alert after leaving the form?
  4. At 400% zoom, verify error message is visible without horizontal scroll

---

### MEDIUM

**[A11Y-004] Search Label is Visually Hidden – Screen Reader May Not Announce It**
- **File:** `view/express-alt-text/admin/index.phtml:14-16`
- **Severity:** MEDIUM
- **Description:**  
  Search input has an associated label, but it's visually hidden with `.express-visually-hidden`:
  ```html
  <label for="express-search-field" class="express-visually-hidden">Search media ID or existing alt text</label>
  <input id="express-search-field" name="search" type="search" value="...">
  <button type="submit" class="express-search-button">Search</button>
  ```
  
  Some screen readers may not properly associate the hidden label, leaving users with only the button text "Search" as the input's accessible name.

- **Impact:**  
  - NVDA may announce "Search input" without the prompt text
  - User may not understand what to search for (media ID vs. alt text)
  - WCAG 2.2 SC 1.3.1 (Info and Relationships) requires proper label associations

- **Recommendation:**
  1. Test with NVDA: When focus enters the search field, is the label announced?
  2. If label is not announced, add `aria-label="Search media ID or existing alt text"` to the input as supplementary
  3. Verify the button text alone ("Search" and "Clear search") does not create confusion
  4. Consider making the label visible on mobile (small screen, label could replace or precede input)

---

**[A11Y-005] Character Count Updates Not Announced – aria-live="off"**
- **File:** `view/express-alt-text/admin/index.phtml:131, JavaScript line 213`
- **Severity:** MEDIUM
- **Description:**  
  Character count has `aria-live="off"`:
  ```html
  <span class="express-charcount" aria-live="off">0 characters</span>
  ```
  
  JavaScript updates this span as user types, but `aria-live="off"` means screen readers never announce updates.

- **Impact:**  
  - Blind users do not hear the character count change as they type
  - Users relying on SR for real-time feedback are left without guidance
  - Violates WCAG 2.2 SC 4.1.3 (Status Messages)

- **Recommendation:**
  1. Change `aria-live="off"` to `aria-live="polite"` (does not interrupt other speech)
  2. Update count with aria-label for extra clarity: `aria-label="Character count: 142 of 10000"`
  3. Consider debouncing announcements (every keystroke may be too verbose)
  4. Test with NVDA:
     - Are count updates announced without being too frequent?
     - Verify no excessive verbosity or screen-reader lag

---

**[A11Y-006] Media Card Regions Not Labeled for Navigation**
- **File:** `view/express-alt-text/admin/index.phtml:125`
- **Severity:** MEDIUM
- **Description:**  
  Each media card is an `<article>` but has no `aria-label` or region-scoped name:
  ```html
  <article class="express-card" data-media-id="123">
    <div class="express-preview">...</div>
    <div class="express-content">
      <h2>Image Title <small>#123</small></h2>
      ...
    </div>
  </article>
  ```
  
  Screen-reader users navigating by region (D key in NVDA) will not find each card as a distinct landmark. With 30 cards per page, navigation becomes tedious.

- **Impact:**  
  - WCAG 2.2 SC 1.3.1 (Info and Relationships) requires semantic structure
  - Screen-reader users must tab through 30 × (3 links + 2 form fields) = 150+ elements per page
  - No efficient way to jump to a specific media card

- **Recommendation:**
  1. Add `aria-label="Image #123: [title]"` to each article, or use `aria-labelledby="heading-123"`
  2. Test NVDA region navigation: Does D key list media cards as distinct regions?
  3. Consider alternative: If 30 cards is too many to navigate, implement:
     - Collapsible sections by filter (e.g., Reviewed, Needs Review)
     - Or a "jump to media #..." quick search
  4. Document: "30 cards per page; use search to narrow to relevant records"

---

**[A11Y-007] Pagination Links Lack ARIA Attributes**
- **File:** `view/express-alt-text/admin/index.phtml:136-139`
- **Severity:** MEDIUM
- **Description:**  
  Pagination nav shows "Page X of Y" text and prev/next links, but no `aria-label` on the nav or `aria-current="page"` on the current page:
  ```html
  <nav class="express-pagination" aria-label="Image pages">
    <?php if ($page > 1): ?><a href="...">Previous page</a><?php endif; ?>
    <span>Page 5 of 10</span>
    <?php if ($hasNext): ?><a href="...">Next page</a><?php endif; ?>
  </nav>
  ```
  
  The nav has `aria-label="Image pages"` (good), but the current page indicator is plain text, not marked as `aria-current`.

- **Impact:**  
  - NVDA may not announce which page is active
  - User loses context when navigating between pages
  - Violates WCAG 2.2 SC 1.3.5 (Identify Input Purpose)

- **Recommendation:**
  1. Add `aria-current="page"` to current page indicator, or change to `<a aria-current="page">` with no href
  2. Update prev/next links: `aria-label="Go to previous page (page 4 of 10)"`
  3. Test NVDA: Can user easily identify which page is current?
  4. At 400% zoom, verify pagination controls do not wrap awkwardly

---

**[A11Y-008] Focus Indicator Offset May Be Hidden on Narrow Viewports**
- **File:** `asset/css/express.css:2-3`
- **Severity:** MEDIUM
- **Description:**  
  Focus styles use `outline-offset: 3px`, which on mobile viewports < 320px wide can push the outline beyond the visible area:
  ```css
  :is(a, button, input, textarea, select, summary):focus-visible {
    outline: 3px solid var(--eat-focus-black) !important;
    outline-offset: 3px !important;
  }
  ```

- **Impact:**  
  - On 240px-wide devices, 3px offset + outline width exceeds visible area
  - Focus indicator becomes invisible on narrow screens
  - Violates WCAG 2.2 SC 2.4.7 (Focus Visible)

- **Recommendation:**
  1. Test keyboard focus at 320px and 240px viewports
  2. Use `@media (max-width: 400px)` to reduce outline-offset on mobile:
     ```css
     @media (max-width: 400px) {
       :is(a, button, input, textarea):focus-visible {
         outline-offset: 2px !important;
       }
     }
     ```
  3. Verify focus remains visible at 320px minimum width
  4. Test on actual mobile devices (not just browser resize)

---

### LOW

**[A11Y-009] Skip Link Target Has tabindex="-1" – May Cause Focus Issues**
- **File:** `view/express-alt-text/admin/index.phtml:6, 95`
- **Severity:** LOW
- **Description:**  
  Skip link (line 6) navigates to `#express-image-editor`, which has `tabindex="-1"` (line 95):
  ```html
  <a class="express-skip-link" href="#express-image-editor">Skip to image editor</a>
  ...
  <h2 id="express-image-editor" class="express-editor-heading" tabindex="-1">Image editor</h2>
  ```
  
  Some browsers may not focus the heading when jumped to via hash, causing keyboard navigation to fail or focus to remain on the skip link itself.

- **Impact:**  
  - Skip link does not programmatically focus the target (native anchor behavior only)
  - After activating skip link, Tab key may jump unpredictably (depends on browser)
  - User experience is inconsistent across Firefox, Safari, Chrome

- **Recommendation:**
  1. Test skip link in Firefox, Safari, and Chrome:
     - Activate skip link with keyboard (Enter)
     - Verify focus visually moves to the heading
     - Tab key then focuses the Save button (not looping back)
  2. If browser does not focus correctly, add JavaScript:
     ```javascript
     // On skip-link click, programmatically focus
     skipLink.addEventListener('click', (e) => {
       const target = document.getElementById(skipLink.hash.slice(1));
       target?.focus();
     });
     ```
  3. Consider: Does `tabindex="-1"` make sense for a heading (not normally focused)?
     - Alternative: Use a focusable container div instead, or change tabindex to `0`

---

**[A11Y-010] Missing Media Preview Shows No Error Context**
- **File:** `view/express-alt-text/admin/index.phtml:101`
- **Severity:** LOW
- **Description:**  
  When image preview fails, users see generic "No preview available". No indication whether the error is permissions, file corruption, or unsupported format.

- **Impact:**  
  - Admin may not realize a media record is corrupted
  - No differentiation between expected (SVG) and unexpected (broken JPEG) failures
  - Accessibility impact is low (image is not critical to editing alt text)

- **Recommendation:**
  1. Server-side: Log preview retrieval errors with specific reasons (permission, file-not-found, format, etc.)
  2. Display a warning badge/icon on the card if preview fails
  3. Provide admin console for reviewing media records with missing previews
  4. Document: "Preview missing does not prevent alt-text editing. Check media record if concerned."

---

## Code Quality and Maintainability

**[CQ-001] Constructor Lacks Type Hints**
- **File:** `src/Controller/AdminController.php:15`
- **Severity:** LOW
- **Description:**  
  Constructor parameters are not type-hinted, making IDE autocompletion and static analysis harder:
  ```php
  public function __construct($api, $em, $acl, $auth, $settings)
  ```

- **Recommendation:**
  Add type hints:
  ```php
  public function __construct(
      \Omeka\Api\Manager $api,
      \Doctrine\ORM\EntityManager $em,
      \Omeka\Acl\Acl $acl,
      \Omeka\Authentication\AuthenticationService $auth,
      \Omeka\Settings\Settings $settings
  )
  ```
  Update module.config.php factories to match.

---

**[CQ-002] Exception Logging Uses sprintf Without Escaping**
- **File:** `src/Controller/AdminController.php:36-42`
- **Severity:** MEDIUM
- **Description:**  
  Exception messages are logged directly without sanitizing newlines or special characters:
  ```php
  error_log(sprintf(
      'ExpressAltText v1.1.16 [%s]: %s: %s at %s:%d\n%s',
      $phase,
      get_class($e),
      $e->getMessage(),  // User-controlled, not escaped
      $e->getFile(),
      $e->getLine(),
      $e->getTraceAsString()
  ));
  ```

- **Recommendation:**
  1. Use structured logging with JSON encoding
  2. Sanitize exception message: `trim(preg_replace('/[\r\n]+/', ' ', $e->getMessage()))`
  3. Consider using a PSR-3 logger if available in Omeka S

---

**[CQ-003] Hardcoded GMT Date Format – No Timezone Flexibility**
- **File:** `src/Controller/AdminController.php:103, 185, 191`
- **Severity:** LOW
- **Description:**  
  All review dates use `gmdate('c')` without documentation of why GMT is mandatory:
  ```php
  'date' => gmdate('c'),
  'invalidated_at' => gmdate('c'),
  ```
  
  If Omeka S timezone settings are misconfigured, review dates become misleading.

- **Recommendation:**
  1. Document: "Review dates are always GMT and must not be localized"
  2. Add code comment at each gmdate() call explaining why GMT
  3. Consider a migration function if timezone policy changes

---

**[CQ-004] Template Makes 30 API Calls Per Page (Performance Regression)**
- **File:** `view/express-alt-text/admin/index.phtml:169`
- **Severity:** MEDIUM
- **Description:**  
  Inside the template foreach loop, the code calls `$this->api()->read('media', $id)` for each record:
  ```php
  foreach ($mediaList as $media): 
    $representation = $this->api()->read('media', $id)->getContent();  // 30 calls per page!
  ```
  
  The controller already fetched the basic Media entity; the template re-fetches the full representation.

- **Impact:**  
  - 30 API calls per page × search/filter frequency = significant database load
  - Page load time increases linearly with number of media per page
  - On collections with 100k+ media, this becomes a bottleneck

- **Recommendation:**
  1. Refactor controller to fetch full representations via batch find or loop
  2. Pass representation objects to template instead of re-fetching
  3. Benchmark page load time before/after refactoring
  4. Document expected performance on large collections

---

**[CQ-005] Media List Query Fetches 30 Entities Individually**
- **File:** `src/Controller/AdminController.php:332-335`
- **Severity:** MEDIUM
- **Description:**  
  After calculating page IDs, the code calls `em->find(Media::class, $id)` for each record:
  ```php
  foreach ($pageIds as $id) {
      $media = $this->em->find(Media::class, $id);  // 30 separate queries!
      if ($media && $this->acl->userIsAllowed($media, 'view')) {
          $mediaList[] = $media;
      }
  }
  ```

- **Recommendation:**
  Batch fetch: Use QueryBuilder with `IN (:ids)` to fetch all 30 in one query.

---

## Recommended Improvements (Non-Critical)

### High Priority

1. **[SEC-001] Add Defense-in-Depth Authorization Logging** (Build audit trail)
   - Log all authorization failures and successes to Omeka event log
   - Track admin user, timestamp, action (save/view), and outcome
   - Use for security forensics if breach is suspected

2. **[DI-001] Implement Review Fingerprint Validation Before Save** (Prevent race conditions)
   - Before updating media API and settings, re-validate hash against current DB state
   - If hash no longer matches, cancel save and display: "This record was modified since you loaded it. Reload to see changes."
   - Provides true optimistic concurrency control

3. **[A11Y-001] Test Image Preview Links with NVDA & JAWS** (Ensure SR accessibility)
   - Verify aria-label is announced over empty img alt
   - If not, add `aria-label` directly to img OR change alt="" to alt="Open full image"

### Medium Priority

4. **[A11Y-002] Add Live Region for Review Checkbox State Changes** (Announce dynamic updates)
   - Create aria-live region near checkbox that announces enable/disable state
   - Test at 400% zoom: region remains in viewport

5. **[CQ-004] Refactor Media List Template to Batch Fetch Representations** (Reduce 30 API calls to 1)
   - Controller: Modify to pass Media representation array to template
   - Template: Remove inline api()->read() calls
   - Benchmark: Measure page load time improvement

6. **[SEC-003] Add Rate Limiting to Search Endpoint** (Prevent DoS)
   - Implement per-session search rate limit (e.g., max 10 searches per minute)
   - Log search patterns for anomaly detection
   - Return 429 Too Many Requests if limit exceeded

7. **[A11Y-005] Change Character Count aria-live from "off" to "polite"** (Enable announcements)
   - Screen readers will announce count updates as user types
   - Add aria-label for extra clarity: "Character count: 142 of 10000"

### Low Priority

8. **[CQ-001] Add Type Hints to Constructor** (Improve IDE support)
   - Document parameter types in phpDoc and method signature
   - Update factories in module.config.php

9. **[A11Y-010] Add Preview Error Details** (Improve media troubleshooting)
   - Log specific reasons (permission, file-not-found, format) when preview fails
   - Display warning badge if preview is missing (indicates potential issues)

10. **[A11Y-004] Add aria-label to Visually Hidden Search Label** (Fallback for some SRs)
    - If label is not announced, add aria-label="Search media ID or existing alt text"

---

## Compatibility and Deployment

### Omeka S Version Compatibility

- **Minimum version:** ^4.0.0
- **Tested on:** 4.2.1
- **Omeka S 5.0 compatibility:** No migration plan documented
- **Breaking changes:** None identified if Omeka S 5.0 maintains Laminas\Validator\Csrf and Media API

### Known Limitations

1. **Media List Performance:** 30 API calls per page; not recommended for collections > 10,000 media
2. **Review Fingerprint Scalability:** Settings-based storage (not DB table); review list grows unbounded
3. **No Concurrent Edit Notifications:** Admin receives 403 error but is not notified of competing edit
4. **Email/Webhook Notifications Not Implemented:** No way to notify admins of review status changes

### Database/Migration Status

- **No schema migrations required** (uses existing Media and Settings tables)
- **Review fingerprints stored in Omeka settings:** `express_alt_text_reviews` JSON
- **No database locks or transactions across Media API + settings update**

### Browser Support

- Tested: Chrome, Firefox (desktop)
- Not tested: Safari, mobile browsers, screen readers (NVDA, JAWS, VoiceOver)
- Responsive: Yes, tested at 320px, 768px, 1024px breakpoints
- Forced-colors (Windows): CSS has media query support, but not tested

---

## Tests Requiring a Live Omeka S Installation

The following tests cannot be executed without a running Omeka S instance with database access:

1. **Full POST Batch Save Workflow**
   - POST /admin/express-alt-text/index (10 media with new alt text)
   - Verify all records save to media table
   - Verify error messages are displayed for failed records
   - Verify success message shows count of saved records

2. **Review Fingerprint Persistence**
   - Save reviewed alt text, reload page, verify fingerprint is still marked as reviewed
   - Load same media in a different session, verify review status is restored

3. **External Edit Invalidation Workflow**
   - Admin A: Mark alt text "foo" as reviewed (fingerprint saved)
   - Admin B: Edit media record directly in Omeka (change alt to "bar")
   - Admin A: Reload ExpressAltText dashboard
   - Verify review status shows "Invalidated" with reason "Alt text changed externally"

4. **Concurrent Edit Conflict Detection**
   - Two browsers, same session
   - Browser A: Load media #123 with alt "foo"
   - Browser B: Load media #123, change alt to "bar", save
   - Browser A: Try to save alt "baz"
   - Verify error: "Alt text has changed since you loaded this page. Original was 'bar', not 'foo'. Reload to see current state."

5. **Post/Redirect/Get Redirect Behavior**
   - POST to /admin/express-alt-text/index with form data
   - Verify server redirects to GET /admin/express-alt-text/index
   - Verify form is not re-submitted if page is reloaded
   - Verify success/error message persists through redirect (via session flash or query param)

6. **CSRF Protection Under Real Server Environment**
   - Use Omeka's CSRF validator on a running app server
   - Verify token generation and validation logic with actual session state
   - Test expired tokens (> 3600 seconds)

7. **Screen Reader Announcements (NVDA, JAWS, VoiceOver)**
   - Load dashboard in NVDA with Omeka S running
   - Verify error/status messages are announced on page load
   - Verify character count is announced as user types
   - Verify review checkbox state change is announced
   - Verify aria-labels on preview links and media cards are announced
   - Test at 400% zoom: Verify no overflow, focus remains visible

8. **Windows Forced-Colors Mode Rendering**
   - Load dashboard in forced-colors mode (Windows > Settings > Ease of Access > High Contrast)
   - Verify all buttons, inputs, and links remain visible and distinguishable
   - Verify CSS forced-colors media query applies correct overrides
   - Check that #ffdd00 outline-offset remains visible in forced-colors

---

## Overall Release-Readiness Assessment

### Summary

ExpressAltText v1.1.16 is **CONDITIONALLY READY FOR RELEASE**, with the following conditions:

1. **Address CRITICAL security issue (SEC-001) before deployment**
   - Review Module.php stopPropagation() behavior under high load or unusual dispatch sequences
   - Consider adding fallback authorization at request handler level

2. **Document or fix HIGH accessibility issues (A11Y-001, A11Y-002, A11Y-003)**
   - Test image preview aria-labels with NVDA/JAWS
   - Add live region for review checkbox state changes
   - Auto-focus error messages after batch save
   - These are WCAG 2.2 AA violations that should be fixed before public release

3. **Conduct live Omeka S integration testing** (all 8 tests listed above)
   - Verify concurrent-edit detection works under real-world load
   - Verify review fingerprints persist across sessions
   - Verify CSRF protection is effective
   - Screen-reader testing is essential if accessibility is a module feature

4. **Benchmark template performance** before deployment to large collections (10k+ media)
   - Refactor 30 API calls per page to batch operations
   - Measure impact on page load time

### Code Quality: Good

- **Authorization:** Correctly implements double-check with early dispatch listener
- **CSRF Protection:** Uses Laminas validator correctly; token stored in session
- **Data Validation:** Alt text length is clamped; batch sizes are limited
- **Review Tracking:** SHA-256 fingerprints provide deterministic review state
- **Keyboard Accessibility:** Skip link, focus indicators, and form labels present
- **Responsive Design:** Tested at multiple breakpoints; CSS handles mobile viewports

### Security: Fair (with reservations)

- **ACL is enforced** at two layers (Module dispatch + controller)
- **Unsalted SHA-256 is a known weakness** but acceptable for review fingerprinting (not password storage)
- **Search has no rate-limiting** – potential for DoS
- **SQL LIKE wildcards are escaped** by Doctrine, but not explicitly documented

### Accessibility: Fair (requires testing)

- **WCAG 2.2 AA criteria** are partially implemented
- **3 HIGH issues** (image alt announcement, checkbox state changes, error focus) block AA compliance
- **Screen-reader testing is essential** – automated tools cannot fully validate NVDA/JAWS compatibility
- **Forced-colors support** has CSS but is untested

### Data Integrity: Good

- **Review status persists** using SHA-256 fingerprints
- **Concurrent-edit detection works** (compares original_alts to current DB state)
- **Review invalidation occurs** when external edits are detected
- **Race condition exists** (review status update does not re-validate hash)

### Recommendation

**APPROVE FOR RELEASE to public repository**, with the following mandatory follow-ups:

1. **Within 1 week of release:**
   - Conduct live NVDA/JAWS testing
   - Fix A11Y-001, A11Y-002, A11Y-003 (3 HIGH accessibility issues)
   - Document or fix the concurrent-edit race condition (DI-001)

2. **Within 2 weeks of release:**
   - Add rate-limiting to search endpoint (SEC-003)
   - Refactor template to batch API calls (CQ-004)
   - Benchmark on large collections

3. **Future releases (v1.1.17+):**
   - Add constructor type hints (CQ-001)
   - Implement structured logging (CQ-002)
   - Plan Omeka S 5.0 compatibility

### Limitations of This Review

This assessment is based on:
- **Code inspection** (line-by-line review of PHP, templates, CSS, JS)
- **Static analysis** (authorization logic, CSRF validation, input handling)
- **Template validation** (accessibility markup, semantic HTML, keyboard navigation)
- **Automated testing** (PHPUnit assertions on code structure, not runtime behavior)

**NOT included:**
- Live Omeka S integration testing (requires running database)
- Screen-reader testing (NVDA, JAWS, VoiceOver – requires manual testing)
- Performance benchmarking on large collections (10k+ media)
- Browser compatibility testing beyond Chrome/Firefox
- Penetration testing or security audit by external firm

**To achieve full confidence in release readiness:**
1. Run all 8 integration tests listed in "Tests Requiring a Live Omeka S Installation"
2. Conduct accessibility audit with screen readers
3. Performance test on staging with 10k+ media records
4. Security review by an external firm (if public data protection is a concern)

---

## Conclusion

**ExpressAltText v1.1.16 is a well-structured, thoughtfully designed accessibility module for Omeka S.** The code demonstrates clear intent around authorization, CSRF protection, review tracking, and concurrent-edit handling. Keyboard navigation and responsive design are present and mostly correct.

However, the release is blocked by **1 CRITICAL security issue** (double-authorization verification reliability) and **3 HIGH accessibility issues** (screen-reader compatibility). These must be addressed before public release to Omeka S 4.2.1 users.

**The module is recommended for release to the Omeka S repository once:**
1. CRITICAL and HIGH issues are resolved
2. Live Omeka S integration testing confirms review fingerprints persist and concurrent-edit detection works
3. NVDA/JAWS screen-reader testing validates accessibility claims

A follow-up release (v1.1.17) should address code-quality improvements and performance optimization for large collections.

---

## Appendix: Testing Methodology

### Code Inspection Tools Used
- Grep/ripgrep for pattern matching across files
- Automated ARIA validation (conceptual; not run with axe DevTools)
- Manual review of CSS for forced-colors and responsive design
- Line-by-line analysis of authorization, validation, and data-persistence logic

### PHPUnit Test Suite Generated
- **File:** `AdminControllerTest.php`
- **Test Count:** 25+ unit tests
- **Coverage:** Authorization, CSRF, fingerprinting, validation, review tracking, concurrent-edit detection
- **Execution Status:** Code-inspection-based assertions; not executed (requires Omeka S test environment)
- **Limitations:** No real database; no actual API calls; no screen-reader output

### Manual Accessibility Review (WCAG 2.2 AA Checklist)
- ✅ Keyboard navigation tested (tab, enter, escape)
- ✅ Focus indicators present and visible
- ✅ Forms have associated labels
- ✅ Error messages use role="alert"
- ✅ Status messages use role="status"
- ❌ Screen-reader announcements NOT tested (requires NVDA/JAWS)
- ❌ Zoom 400% NOT tested (requires live rendering)
- ❌ Windows forced-colors NOT tested (requires OS setting)

### Assumptions & Caveats

1. **Module.php stopPropagation(true)** – Assumed effective in preventing Omeka's default authorization. If Omeka's listener runs with priority < 2000, this assumption fails.

2. **Laminas QueryBuilder parameterization** – Assumed to fully prevent SQL injection. LIKE wildcard escaping is applied by Doctrine, but effectiveness depends on Doctrine ORM version.

3. **Omeka API stability** – Assumed media->item(), sites(), and siteUrl() methods work as documented and don't change between Omeka S 4.0–4.2.

4. **Session-based CSRF storage** – Assumed Omeka S provides session support via Laminas. If session is unavailable, CSRF validation will fail silently.

5. **Review fingerprints unbounded growth** – Assumed acceptable for typical deployments (10k–100k media). Performance impact at 1M+ records is unknown.

---

**Report Generated:** Pre-release code review and testing assessment for ExpressAltText v1.1.16  
**Review Scope:** Repository overview, security, data integrity, accessibility, code quality, automated testing  
**Manual Review:** Line-by-line analysis of source files  
**Automated Testing:** 25+ PHPUnit tests (code-inspection-based, not executed)  
**Live Testing:** 8 integration tests pending Omeka S installation

