# Technical Audit - PsicoAsistente MVP

## Executive Summary
PsicoAsistente MVP is a FastAPI-based mental-health-oriented support assistant that provides authenticated chat guidance with crisis keyword detection, intent-based canned responses, and optional ElevenLabs text-to-speech output. The architecture is a small monolith with clear layer separation (routers/services/models/security/frontend), using JWT auth and SQLAlchemy persistence for users and chat history. It is portfolio-ready as an MVP reference for AI-adjacent UX patterns, but needs hardening (CORS policy, rate limiting, migration strategy, and stronger operational controls) before production use.

## First Impression (Dependency File + Entry Point)
- `requirements.txt` shows a backend-first stack: FastAPI/Uvicorn + SQLAlchemy + Pydantic Settings + JWT/crypto + bcrypt + requests, which strongly suggests an API-centric app with auth, DB persistence, and external TTS integration.
- The entry point `app/main.py` confirms this: FastAPI app setup, DB table auto-create, CORS middleware, exception handlers, router registration, and static serving for frontend and media.
- The startup flow indicates a single deployable service that serves both API and static frontend from the same process.

## Tech Stack
| Technology | Version | Purpose |
|---|---:|---|
| Python | 3.11+ (README recommendation) | Runtime for backend and tooling |
| FastAPI | 0.116.1 | REST API and app framework |
| Uvicorn | 0.35.0 | ASGI server |
| SQLAlchemy | 2.0.43 | ORM and DB access |
| Pydantic | 2.11.7 | Data validation / schemas |
| pydantic-settings | 2.10.1 | `.env` configuration loading |
| python-jose[cryptography] | 3.5.0 | JWT token encoding/decoding |
| passlib[bcrypt] + bcrypt | 1.7.4 + 4.1.3 | Password hashing/verification |
| requests | 2.32.4 | External HTTP calls (ElevenLabs API) |
| HTML/CSS/Vanilla JS | n/a | Frontend auth/chat UI |
| ElevenLabs API | external service | TTS generation |

## Project Map
- `app/`: Backend source with configuration, DB, auth/security, API routers, and services.
- `app/routers/`: HTTP boundary layer (`/auth`, `/chat`, `/tts`) handling request/response and dependencies.
- `app/services/`: Domain logic for assistant response generation and TTS integration.
- `frontend/`: Static client (login and chat views) consuming backend APIs via fetch + JWT in localStorage.
- `scripts/`: Windows scripts for local execution and packaging.
- `requirements.txt`: Python dependencies with pinned versions.
- `README.md`: Setup and manual test guidance, mostly Windows-focused.

## Architecture & Data Flow
1. User registers/logs in on `/` frontend; JS calls `/auth/register` or `/auth/login`.
2. Backend creates/verifies user, returns JWT and user payload.
3. Frontend stores JWT in localStorage and navigates to `/chat`.
4. Chat message posts to `/chat/message` with Bearer token.
5. Router stores user message in DB, fetches recent history, runs rule-based assistant engine (crisis + intent), then attempts TTS synthesis.
6. Assistant response and optional audio URL are persisted and returned to UI.
7. Frontend renders response metadata and auto-plays audio when available.

## Critical / Complex Logic
- `generate_safe_reply(...)` in `assistant_engine.py`: central decision tree for crisis detection, intent classification, and curated response templates.
- `chat_message(...)` in `app/routers/chat.py`: orchestrates DB writes, history retrieval, response generation, TTS fallback, and API output shaping.
- `get_current_user(...)` + JWT helpers in `app/security.py`: core auth gate for protected endpoints.
- `TTSService.synthesize(...)` in `tts_service.py`: external dependency boundary with network error handling and media persistence.

## Security & Operational Findings
### Strengths
- Password hashing via passlib/bcrypt and token-based auth gate on protected routes.
- Anti-enumeration behavior in login endpoint (generic credential error).
- Input schema limits on message/password/text lengths reduce abuse surface.

### Risks / Gaps
- CORS is fully open (`allow_origins=["*"]`) while using bearer tokens, increasing exposure for browser clients.
- No visible rate limiting, brute-force protection, or account lockout on auth endpoints.
- Generic exception handler may hide server details for clients but can reduce observability without structured logging.
- JWT `sub` is cast with `int(...)` in `get_current_user`; malformed-but-decodable tokens could raise unhandled `ValueError` and return 500.
- DB tables are auto-created at startup (`Base.metadata.create_all`) instead of migrations; risky for production evolution.
- Chat/history content is stored as plain text with no retention policy or encryption-at-rest strategy documented.

## Documentation Quality
### What works
- README clearly explains local run flow and endpoint smoke tests.
- `.env.example` documents required/optional configuration keys.

### Gaps
- README references files not present in repository (`test_elevenlabs_tts.py`).
- No testing strategy (unit/integration), CI instructions, or production deployment guidance.
- Limited cross-platform instructions (heavily PowerShell/Windows oriented).

## Code Quality Snapshot
### 3 Pros
1. Clear modular separation across routers/services/security/models.
2. Defensive API design in several areas (validation models, auth dependency, graceful TTS failure handling).
3. Good MVP UX continuity: text response still works when TTS fails.

### 3 Cons
1. Rule-based assistant logic is monolithic and hard-coded, making future scaling/model upgrades harder.
2. Security hardening is incomplete for internet-facing deployment (CORS, rate limits, auth abuse controls).
3. Missing automated tests and migration tooling introduce maintainability and reliability debt.
