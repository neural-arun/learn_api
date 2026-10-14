# 1. Fundamentals — Concept Notes

> Goal: understand **how and why** APIs work in FastAPI. Syntax is secondary; concepts are everything.

---

## 1.1 What is an API (the big picture)

**Concept:** An API (Application Programming Interface) is a contract. The client (browser, mobile app, another server) asks for something using a defined format, and the server promises to respond in a defined format.

**Real-world analogy:** A restaurant waiter.
- You (client) don't walk into the kitchen (server/database).
- You tell the waiter (API) your order (request: method + URL + data).
- The waiter brings back food (response: status code + body).

**Why it matters:** Every HTTP API is just: **Request in → Logic → Response out**. FastAPI is only a framework for writing that middle step cleanly.

---

## 1.2 FastAPI setup, virtual environment & Uvicorn

**Virtual environment:**
- Concept: a private, project-local Python installation. Different projects can use different package versions without breaking each other.
- Analogy: each project gets its own toolbox instead of sharing one messy garage.

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install fastapi uvicorn
```

**Uvicorn:**
- Concept: an **ASGI server** — the program that actually listens on a network port and delivers HTTP requests to your Python code.
- FastAPI *writes* the logic; Uvicorn *serves* it.
- Analogy: FastAPI is the chef; Uvicorn is the dining hall that accepts customers.

```bash
uvicorn main:app --reload
```
- `main:app` → file `main.py`, object `app`.
- `--reload` → restart on code change (dev only).

**Advanced concept — WSGI vs ASGI:**
- WSGI (Flask/Django) = one request at a time, sync only.
- ASGI (FastAPI) = can handle async code, WebSockets, long connections.
- This is *why* FastAPI is fast under concurrency — but only if you use `async def` correctly (Section 6).

---

## 1.3 Routes and HTTP methods

**Concept:** A **route** = URL + HTTP method that triggers a function.

```python
@app.get("/items/{item_id}")      # route
def read_item(item_id: int):      # handler
    ...
```

**The 5 standard methods (REST conventions):**

| Method   | Meaning              | Idempotent? | Typical use                |
|----------|----------------------|-------------|----------------------------|
| GET      | Read                 | Yes         | Fetch a list or one item   |
| POST     | Create               | No          | Add new resource           |
| PUT      | Replace (full update)| Yes         | Overwrite whole resource   |
| PATCH    | Partial update       | Yes         | Change 1-2 fields          |
| DELETE   | Remove               | Yes         | Delete resource            |

**Key concept — Idempotent:** doing the operation 10 times = same result as once. This matters for networks: browsers/proxies safely retry idempotent requests, but not POST.

**Real-world:** an online store
- `GET /products` → see the catalog (everyone can read)
- `POST /cart/items` → add to cart (creates something new)
- `PATCH /profile` → change only your phone number
- `DELETE /cart/items/9` → remove item 9

**Why not just one GET for everything?** Because the HTTP method carries *intent*. The server, logs, caches, and security layers can make decisions before your code even runs.

---

## 1.4 Path parameters, query parameters & request body

This is the single most important fundamentals concept: **three ways data enters your API.**

### Path parameter — *which resource*
```python
@app.get("/users/{user_id}")
```
- Part of the URL, required, defines **identity**: `/users/42`.
- Use for: selecting a specific resource.
- Rule of thumb: path = **noun identity**, never actions. Prefer `DELETE /users/42` over `GET /delete_user/42`.

### Query parameter — *how to modify the read*
```
GET /products?category=shoes&limit=20&sort=-price
```
- Comes after `?`, optional, used for **filtering, sorting, pagination, search**.
- Analogy: same book (path), but "page 5, English edition" (query) changes *which view* you get.
- Conceptually: path selects **which object**, query selects **which representation/variant**.

### Request body — *the data payload*
```python
@app.post("/products")
def create(product: Product):   # Pydantic model from JSON body
```
- Structured data sent (almost always as JSON) with POST/PUT/PATCH.
- Use for: creating or fully updating resources.
- GET requests should not have bodies — many proxies/servers ignore them.

**Decision cheat-sheet:**
- Identifying a thing → **path**
- Optional knobs on a request → **query**
- Actual data being created/sent → **body**

**Advanced:** FastAPI auto-converts types from the path/query (e.g. `item_id: int`), and if conversion fails you get a **422** automatically — validation for free (deep dive in Section 2).

---

## 1.5 HTTP status codes, headers and cookies

**Concept:** The status code tells the client *what happened* — it is the API's honest report card. Never return 200 for a failure.

**Families (memorize the first digit):**

| Range | Meaning | Examples |
|-------|---------|----------|
| 2xx | Success | `200 OK`, `201 Created`, `204 No Content` |
| 3xx | Redirection / caching | `301 Moved`, `304 Not Modified` |
| 4xx | **Client's fault** | `400 Bad Request`, `401 Unauthenticated`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity` |
| 5xx | **Server's fault** | `500 Internal Server Error`, `503 Unavailable` |

**Real-world distinction that confuses everyone:**
- **401** = "I don't know who you are" (no/invalid login) → go log in.
- **403** = "I know who you are, and you're not allowed" → forbidden.
- **404** = "that thing doesn't exist (or I'm hiding it)".
- **422** = "the request is syntactically fine but the data failed validation" (FastAPI's signature).

**Headers** = metadata alongside the request/response (key: value).
- Request headers: `Authorization: Bearer <token>`, `Content-Type: application/json`, `Accept`.
- Response headers: `Content-Type`, `Cache-Control`, custom `X-Request-Id`.
- Concept: headers describe **how to handle** the message; the body is **the message**.

**Cookies** = a small piece of data the server asks the browser to store and send back automatically on every request.
- Concept: HTTP is **stateless** (each request is independent, the server "forgets" you). Cookies are one trick to remember identity across requests (classic session login). Modern APIs more often use JWT tokens in the `Authorization` header instead (Section 5).

**Advanced:** `Set-Cookie` with `HttpOnly`, `Secure`, `SameSite` flags defends against XSS/CSRF — security lives in headers too.

---

## 1.6 Swagger UI, ReDoc & OpenAPI

**Concept chain:**
- **OpenAPI** = a JSON/YAML *specification* — a machine-readable description of your API (routes, params, schemas, examples).
- FastAPI generates this **automatically** from your code and type hints.
- **Swagger UI** (`/docs`) = interactive playground generated from that spec.
- **ReDoc** (`/redoc`) = pretty read-only documentation page.

**Why this is a big deal (the real reason FastAPI exists):**
Your code is the single source of truth. Add a route → docs update themselves. Clients can test endpoints without you writing a single line of doc.

**Principles behind it:**
1. **Self-documenting code** — type hints and Pydantic models *are* the documentation.
2. **Contract-first thinking** — the API surface is visible and reviewable.
3. **Separation**: OpenAPI (spec) ≠ Swagger (UI for the spec) — people use the words interchangeably, but they're different things.

**Advanced:**
- In production you can disable `/docs` for security: `FastAPI(docs_url=None, redoc_url=None)`.
- Customize with `title`, `version`, `openapi_tags`, and `responses={...}` to document possible errors per endpoint.

---

## 1.7 How it all fits together (request lifecycle)

```
Client sends request
   → Uvicorn receives it
      → FastAPI matches method + URL to a route
         → path/query/body are parsed & validated
            → your function runs (returns data)
               → FastAPI serializes it to JSON
                  → status code + headers + body sent back
```

Knowing this map makes debugging easy:
- 404 → matching failed (route/URL wrong)
- 422 → parsing/validation failed (input wrong)
- 500 → your function raised an exception (logic wrong)

---

## Quick recap — concepts over syntax

1. API = contract; HTTP = request/response with **method, URL, headers, body, status code**.
2. Uvicorn serves, FastAPI handles; virtual envs isolate projects.
3. Routes map URL + method → function; methods carry **intent** (and idempotency).
4. Data enters via **path** (identity), **query** (variant), **body** (payload).
5. Status codes report honestly: 4xx client, 5xx server.
6. Headers = metadata, cookies = statefulness hack on a stateless protocol.
7. OpenAPI auto-generated from code → Swagger UI / ReDoc for free.
