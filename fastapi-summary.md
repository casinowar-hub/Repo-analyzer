# FastAPI — Project Analysis Summary

> Analysed from: https://github.com/fastapi/fastapi (shallow clone, March 2026)

---

## What the Project Does

FastAPI is a modern, high-performance Python web framework for building HTTP APIs. It is built on top of **Starlette** (ASGI foundation) and **Pydantic** (data validation), and is designed around Python type annotations. Its headline features are:

- **Automatic data validation** — request parameters, query strings, headers, bodies, and responses are all validated using Pydantic models derived directly from function signatures.
- **Automatic OpenAPI docs** — every route is introspected and a live OpenAPI 3.x schema is generated, served at `/openapi.json`, with interactive Swagger UI at `/docs` and ReDoc at `/redoc`.
- **Dependency injection** — a first-class `Depends()` system resolves nested, cacheable dependencies (auth, DB sessions, config, etc.) before calling each endpoint.
- **Async-first** — built on ASGI; supports `async def` endpoints, WebSockets, Server-Sent Events, and streaming responses natively.
- **Standards-based** — JSON Schema, OpenAPI 3.x, OAuth2, HTTP Basic/Bearer, API Keys, and OpenID Connect are all supported out of the box.

---

## Key Modules and What They Do

### Core Application

| Module | Path | Description |
|--------|------|-------------|
| `applications.py` | `fastapi/applications.py` | Defines `FastAPI`, the root ASGI application class. Inherits from Starlette. Owns the router, middleware stack, exception handlers, OpenAPI metadata, and auto-generated `/docs`, `/redoc`, `/openapi.json` endpoints. |
| `routing.py` | `fastapi/routing.py` | Defines `APIRouter`, `APIRoute`, and `APIWebSocketRoute`. Routes parse their handler's function signature into a `Dependant`, build the ASGI request handler, drive the dependency/validation pipeline, and produce serialised responses. The largest module at ~4,900 lines. |
| `params.py` | `fastapi/params.py` | Pydantic `FieldInfo` subclasses used as parameter annotations: `Path`, `Query`, `Header`, `Cookie`, `Body`, `Form`, `File`, `Depends`, `Security`. |
| `param_functions.py` | `fastapi/param_functions.py` | Thin factory functions (`Path()`, `Query()`, `Depends()`, etc.) that construct the param classes above and are what users actually call in function signatures. |
| `exceptions.py` | `fastapi/exceptions.py` | `HTTPException`, `WebSocketException`, `RequestValidationError`, `ResponseValidationError`, `FastAPIError`. |
| `exception_handlers.py` | `fastapi/exception_handlers.py` | Default handlers that serialise validation errors and HTTP exceptions into JSON responses. |
| `encoders.py` | `fastapi/encoders.py` | `jsonable_encoder()` — recursively converts arbitrary Python objects (Pydantic models, datetimes, UUIDs, etc.) to JSON-safe types. |
| `datastructures.py` | `fastapi/datastructures.py` | `UploadFile`, `Default`, `DefaultPlaceholder` — file upload wrapper and sentinel defaults. |
| `concurrency.py` | `fastapi/concurrency.py` | Helpers for running sync functions in a threadpool and async utilities. |
| `background.py` | `fastapi/background.py` | `BackgroundTasks` — wraps Starlette's `BackgroundTasks` to schedule work that runs after the response is sent. |
| `websockets.py` | `fastapi/websockets.py` | Re-exports `WebSocket`, `WebSocketDisconnect`, `WebSocketState` from Starlette. |
| `responses.py` | `fastapi/responses.py` | Re-exports Starlette response classes plus `ORJSONResponse`; `JSONResponse` is the default. |
| `requests.py` | `fastapi/requests.py` | Re-exports Starlette `Request`. |
| `types.py` | `fastapi/types.py` | Type aliases used internally (`DecoratedCallable`, `IncEx`, `DependencyCacheKey`). |
| `utils.py` | `fastapi/utils.py` | Utilities: `generate_unique_id()`, `create_model_field()`, `generate_operation_id_for_path()`. |
| `sse.py` | `fastapi/sse.py` | `EventSourceResponse` for Server-Sent Events streaming. |
| `cli.py` | `fastapi/cli.py` | CLI entry point (delegates to `fastapi-cli`). |

### Dependencies Sub-package (`fastapi/dependencies/`)

| Module | Description |
|--------|-------------|
| `models.py` | `Dependant` dataclass — the internal representation of a parsed endpoint or dependency function. Holds categorised parameter lists (`path_params`, `query_params`, `body_params`, …) and a list of nested `Dependant` sub-dependencies. |
| `utils.py` | `get_dependant()` — analyses a function's signature and builds a `Dependant`. `get_flat_dependant()` — flattens the dependency tree. `solve_dependencies()` — executes the full tree, validates all params, returns resolved values. |

### OpenAPI Sub-package (`fastapi/openapi/`)

| Module | Description |
|--------|-------------|
| `utils.py` | `get_openapi()` — iterates all routes, extracts parameters, request bodies, responses, and security schemes, and assembles a complete OpenAPI 3.x dict. |
| `models.py` | Pydantic models for every OpenAPI object (`OpenAPI`, `PathItem`, `Operation`, `Parameter`, `RequestBody`, `Response`, …). |
| `docs.py` | `get_swagger_ui_html()`, `get_redoc_html()`, `get_swagger_ui_oauth2_redirect_html()` — generates HTML pages for the interactive docs UIs. |
| `constants.py` | Shared OpenAPI constant values. |

### Security Sub-package (`fastapi/security/`)

| Module | Scheme | Description |
|--------|--------|-------------|
| `oauth2.py` | OAuth2 | `OAuth2`, `OAuth2PasswordBearer`, `OAuth2AuthorizationCodeBearer`, `OAuth2PasswordRequestForm`, `SecurityScopes` |
| `api_key.py` | API Key | `APIKeyQuery`, `APIKeyHeader`, `APIKeyCookie` |
| `http.py` | HTTP | `HTTPBasic`, `HTTPBearer`, `HTTPDigest`, `HTTPBasicCredentials`, `HTTPAuthorizationCredentials` |
| `open_id_connect_url.py` | OIDC | `OpenIdConnect` |
| `base.py` | — | `SecurityBase` — base class linking a security scheme to its OpenAPI model |

### Middleware Sub-package (`fastapi/middleware/`)

| Module | Description |
|--------|-------------|
| `asyncexitstack.py` | `AsyncExitStackMiddleware` — injects an `AsyncExitStack` into the request scope so that `yield`-based dependencies and uploaded files are cleaned up after the response is sent. |
| `cors.py` | Re-exports Starlette `CORSMiddleware` |
| `gzip.py` | Re-exports Starlette `GZIPMiddleware` |
| `httpsredirect.py` | Re-exports Starlette `HTTPSRedirectMiddleware` |
| `trustedhost.py` | Re-exports Starlette `TrustedHostMiddleware` |
| `wsgi.py` | Re-exports Starlette `WSGIMiddleware` (mounts WSGI apps inside FastAPI) |

---

## Main Dependencies

### Required

| Package | Version | Role |
|---------|---------|------|
| `starlette` | `>=0.46.0` | ASGI foundation — request/response primitives, routing base, WebSocket, middleware |
| `pydantic` | `>=2.7.0` | Data validation, serialisation, JSON schema generation |
| `typing-extensions` | `>=4.8.0` | Backports of newer `typing` features |
| `typing-inspection` | `>=0.4.2` | Runtime type introspection utilities |

### Optional (installed via `fastapi[standard]`)

| Package | Role |
|---------|------|
| `uvicorn[standard]` | Production ASGI server |
| `fastapi-cli` | `fastapi dev` / `fastapi run` CLI commands |
| `httpx` | Test client (`from fastapi.testclient import TestClient`) |
| `jinja2` | HTML templating |
| `python-multipart` | Form data and file upload parsing |
| `email-validator` | Validates `EmailStr` Pydantic type |
| `pydantic-settings` | `BaseSettings` for environment-based config |
| `pydantic-extra-types` | Extra Pydantic types (Color, PaymentCard, etc.) |

### Indirect (via Starlette)

| Package | Role |
|---------|------|
| `anyio` | Async concurrency abstraction (trio / asyncio) |
| `idna` | Internationalized domain names |

---

## Entry Points

### 1. Python API (primary)

```python
from fastapi import FastAPI, APIRouter, Depends, HTTPException
from pydantic import BaseModel

app = FastAPI(title="My API", version="1.0.0")

class Item(BaseModel):
    name: str
    price: float

@app.get("/items/{item_id}", response_model=Item)
async def read_item(item_id: int):
    return Item(name="foo", price=3.14)
```

Run with Uvicorn:
```bash
uvicorn main:app --reload
```

### 2. CLI (via `fastapi-cli`)

Defined in `pyproject.toml`:
```toml
[project.scripts]
fastapi = "fastapi.cli:main"
```

```bash
fastapi dev main.py     # development server with auto-reload
fastapi run main.py     # production server
```

### 3. Sub-routers (modular apps)

```python
router = APIRouter(prefix="/users", tags=["users"])

@router.get("/{user_id}")
async def get_user(user_id: int): ...

app.include_router(router)
```

### 4. Lifespan (startup / shutdown)

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup
    yield
    # shutdown

app = FastAPI(lifespan=lifespan)
```

### 5. ASGI interface (direct)

`FastAPI` is a valid ASGI app and can be mounted directly in any ASGI server or composed with other ASGI apps:
```python
# Gunicorn + uvicorn workers
# gunicorn main:app --worker-class uvicorn.workers.UvicornWorker
```

---

## Request Lifecycle (summary)

```
HTTP Request
  → Middleware stack (ServerError → User → Exception → AsyncExitStack)
  → APIRouter matches path → APIRoute
  → get_request_handler()
      1. Parse request body (JSON / form / multipart)
      2. solve_dependencies() — validate all params, run Depends() tree
      3. run_endpoint_function() — call the handler
      4. serialize_response() — Pydantic validate + filter
      5. Construct response (JSONResponse / StreamingResponse / …)
      6. Send response to client
      7. Execute BackgroundTasks
      8. AsyncExitStack cleanup (yield deps, uploaded files)
```
