# Agentic Learning Path Generator

Backend for AI-driven personalized education using **NVIDIA Nemotron-3-Ultra** (primary) with **Gemini 2.5 Flash** fallback. Built with **FastAPI**, **SQLModel**, and **PostgreSQL**.

---

## 🎯 Core Features

### 1. Multi-Agent Learning Path Generation
Four specialized AI agents collaborate to create personalized roadmaps:

| Agent | Role | Key Capability |
|-------|------|----------------|
| **Market Analyst** | Scans job market trends | Identifies top 5-7 critical/emerging skills for target role with demand levels and growth metrics |
| **Architect** | Designs curriculum | Creates 4-6 sequential modules with scaffolding, gap analysis, and explainable `why_needed` rationale |
| **Curator** | Sources resources | Matches user's VARK preference (Video/Text/Interactive); generates real URLs for non-video; uses `AUTO_YOUTUBE` placeholder for videos |
| **Critic** | Validates quality | Checks logical flow, prerequisites, resource diversity; re-orders/fixes modules if needed |

### 2. Two Generation Modes

#### A. Manual Profile Mode (`POST /api/v1/generate-roadmap`)
User provides structured profile:
```json
{
  "name": "Alex",
  "current_role": "Student",
  "target_role": "React Developer",
  "current_skills": ["HTML", "CSS", "JavaScript Basics"],
  "preferred_style": "Video",
  "experience_level": "Beginner"
}
```
**Flow:** Market Analyst → Architect → Curator → YouTube Enrichment → Critic

#### B. Resume + Job Description Mode (`POST /api/v1/generate-roadmap-from-docs`)
Upload resume + provide JD (3 ways):
- `jd_text`: pasted text (most reliable)
- `jd_file`: uploaded PDF/DOCX/TXT
- `job_url`: link to job posting (auto-scraped, best-effort)

**Flow:** Gap Analyst → Architect (gap-driven) → Curator → YouTube Enrichment → Critic

**Gap Analysis Output:**
- `matched_skills`: skills candidate already has
- `gaps`: each with `skill`, `status` (Missing/Partial/Met), `importance` (Critical/High/Medium), `note`
- `resume_summary`, `jd_summary`

---

### 3. Real YouTube Video Integration
- **YouTube Data API v3** (primary) or **Serper.dev** (fallback)
- Fetches high-quality videos per module (≥1000 views, likes-ratio sorted)
- Filters: English content, embeddable, syndicated, moderate safe search
- Duration preference: `medium` (10-20 min tutorials)
- Concurrent fetching via `ThreadPoolExecutor` (5 modules ≈ 1 slowest lookup)

### 4. Explainable AI (Feature C)
Every module includes `why_needed` explaining **why** it exists:
- Manual mode: connects to career goal + market demand
- Gap mode: references specific skill gap + JD requirement

### 5. VARK Learning Style Support (Feature D)
`preferred_style` enum: `Video` | `Text` | `Interactive`
- Curator adjusts resource mix accordingly
- YouTube enrichment respects preference (more videos for Video style)

### 6. Session Management & Persistence
| Endpoint | Purpose |
|----------|---------|
| `POST /api/v1/sessions` | Create session from profile (returns `roadmap_id`, `conversation_id`) |
| `POST /api/v1/sessions/{id}/messages` | Append message to session log |
| `POST /api/v1/sessions/{id}/progress` | Mark module completed (`module_id`, `status`) |
| `POST /api/v1/sessions/{id}/regenerate` | Regenerate roadmap preserving completed modules |

**Database Models:** User, Roadmap, Module, Resource, AgentLog, ModuleProgress

---

### 7. Document Processing
- **Resume extraction**: PDF/DOCX/TXT via `pypdf` + `python-docx` (15k char limit)
- **JD extraction**: Same pipeline (8k char limit)
- **Job URL scraping**: `httpx` + `BeautifulSoup` with heuristic selectors (JS-rendered pages return `None` → caller falls back to manual paste)

---

### 8. Dual LLM Provider with Failover
| Provider | Model | Use Case |
|----------|-------|----------|
| **NVIDIA NIM** | `nvidia/nemotron-3-ultra-550b-a55b` | Primary (180s timeout, 8192 max tokens for Curator) |
| **Google Gemini** | `gemini-2.5-flash-lite` | Fallback on auth/quota/404 errors (120s timeout) |

Lazy initialization + masked key logging at startup.

---

### 9. Progress-Aware Regeneration
`POST /api/v1/sessions/{id}/regenerate`:
- Loads completed modules from `ModuleProgress`
- Detects original flow (gap-based vs manual) from stored `resume_text`/`jd_text`
- Re-runs matching pipeline with `completed_module_ids` exclusion
- Returns full `RoadmapResponse` with updated modules

---

## 📡 API Endpoints Summary

```
POST   /api/v1/generate-roadmap              # Manual profile → roadmap
POST   /api/v1/generate-roadmap-from-docs    # Resume + JD → gap-based roadmap
POST   /api/v1/sessions                      # Create session
POST   /api/v1/sessions/{id}/messages        # Add message
POST   /api/v1/sessions/{id}/progress        # Mark module done
POST   /api/v1/sessions/{id}/regenerate      # Regenerate with progress
GET    /api/v1/status                        # Health check (ad-blocker safe)
GET    /health                               # Root health check
GET    /                                     # Root info
GET    /api/v1/debug/gen-init                # Debug Gemini init (env-gated)
```

---

## 🗄️ Database Schema (SQLModel)

```text
User (id, name, created_at)
  └─ Roadmap (id, user_id, target_role, market_analysis, profile,
              resume_text, jd_text, jd_source_url, gap_analysis, created_at)
      ├─ Module (id, roadmap_id, module_index, module_name, description,
                 skills_covered, why_needed, estimated_time)
      │   └─ Resource (id, module_id, title, url, type, duration, reason)
      ├─ AgentLog (id, roadmap_id, agent_name, action, timestamp)
      └─ ModuleProgress (id, roadmap_id, module_id, status, completed_at)
```

---

## ⚙️ Configuration (Environment Variables)

| Variable | Required | Description |
|----------|----------|-------------|
| `NVIDIA_API_KEY` / `NVIDIA_NIM_API` | Yes* | NVIDIA NIM API key |
| `NVIDIA_BASE_URL` | No | Default: `https://integrate.api.nvidia.com/v1` |
| `NVIDIA_MODEL` | No | Default: `nvidia/nemotron-3-ultra-550b-a55b` |
| `GEMINI_API_KEY` | Yes* | Google Gemini API key (fallback) |
| `YOUTUBE_API_KEY` | No | YouTube Data API v3 (real videos) |
| `SERPER_API_KEY` | No | Serper.dev API (fallback video search) |
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `ALLOWED_ORIGINS` | No | Comma-separated CORS origins |
| `ALLOW_ALL_ORIGINS` | No | `true` to allow all (dev only) |
| `ENABLE_GENIE_DEBUG` | No | Enable `/debug/gen-init` endpoint |

*At least one of NVIDIA or Gemini key required.

---

## 🚀 Quick Start

```bash
# 1. Clone & install
git clone <repo>
cd eduWay
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# 2. Configure environment
cp .env.example .env
# Edit .env with your API keys and DATABASE_URL

# 3. Run
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

Docker:
```bash
docker build -t eduway .
docker run -p 8000:8000 --env-file .env eduway
```

---

## 📦 Key Dependencies

| Package | Purpose |
|---------|---------|
| `fastapi`, `uvicorn` | Web framework + ASGI server |
| `sqlmodel`, `psycopg` | ORM + PostgreSQL driver |
| `openai` | NVIDIA NIM client (OpenAI-compatible) |
| `google-generativeai` | Gemini client |
| `pypdf`, `python-docx` | Document parsing |
| `beautifulsoup4`, `httpx` | Job URL scraping |
| `python-multipart` | File upload handling |

---

## 🧪 Testing Endpoints

```bash
# Health
curl http://localhost:8000/api/v1/status

# Manual roadmap
curl -X POST http://localhost:8000/api/v1/generate-roadmap \
  -H "Content-Type: application/json" \
  -d '{"name":"Alex","current_role":"Student","target_role":"React Developer",
       "current_skills":["HTML","CSS"],"preferred_style":"Video","experience_level":"Beginner"}'

# Gap-based (multipart)
curl -X POST http://localhost:8000/api/v1/generate-roadmap-from-docs \
  -F "name=Alex" \
  -F "target_role=React Developer" \
  -F "preferred_style=Video" \
  -F "jd_text=We need React, TypeScript, Next.js..." \
  -F "resume=@resume.pdf"
```

---

## 📁 Project Structure

```
eduWay/
├── app/
│   ├── main.py              # FastAPI app, CORS, startup
│   ├── apis/routes.py       # All API endpoints
│   ├── services/
│   │   ├── agent_service.py      # Multi-agent workflow orchestration
│   │   ├── document_parser_service.py  # PDF/DOCX/TXT extraction
│   │   ├── job_fetcher_service.py      # JD URL scraping
│   │   └── youtube_service.py        # YouTube/Serper video search
│   ├── models/
│   │   ├── schemas.py       # Pydantic request/response models
│   │   └── __init__.py
│   ├── db/
│   │   ├── models.py        # SQLModel tables
│   │   ├── session.py       # DB engine/session
│   │   └── __init__.py
│   └── utils/prompts.py     # All agent prompts
├── requirements.txt
├── .env.example
├── Dockerfile
└── README.md
```

---

## 🔒 Security Notes

- API keys loaded from `.env` (never committed)
- CORS restricted to configured origins (defaults to common dev ports)
- Input validation via Pydantic models
- File upload size limits via character truncation (not byte limits)
- No authentication/authorization layer yet (add for production)

---

## 🛣️ Roadmap / Known Limitations

- [ ] Add authentication (JWT/OAuth)
- [ ] Rate limiting on generation endpoints
- [ ] Background job queue (Celery/RQ) for long generations
- [ ] Frontend repository integration
- [ ] More LLM providers (Anthropic, Ollama local)
- [ ] Vector DB for semantic resource search
- [ ] Multi-language support