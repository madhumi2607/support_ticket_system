# Support Ticket System

A full-stack support ticket application with AI-powered auto-classification.

## Quick Start

### 1. Clone / unzip the project

### 2. Set your LLM API key

Copy the example env file and add your key:

```bash
cp .env.example .env
# Edit .env and set your LLM_API_KEY
```

Or export it directly:

```bash
export LLM_API_KEY=your_api_key_here
```

### 3. Run with Docker

```bash
docker-compose up --build
```

- Frontend: http://localhost:3000
- Backend API: http://localhost:8000/api/tickets/

That's it. Migrations run automatically on startup.

---

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `LLM_API_KEY` | _(empty)_ | Your LLM API key |
| `LLM_PROVIDER` | `anthropic` | One of: `anthropic`, `openai`, `google` |

The app works without an API key — you just won't get auto-suggestions.

---

## LLM Choice: Anthropic (Claude)

I chose **Anthropic's Claude** (`claude-3-haiku-20240307`) as the default provider for the following reasons:

1. **Speed**: Haiku is the fastest Claude model, keeping classification latency low.
2. **Instruction following**: Claude follows structured output instructions reliably, making JSON extraction from responses consistent.
3. **Cost**: Haiku is very cheap, well-suited for high-frequency classification calls.
4. **Flexibility**: The classifier also supports OpenAI (GPT-3.5-turbo) and Google (Gemini Pro) — set `LLM_PROVIDER` accordingly.

### The Prompt

The prompt (in `backend/tickets/classifier.py`) gives the model:
- Explicit definitions of each category and priority level
- A strict instruction to respond **only** in JSON
- Clear examples of what each level means

This approach avoids ambiguity and makes parsing reliable. If the model returns garbage, the system falls back gracefully without blocking ticket submission.

---

## Architecture & Design Decisions

### Backend (Django + DRF + PostgreSQL)
- **DB-level constraints** enforced via `CheckConstraint` on all enum fields
- **Stats endpoint** uses ORM `aggregate` + `annotate` — no Python loops
- **Filtering** is fully combinable: category + priority + status + search
- Migrations ship pre-generated so `manage.py migrate` works in Docker with no extra steps

### Frontend (React)
- **Debounced classify call**: fires 800ms after the user stops typing in the description field — avoids hammering the API on every keystroke
- **Graceful degradation**: if the LLM call fails, the form still works; the user just selects category/priority manually
- **No page reloads**: new tickets appear immediately in the list; stats refresh automatically after each submission or status change

### Docker
- Health check on Postgres ensures the backend waits for the DB to be ready before running migrations
- API key is only ever passed as an environment variable — never hardcoded

---

## API Reference

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/tickets/` | Create ticket |
| `GET` | `/api/tickets/` | List tickets (supports `?category=`, `?priority=`, `?status=`, `?search=`) |
| `PATCH` | `/api/tickets/<id>/` | Update ticket |
| `GET` | `/api/tickets/stats/` | Aggregated stats |
| `POST` | `/api/tickets/classify/` | LLM classify description |
