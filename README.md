# RESTful API Planning Document
## Drug Recall Watchlist – openFDA Client

This document plans the client side of a REST-style service that consumes a public API from the [public-apis](https://github.com/public-apis/public-apis) repository. The client fetches U.S. FDA drug recall data, displays it in a dynamic table, and lets the user save medications to a watchlist through a validated form. Saved items are sent as JSON to a Python (Flask) backend, whose contract is described here but not built.

---

## 1. API Selection

**Chosen API:** openFDA, from the Health category of the public-apis repository.

| Item | Detail |
|---|---|
| Provider | U.S. Food and Drug Administration |
| Endpoint | `https://api.fda.gov/drug/enforcement.json` (drug recall enforcement reports) |
| Format | JSON over HTTPS |
| Authentication | Optional API key, sent as the `api_key` query parameter |
| Rate limits | Without a key: 240 requests per minute and 1,000 per day per IP address. With a free key: 240 per minute and 120,000 per day |
| Documentation | https://open.fda.gov/apis/drug/enforcement/ |

**Purpose:** openFDA publishes official FDA data about drugs, medical devices and foods. The Drug Enforcement endpoint returns recall reports, including what was recalled, why, how serious the recall is, and its current status.

**Why it suits this assignment:**
- **Free and open.** It works without a key, so setup is simple.
- **RESTful.** It uses standard GET requests with query parameters (`search`, `sort`, `limit`) and returns consistent JSON.
- **Clinically relevant.** It continues the health-data theme of this course.
- **Good for UI logic.** Fields such as `classification` (Class I, II, III) and `status` (Ongoing, Completed, Terminated) are natural candidates for conditional styling.
- **Safe.** It contains no patient-identifiable information.

**Fields the client will use:**

| Field | Description | Example |
|---|---|---|
| `recall_number` | Unique recall ID | `D-0595-2026` |
| `product_description` | Product recalled | `Metformin HCl ER Tablets, 500 mg…` |
| `reason_for_recall` | Reason for the recall | `Failed impurities specifications` |
| `classification` | Severity: Class I is the most serious | `Class II` |
| `status` | Recall status | `Ongoing` |
| `recalling_firm` | Company issuing the recall | `Example Pharma Inc.` |
| `report_date` | Report date (`YYYYMMDD`) | `20260604` |

---

## 2. Endpoint Mapping

### 2a. External endpoint (public API, read-only)

| Verb | Endpoint | Parameters | Returns |
|---|---|---|---|
| `GET` | `https://api.fda.gov/drug/enforcement.json` | `search=openfda.generic_name:"<drug>"`, `sort=report_date:desc`, `limit=<1–100>`, optional `api_key` | `{ meta: {...}, results: [ recall, ... ] }` |

Example:
```
GET https://api.fda.gov/drug/enforcement.json?search=openfda.generic_name:"metformin"&sort=report_date:desc&limit=10
```

Note: when nothing matches, openFDA returns **HTTP 404** with error code `NOT_FOUND`. The client will treat this as "no recalls found", not as a failure.

### 2b. Backend endpoints (Python/Flask contract, to be built later)

| Verb | Endpoint | Accepts | Returns | Status codes |
|---|---|---|---|---|
| `GET` | `/api/recalls?drug=<name>&limit=<n>` | Query parameters | Normalized recall list (an optional proxy for openFDA) | `200`, `400`, `502` |
| `GET` | `/api/watchlist` | — | Array of saved watchlist items | `200` |
| `GET` | `/api/watchlist/<id>` | — | A single item | `200`, `404` |
| `POST` | `/api/submit` (alias `/api/watchlist`) | JSON watchlist item | The created item with `id` and `created_at` | `201`, `400`, `409`, `422` |
| `PUT` | `/api/watchlist/<id>` | JSON fields to update | The updated item | `200`, `404`, `422` |
| `DELETE` | `/api/watchlist/<id>` | — | Empty body | `204`, `404` |

**Response shape for a saved item:**
```json
{
  "id": 1,
  "drug_name": "metformin",
  "name_type": "generic",
  "priority": "high",
  "min_classification": "Class II",
  "notes": "Used by diabetes clinic",
  "created_at": "2026-11-10T14:40:00Z"
}
```

**Error response shape (all endpoints):**
```json
{ "error": "Validation failed", "details": [ { "field": "drug_name", "message": "Required" } ] }
```

---

## 3. Data Flow Diagram (textual)

```
[User enters drug name and clicks "Search"]
          │
          ▼
(1) Validate the search form locally ──invalid──► show inline field errors and stop
          │ valid
          ▼
(2) Enter the LOADING state
    - disable the Search button and show a spinner with "Loading recalls…"
    - clear the old table and error banner
          │
          ▼
(3) async fetch()  GET api.fda.gov/drug/enforcement.json?search=...&limit=...
    (wrapped in try/catch, with a 10-second timeout via AbortController)
          │
     ┌────┴───────────────┬──────────────────────┬───────────────────────┐
     ▼                    ▼                      ▼                       ▼
  200 OK             404 NOT_FOUND           429 Too Many            Network error,
  with results       (no matches)            Requests                timeout, 5xx
     │                    │                      │                       │
     ▼                    ▼                      ▼                       ▼
(4) PROCESS          EMPTY state:            ERROR state:            ERROR state:
  - check that       "No recalls found       "Rate limit reached,    "Could not reach
    results is an      for metformin"          try again in a          openFDA" with a
    array                                      minute"                 Retry button
  - keep only the
    needed fields
  - format dates as
    YYYY-MM-DD
  - sort newest first
     │
     ▼
(5) RENDER
  - build table rows with document.createElement and textContent
    (never innerHTML with API data, to prevent script injection)
  - show the summary "Showing 10 recalls for metformin"
  - apply conditional styling (below)
          │
          ▼
(6) FINALLY: leave the loading state (hide the spinner, enable the button)
```

**Loading states**

| State | What the user sees |
|---|---|
| Idle | Search form, empty results area |
| Loading | Spinner, "Loading recalls…", Search button disabled, `aria-busy="true"` on the results area |
| Success | Results table and summary line |
| Empty | Neutral message: "No recalls found" |
| Error | Red banner with a clear message and a **Retry** button |

**Error handling rules**
- `fetch()` only rejects on network failure, so the client also checks `response.ok`.
- A `404` from openFDA becomes the Empty state.
- A `429` shows a rate-limit message and disables Retry for 60 seconds.
- An unexpected JSON shape, a timeout or a `5xx` response shows the Error state with Retry. The details are logged with `console.error`.
- Every path ends in a `finally` block so the UI never stays stuck in Loading.

**Conditional styling logic**

| Condition | Styling |
|---|---|
| `classification === "Class I"` | Red row highlight and a red **Class I – Serious** badge |
| `classification === "Class II"` | Orange badge |
| `classification === "Class III"` | Yellow badge |
| `status === "Ongoing"` | Bold, with an **Ongoing** pill |
| `status === "Terminated"` or `"Completed"` | Greyed-out text |
| Report date within the last 90 days | **New** tag |
| Drug is already on the watchlist | **Add to Watchlist** button disabled and labelled **Saved** |
| Form field is invalid | Red border, error text below it, and `aria-invalid="true"` |

Styling is done by adding CSS classes (for example, `row--class1`), not inline styles. Badges also contain text, so meaning never depends on color alone.

---

## 4. Form Handling

### 4a. Recall Search form (feeds the GET request)

| Control | Type | Validation (local) |
|---|---|---|
| Drug name | `<input type="text">` | Required. 2–100 characters. Letters, numbers, spaces and hyphens only. Trimmed |
| Name type | `<select>`: Generic / Brand | Required. Default: Generic |
| Results limit | `<input type="number">` | Whole number from 1 to 100. Default: 10 |
| Classification filter | `<select>`: All / Class I / Class II / Class III | Optional. Applied in the browser after the fetch |

The values are turned into query parameters with `URLSearchParams`, and the drug name is wrapped in quotes for openFDA's search syntax.

### 4b. Add to Watchlist form (feeds the POST request to the backend)

| Control | Type | Validation (local) |
|---|---|---|
| Drug name | `<input type="text">` | Required, same rules as the search form. Pre-filled from the search |
| Name type | `<select>` | Required: `generic` or `brand` |
| Priority | Radio buttons: Low / Medium / High | Required |
| Alert threshold | `<select>`: Class I / Class II / Class III | Required. The minimum recall class to flag |
| Notes | `<textarea>` | Optional. Maximum 255 characters, with a live character counter |
| Confirm | `<input type="checkbox">` "I understand this data is not medical advice" | Must be checked |

**Validation behavior**
- Validate each field on `blur` and all fields on `submit`. HTML5 attributes (`required`, `minlength`, `maxlength`, `pattern`) are backed up by JavaScript checks.
- The Submit button stays disabled until the form is valid.
- Errors appear inline under each field and are announced with `aria-live="polite"`.
- No personal or patient data is collected.

**Transforming the form into a JSON payload**

```js
const form = new FormData(watchlistForm);
const payload = {
  drug_name: form.get("drug_name").trim().toLowerCase(),
  name_type: form.get("name_type"),                 // "generic" | "brand"
  priority: form.get("priority"),                   // "low" | "medium" | "high"
  min_classification: form.get("min_classification"), // "Class I" | "Class II" | "Class III"
  notes: form.get("notes").trim() || null,
  acknowledged_disclaimer: form.get("confirm") === "on"
};

const response = await fetch("/api/submit", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(payload)
});
```

**Contract with the Python backend**

| Item | Contract |
|---|---|
| Request | `POST /api/submit`, `Content-Type: application/json`, body as above |
| Backend validation | A Pydantic schema enforces the same rules as the form (the backend never trusts the client) |
| Success | `201 Created` with the saved item, including `id` and `created_at` |
| Duplicate drug | `409 Conflict`. The client shows "Already on your watchlist" |
| Invalid data | `422` with `details[]`. The client maps each error back to its field |
| Server error | `500`. The client shows a banner and keeps the form data so the user can retry |

---

## 5. Technology Stack

| Area | Tools |
|---|---|
| Markup and styling | HTML5 and CSS3 (Flexbox/Grid, CSS classes for conditional styling) |
| Client logic | Vanilla JavaScript (ES2020+): `fetch`, `async`/`await`, `AbortController`, `URLSearchParams`, `FormData` |
| Backend (contract only) | Python 3, **Flask** (serves the page and the `/api/*` endpoints), Pydantic (validation), SQLAlchemy + SQLite (storage) |
| Version control | Git and GitHub |
| Editor | Visual Studio Code with the Live Server extension |
| Testing: client | **Jest** with jsdom for unit tests (form validation, data transformation, styling rules), using a mocked `fetch` |
| Testing: API | **curl** and **Postman** to check openFDA requests and the backend contract |
| Testing: backend | **pytest** with Flask's `test_client()` once the backend is built |
| Testing: manual | Browser DevTools (Network tab for requests, Console for errors, network throttling to see loading states) |
| Quality | Lighthouse (accessibility and performance) and the W3C HTML validator |
| CI | GitHub Actions running the test suite on every push |

---

## 6. Implementation Steps

**Step 1 – Set up the project**
- Create the `restful-api-planning` repository with `README.md` and `.gitignore`.
- Create the folder structure: `index.html`, `css/styles.css`, `js/api.js`, `js/ui.js`, `js/form.js`, `tests/`.
- Create a Python virtual environment and a minimal Flask app that serves `index.html`.
- Test a call to openFDA with curl to confirm the response shape.

**Step 2 – Implement the fetch GET with loading and error states**
- Write `fetchRecalls(drug, nameType, limit)` in `api.js` using `async`/`await`, `response.ok` checks and a 10-second `AbortController` timeout.
- Handle `200`, `404` (empty), `429` and network or `5xx` errors as described in Section 3.
- Add the Loading, Empty and Error UI states with a `finally` block.
- Check in the browser whether openFDA allows direct calls (CORS). If it doesn't, route requests through the Flask `/api/recalls` proxy.

**Step 3 – Render dynamic lists and tables**
- Normalize the records (select fields, format dates, sort).
- Build the results table with `createElement` and `textContent`.
- Add the summary line and the in-browser classification filter.
- Show the watchlist as a list loaded from `GET /api/watchlist`, using mock data until the backend exists.

**Step 4 – Add form controls and validation**
- Build the Search and Add to Watchlist forms from Section 4.
- Add HTML5 validation attributes plus JavaScript validators on blur and submit.
- Add inline error messages, the character counter and the disabled-until-valid Submit button.
- Convert the form data to the JSON payload and POST it to `/api/submit`. Handle `201`, `409`, `422` and `500` responses.

**Step 5 – Integrate conditional styling**
- Add CSS classes for classification badges, status pills, the **New** tag, the **Saved** state and invalid fields.
- Apply the classes in the render functions based on the data.
- Check color contrast and confirm every badge has text.

**Step 6 – Final review and documentation**
- Write Jest tests for the validators, data transformation and styling rules, and run them in GitHub Actions.
- Test manually: slow-network throttling, offline mode, an empty result, invalid input and a rate-limit message.
- Run Lighthouse and fix any accessibility issues.
- Update this README with setup instructions, screenshots and any changes to the plan, then push the final commit.

---

## References
- public-apis repository: https://github.com/public-apis/public-apis
- openFDA API overview: https://open.fda.gov/apis/
- openFDA authentication and rate limits: https://open.fda.gov/apis/authentication/
- openFDA Drug Enforcement endpoint: https://open.fda.gov/apis/drug/enforcement/

*Disclaimer: openFDA data is for information only and must not be used to make medical decisions.*
