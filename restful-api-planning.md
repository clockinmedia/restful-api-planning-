# RESTful API Planning Document: Drug Recall Watchlist Service

## 1. Overview

**Goal:** Build a small REST-style service that lets a user keep a *watchlist* of medications and check each one against official FDA drug recall data.

The service is a **client** of a public API (openFDA) and also exposes its own **CRUD** endpoints for managing the watchlist. It builds on earlier work in this course: Flask, Pydantic validation, and dual SQLite + JSON Lines storage.

**Example use case:** A clinic coordinator adds "metformin" and "lisinopril" to the watchlist, then calls one endpoint to see whether either has recent FDA recall enforcement reports.

---

## 2. Selected Public API

| Item | Detail |
|---|---|
| API | **openFDA** (Health category, [public-apis](https://github.com/public-apis/public-apis)) |
| Provider | U.S. Food and Drug Administration |
| Endpoint used | `https://api.fda.gov/drug/enforcement.json` (drug recall enforcement reports) |
| Format | JSON over HTTPS (GET only) |
| Auth | Optional API key passed as `api_key` query parameter |
| Rate limits | No key: 240 requests/min and 1,000/day per IP. With a free key: 240/min and 120,000/day |
| Docs | https://open.fda.gov/apis/drug/enforcement/ |

**Why openFDA:** It is free, authoritative, well documented, contains no patient-identifying data, and fits the clinical theme of this course.

**Key response fields used:**

| openFDA field | Meaning |
|---|---|
| `recall_number` | Unique recall identifier, e.g. `D-0595-2026` |
| `product_description` | Product that was recalled |
| `reason_for_recall` | Why it was recalled |
| `classification` | `Class I` (most serious), `Class II`, `Class III` |
| `status` | `Ongoing`, `Completed`, or `Terminated` |
| `recalling_firm` | Company issuing the recall |
| `report_date` | Date in `YYYYMMDD` format |

**Example external request:**

```
GET https://api.fda.gov/drug/enforcement.json?search=openfda.generic_name:"metformin"&sort=report_date:desc&limit=5
```

---

## 3. Architecture

```
 ┌──────────────┐   HTTP/JSON   ┌───────────────────────┐   HTTPS GET   ┌──────────────┐
 │ Client       │ ────────────► │ Flask Watchlist API   │ ────────────► │ openFDA API  │
 │ (browser UI, │ ◄──────────── │ (Python)              │ ◄──────────── │ api.fda.gov  │
 │  curl)       │               │  - Pydantic schemas   │               └──────────────┘
 └──────────────┘               │  - openFDA client     │
                                │  - SQLite (watchlist) │
                                │  - JSONL audit log    │
                                └───────────────────────┘
```

| Layer | Technology |
|---|---|
| Web framework | Flask |
| Validation | Pydantic |
| Database | SQLite via SQLAlchemy (`watchlist.db`) |
| Audit log | JSON Lines (`data/audit_log.jsonl`) |
| HTTP client | `requests`, with timeouts and retries |
| Front end | Single HTML page using `async`/`await` `fetch()` |
| Tests | `pytest`, with `unittest.mock` to fake openFDA |

**Project structure:**

```
restful-api-planning/
├── app.py               # Flask routes
├── models.py            # SQLAlchemy model
├── schemas.py           # Pydantic schemas
├── fda_client.py        # Wrapper around openFDA calls
├── static/index.html    # Front-end client
├── tests/               # pytest tests
├── requirements.txt
├── .env.example         # FDA_API_KEY=
└── restful-api-planning.md
```

---

## 4. Data Model

**Table: `watchlist_item`**

| Column | Type | Notes |
|---|---|---|
| `id` | Integer | Primary key |
| `drug_name` | String(100) | Generic or brand name, required |
| `name_type` | String(10) | `generic` or `brand` |
| `notes` | String(255) | Optional |
| `last_checked` | DateTime | When recalls were last fetched |
| `recall_count` | Integer | Recalls found at last check |
| `created_at` | DateTime | Set on create |
| `updated_at` | DateTime | Set on update |

**Pydantic schemas:** `WatchlistCreate` (input), `WatchlistUpdate` (all fields optional), `WatchlistRead` (output), and `RecallRead` (normalized openFDA record).

---

## 5. REST Endpoint Design (CRUD + External Data)

| Method | Endpoint | Purpose | Success | Errors |
|---|---|---|---|---|
| `POST` | `/watchlist/` | **Create** a watchlist item | `201` | `422` invalid data, `409` duplicate |
| `GET` | `/watchlist/` | **Read** all items | `200` | — |
| `GET` | `/watchlist/<id>` | **Read** one item | `200` | `404` |
| `PUT` | `/watchlist/<id>` | **Update** an item | `200` | `404`, `422` |
| `DELETE` | `/watchlist/<id>` | **Delete** an item | `204` | `404` |
| `GET` | `/watchlist/<id>/recalls` | Fetch openFDA recalls for that drug | `200` | `404`, `502`, `503` |
| `GET` | `/recalls?drug=<name>&limit=5` | Ad-hoc recall search | `200` | `400`, `502` |

**Example create request:**

```bash
curl -X POST http://127.0.0.1:5001/watchlist/ \
  -H "Content-Type: application/json" \
  -d '{"drug_name":"metformin","name_type":"generic","notes":"Diabetes patients"}'
```

**Example recalls response (normalized by our service):**

```json
{
  "drug_name": "metformin",
  "count": 1,
  "recalls": [
    {
      "recall_number": "D-0000-2026",
      "classification": "Class II",
      "status": "Ongoing",
      "reason_for_recall": "Example reason",
      "recalling_firm": "Example Pharma",
      "report_date": "2026-06-04"
    }
  ],
  "source": "openFDA",
  "disclaimer": "Public FDA data; not medical advice."
}
```

---

## 6. Consuming the External API (Client-Side Plan)

1. **Build the request:** use `params={...}` so `requests` handles URL encoding. Search `openfda.generic_name` or `openfda.brand_name` depending on `name_type`. Sort with `sort=report_date:desc` and cap results with `limit` (max 1,000).
2. **API key:** read `FDA_API_KEY` from an environment variable. If present, add it as `api_key`; if not, run keyless.
3. **Timeouts:** 10-second timeout on every call.
4. **Response handling:**

| openFDA result | Our service returns |
|---|---|
| `200` with `results` | `200` with normalized recalls |
| `404` `NOT_FOUND` (openFDA's response for *no matches*) | `200` with `count: 0`, an empty list |
| `429` rate limit | Retry with exponential backoff (1s, 2s, 4s), then `503` |
| `5xx` or timeout | `502 Bad Gateway` with a clear message |
| Unexpected JSON | `502` and log the error |

5. **Normalize:** convert `report_date` from `YYYYMMDD` to ISO `YYYY-MM-DD` and keep only the fields our schema defines.
6. **Cache:** keep results for each drug in memory for 15 minutes to stay well under the 1,000/day keyless limit.
7. **Front end:** an HTML page uses `async`/`await` `fetch()` with a loading indicator, error messages, and a table, following the pattern from the earlier Async Fetch assignment.

---

## 7. Interoperability Layers

| Layer | How this design addresses it |
|---|---|
| **Technical** | HTTP, REST verbs, JSON, HTTPS to openFDA, SQLite |
| **Semantic** | Pydantic schemas define field meaning and types. openFDA's `classification` and `status` values are passed through unchanged, and dates are normalized to ISO 8601 |
| **Organizational** | JSONL audit log of every create, update and delete. openFDA terms and disclaimers respected. No patient data stored |

---

## 8. Testing Plan

| Type | What is tested | Tool |
|---|---|---|
| Unit | Pydantic validation (empty name, bad `name_type`, too-long notes) | `pytest` |
| Unit | `fda_client` parsing, date conversion, and 404 handled as "no results" | `pytest` + `unittest.mock` |
| Unit | Retry logic on `429` and timeout handling | `pytest` + mock |
| Integration | Full CRUD cycle against a temporary SQLite database | Flask `test_client()` |
| Integration | Each endpoint's status codes (`201`, `200`, `204`, `404`, `409`, `422`, `502`) | Flask `test_client()` |
| Live smoke test | One real call to openFDA for a common drug (run manually, not in CI) | `curl` / `pytest -m live` |
| Manual | Front-end loading states and error messages | Browser |

**Rules:**
- Automated tests never call the real openFDA API. They use mocked responses so they are fast and don't use up rate limits.
- Aim for at least 80% coverage, checked with `pytest --cov`.
- Run every test on each push with GitHub Actions.

**Manual curl test script (happy path):**

```bash
curl -X POST http://127.0.0.1:5001/watchlist/ -H "Content-Type: application/json" -d '{"drug_name":"metformin","name_type":"generic"}'
curl http://127.0.0.1:5001/watchlist/
curl -X PUT http://127.0.0.1:5001/watchlist/1 -H "Content-Type: application/json" -d '{"notes":"Updated note"}'
curl http://127.0.0.1:5001/watchlist/1/recalls
curl -X DELETE http://127.0.0.1:5001/watchlist/1
```

---

## 9. Deployment Plan

1. **Local development:** `python3 -m venv venv`, `pip install -r requirements.txt`, `flask run --port 5001`. Port 5001 avoids the macOS AirPlay conflict on port 5000.
2. **Configuration:** settings come from environment variables (`FDA_API_KEY`, `DATABASE_URL`). `.env` is listed in `.gitignore`, and only `.env.example` is committed.
3. **Production server:** `gunicorn app:app` instead of Flask's development server.
4. **Hosting:** a free tier on Render or PythonAnywhere, with the API key set in the host's environment settings.
5. **CI/CD:** a GitHub Actions workflow runs `pytest` on every push and pull request. Deployment happens only when tests pass on `main`.
6. **Database:** SQLite is fine for this project. If it moves to multiple users, switch to PostgreSQL by changing `DATABASE_URL`.
7. **Monitoring:** log every openFDA call's status code and duration, and track remaining daily requests.

---

## 10. Security and Compliance

- Never commit the API key to GitHub.
- Validate all input with Pydantic before it reaches the database or openFDA.
- Use SQLAlchemy queries, never string-built SQL, to prevent SQL injection.
- Store no patient-identifiable information. The watchlist holds drug names only.
- Show the disclaimer that openFDA data is for information only and not for medical decisions.
- Enable CORS only for the front end's own origin.

---

## 11. Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Hitting the keyless rate limit (1,000/day) | Caching, a free API key, and backoff on `429` |
| openFDA is down or slow | Timeouts, a `502` response with a friendly message, and the last cached result shown |
| openFDA changes its field names | All parsing lives in one place (`fda_client.py`), plus a live smoke test |
| Drug-name mismatches (brand vs generic) | `name_type` field, and searching both fields when unsure |

---

## 12. Milestones

| Week | Milestone |
|---|---|
| 1 | Repository set up, models, schemas, CRUD endpoints, and unit tests |
| 2 | openFDA client, recalls endpoints, error handling, and mocked tests |
| 3 | Front-end page, GitHub Actions CI, and deployment |
| 4 | Final testing, documentation, and demo |

---

## 13. References

- public-apis list: https://github.com/public-apis/public-apis
- openFDA API overview: https://open.fda.gov/apis/
- openFDA authentication and rate limits: https://open.fda.gov/apis/authentication/
- openFDA Drug Enforcement endpoint: https://open.fda.gov/apis/drug/enforcement/
