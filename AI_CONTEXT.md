# AI Content

Status: canonical AI/system context for this repository
Last verified: 2026-07-02
Audience: humans, agents, and LLMs working inside this repo

If this file conflicts with README.md, docs/architecture.md, or older comments in the code, treat this file as the current single source of truth.

## 1. Identity And Naming

- Product name: DoclineAi
- Repository name: Agentic-PVS
- Local workspace/folder naming: current checkout may be `Docline/Docline`; older scripts/docs may still say `DoclineAi`. Both refer to the same project.
- Domain: on-premise, voice-first, autonomy-targeted clinical operating system for outpatient care; current repo reality is still a Germany-oriented PVS prototype, while the strategic launch path now starts in Austria

The naming is historically inconsistent. In code and docs, the product is primarily called DoclineAi. In Git/GitHub context, the repository currently appears as Agentic-PVS, while older history or docs may still mention Docline-PVS. Inside the local workspace, the folder name is DoclineAi. These names refer to the same project.

## 2. What This Project Is

DoclineAi is currently an AI-assisted Praxisverwaltungssystem prototype for outpatient care. The present codebase is still strongest as a local-first, voice-first, safety-constrained PVS foundation. The strategic product target is larger: Docline should become a fully autonomous clinical operating system.

That target does not replace the current truth. It adds a transformation sequence on top of it:

1. functional base PVS,
2. Austria-first SVC / e-card integration,
3. ELGA interoperability,
4. transition into an ambient, policy-bound autonomous clinical operating system.

The canonical roadmap for that transition lives in [docs/autonomous-clinical-os.md](docs/autonomous-clinical-os.md). The expanded implementation backlog (M5-M9) lives in [docs/clinical-os-backlog.md](docs/clinical-os-backlog.md).

A gap analysis against the Austrian general-practice (Hausarzt) PVS gold standard lives in [docs/austria-hausarzt-gap.md](docs/austria-hausarzt-gap.md). It identifies Austrian daily-value features still missing in both code and the prior roadmap - Austrian billing/Honorarnoten/Registrierkasse, e-Impfpass, recall/prevention, chronic-disease management, online self-service, telemedicine, and mobile/house visits - now planned as milestone M9 - Austria GP Daily-Value. The DE → AT localization layer (billing catalogs, connector stubs, provider/payer identifiers, terminology) has since been reoriented to Austria - EBM/GOÄ → ÖGK/BVAEB/SVS Kassentarif + Wahlarzt-Honorarnote, gematik/KIM/eAU → e-card/ELGA/Krankmeldung, LANR → ÖÄK/GDA, GKV/PKV → e-card vs. Wahlarzt; only the real external connectors remain outstanding.

The system is designed to let doctors and medical assistants trigger workflows by voice or text while keeping patient data local to the practice environment.

The core idea is not "chat with an LLM" in the generic sense. The core idea is:

1. capture structured practice intent from spoken or typed German commands,
2. route that intent into safe workflow tools,
3. require explicit confirmation for risky or clinical writes,
4. persist results as FHIR-shaped data in PostgreSQL,
5. keep a full audit trail of AI-proposed and AI-executed actions.

This repository contains both the frontend application and the backend services required for that loop.

## 3. USPs

- On-premise first: the architecture is designed so patient and speech data stay local by default.
- Voice-first UX: voice is a primary interaction mode, not a thin add-on.
- Safety-by-confirmation: clinical and write actions are previewed first and require explicit confirmation.
- FHIR-native persistence: domain data is stored as full FHIR JSONB resources plus search columns.
- Current German practice context with Austria-first roadmap: much of the checked-in workflow naming still targets German outpatient care, while the delivery roadmap now shifts national integration work toward Austria first.
- Auditability: prompt version, model, action, result, and initiator are recorded for AI actions.
- Offline-leaning operational model: queueing and local runtime assumptions are part of the design.

## 4. Current Product Scope

The repository covers these functional surfaces:

- practice foundation: organizations, practices, practitioner profiles, role assignments, consent, break-glass, policy-aware administration, and tenant-scoped auth/session flows
- patient lookup, candidate resolution, patient opening, identifier lookup, patient master data management, allergy/coverage subresources, and longitudinal patient chart aggregation
- appointment listing with patient/resource/date-range filters, slot search, follow-up lookahead, booking preview/execution, calendar-block previews, and scheduling administration for resources, visit types, availability rules, and calendar blocks
- ePA/encounter workspace flows, especially structured Anamnese/Befund/Diagnose/Therapie documentation, artifact generation, artifact lifecycle transitions, diagnosis and observation capture, encounter finalization, ambient capture, and persisted supervision queue items
- prescriptions and TI-related scaffolding
- billing-related UI and backend surfaces, including encounter-linked billing event capture and Austrian Kassentarif validation
- today workspace, communications, controlling, laboratory, documents, tools, and administration modules
- voice-assisted document lookup/preview, direct structured lab-value retrieval, medication lookup, and patient-chart search

Important reality check:

- practice foundation, patient chart, and encounter workspace slices are now concrete REST/UI surfaces, not just roadmap items
- today workspace, scheduling administration, and runtime auth/session identity are also concrete repo surfaces, not just roadmap language
- the voice path has been split into a separate `voice-service` process on port 7002; the core `pvs-api` on port 7001 remains REST-only and does not depend on live voice
- not every visible module has equally deep AI workflow coverage yet
- the most concretely wired end-to-end AI flows today are patient, appointment, selected documentation/lab workflows, patient master-data preview, calendar-block preview, and the first ambient-supervision materialization path
- the newest ambient slice turns selected AmbientPlanSteps into persisted supervision queue items linked to preview IDs, evidence, policy status, and confirmed execution outcomes
- the current assisted core flows now carry confidence, uncertainty, fallback reason, and manual-target metadata through the shared preview contract
- billing is routed as a first-class domain in the orchestrator, but currently degrades to a guided manual billing path when no billing tool matches
- document and encounter artifacts now carry explicit lifecycle state through `generation_status` and read-model `artifact_status`
- patient search/opening now returns candidate lists for ambiguous matches instead of forcing an unsafe single-patient choice
- voice results are rendered in a dedicated results panel, and browser speech output can be cancelled from the shell/hook path
- some modules are more UI/API scaffolding than fully realized agent automation

Transformation direction from this current scope:

- first harden the repo into a clinic-runnable base PVS
- then add Austria-first national infrastructure via SVC / e-card
- then add ELGA interoperability
- then replace command-style interaction with ambient, policy-bound autonomy

## 5. Architecture Summary

At a high level, the system is a monorepo with:

- React + TypeScript frontend
- FastAPI `pvs-api` backend on port 7001 for the core REST/PVS surface
- separate FastAPI `voice-service` on port 7002 for browser WebSocket audio, STT, voice session context, and gateway calls into the backend domain library
- PostgreSQL as the primary application datastore
- FHIR resources stored as JSONB plus indexed projection columns
- optional HAPI FHIR server running alongside the app stack
- local or configurable AI providers for LLM and speech

Canonical runtime flow:

1. The frontend opens a WebSocket to the `voice-service` endpoint on port 7002.
2. The frontend sends either text input or PCM audio chunks.
3. The voice-service keeps voice-local session context and delegates domain work through `voice_app/gateway.py` into `backend/app.*`.
4. Audio is transcribed by the configured STT service.
5. Direct deterministic paths handle very concrete lab-value questions before LLM routing.
6. With `VOICE_FAST_PATH=true`, the orchestrator first tries a single LLM tool-routing call across exposed tools.
7. If the fast path misses, the IntentPlanner produces a structured IntentPlan and the SupervisorAgent routes the intent domain (patient, appointment, documentation, prescription, billing, general).
8. The domain-specific sub-agent runs against the tool registry and produces a preview, query result, or workflow-specific manual fallback target.
9. If no tool matches for a core workflow, the orchestrator degrades to a deterministic manual handoff; only non-core/general paths fall through to a direct LLM text response.
10. If confirmation is required, the frontend receives a confirmation_request event bound to a server-issued preview_id.
11. After confirmation, the backend resolves that preview_id against pending server state, executes the write action, updates any linked supervision queue item, and emits UI updates.
12. Every relevant action is eligible for monitoring, queue lifecycle tracking, and audit logging.

## 6. Frontend Architecture

Stack:

- React 18
- TypeScript
- Vite
- Zustand for app state
- Tailwind is present in the toolchain, but the app also uses substantial custom CSS

Main entrypoints:

- frontend/src/main.tsx
- frontend/src/app/App.tsx
- frontend/src/app/DoclineShell.tsx
- frontend/src/app/Shell.tsx

Frontend mental model:

- App.tsx mounts `AuthGate` and `DoclineShell` directly.
- DoclineShell is the current application shell.
- The UI has two major modes:
   - a voice-first home mode centered around the Today workspace, agenda/inbox context, and a primary voice board
   - a module mode with grouped sidebar navigation, a sticky workspace toolbar, and a floating voice dock
- Global state lives in frontend/src/stores/appStore.ts.
- Runtime auth/session identity is mirrored into frontend state via active organization, active practice, session user, and available tenant data.
- PatientRecord includes a patient master-data panel, allergy/coverage management, and a timeline/chart view backed by frontend/src/modules/patients/PatientChart.tsx.
- The ePA module is now centered on frontend/src/modules/epa/EncounterWorkspace.tsx with Anamnese/Befund/Diagnose/Therapie editing, artifact generation, artifact status transitions, billing event visibility, and finalize actions.
- The documents module supports richer document detail previews and voice-driven document selection.
- The administration module exposes OrganizationOverview, UserManagement, PracticeSettings, SchedulingAdministration, LandingPreferences, and SecurityCenter for practice foundation, scheduling, voice landing behavior, and security primitives.
- The Today workspace in frontend/src/modules/today/TodayWorkspace.tsx aggregates queue, waiting room, tasks, inbox, and metrics from controlling and communications services.
- Voice transport and browser audio capture live in frontend/src/hooks/useVoiceSession.ts.
- WebSocket connection creation lives in frontend/src/services/websocket.ts.

Frontend modules currently present in frontend/src/modules:

- today
- patients
- appointments
- epa
- documents
- prescriptions
- billing
- laboratory
- communications
- controlling
- tools
- administration

Frontend state that matters for AI flows:

- active module
- active patient
- active encounter
- active practitioner
- active organization and active practice
- session user and available tenants
- user role / workflow mode
- pending confirmations and pending actions with confidence, uncertainty, fallback reason, and manual-target metadata
- persisted ambient supervision queue items surfaced inside the encounter workspace
- voice result panel state, patient candidate lists, document previews, and landing preferences for voice-driven navigation
- voice previews
- voice-derived appointments
- voice-derived diagnoses
- assisted execution status for preview, execute, cancel, or manual handoff states

Important frontend truth:

- the frontend speaks assistant responses via browser Web Speech by default, but prefers non-empty backend WAV audio when `TTS_ENGINE=piper` is available
- the frontend can cancel currently spoken assistant output through useVoiceSession.cancelSpeech(), which clears browser speechSynthesis and any HTMLAudio fallback
- voice appointment results such as `slot_search_results` and `show_schedule` are persisted in appStore and rendered in the calendar UI
- non-empty backend `voice_response.audio` is now the primary spoken path when Piper is configured; browser speechSynthesis remains the default/fallback path

## 7. Backend Architecture

Stack:

- Python 3.12
- FastAPI
- SQLAlchemy async
- Alembic
- LangGraph-compatible orchestration layer

Main backend entrypoint:

- backend/app/main.py

Routers exposed from main.py:

- documents
- auth
- patients
- appointments
- audit
- epa
- prescriptions
- ti
- billing
- laboratory
- communications
- controlling
- tools
- monitoring
- session
- practice-foundation
- consents
- break-glass

The live voice WebSocket router is no longer mounted in `backend/app/main.py`; it lives in `apps/voice-service/voice_app/ws.py` and reaches the backend domain through `apps/voice-service/voice_app/gateway.py`.

Important backend package layout:

- backend/app/api/v1: HTTP and WebSocket API surface
- backend/app/agents: orchestration, IntentPlanner, SupervisorAgent, ToolCallValidator, sub-agents, tool registry, workflow helpers
- backend/app/agents/sub_agents: domain sub-agents (PatientAgent, AppointmentAgent, DocumentationAgent, PrescriptionAgent, BillingAgent)
- backend/app/services: LLM, STT, TTS, auth, session identity/context, FHIR, patient chart, encounter workspace, scheduling, practice foundation, audit, monitoring, external integration services
- backend/app/repositories: persistence and query layer
- backend/app/core: config, database, middleware, security
- backend/app/models: DB and FHIR models

How process_voice_message() works:

- entry point is process_voice_message() in backend/app/agents/orchestrator.py
- concrete lab-value questions can be routed to `documentation_get_lab_value` before LLM planning
- with `VOICE_FAST_PATH=true`, it first attempts a single LLM tool-routing call against the exposed tool catalog
- if the fast path misses, it compiles and invokes a LangGraph StateGraph as the full runtime path
- the graph is a singleton compiled once via _get_graph() and reused across requests
- the legacy _legacy_process() function exists as a fallback if graph construction fails
- each graph node appends to a shared events list in DoclineState; the final events list is returned through the voice-service gateway

## 8. Voice And Interaction Protocol

Primary endpoint:

- WebSocket: ws://localhost:7002/api/v1/ws/voice/{session_id} in local development
- The legacy backend port 7001 is REST/core API only after the service split.

Client-to-server message types:

- audio_start
- audio_chunk
- audio_end
- text_input
- ambient_step_preview
- confirmation, resolved by preview_id against the server-side pending preview store
- session_resume
- vad_control
- health_request

Server-to-client event types:

- status
- partial_transcription
- transcription
- intent_detected
- ui_update
- tool_call
- voice_response
- confirmation_request
- action_executed
- action_cancelled
- ambient_capture
- ambient_candidates
- session_welcome
- session_health
- vad_config
- error
- session_context

Session context is tracked server-side in apps/voice-service/voice_app/session_context.py and mirrored into the frontend store. This is what keeps the active patient, module, practitioner, and pending confirmation aligned across turns.

Important protocol truth:

- confirmation payloads are no longer trusted as executable client input; the backend resolves confirmations against server-issued preview IDs
- preview and manual-handoff payloads can carry workflow, confidence, confidence_band, uncertainty, fallback_reason, and manual_target metadata

## 9. AI Runtime: What Models Exist, What Is Actually Used, And How

### 9.1 Checked-in canonical defaults

Defaults come from backend/app/core/config.py and .env.example, not from any developer-specific local .env.

- LLM_PROVIDER = gemini
- LLM_MODEL = gemini-3.1-flash-lite
- GEMINI_API_KEY = (required when LLM_PROVIDER=gemini; set in .env)
- GOOGLE_API_KEY = (legacy alias for GEMINI_API_KEY; accepted but not preferred)
- OLLAMA_BASE_URL = `http://localhost:11434` (used when LLM_PROVIDER=ollama)
- LLM_ROUTER_MODEL = empty by default, meaning router/tool selection uses LLM_MODEL
- VOICE_FAST_PATH = true
- STT_PROVIDER = parakeet
- VOICE_STT_PROVIDER = whisper for the Docker voice-service default
- NVIDIA_PARAKEET_MODEL = nvidia/parakeet-tdt-0.6b-v3
- WHISPER_MODEL = medium
- WHISPER_CACHE_DIR = ./models/whisper by default
- MLX_WHISPER_MODEL = mlx-community/whisper-large-v3-turbo
- TTS_ENGINE = browser by default; piper enables local backend WAV synthesis

On-premise deployments that cannot use Gemini should set LLM_PROVIDER=ollama and LLM_MODEL=llama3.2:3b (or any locally available model). The full Ollama code path is maintained and tested.

### 9.2 LLM providers

Configured enum values in settings:

- ollama
- mistral
- openai
- gemini

What the current LLMService actually implements in code:

- ollama
- gemini

What this means in practice:

- Ollama and Gemini are real current code paths.
- Mistral and OpenAI are configuration-level options today, not fully implemented runtime branches in backend/app/services/llm_service.py.
- Any claim that Mistral EU is a current first-class runtime path is aspirational or architectural intent, not the most precise description of the present code.

### 9.3 LLM model usage pattern

The LLM is used in several distinct ways inside and around the agent graph:

0. Fast single-call tool routing (orchestrator fast path)
   - Enabled by VOICE_FAST_PATH.
   - Uses the normal tool catalog and validator, but skips the separate IntentPlanner/Supervisor roundtrip when exactly one tool call is selected.
   - Falls back to the full graph on a miss.

1. Intent planning (IntentPlanner)
   - Gemini: with_structured_output(IntentPlan) via LangChain, zero temperature.
   - Ollama: httpx JSON-mode completion to /api/chat with format="json".
   - Output: IntentPlan with one or more PlanStep objects, estimated risk, and parallel_safe flag.
   - Fallback: deterministic single-step plan derived from session_context.active_module.

2. Domain routing (SupervisorAgent)
   - Gemini: with_structured_output(IntentClassification) via LangChain.
   - Ollama: httpx JSON-mode completion.
   - Output: domain string (patient | appointment | documentation | prescription | billing | general).
   - Level-1: accepted when LLM confidence >= 0.6.
   - Level-2 fallback: deterministic mapping from session_context.active_module.

3. Tool selection and direct response (sub-agents + fallback_responder)
   - Sub-agents select exactly one tool call when a structured workflow should run.
   - Gemini uses bind_tools via LangChain; Ollama uses function calling via /api/chat.
   - ToolCallValidator sanitizes arguments (UUID patient_ids, ICD-10-GM codes, ISO 8601 dates) before execution.
   - If no tool matches in a core workflow, the orchestrator emits a deterministic manual fallback target instead of silently drifting into free-text agent behavior.
   - Only non-core/general flows fall through to fallback_responder_node for a short direct German response.
   - The system never invents tool results or uses heuristic fallbacks at the sub-agent level.

A test in backend/tests/test_agents/test_workflow_tools.py guards the no-heuristic rule explicitly.

### 9.4 STT providers

Configured STT providers:

- parakeet
- whisper
- mlx

Actual behavior:

- core default STT is NVIDIA Parakeet via NeMo, loaded locally from the Hugging Face model id in NVIDIA_PARAKEET_MODEL
- Docker voice-service default is Whisper via VOICE_STT_PROVIDER=whisper, with model cache under WHISPER_CACHE_DIR
- on Apple Silicon, STT_PROVIDER=mlx enables MLX-Whisper with the configured MLX_WHISPER_MODEL
- if STT_PROVIDER is parakeet or mlx but the required module is unavailable, the service falls back to local Whisper
- this fallback is explicitly implemented and tested

The current canonical speech model is therefore:

- native core default: nvidia/parakeet-tdt-0.6b-v3
- Apple Silicon performance path: MLX-Whisper, default mlx-community/whisper-large-v3-turbo
- Docker voice-service default and fallback: Whisper, default size medium

### 9.5 TTS reality

There is a TTS abstraction in backend/app/services/tts_service.py and settings expose TTS_ENGINE values:

- browser
- kokoro
- piper
- elevenlabs_eu

The current real backend TTS implementation is Piper: when TTS_ENGINE=piper and a local Piper CLI/model are available, backend/app/services/tts_service.py synthesizes WAV bytes and the orchestrator attaches them to voice_response.audio.

Default UX still uses the frontend browser speechSynthesis path. The frontend now prefers non-empty backend audio first and falls back to browser speechSynthesis when Piper is disabled, missing, or cannot produce audio.

Do not assume Kokoro is operationally wired end-to-end just because the setting exists.

## 10. Tooling, Safety Model, And Confirmation Rules

The core AI safety pattern is implemented in the tool registry and workflow preview system.

Tool definitions live in backend/app/agents/tool_registry.py and are classified by:

- risk: read, administrative, clinical
- mode: query, preview, confirmed_execution
- requires_confirmation
- llm_exposed

Current tool families include:

- patient search, candidate resolution, patient open, and patient master-data update preview
- appointment list with patient/resource/date filters
- appointment slot search
- appointment booking preparation and booking execution
- calendar block preparation
- document listing, document search, structured lab-value retrieval, medication listing, and patient-chart search
- diagnosis preparation and diagnosis write
- observation preparation and observation write

Current preview contracts can additionally carry:

- workflow identity for the current assisted flow
- confidence and confidence_band
- uncertainty reasons
- fallback_reason
- manual_target for deterministic handoff into the corresponding frontend module

Separate REST APIs, not LLM tool calls, currently handle patient chart aggregation, encounter workspace lifecycle, practice foundation administration, consent, break-glass, and policy evaluation.

Current coverage nuance:

- billing now has a routed domain sub-agent and deterministic manual fallback path
- full billing and practice-foundation LLM tool coverage is still shallower than patient, appointment, and selected documentation workflows

Important rule:

- tools that perform confirmed execution are not directly exposed to the LLM for immediate execution
- instead, the LLM can usually produce a preview tool call first
- the user then confirms or cancels
- only after confirmation does the backend execute the write path
- confirmation is bound to a server-issued preview_id rather than arbitrary client payload contents
- if a core workflow is uncertain or unsupported, the system should degrade to preview or manual target instead of pretending full automation exists

This is the core mechanism behind the "doctor confirms clinical actions" principle.

## 11. Data Model And Persistence

Primary datastore:

- PostgreSQL 16

Schema:

- FHIR resources are stored under schema fhir

Alembic migration 0001 creates JSONB resource tables for:

- patient
- practitioner
- encounter
- condition
- medication_request
- observation
- appointment
- communication
- coverage
- allergy_intolerance
- claim
- invoice

Each table stores:

- resource_id
- patient_id
- status
- code
- full JSONB resource
- created_at / updated_at

Plus indexes, including a GIN index on the JSONB payload.

Additional projection/search columns are added for selected resources, for example:

- patient given_name, family_name, display_name, birth_date, insurance_number, insurance_type
- appointment starts_at
- encounter started_at, ended_at, practitioner_id
- condition recorded_at

Important implementation truth:

- the repository-backed FHIR service currently operates on an expanded subset of resource tables in active application code:
  - patient
  - appointment
  - encounter
  - condition
  - observation
   - allergy_intolerance
   - coverage
- the broader migration footprint is larger than the currently repository-backed service surface

Additional persisted surfaces beyond the core FHIR JSONB resource tables now include:

- organization, practice, practitioner_profile, role_assignment, user_account, auth_identity, auth_session, auth_invitation, consent, and break_glass_event in the practice foundation/auth layer
- schedule_resource, visit_type, availability_rule, and calendar_block for scheduling administration
- encounter_workspace and encounter_billing_event under schema fhir
- document metadata columns for encounter_id, artifact_kind, generation_status, and origin; current read models additionally expose artifact_status from lifecycle data
- ambient_supervision_queue, ambient_full_consult/ambient plan store, voice_transcript_turn, lab_alert_queue, prescription, and medication_change_proposal surfaces through migrations up to 0021

## 12. HAPI FHIR: Present But Not The Active Source Of Truth

The Docker stack starts a HAPI FHIR R4 server.

However, in the current backend application code:

- FHIR_SERVER_URL exists in settings
- but there is no active usage of FHIR_SERVER_URL inside backend/app
- the operational persistence path used by the app is the custom PostgreSQL-backed FHIR repository/service layer

So the current truth is:

- HAPI FHIR is part of the deployment topology
- custom PostgreSQL FHIR tables are the active application persistence layer
- do not assume the app is currently HAPI-first

## 13. Audit, Monitoring, And Compliance Posture

Audit logging is part of the runtime design, not an afterthought.

Alembic migration 0002 adds fhir.audit_log with fields including:

- user_id
- user_roles
- session_id
- timestamp
- resource_type
- resource_id
- action
- initiated_by_ai
- prompt_version
- model
- metadata_json

The system records model and prompt metadata for AI-driven actions. This is directly aligned with the compliance posture documented in docs/regulatory/eu-ai-act.md.

Core compliance/product guardrails repeated across the codebase and docs:

- no silent clinical writes
- explicit human confirmation for risky actions
- local-first data handling
- auditable AI behavior
- high-risk medical AI assumptions under EU AI Act framing

Concrete current-state compliance primitives now include:

- permission evaluation with role/permission checks plus consent and break-glass handling
- audit reads filterable by actor_kind, organization_id, practice_id, policy_decision, and resource_type

## 14. Runtime And Deployment Model

The canonical local startup script is start.sh.

What start.sh does conceptually:

- ensures .env exists
- starts PostgreSQL and HAPI FHIR through Docker Compose
- decides whether backend/frontend run natively or in Docker
- clears stale native `uvicorn app.main:app --reload` listeners on the backend port before relaunch
- refreshes missing backend dependencies in backend/.venv when native startup detects drift
- when LLM_PROVIDER=ollama, ensures local Ollama is running on the host and pulls the configured model if needed
- when native backend uses Parakeet on Apple Silicon, installs backend/requirements.native.txt if necessary
- when native backend uses MLX-Whisper on Apple Silicon, installs backend/requirements.native.mlx.txt if necessary
- falls back to Whisper if Parakeet/MLX dependencies are unavailable but Whisper exists locally

Default local development pattern today:

- database and HAPI in Docker
- backend native on the host
- voice-service native on the host for live WebSocket/STT when using native startup
- frontend native via Vite
- Ollama native on the host if used

Supporting scripts:

- start.sh
- stop.sh
- restart.sh

Ports used by default:

- frontend: 3000
- backend: 7001
- voice-service: 7002
- HAPI FHIR: 8080
- Ollama: 11434

Important Docker truth:

- Docker Compose points backend Ollama traffic to host.docker.internal by default
- this allows a Dockerized backend to call a host-native Ollama instance

## 15. Security And Auth Posture

Current checked-in defaults are runtime-oriented, not developer-friendly.

- AUTH_MODE defaults to runtime
- AUTH_REQUIRED defaults to true
- local development must explicitly opt into AUTH_MODE=dev to rely on x-user-id and x-user-roles headers or DEFAULT_DEV_ROLES fallback without bearer tokens
- runtime auth uses JWT access tokens, refresh sessions, tenant bootstrap/invite flows, and explicit tenant selection

When the DB-backed policy layer is in play, role/permission checks can still be combined with patient consent and active break-glass events even in local development mode.

Relevant security behavior:

- request logging middleware exists
- rate limiting middleware exists
- /api/v1/auth provides register, login, refresh, logout, tenant selection, invitation, and admin user flows
- /api/v1/session/me resolves practitioner context, permissions, auth_mode, and available_tenants into the frontend store
- role/permission catalog and /permissions/me endpoints exist under the practice-foundation router
- consent and break-glass REST flows exist alongside audit filters for policy decisions
- tenant-sensitive admin and encounter routes enforce active tenant scoping in addition to role checks
- confirmation guardrails exist for risky actions
- offline queue path exists for TI-related resilience

## 16. Tests And Validation Surface

Backend tests exist for:

- agents (orchestrator, intent_planner, supervisor_agent, tool_call_validator, query_agent, workflow_tools)
- voice-service startup, WebSocket helpers, session context, and STT behavior
- API
- auth/session and scheduling services
- billing
- services, including permissions, policy, patient chart, encounter workspace, seed data, and STT/LLM adapters
- evaluation datasets

Frontend tests exist for:

- voice bar behavior and voice store synchronization
- patient search and patient-record flows
- patient master-data, allergy, and coverage flows
- appointment quick book, scheduling, and calendar mapping/views
- encounter workspace and artifact status behavior
- voice results panel, speech cancellation, and voice-driven document/result handling
- billing UI
- tool console
- lab report prototype

Examples of what is explicitly tested:

- tools endpoint exposes the default gemini/gemini-3.1-flash-lite model metadata
- LLM tool routing has no heuristic fallback when no tool is selected
- the single-call voice fast path routes one selected tool through the same validator/confirmation gate as the full graph
- direct lab-value questions such as Kreatinin can bypass broad document search and call `documentation_get_lab_value`
- IntentPlanner produces single-step and multi-step plans with correct domain and risk classification
- IntentPlanner falls back to deterministic single-step plan when LLM is unavailable
- SupervisorAgent accepts LLM classification when confidence >= 0.6, falls back to active_module otherwise
- ToolCallValidator rejects non-UUID patient_ids, fills them from session context, normalizes ICD-10-GM codes, and validates ISO 8601 dates
- confirmation resolution rejects client payload tampering and executes only server-tracked previews
- patient candidate resolution returns search results for ambiguous patient matches
- appointment queries respect patient, resource, and date-range context
- runtime auth resolves session identity, tenant context, and guarded admin actions
- patient chart aggregation returns identifiers, sections including allergies and coverages, timeline, and provenance summary
- encounter workspace service loads/persists Anamnese/Befund/Diagnose/Therapie docs, artifacts, artifact status transitions, billing events, and finalize state
- policy service combines roles, consent, and break-glass for access decisions
- break-glass lifecycle and seed test data plan/dry-run coverage
- Parakeet/MLX fall back to Whisper when native STT dependencies are unavailable
- streaming STT buffer behavior for container audio formats
- scheduling service covers resources, visit types, availability rules, and calendar blocks
- frontend voice store keeps confidence and manual fallback metadata in sync with pending assisted actions

## 17. What Is Fully Real Today Vs What Is Directional

### Real in current code

- split `pvs-api` and `voice-service` runtime, with voice WebSocket transport on port 7002
- text and audio input handling
- local Parakeet STT, MLX-Whisper support on Apple Silicon, and Whisper fallback/default in the Docker voice-service
- Ollama and Gemini code paths for LLM usage (Gemini is now the default)
- voice fast path for single-call tool routing plus deterministic direct lab-value routing
- LangGraph StateGraph as the active voice request execution engine
- IntentPlanner: structured multi-step plan from voice input (Gemini structured output or Ollama JSON mode)
- SupervisorAgent: 2-level domain routing (LLM confidence >= 0.6, then deterministic fallback)
- ToolCallValidator: argument sanitization before tool execution (UUID, ICD-10-GM, ISO 8601)
- domain sub-agents: PatientAgent, AppointmentAgent, DocumentationAgent, PrescriptionAgent, BillingAgent
- tool-based preview/confirmation/execute workflow model with confidence, uncertainty, fallback reason, and manual-target metadata
- deterministic manual fallback for core workflows when tool coverage is missing or confidence is insufficient
- JWT-based runtime auth with refresh sessions, tenant selection, invitation/bootstrap onboarding, and session identity hydration
- PostgreSQL-backed FHIR persistence for core resource types
- longitudinal patient chart aggregation, patient identifier endpoints, allergy/coverage subresources, and patient master-data UI
- encounter workspace, artifact generation, artifact lifecycle updates, finalize flow, and encounter billing events
- scheduling administration for schedule resources, visit types, availability, and calendar blocks
- voice-assisted patient candidate resolution, appointment resource/date queries, document search/listing, lab-value lookup, medication lookup, and patient-chart search
- preview tools for patient master-data changes and calendar blocks
- practice foundation REST surfaces for organizations, practices, practitioners, and roles
- consent, break-glass, and permission/policy services with audit integration
- audit logging with prompt/model metadata
- deterministic seed data script and coverage for the base PVS surface
- frontend voice-first shell with Today workspace, module mode, pending confirmation UX, and manual fallback hints

### Present but partial, stubbed, or directional

- Mistral and OpenAI as fully implemented LLM runtime branches
- Kokoro or ElevenLabs-EU as real end-to-end spoken output engines
- HAPI FHIR as the backend application's active persistence source
- equal AI depth across every business module shown in the UI
- multi-step IntentPlan execution (parallel_safe routing; currently single-step is the dominant path)
- Austria-first SVC / e-card integration
- ELGA interoperability and e-Medikation exchange
- ambient full-consultation listening with diarization and encounter memory
- autonomy-tiered policy engine with explicit execution envelopes beyond today's role/permission/consent/break-glass evaluator
- post-GUI supervisory workspace where review and intervention replace manual GUI driving as the primary operating model

## 18. Files To Read When Changing Specific Behavior

If you need to change voice transport or session protocol:

- apps/voice-service/voice_app/ws.py
- apps/voice-service/voice_app/gateway.py
- apps/voice-service/voice_app/session_context.py
- frontend/src/hooks/useVoiceSession.ts
- frontend/src/stores/appStore.ts
- frontend/src/services/websocket.ts

If you need to change LLM routing or provider behavior:

- backend/app/services/llm_service.py
- backend/app/agents/orchestrator.py
- backend/app/agents/intent_planner.py
- backend/app/agents/supervisor_agent.py
- backend/app/agents/tool_call_validator.py
- backend/app/agents/sub_agents/_base.py
- backend/app/agents/sub_agents/billing_agent.py
- backend/app/agents/tool_registry.py

If you need to change STT behavior:

- apps/voice-service/voice_app/stt.py
- start.sh
- backend/requirements.native.txt
- backend/requirements.native.mlx.txt

If you need to change confirmation and safe execution behavior:

- backend/app/agents/tool_registry.py
- backend/app/agents/workflow_tools.py
- backend/app/agents/guardrails.py
- backend/app/agents/orchestrator.py
- apps/voice-service/voice_app/ws.py
- apps/voice-service/voice_app/gateway.py
- frontend/src/hooks/useVoiceSession.ts
- frontend/src/stores/appStore.ts

If you need to change patient chart or encounter workspace behavior:

- backend/app/api/v1/patients.py
- backend/app/api/v1/epa.py
- backend/app/services/patient_service.py
- backend/app/services/encounter_workspace_service.py
- frontend/src/modules/patients/PatientChart.tsx
- frontend/src/modules/epa/EncounterWorkspace.tsx
- frontend/src/services/patientService.ts
- frontend/src/services/encounterWorkspaceService.ts

If you need to change FHIR persistence or search behavior:

- backend/app/services/fhir_service.py
- backend/app/repositories/fhir_repository.py
- backend/alembic/versions/

If you need to change practice foundation, consent, break-glass, or permission logic:

- backend/app/api/v1/practices.py
- backend/app/api/v1/consents.py
- backend/app/api/v1/break_glass.py
- backend/app/api/v1/audit.py
- backend/app/core/permissions.py
- backend/app/services/policy_service.py
- backend/app/services/practice_service.py
- backend/app/services/consent_service.py
- backend/app/services/break_glass_service.py
- frontend/src/modules/administration/PracticeSettings.tsx
- frontend/src/modules/administration/OrganizationOverview.tsx
- frontend/src/modules/administration/SchedulingAdministration.tsx
- frontend/src/modules/administration/SecurityCenter.tsx
- frontend/src/modules/administration/UserManagement.tsx

If you need to change runtime auth or tenant/session behavior:

- backend/app/api/v1/auth.py
- backend/app/api/v1/session.py
- backend/app/services/auth_service.py
- backend/app/services/session_identity.py
- backend/app/core/security.py
- frontend/src/services/auth.ts
- frontend/src/stores/appStore.ts

If you need to change scheduling administration or Today workspace behavior:

- backend/app/api/v1/appointments.py
- backend/app/services/scheduling_service.py
- backend/app/api/v1/communications.py
- backend/app/api/v1/controlling.py
- frontend/src/modules/today/TodayWorkspace.tsx
- frontend/src/modules/today/usePracticeWorkspaceData.ts
- frontend/src/modules/administration/SchedulingAdministration.tsx
- frontend/src/services/appointmentService.ts
- frontend/src/services/communicationService.ts
- frontend/src/services/controllingService.ts

If you need to change frontend workflow state sync:

- frontend/src/stores/appStore.ts
- frontend/src/app/DoclineShell.tsx

## 19. Canonical Mental Model For Future LLMs

If you only remember one thing about this repo, remember this:

Current reality: DoclineAi is a local-first, voice-first, safety-constrained medical workflow system. The real value today is not generic chat quality. The real value today is reliable conversion of practice commands into auditable, FHIR-backed workflow actions with confirmation gates for anything risky.

Target reality: the product is being steered toward a fully autonomous clinical operating system where the consultation itself becomes the primary interface, but autonomy always remains bounded by policy, auditability, consent, signatures, and human supervision.

And if you only remember one implementation nuance, remember this:

the present codebase is more concrete in its voice transport, STT pipeline, tool registry, and PostgreSQL FHIR persistence than in some of its broader architecture claims around cloud LLM providers, backend TTS, or HAPI FHIR integration.
