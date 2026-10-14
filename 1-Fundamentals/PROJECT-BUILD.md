# 1. Fundamentals — Step-by-Step Project Build

**Project: "Book Shelf" — a small in-memory CRUD API for books.**

It deliberately uses ONLY Fundamentals concepts:
setup → routes → path/query/body → status codes → headers/cookies → Swagger.

No databases, no Pydantic deep-dives, no auth — those come in later sections.

---

## Step 0 — Setup (concepts: venv, Uvicorn)

```bash
mkdir bookshelf && cd bookshelf
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install fastapi uvicorn
```

Create `main.py`. Everything below goes into this one file so you can *see* the whole request lifecycle in one place.

Run it:
```bash
uvicorn main:app --reload
```
Open http://127.0.0.1:8000/docs — this is Swagger UI, generated automatically.

---

## Step 1 — The empty app (concept: app object = the contract)

```python
from fastapi import FastAPI

app = FastAPI(title="Book Shelf")
```

**What just happened:** `app` is the object Uvicorn will call for every request. The `title` shows up in the docs — proof that code drives documentation.

**Check:** visit `/docs`. You should see "Book Shelf" with zero endpoints.

---

## Step 2 — First GET route (concept: route = method + URL)

```python
BOOKS = [
    {"id": 1, "title": "Clean Code", "author": "Robert Martin", "pages": 464},
    {"id": 2, "title": "The Pragmatic Programmer", "author": "Hunt & Thomas", "pages": 352},
]

@app.get("/")
def home():
    return {"message": "Welcome to Book Shelf"}
```

**Concepts in play:**
- `@app.get("/")` binds HTTP GET on `/` to the function below.
- Returning a `dict` → FastAPI serializes it to JSON and sets `200 OK` automatically.

**Check:** visit http://127.0.0.1:8000/ directly in the browser, or hit it from Swagger's "Try it out".

---

## Step 3 — Path parameter (concept: "which resource")

```python
@app.get("/books/{book_id}")
def get_book(book_id: int):
    for book in BOOKS:
        if book["id"] == book_id:
            return book
    # concept: honest status codes — don't return 200 for "not found"
```

Replace the body with a proper 404:

```python
from fastapi import HTTPException

@app.get("/books/{book_id}")
def get_book(book_id: int):
    for book in BOOKS:
        if book["id"] == book_id:
            return book
    raise HTTPException(status_code=404, detail="Book not found")
```

**Concepts in play:**
- `{book_id}` is a **path parameter** — required, identifies *which* book.
- `book_id: int` → FastAPI converts `"abc"` → **422 Unprocessable Entity** automatically. Validation comes from the type hint.
- 404 for missing resource: the status code *is* part of the API contract.

**Try it in Swagger:**
- `/books/1` → 200 with book JSON
- `/books/99` → 404 `{"detail": "Book not found"}`
- `/books/abc` → 422 validation error

---

## Step 4 — Query parameters (concept: "which variant")

```python
@app.get("/books")
def list_books(author: str | None = None, limit: int = 10):
    results = BOOKS
    if author:
        results = [b for b in results if author.lower() in b["author"].lower()]
    return results[:limit]
```

**Concepts in play:**
- No `{}` in the URL → these are **query parameters**: `/books?author=robert&limit=1`
- Optional (`= None`) and defaults (`limit: int = 10`) — the client only sends what it cares about.
- Path says *which collection* (`/books`); query says *which view of it*.

**Try it:**
- `/books` → all books
- `/books?author=hunt` → filtered
- `/books?limit=1` → paginated (concept: pagination is just a query param)

---

## Step 5 — POST with a body (concept: "the payload")

For now, a simple dict body works:

```python
from fastapi import Body

@app.post("/books", status_code=201)
def create_book(title: str = Body(...), author: str = Body(...), pages: int = Body(...)):
    new_id = max(b["id"] for b in BOOKS) + 1 if BOOKS else 1
    book = {"id": new_id, "title": title, "author": author, "pages": pages}
    BOOKS.append(book)
    return book
```

**Concepts in play:**
- POST = **create** → returns **201 Created** (not 200) — status codes carry meaning.
- `Body(...)` = data comes from the **request body** (JSON), not URL.
- Server assigns the `id` — clients don't pick identities.

**Try it in Swagger (POST → Try it out → Execute):**
```json
{"title": "Refactoring", "author": "Martin Fowler", "pages": 448}
```

**Note:** Section 2 will replace these individual `Body(...)` fields with a proper Pydantic model — remember this moment so you feel *why* it's better.

---

## Step 6 — PUT and DELETE (concept: idempotency)

```python
@app.put("/books/{book_id}")
def replace_book(book_id: int, title: str = Body(...), author: str = Body(...), pages: int = Body(...)):
    for book in BOOKS:
        if book["id"] == book_id:
            book.update({"title": title, "author": author, "pages": pages})
            return book
    raise HTTPException(status_code=404, detail="Book not found")


@app.delete("/books/{book_id}", status_code=204)
def delete_book(book_id: int):
    for i, book in enumerate(BOOKS):
        if book["id"] == book_id:
            del BOOKS[i]
            return None          # 204 = deleted, nothing to return
    raise HTTPException(status_code=404, detail="Book not found")
```

**Concepts in play:**
- PUT = full replace → **idempotent** (sending it twice = same end state).
- DELETE twice → second returns 404, but the *desired state* (gone) held.
- **204 No Content** = success with empty body — "done, nothing to show you."

---

## Step 7 — Custom headers (concept: metadata vs message)

```python
from fastapi import Response

@app.get("/books/{book_id}/with-header")
def get_book_with_header(book_id: int, response: Response):
    for book in BOOKS:
        if book["id"] == book_id:
            response.headers["X-Book-Source"] = "in-memory-shelf"
            response.headers["Cache-Control"] = "no-store"
            return book
    raise HTTPException(status_code=404, detail="Book not found")
```

**Concept:** the body is *the message*; headers describe *how to handle it*. Check the `Headers` tab of the response in Swagger to see them.

---

## Step 8 — Cookies (concept: statelessness & remembering a client)

```python
from fastapi import Cookie

@app.get("/greet")
def greet(response: Response, visitor: str | None = Cookie(default=None)):
    if visitor is None:
        visitor = "first-timer"
        response.set_cookie("visitor", visitor, max_age=3600)
    return {"message": f"Hello {visitor}"}
```

**Concept in action:**
1. First call → server sets a cookie, response says "Hello first-timer".
2. Second call → browser *automatically* sends the cookie back → "Hello first-timer" again.
3. This is the classic fix for HTTP being **stateless** — the server doesn't remember you, the *client* carries the ID card.

**Try it:** run twice in Swagger and watch `Set-Cookie` in the response headers.

---

## Step 9 — Read your own documentation (concept: OpenAPI as source of truth)

Revisit `/docs` and `/redoc` now that all routes exist. Notice:

- Every parameter, type, default, and body field was generated **from your code**.
- "Try it out" executes **real** requests against your running server.
- The spec behind it: http://127.0.0.1:8000/openapi.json

**Exercise:** add `@app.get("/health")` returning `{"status": "ok"}`, save, and watch it appear in the docs instantly. That's self-documenting code.

---

## Step 10 — Full lifecycle checklist

Run through this with any endpoint you built:

| Phase | What to observe | If it fails you see |
|---|---|---|
| Match route | correct method + URL | **404** |
| Parse input | path/query/body types | **422** |
| Run logic | your function | **500** (unhandled exception) |
| Report result | honest status code + headers + body | wrong 2xx/4xx |

**Manual test script** (no framework needed yet — just curl):
```bash
curl http://127.0.0.1:8000/books
curl http://127.0.0.1:8000/books/1
curl -X POST http://127.0.0.1:8000/books \
  -H "Content-Type: application/json" \
  -d '{"title": "Clean Architecture", "author": "Robert Martin", "pages": 432}'
curl -X DELETE http://127.0.0.1:8000/books/1
curl -i http://127.0.0.1:8000/books/99    # -i shows status code + headers
```

---

## Final state of `main.py`

```python
from fastapi import FastAPI, HTTPException, Body, Response, Cookie

app = FastAPI(title="Book Shelf")

BOOKS = [
    {"id": 1, "title": "Clean Code", "author": "Robert Martin", "pages": 464},
    {"id": 2, "title": "The Pragmatic Programmer", "author": "Hunt & Thomas", "pages": 352},
]


@app.get("/")
def home():
    return {"message": "Welcome to Book Shelf"}


@app.get("/books")
def list_books(author: str | None = None, limit: int = 10):
    results = BOOKS
    if author:
        results = [b for b in results if author.lower() in b["author"].lower()]
    return results[:limit]


@app.get("/books/{book_id}")
def get_book(book_id: int):
    for book in BOOKS:
        if book["id"] == book_id:
            return book
    raise HTTPException(status_code=404, detail="Book not found")


@app.post("/books", status_code=201)
def create_book(title: str = Body(...), author: str = Body(...), pages: int = Body(...)):
    new_id = max(b["id"] for b in BOOKS) + 1 if BOOKS else 1
    book = {"id": new_id, "title": title, "author": author, "pages": pages}
    BOOKS.append(book)
    return book


@app.put("/books/{book_id}")
def replace_book(book_id: int, title: str = Body(...), author: str = Body(...), pages: int = Body(...)):
    for book in BOOKS:
        if book["id"] == book_id:
            book.update({"title": title, "author": author, "pages": pages})
            return book
    raise HTTPException(status_code=404, detail="Book not found")


@app.delete("/books/{book_id}", status_code=204)
def delete_book(book_id: int):
    for i, book in enumerate(BOOKS):
        if book["id"] == book_id:
            del BOOKS[i]
            return None
    raise HTTPException(status_code=404, detail="Book not found")


@app.get("/greet")
def greet(response: Response, visitor: str | None = Cookie(default=None)):
    if visitor is None:
        visitor = "first-timer"
        response.set_cookie("visitor", visitor, max_age=3600)
    return {"message": f"Hello {visitor}"}
```

---

## You're done with Fundamentals when you can answer:

1. What does Uvicorn do that FastAPI doesn't?
2. Why is `DELETE` idempotent but `POST` not?
3. When do you use a path param vs a query param vs a body?
4. Difference between 401 and 403? Between 422 and 400?
5. Where does Swagger UI's data come from — and why does it never need manual updating?
6. HTTP is stateless — how did the cookie trick work around that?
