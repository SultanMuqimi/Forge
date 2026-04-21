# CLAUDE.md — FastAPI Backend (Python)
### Forge Stack Rules | Python FastAPI Backend

---

This file extends the universal CLAUDE.md. Read that first.

---

## WHEN TO USE THIS STACK

- **AI-powered apps** (always — Python has the best AI/ML ecosystem)
- Data-heavy applications (pandas, numpy integration)
- APIs that interact with Python-based services
- Teams with Python expertise

---

## TECHNOLOGY STANDARDS

### Required
- **Python 3.11+**
- **FastAPI** (latest stable)
- **Uvicorn** as the ASGI server (dev) or **Gunicorn + Uvicorn workers** (production)
- **PostgreSQL** as the database
- **SQLAlchemy 2.0+** with async support, or **SQLModel** (SQLAlchemy + Pydantic combined)
- **Alembic** for migrations
- **Pydantic v2** for validation

### Required Libraries
- **python-jose[cryptography]** for JWT
- **passlib[bcrypt]** for password hashing
- **python-multipart** for file uploads
- **slowapi** for rate limiting
- **structlog** for structured logging
- **pytest** + **pytest-asyncio** for testing

### Package Management
- Use **Poetry** or **uv** for dependency management — not raw pip
- Lock files committed to the repo
- Never use `requirements.txt` alone (no version locking)

---

## FOLDER STRUCTURE

```
backend/
├── CLAUDE.md
├── .env.example
├── pyproject.toml
├── poetry.lock
├── alembic.ini
├── alembic/
│   └── versions/
└── src/
    ├── main.py                    (FastAPI app entry)
    ├── config/
    │   ├── __init__.py
    │   ├── settings.py            (Pydantic BaseSettings)
    │   └── database.py
    ├── modules/                   (organize by feature)
    │   └── users/
    │       ├── __init__.py
    │       ├── router.py
    │       ├── controller.py      (optional — can live in router)
    │       ├── service.py
    │       ├── repository.py
    │       ├── schemas.py         (Pydantic models)
    │       ├── models.py          (SQLAlchemy models)
    │       └── dependencies.py    (FastAPI dependencies)
    ├── core/
    │   ├── security.py            (JWT, password hashing)
    │   ├── exceptions.py          (custom exceptions)
    │   ├── middleware.py
    │   └── logging.py
    ├── shared/
    │   ├── base_model.py          (SQLAlchemy base)
    │   └── base_schema.py         (Pydantic base)
    └── tests/
        ├── conftest.py
        └── modules/
```

---

## LAYERED ARCHITECTURE

Same as Node.js backend:
```
Router → Service → Repository → Database
```

### Example — Users Module

**schemas.py** (Pydantic models for API I/O):
```python
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime

class UserCreate(BaseModel):
    email: EmailStr
    name: str = Field(min_length=1, max_length=100)
    password: str = Field(min_length=8)

class UserResponse(BaseModel):
    id: str
    email: EmailStr
    name: str
    created_at: datetime
    
    model_config = {"from_attributes": True}
```

**models.py** (SQLAlchemy ORM):
```python
from sqlalchemy import String, DateTime
from sqlalchemy.orm import Mapped, mapped_column
from datetime import datetime
from src.shared.base_model import Base
import uuid

class User(Base):
    __tablename__ = "users"
    
    id: Mapped[str] = mapped_column(String, primary_key=True, default=lambda: str(uuid.uuid4()))
    email: Mapped[str] = mapped_column(String, unique=True, index=True)
    name: Mapped[str] = mapped_column(String)
    password_hash: Mapped[str] = mapped_column(String)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
```

**repository.py** (database access):
```python
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from .models import User

class UsersRepository:
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def find_by_email(self, email: str) -> User | None:
        result = await self.db.execute(select(User).where(User.email == email))
        return result.scalar_one_or_none()
    
    async def create(self, email: str, name: str, password_hash: str) -> User:
        user = User(email=email, name=name, password_hash=password_hash)
        self.db.add(user)
        await self.db.commit()
        await self.db.refresh(user)
        return user
```

**service.py** (business logic):
```python
from .repository import UsersRepository
from .schemas import UserCreate
from src.core.security import hash_password
from src.core.exceptions import ConflictError

class UsersService:
    def __init__(self, repo: UsersRepository):
        self.repo = repo
    
    async def create(self, data: UserCreate) -> User:
        existing = await self.repo.find_by_email(data.email)
        if existing:
            raise ConflictError("Email already registered")
        
        password_hash = hash_password(data.password)
        return await self.repo.create(data.email, data.name, password_hash)
```

**router.py** (HTTP layer):
```python
from fastapi import APIRouter, Depends, status
from .schemas import UserCreate, UserResponse
from .service import UsersService
from .dependencies import get_users_service

router = APIRouter(prefix="/users", tags=["users"])

@router.post("", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    data: UserCreate,
    service: UsersService = Depends(get_users_service),
) -> UserResponse:
    user = await service.create(data)
    return UserResponse.model_validate(user)
```

---

## EXCEPTION HANDLING — GLOBAL

```python
# src/core/exceptions.py
class AppException(Exception):
    def __init__(self, message: str, status_code: int):
        self.message = message
        self.status_code = status_code

class NotFoundError(AppException):
    def __init__(self, message: str = "Resource not found"):
        super().__init__(message, 404)

class ConflictError(AppException):
    def __init__(self, message: str = "Conflict"):
        super().__init__(message, 409)

class UnauthorizedError(AppException):
    def __init__(self, message: str = "Unauthorized"):
        super().__init__(message, 401)
```

```python
# src/main.py
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from src.core.exceptions import AppException

app = FastAPI()

@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.message},
    )
```

**No try/except in routers for business logic errors.** Let exceptions bubble to the global handler.

---

## AUTHENTICATION

### JWT with python-jose
```python
from jose import jwt
from datetime import datetime, timedelta

def create_access_token(user_id: str) -> str:
    expire = datetime.utcnow() + timedelta(minutes=15)
    payload = {"sub": user_id, "exp": expire}
    return jwt.encode(payload, settings.JWT_SECRET, algorithm="HS256")
```

### Password Hashing
```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)
```

### Dependency-Based Auth
```python
from fastapi import Depends, Header
from src.core.exceptions import UnauthorizedError

async def get_current_user(authorization: str = Header()) -> User:
    if not authorization.startswith("Bearer "):
        raise UnauthorizedError("Missing token")
    token = authorization[7:]
    payload = jwt.decode(token, settings.JWT_SECRET, algorithms=["HS256"])
    user = await users_repository.find_by_id(payload["sub"])
    if not user:
        raise UnauthorizedError("User not found")
    return user

# Use it
@router.get("/me")
async def me(user: User = Depends(get_current_user)):
    return user
```

---

## DATABASE — ASYNC SQLAlchemy

### Engine and Session
```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker

engine = create_async_engine(settings.DATABASE_URL, echo=settings.DEBUG)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_db() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session
```

### Always Async
- Use `AsyncSession` everywhere
- Use `select()` with `await session.execute()`
- Use `asyncpg` as the driver: `postgresql+asyncpg://...`

### Alembic Migrations
- Create: `alembic revision --autogenerate -m "description"`
- Apply: `alembic upgrade head`
- Review autogenerated migrations carefully — they are not always perfect

---

## VALIDATION — PYDANTIC V2

- All request/response models inherit from `BaseModel`
- Use `Field()` for constraints
- Use `EmailStr`, `HttpUrl`, `UUID` for typed strings
- Use `model_validator` for cross-field validation
- Response models use `model_config = {"from_attributes": True}` for ORM integration

---

## CONFIGURATION

```python
# src/config/settings.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    DATABASE_URL: str
    JWT_SECRET: str
    JWT_ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 15
    CORS_ORIGINS: list[str] = ["http://localhost:5173"]
    LOG_LEVEL: str = "INFO"
    DEBUG: bool = False
    
    model_config = {"env_file": ".env"}

settings = Settings()
```

---

## LOGGING — STRUCTLOG

```python
import structlog

logger = structlog.get_logger()

logger.info("user_created", user_id=user.id, email=user.email)
```

- Structured logs (JSON in production)
- Never log passwords, tokens, or PII
- Include request IDs for tracing

---

## AI-POWERED APPS — SPECIAL RULES

When the app uses AI features (LLMs, embeddings, RAG):

### SDK Choice
- **Anthropic SDK** for Claude: `anthropic`
- **OpenAI SDK** for GPT: `openai`
- **LangChain** for complex pipelines or multi-provider setups

### Vector Database
- Use **pgvector** extension in PostgreSQL (already the chosen DB)
- Never spin up a separate vector DB unless scale demands it

### Separation
- AI logic lives in `src/modules/ai/` — never mixed with business logic
- Prompts are in dedicated files (`src/modules/ai/prompts/`)
- Caching for embeddings and expensive completions
- Rate limiting on AI endpoints (they're expensive)

### Streaming
- For chat-style interactions, use Server-Sent Events or WebSockets
- FastAPI supports SSE natively with `StreamingResponse`

---

## TESTING

### Stack
- **pytest** + **pytest-asyncio**
- **httpx** for API testing (async HTTP client)
- **pytest-cov** for coverage

### Fixtures
- `conftest.py` with shared fixtures
- Separate test database
- Transaction rollback between tests for speed

### Example
```python
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_create_user(client: AsyncClient):
    response = await client.post("/users", json={
        "email": "test@example.com",
        "name": "Test User",
        "password": "securepassword123",
    })
    assert response.status_code == 201
    assert response.json()["email"] == "test@example.com"
```

---

## SECURITY CHECKLIST

- [ ] CORS configured with specific origins
- [ ] Rate limiting on public endpoints
- [ ] Password hashing with bcrypt (cost 12)
- [ ] JWT secrets are strong (32+ bytes) and in env vars
- [ ] Input validation via Pydantic on every endpoint
- [ ] Output via Pydantic response models — never raw ORM objects
- [ ] SQL injection prevented via SQLAlchemy (never raw SQL with user input)
- [ ] HTTPS enforced in production
- [ ] Security headers middleware configured
- [ ] Dependency audit (`pip-audit` or `safety`)

---

## ENVIRONMENT VARIABLES

```
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/db
JWT_SECRET=<32+ random bytes>
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=15
CORS_ORIGINS=http://localhost:5173
LOG_LEVEL=INFO
DEBUG=false

# If AI features:
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
```

---

## SPECIFIC DO NOTS

- Never use sync SQLAlchemy in an async FastAPI app
- Never use `requirements.txt` alone — use Poetry or uv
- Never skip Pydantic validation with raw dict responses
- Never use `print()` for logging — use structlog
- Never put business logic in router functions
- Never return SQLAlchemy models directly — use Pydantic response models
- Never use `*` imports
- Never skip type hints — every function annotated

---

## SESSION COMPLETION CHECKLIST — FASTAPI SPECIFIC

- [ ] App starts without errors (`uvicorn src.main:app --reload`)
- [ ] OpenAPI docs accessible at `/docs`
- [ ] All endpoints have proper Pydantic schemas
- [ ] All protected endpoints use the auth dependency
- [ ] Rate limiting applied
- [ ] Global exception handler registered
- [ ] Logger configured
- [ ] `/health` endpoint exists
- [ ] Alembic migrations are current
- [ ] Tests pass (`pytest`)
- [ ] Type checks pass (`mypy` if configured)
- [ ] No linter errors (`ruff check`)

---

*This file is part of Forge by Sultan Al-Muqimi / NQTH LLC.*
