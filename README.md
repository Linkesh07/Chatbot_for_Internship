# TileShop and Tile Buddy

An online tile store with an AI agent that answers from the store's own data and the web, and acts on the site for the customer.

| Part | Technology |
|---|---|
| Frontend | Next.js, TypeScript |
| API | FastAPI, async SQLAlchemy, Alembic |
| Database | Local SQLite (WAL mode) with FTS5 search |
| Model | `openai/gpt-oss-120b` on Groq, `qwen/qwen3.8-27b` for images |
| Web search | SerpApi (Google) |

## Run locally

Copy `.env.example` to `.env` and fill it in. Then start the backend from `backend/`:

```bash
python -m venv .venv
```

```bash
.venv/Scripts/python -m pip install -e ".[dev]"
```

```bash
.venv/Scripts/alembic upgrade head
```

```bash
.venv/Scripts/python -m app.seed
```

```bash
.venv/Scripts/uvicorn app.main:app --port 8001
```

And the frontend from `frontend/`:

```bash
npm install
```

```bash
BACKEND_URL=http://localhost:8001 npm run dev
```

Open http://localhost:3000. Or run both in Docker with `docker compose up --build`.

## What the agent can do

| Area | Tools |
|---|---|
| Answer | `search_products`, `get_product`, `search_knowledge`, `web_search`, `estimate_tiles` |
| Navigate | `navigate` (fixed list of site pages) |
| Cart | `view_cart`, `add_to_cart`, `set_cart_quantity`, `remove_from_cart` |
| Orders (login needed) | `place_order`, `list_orders`, `track_order`, `cancel_order` |
| Staff requests | `create_request` (quote, callback, showroom visit, human handoff) |

The agent is **fully autonomous**: it never asks the customer to confirm. These server-side limits apply instead:

- The customer's identity comes from signed cookies, never from the model.
- Prices and stock always come from the database.
- `place_order`, `cancel_order` and `create_request` are refused for the rest of a reply once it has used web or image content, so a malicious page can't trigger them. The customer can repeat the request in a new message.
- One order per reply, 5 agent orders per customer per day, ₹500,000 per agent order. Configure with `AGENT_MAX_ORDERS_PER_DAY` and `AGENT_MAX_ORDER_VALUE_INR`.
- Every tool call, including refused ones, is stored in the `audit_log` table.

Payment is pay on delivery. No online payment is connected.

## Change the store's content

- Products: `backend/data/products.json`
- Policies and guides: `backend/data/knowledge/*.md`, split into sections at each `##` heading

Then run `python -m app.seed`. Seeding keeps live stock levels for existing products.

**The products, policies and showroom details are sample data. Replace them before real customers use the site.**

## Tests

From `backend/`:

```bash
.venv/Scripts/python -m pytest
```

```bash
.venv/Scripts/ruff check .
```

```bash
.venv/Scripts/mypy
```

Tests use a fake model and a temporary database, so they spend no API quota.

## Limits to know before going live

- **Groq rate limit:** the published limit for this model is 8K tokens per minute, and one agent reply used 1,255 to 4,201 tokens in testing. Expect "try again" messages with more than a few active customers until your Groq plan is upgraded.
- **SQLite:** every transaction takes the write lock, so it suits one server and modest traffic. Moving to many servers needs a server database, plus `REDIS_URL` for shared rate limits.
- **Knowledge search:** keyword search, so a Tamil or Hindi question relies on the model searching in English.
- **Before public hosting:** set `ENVIRONMENT=production`, `COOKIE_SECURE=true` and your real `ALLOWED_ORIGINS`, and serve over HTTPS.

Coding rules from [ECC](https://github.com/affaan-m/ECC) (MIT) are in `.claude/rules/ecc/`.
