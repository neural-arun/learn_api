

## FastAPI — Complete Study Roadmap

## My FastAPI checklist

0/47

### 1. Fundamentals

FastAPI setup, virtual environment & Uvicorn/CLI

Routes and HTTP methods: GET, POST, PUT, PATCH, DELETE

Path parameters, query parameters & request body

HTTP status codes, headers and cookies

Swagger UI, ReDoc & OpenAPI

### 2. Data Validation

Python type hints & Annotated

Pydantic BaseModel and field types

Required vs optional fields, defaults & constraints

Request validation and response_model

Nested models, lists and enums

### 3. API Design

CRUD API design and REST principles

APIRouter and modular project structure

Dependency injection: Depends()

Error handling: HTTPException and custom handlers

Pagination, filtering, sorting and search

File uploads, form data and JSON responses

### 4. Databases

SQLAlchemy ORM and database sessions

PostgreSQL integration

Models, relationships and CRUD operations

Transactions, commit, rollback and connection lifecycle

Alembic database migrations

Async database access (learn after synchronous basics)

### 5. Authentication & Security

OAuth2 concepts and password hashing

JWT access tokens and refresh-token concepts

Authentication vs authorization

Role-based access control (RBAC)

CORS, environment variables and secrets

Input validation, rate limiting and OWASP API risks

### 6. Async & Background Work

async/await and event-loop basics

async def vs def; blocking vs non-blocking I/O

Calling external APIs with HTTPX

BackgroundTasks and their limitations

Lifespan events and resource initialization

Task queues (Celery/ARQ) when needed

### 7. Production Engineering

Automated tests with pytest and TestClient/httpx

Dependency overrides and mocking

Logging, debugging and exception tracking

Middleware and request lifecycle

Configuration and environment-specific settings

Docker, deployment, HTTPS and reverse proxies

Health checks, graceful shutdown and monitoring

### 8. Advanced / Optional

WebSockets and streaming responses

Server-Sent Events (SSE)

Custom middleware and exception handlers

API versioning and advanced OpenAPI customization

Performance profiling and concurrency limits

Distributed tasks and microservice integration

