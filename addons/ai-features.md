# AI Features Addon
### Forge Addon | Append to your project's CLAUDE.md when the app uses AI

---

## WHEN TO USE THIS ADDON

Add this to your project when the app has features like:
- Chatbots or conversational interfaces
- Document analysis (PDFs, Word files, etc.)
- Content generation
- Semantic search / RAG (Retrieval Augmented Generation)
- Image analysis or generation
- Recommendation systems

---

## CORE RULES

### Backend Must Be Python FastAPI
AI-powered apps always use Python FastAPI as backend — no exceptions. Python has the best AI/ML ecosystem.

### Vector Database
- Use **pgvector** extension inside PostgreSQL (already your primary database)
- Never add a separate vector DB (Pinecone, Weaviate) unless scale explicitly demands it
- Setup: `CREATE EXTENSION IF NOT EXISTS vector;`

### Folder Separation
AI logic lives in its own module — never mixed with business logic:

```
src/modules/
├── users/                         (business logic)
├── orders/                        (business logic)
└── ai/                            (all AI code)
    ├── router.py
    ├── service.py
    ├── embeddings.py              (vector generation)
    ├── rag.py                     (retrieval pipeline)
    ├── prompts/                   (prompt templates)
    │   ├── chat.py
    │   └── summarize.py
    └── providers/                 (API clients)
        ├── anthropic_client.py
        └── openai_client.py
```

---

## PROVIDER SDKS

### Anthropic (Claude)
```python
from anthropic import AsyncAnthropic

client = AsyncAnthropic(api_key=settings.ANTHROPIC_API_KEY)

async def complete(messages: list[dict], system: str | None = None) -> str:
    response = await client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        system=system,
        messages=messages,
    )
    return response.content[0].text
```

### OpenAI (GPT)
```python
from openai import AsyncOpenAI

client = AsyncOpenAI(api_key=settings.OPENAI_API_KEY)

async def complete(messages: list[dict]) -> str:
    response = await client.chat.completions.create(
        model="gpt-4",
        messages=messages,
    )
    return response.choices[0].message.content
```

### LangChain (Multi-Provider or Complex Pipelines)
Use LangChain only when:
- You need provider abstraction (swap between Anthropic/OpenAI)
- You're building complex agent workflows
- You need pre-built RAG pipelines

For simple cases, use the provider SDKs directly — less overhead.

---

## PROMPTS

### Store Prompts in Files, Not Code
```python
# src/modules/ai/prompts/chat.py
SYSTEM_PROMPT = """You are a helpful assistant for {app_name}.
Your job is to {role_description}.
Always respond in the language the user writes in."""

def build_system_prompt(app_name: str, role_description: str) -> str:
    return SYSTEM_PROMPT.format(
        app_name=app_name,
        role_description=role_description,
    )
```

### Version Prompts
- Track prompt changes in git with clear commit messages
- A/B test significant changes before rolling out
- Include the prompt version in logs

### Never Include User Input Directly
- Always use parameterized templates
- Escape user input that goes into prompts to prevent prompt injection
- Consider user input as untrusted — validate and sanitize

---

## RAG (RETRIEVAL AUGMENTED GENERATION)

### When to Use RAG
- User has proprietary documents (manuals, policies, knowledge base)
- Need AI to answer from specific data, not general knowledge
- Need citations back to source documents

### Embedding Model
- **OpenAI:** `text-embedding-3-small` (good balance) or `text-embedding-3-large` (better quality)
- **Anthropic:** Use Voyage AI embeddings (their partner)
- Choose once and stick with it — embeddings from different models are not compatible

### Chunking Strategy
```python
def chunk_text(text: str, chunk_size: int = 1000, overlap: int = 200) -> list[str]:
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start = end - overlap
    return chunks
```

- Default chunk size: 1000 characters
- Overlap: 200 characters (preserves context across chunks)
- For structured documents (markdown), split by sections when possible

### Storage Schema
```sql
CREATE TABLE document_chunks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    embedding vector(1536) NOT NULL,  -- dimension depends on model
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON document_chunks USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

### Retrieval
```python
async def retrieve_relevant_chunks(
    query: str, 
    limit: int = 5,
) -> list[DocumentChunk]:
    query_embedding = await generate_embedding(query)
    
    result = await db.execute(
        select(DocumentChunk)
        .order_by(DocumentChunk.embedding.cosine_distance(query_embedding))
        .limit(limit)
    )
    return result.scalars().all()
```

### Prompt with Context
```python
def build_rag_prompt(question: str, chunks: list[DocumentChunk]) -> str:
    context = "\n\n".join([f"Source: {c.metadata.get('source')}\n{c.content}" for c in chunks])
    return f"""Answer the question based only on the context below. If the context doesn't contain the answer, say so.

Context:
{context}

Question: {question}"""
```

---

## STREAMING RESPONSES

### Backend (FastAPI SSE)
```python
from fastapi.responses import StreamingResponse

@router.post("/chat/stream")
async def chat_stream(request: ChatRequest):
    async def generate():
        async with client.messages.stream(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            messages=request.messages,
        ) as stream:
            async for text in stream.text_stream:
                yield f"data: {json.dumps({'text': text})}\n\n"
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(generate(), media_type="text/event-stream")
```

### Frontend (React)
Use the `fetch` API with `ReadableStream` or the `@anthropic-ai/sdk` browser SDK for direct streaming.

---

## CONVERSATION MEMORY

### Persistent (User-Specific)
Store in database:
```sql
CREATE TABLE conversations (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id),
    title TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE messages (
    id UUID PRIMARY KEY,
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    role TEXT NOT NULL,  -- 'user' or 'assistant'
    content TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON messages (conversation_id, created_at);
```

### Context Window Management
- Token limits are real — monitor and truncate
- Summarize old messages when approaching limits
- Keep recent messages in full, summarize older ones

---

## CACHING

### What to Cache
- **Embeddings** — don't regenerate for the same text
- **Expensive completions** — cache by prompt hash (Redis)
- **Retrieved chunks** — cache for identical queries for a short period

### What NOT to Cache
- User-specific conversational responses
- Anything with randomness/temperature > 0

---

## RATE LIMITING — CRITICAL

AI endpoints are EXPENSIVE. Rate limit aggressively.

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@router.post("/chat")
@limiter.limit("10/minute")  # adjust based on cost
async def chat(request: Request, data: ChatRequest):
    # ...
```

Consider:
- Per-user limits (not just per-IP)
- Daily/monthly caps for paid tiers
- Token-based limits for more accurate cost control

---

## SECURITY — AI SPECIFIC

### Prompt Injection Defense
- Validate user input before passing to the AI
- Use system prompts to establish boundaries
- Check outputs before executing any AI-suggested actions
- Never let AI output directly trigger code execution or database changes — always a human-validated step

### API Key Security
- Keys in environment variables only
- Never expose keys to the frontend — proxy all AI calls through your backend
- Rotate keys periodically
- Use separate keys for dev and production
- Monitor usage for anomalies

### Data Privacy
- Be explicit about what user data goes to AI providers
- Comply with GDPR, CCPA, etc.
- Consider data residency (some users need EU-only processing)
- Disclose AI usage to users where relevant

---

## OBSERVABILITY

### What to Log
- Prompt hash (not full prompt, for privacy)
- Response time
- Token usage (input and output)
- Cost estimate per call
- Model used
- Request ID for tracing

### What NOT to Log
- Full user prompts (PII risk)
- Full AI responses (PII + storage costs)

### Metrics to Track
- Requests per minute
- Average latency
- Token consumption
- Cost per user
- Error rate by provider

---

## COST MANAGEMENT

- Set hard spending limits at the provider level
- Alert when costs exceed thresholds
- Monitor cost per user — identify abuse patterns
- Use smaller/cheaper models for simple tasks (Haiku, GPT-3.5) and premium models only when needed
- Cache aggressively

---

## TESTING AI FEATURES

### What to Test
- Business logic around AI calls (with mocked AI responses)
- Rate limiting
- Error handling for API failures
- Prompt template rendering

### What NOT to Test in Unit Tests
- AI responses themselves (they're non-deterministic)
- Real API calls (too slow, too expensive)

### How to Test AI Quality
- Evaluation sets with expected outputs
- Run against production prompts periodically
- Track regression when changing prompts or models

---

## ENVIRONMENT VARIABLES

```
# Choose your provider(s)
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
VOYAGE_API_KEY=                  # if using Voyage embeddings

# Model configuration
AI_DEFAULT_MODEL=claude-sonnet-4-20250514
AI_EMBEDDING_MODEL=text-embedding-3-small

# Limits
AI_MAX_TOKENS=4096
AI_DAILY_COST_LIMIT_USD=100
```

---

## SPECIFIC DO NOTS

- Never expose AI API keys to the frontend
- Never let the AI execute arbitrary code
- Never send user data to AI without explicit consent for sensitive apps
- Never store embeddings from one model and try to use with another
- Never skip rate limiting on AI endpoints
- Never log full prompts or responses with PII
- Never trust AI output without validation (especially for code generation, SQL, commands)
- Never use temperature > 0 when you need consistent outputs
- Never forget to monitor costs

---

*This addon is part of Forge by Sultan Al-Muqimi / NQTH LLC.*
