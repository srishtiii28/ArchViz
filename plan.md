# AI-Powered Code Smell Visualizer — System Design Document

## Context

We are building a **browser extension** that transforms any GitHub repository into an interactive 3D visualization of code quality. The extension injects into GitHub pages and also provides a standalone popup, analyzes codebases using a hybrid in-browser + cloud approach, and supports JS/TS, Python, Java, Go, and Rust. It must work across Chrome, Firefox, Edge, and Safari.

This document covers every architectural layer, the technology choices at each layer, and the rationale behind each decision.

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    BROWSER EXTENSION                         │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌───────────┐  │
│  │ Content   │  │ Popup /  │  │ Background│  │ Web       │  │
│  │ Script    │  │ SidePanel│  │ Service   │  │ Workers   │  │
│  │ (GitHub   │  │ (Manual  │  │ Worker    │  │ (In-Browser│  │
│  │  Inject)  │  │  URL)    │  │ (Router)  │  │  Analysis)│  │
│  └─────┬─────┘  └────┬─────┘  └─────┬─────┘  └─────┬─────┘  │
│        │              │              │              │         │
│        └──────────────┴──────┬───────┴──────────────┘         │
│                              │                                │
│              ┌───────────────┴────────────────┐               │
│              │     VISUALIZATION ENGINE        │               │
│              │     Three.js (WebGL)            │               │
│              │  (Rendered in extension panel)   │               │
│              └───────────────┬────────────────┘               │
└──────────────────────────────┼────────────────────────────────┘
                               │ HTTPS
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                     CLOUD BACKEND                            │
│  ┌──────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │ API      │  │ Analysis     │  │ AI/ML Layer            │ │
│  │ Gateway  │──│ Engine       │──│ - OpenAI GPT-4o        │ │
│  │ (FastAPI)│  │ (Workers)    │  │ - Code Smell Classifier│ │
│  └────┬─────┘  └──────────────┘  └────────────────────────┘ │
│       │                                                       │
│  ┌────┴─────┐  ┌──────────────┐                              │
│  │PostgreSQL│  │ Redis Cache  │                              │
│  └──────────┘  └──────────────┘                              │
└──────────────────────────────────────────────────────────────┘
```

---

## LAYER 1: Browser Extension Shell

### What This Layer Does
The outermost container — the extension itself that lives in the browser toolbar. It handles permissions, lifecycle, cross-browser compatibility, and coordinates all other layers.

### Technology: WXT (Web Extension Toolkit)

**Why WXT over raw Manifest V3:**
- **Cross-browser from a single codebase.** WXT compiles to Chrome (Manifest V3), Firefox (MV3), Edge (MV3, same as Chrome), and Safari (via `web-ext` + Xcode wrapper). Without WXT, you would write 4 separate manifests and handle API differences manually.
- **Vite-based build system.** Hot Module Reload during development — you change code, the extension reloads automatically. Raw extension development requires manual reload every time.
- **TypeScript-first.** Full type safety across all extension APIs.
- **Module support.** Import npm packages, use modern JS features. Raw MV3 has severe restrictions on module loading.
- **Auto-generated manifest.** Define your entrypoints in code, WXT generates the correct manifest per browser.

**Why not Plasmo (the other popular framework):**
- Plasmo is React-only. WXT is framework-agnostic — important because our visualization layer uses Three.js which works better outside React's reconciler.
- WXT gives more low-level control over the service worker, which we need for the hybrid compute model.
- WXT has better Safari support.

### Extension Entrypoints

```
extension/
├── entrypoints/
│   ├── background.ts          # Service worker (persistent brain)
│   ├── content.ts             # Injected into github.com pages
│   ├── popup/                 # Toolbar popup (manual URL entry)
│   │   ├── index.html
│   │   └── main.tsx
│   └── sidepanel/             # Side panel for visualization
│       ├── index.html
│       └── main.tsx
├── components/                # Shared UI components
├── lib/
│   ├── analyzer/              # In-browser analysis (Web Workers)
│   ├── api/                   # Backend API client
│   └── visualization/         # Three.js rendering
└── assets/
```

### Cross-Browser Strategy

| Browser | Engine  | Extension API         | Notes                              |
|---------|---------|----------------------|------------------------------------|
| Chrome  | V8      | Manifest V3 native   | Primary target, largest market     |
| Edge    | V8      | Identical to Chrome   | Free — same build as Chrome        |
| Firefox | Gecko   | MV3 with differences  | WXT handles polyfills              |
| Safari  | WebKit  | Web Extension API     | Needs Xcode project wrapper        |

**Key decision:** We use `webextension-polyfill` (bundled in WXT) to normalize the `browser.*` API across all browsers. Code is written once using the `browser.*` namespace, and WXT compiles it correctly for each target.

---

## LAYER 2: Content Script (GitHub Integration)

### What This Layer Does
When the user navigates to a GitHub repository page, this script injects UI elements directly into the GitHub DOM — an "Analyze" button in the repo header, inline badges, and a panel for results.

### Technology: Vanilla TypeScript + Mutation Observer

**Why not React/Solid here:**
- Content scripts should be **tiny** (< 50KB). A React runtime adds ~40KB alone.
- GitHub is a single-page app that mutates its DOM via Turbo. We need `MutationObserver` to detect page navigations — a framework adds unnecessary complexity for what is essentially "find element, append button."
- Less collision risk with GitHub's own React instance.

### What It Does

1. **Detects repo pages:** Watches URL changes (GitHub uses `turbo:load` events) for patterns like `github.com/{owner}/{repo}`.
2. **Injects "Analyze" button:** Adds a button next to "Code" / "Issues" / "Pull requests" tabs.
3. **Communicates with background service worker:** Sends `chrome.runtime.sendMessage({ type: 'ANALYZE_REPO', payload: { owner, repo } })`.
4. **Opens side panel:** When results are ready, opens the extension's side panel with the 3D visualization.
5. **Inline annotations (Phase 2):** Badges on file listings showing complexity scores.

### GitHub-Specific Challenges

- **CSP (Content Security Policy):** GitHub has strict CSP. We cannot inject `<script>` tags. All our code runs in the content script's isolated world.
- **SPA Navigation:** GitHub uses Turbo for navigation. We must re-inject on every Turbo page load, not just initial page load.
- **Dark mode:** Must detect and match GitHub's theme. We read the `data-color-mode` attribute on `<html>`.

---

## LAYER 3: Popup & Side Panel (Standalone UI)

### What This Layer Does
Two UI surfaces:
1. **Popup** — Small window from toolbar icon. Enter a GitHub URL manually, see recent analyses, quick settings.
2. **Side Panel** — Full-height panel (Chrome 114+, Firefox 115+) where the 3D visualization and detailed results live.

### Technology: SolidJS + TailwindCSS

**Why SolidJS over React:**
- **Bundle size:** SolidJS is ~7KB. React is ~40KB. In an extension, every KB matters — users feel sluggish extensions.
- **True reactivity:** SolidJS compiles to direct DOM updates with no virtual DOM diffing. The visualization panel will have rapidly updating data (hover states, selections, live metrics) — SolidJS handles this without frame drops.
- **No re-renders:** React re-renders entire component trees. SolidJS only updates the exact DOM nodes that changed. Critical when we have a Three.js canvas that must NOT be unmounted/remounted.
- **Familiar JSX syntax:** Team members coming from React will find SolidJS almost identical in appearance.

**Why TailwindCSS:**
- Scoped by design — utility classes won't conflict with GitHub's CSS in the content script.
- No runtime CSS-in-JS overhead.
- Consistent design system out of the box.

### Popup UI
```
┌─────────────────────────┐
│  Code Smell Visualizer   │
├─────────────────────────┤
│ Enter GitHub URL:        │
│ [github.com/repo______ ]│
│ [     Analyze ->       ] │
├─────────────────────────┤
│ Recent:                  │
│ * facebook/react    Done │
│ * vercel/next.js   ...   │
│ * denoland/deno    Done  │
├─────────────────────────┤
│ Settings                 │
└─────────────────────────┘
```

### Side Panel UI
```
┌───────────────────────────────────────┐
│ repo: facebook/react                  │
│ 3,847 issues found                    │
├───────────────────────────────────────┤
│                                       │
│        [3D VISUALIZATION]             │
│        Three.js Canvas                │
│        (Interactive Galaxy)           │
│                                       │
├───────────────────────────────────────┤
│ Selected: ReactFiber.js               │
│ Complexity: 87/100 (critical)         │
│ Duplications: 3 blocks                │
│ Dependencies: 12 files depend on this │
├───────────────────────────────────────┤
│ AI Analysis:                          │
│ "This file violates SRP. It handles  │
│  reconciliation, scheduling, AND      │
│  error boundaries. Recommend split..." │
│                                       │
│ [View Suggestions] [Export Report]    │
└───────────────────────────────────────┘
```

---

## LAYER 4: Background Service Worker (The Router)

### What This Layer Does
The "brain" of the extension. It runs persistently (within MV3 constraints), coordinates between content scripts, popup, side panel, web workers, and the cloud backend. Think of it as a message bus + orchestrator.

### Technology: Plain TypeScript (no framework)

**Why no framework:**
- Service workers have no DOM. Frameworks are useless here.
- Must be as lightweight as possible — MV3 service workers are killed after 30 seconds of inactivity (Chrome) or 5 minutes (Firefox).
- Logic is pure message routing and state management.

### Responsibilities

1. **Message routing:**
   ```
   Content Script --message--> Service Worker --message--> Web Worker
                                    |
                                    |--> Side Panel
                                    `--> Cloud Backend (fetch)
   ```

2. **Analysis orchestration:**
   - Receives "ANALYZE_REPO" request
   - Checks Redis-backed cache (via API call) — already analyzed?
   - If small repo (< 500 files): route to in-browser Web Workers
   - If large repo: route to cloud backend
   - Merge results and send to visualization layer

3. **State persistence:**
   - Uses `chrome.storage.local` for analysis history, user preferences
   - Uses `chrome.storage.session` for in-progress analysis state (survives service worker restarts)

4. **Alarm-based keep-alive:**
   - For long analyses (> 30 seconds), uses `chrome.alarms` to prevent service worker termination
   - Polls backend for job status on alarm ticks

---

## LAYER 5: In-Browser Analysis (Web Workers + WASM)

### What This Layer Does
Performs lightweight, fast analysis directly in the browser — no server round-trip needed. Handles basic metrics that don't require ML/AI.

### Technology: Web Workers + Tree-sitter (WASM)

**Why Web Workers:**
- Analysis is CPU-intensive. Running on the main thread would freeze the UI.
- Web Workers run in a separate thread with zero UI impact.
- We can spawn multiple workers for parallel file analysis.

**Why Tree-sitter (compiled to WASM):**
- Tree-sitter is the industry standard for code parsing (used by GitHub, Neovim, Zed editor).
- It has grammars for ALL our target languages (JS/TS, Python, Java, Go, Rust).
- Compiled to WASM, it runs at near-native speed in the browser.
- Produces a concrete syntax tree (CST) that we can walk to extract metrics.
- **Why not Babel/ESLint parsers:** They only handle JavaScript. Tree-sitter handles all 5 languages with the same API.

### What Runs In-Browser

| Metric                     | How It Works                                       | Speed       |
|----------------------------|----------------------------------------------------|-------------|
| Lines of Code              | Count non-blank, non-comment lines                 | Instant     |
| Cyclomatic Complexity      | Count decision points in AST (if/for/while/&&/or)  | < 1ms/file  |
| Function Length            | Walk AST, measure function node spans              | < 1ms/file  |
| Nesting Depth              | Walk AST, track max depth                          | < 1ms/file  |
| Import/Dependency Map      | Extract import/require/use statements              | < 2ms/file  |
| Duplicate Detection (basic)| Hash function bodies, find collisions              | ~50ms/repo  |

### What Does NOT Run In-Browser (sent to cloud)

- Semantic similarity (needs ML embeddings)
- AI explanations (needs GPT-4o)
- Cross-file type flow analysis
- Advanced duplicate detection (code clones with variable renaming)
- Bug prediction model

### Architecture

```
Main Thread                    Worker Thread
    |                              |
    |--postMessage(files[])-->     |
    |                              |-- Load Tree-sitter WASM
    |                              |-- Load language grammar WASM
    |                              |-- Parse each file -> AST
    |                              |-- Walk AST -> metrics
    |                              |-- Build dependency graph
    |                              |-- Hash functions -> detect dupes
    |                              |
    |<--postMessage(results)---    |
    |                              |
```

### Worker Pool Strategy
- Spawn `navigator.hardwareConcurrency - 1` workers (leave 1 core for UI)
- Distribute files round-robin across workers
- Each worker loads Tree-sitter once, reuses for all files
- Results stream back as files complete (not batch)

---

## LAYER 6: Cloud Backend API

### What This Layer Does
Handles the heavy analysis that can't run in-browser: AI explanations, ML models, large repo processing, and persistent storage.

### Technology: Python + FastAPI

**Why Python over Node.js for the backend:**
- **ML ecosystem:** scikit-learn, sentence-transformers, and all ML libraries are Python-native. Using them from Node requires awkward Python subprocess calls or inferior JS ports.
- **Tree-sitter bindings:** `py-tree-sitter` is mature and fast for server-side parsing.
- **FastAPI specifically:** Async by default (critical for I/O-heavy workloads like GitHub API + OpenAI API calls), auto-generated OpenAPI docs, Pydantic validation, and arguably the best Python web framework for APIs.
- **Why not Django:** Too heavy. We don't need templates, ORM migrations, admin panel. FastAPI is minimal and fast.
- **Why not Go/Rust:** Your team doesn't know them yet. Python has the lowest learning curve AND the best ML tooling. Performance-sensitive paths use WASM workers in the browser anyway.

### API Endpoints

```
POST   /api/v1/analyze              # Start analysis job
GET    /api/v1/analyze/{job_id}     # Poll job status
GET    /api/v1/results/{repo_hash}  # Get cached results
POST   /api/v1/explain              # AI explanation for a specific file/issue
POST   /api/v1/suggest              # AI refactoring suggestion
GET    /api/v1/health               # Health check

WebSocket /ws/analyze/{job_id}      # Real-time progress updates
```

### Request/Response Flow

```
Extension                    Backend                     External
    |                          |                            |
    |--POST /analyze-->        |                            |
    |                          |--Clone repo via-->    GitHub API
    |                          |<-Repo files------         |
    |                          |                            |
    |                          |--Parse + Analyze-->  Tree-sitter
    |                          |--ML Inference-->     scikit-learn
    |                          |--Embeddings-->       sentence-transformers
    |                          |                            |
    |<-WS: progress 45%--     |                            |
    |<-WS: progress 78%--     |                            |
    |<-WS: complete------     |                            |
    |                          |                            |
    |--GET /results/{hash}-->  |                            |
    |<-JSON (graph data)--    |                            |
    |                          |                            |
    |--POST /explain-->        |                            |
    |                          |--Prompt-->           OpenAI GPT-4o
    |                          |<-Response--                |
    |<-AI explanation--       |                            |
```

### Server Architecture

```
FastAPI Application
├── routers/
│   ├── analyze.py          # Analysis endpoints
│   ├── explain.py          # AI explanation endpoints
│   └── health.py           # Health/status
├── services/
│   ├── github_service.py   # Clone + fetch repos
│   ├── parser_service.py   # Tree-sitter parsing (server-side for large repos)
│   ├── metrics_service.py  # Complexity, duplication, etc.
│   ├── graph_service.py    # Dependency graph + NetworkX algorithms
│   ├── ml_service.py       # Code smell classification + embeddings
│   └── ai_service.py       # OpenAI API integration
├── models/
│   ├── schemas.py          # Pydantic request/response models
│   └── db_models.py        # SQLAlchemy ORM models
├── workers/
│   └── analysis_worker.py  # ARQ background task worker
└── core/
    ├── config.py           # Environment config
    └── cache.py            # Redis cache layer
```

### Background Job Processing

**Technology: ARQ (Async Redis Queue)**

**Why ARQ over Celery:**
- Celery is synchronous by default. Our entire stack is async (FastAPI, aiohttp for GitHub API, aioredis).
- ARQ is built for async Python — native `asyncio` support.
- Celery requires a separate broker (RabbitMQ). ARQ uses Redis, which we already have for caching.
- ARQ is much lighter weight (single dependency vs Celery's many).

**Why a job queue at all:**
- Analysis of a large repo (10k+ files) takes 1-5 minutes.
- HTTP requests should not hang for 5 minutes — they timeout.
- The job queue lets us: accept the request instantly, process in the background, push progress via WebSocket.

---

## LAYER 7: Data Layer

### PostgreSQL — Persistent Storage

**Why PostgreSQL:**
- We need to store structured data: repos, analysis results, user accounts (future), analysis history.
- PostgreSQL handles JSON natively (`jsonb` column type) — perfect for storing flexible analysis results that vary per language.
- Best open-source relational DB. Massive community, runs everywhere.
- **Why not MongoDB:** Our data IS relational (repos have many files, files have many issues, issues reference other files). Forcing this into documents creates duplication and consistency problems.
- **Why not SQLite:** Can't handle concurrent writes from multiple analysis workers. Fine for prototyping, not for production.

### Schema (Core Tables)

```sql
-- Repositories table
CREATE TABLE repositories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    github_url      TEXT UNIQUE NOT NULL,
    owner           TEXT NOT NULL,
    name            TEXT NOT NULL,
    default_branch  TEXT DEFAULT 'main',
    last_analyzed   TIMESTAMP,
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Analysis results table
CREATE TABLE analysis_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repo_id         UUID REFERENCES repositories(id) ON DELETE CASCADE,
    commit_sha      TEXT NOT NULL,
    summary_json    JSONB,        -- overall scores, top issues
    graph_json      JSONB,        -- full node/edge data for visualization
    status          TEXT CHECK (status IN ('pending', 'processing', 'complete', 'failed')),
    file_count      INT,
    duration_ms     INT,
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Per-file metrics table
CREATE TABLE file_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id     UUID REFERENCES analysis_results(id) ON DELETE CASCADE,
    file_path       TEXT NOT NULL,
    language        TEXT NOT NULL,
    loc             INT,
    complexity      FLOAT,
    nesting_depth   INT,
    duplicate_hash  TEXT,          -- for grouping duplicates
    issues_json     JSONB,        -- array of detected issues
    embedding       VECTOR(384)   -- for semantic similarity (pgvector)
);

-- Dependency edges table
CREATE TABLE dependencies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id     UUID REFERENCES analysis_results(id) ON DELETE CASCADE,
    source_file     TEXT NOT NULL,
    target_file     TEXT NOT NULL,
    dependency_type TEXT NOT NULL  -- import, inherit, call, etc.
);

-- Indexes for performance
CREATE INDEX idx_analysis_repo ON analysis_results(repo_id);
CREATE INDEX idx_analysis_status ON analysis_results(status);
CREATE INDEX idx_metrics_analysis ON file_metrics(analysis_id);
CREATE INDEX idx_deps_analysis ON dependencies(analysis_id);
CREATE INDEX idx_metrics_embedding ON file_metrics USING ivfflat (embedding vector_cosine_ops);
```

**Note on pgvector:** PostgreSQL extension that stores ML embeddings directly in the database. Lets us do `SELECT * FROM file_metrics ORDER BY embedding <-> query_vector LIMIT 10` to find semantically similar files. No separate vector DB needed.

### Redis — Caching & Real-Time

**Why Redis:**
- **Analysis cache:** If someone already analyzed `facebook/react` at commit `abc123`, serve cached results instantly. TTL: 24 hours.
- **Job queue:** ARQ uses Redis as its job broker.
- **WebSocket pub/sub:** Progress updates from workers -> API server -> client. Redis pub/sub is the standard way to do this.
- **Rate limiting:** Track API calls per user/IP to prevent abuse.

**Cache strategy:**
```
Key: analyze:{owner}:{repo}:{commit_sha}
Value: JSON analysis results
TTL: 24 hours

Key: github:{owner}:{repo}:tree
Value: Repo file tree from GitHub API
TTL: 1 hour
```

---

## LAYER 8: AI/ML Layer

### What This Layer Does
The "intelligence" — turns raw metrics into actionable, human-readable insights.

### Component 1: OpenAI GPT-4o — Natural Language Explanations

**Why GPT-4o:**
- Best code understanding of any LLM as of 2025.
- Can explain code issues in natural language that junior developers understand.
- Can generate specific refactoring suggestions with code examples.
- Structured output mode ensures consistent JSON responses.

**How we use it:**
- User clicks a file node in the visualization
- We send a prompt with: the code, the detected metrics, the specific issues
- GPT-4o returns: explanation, severity, refactoring suggestion, estimated effort

**Prompt template:**
```
You are a senior software architect reviewing code quality.

File: {file_path}
Language: {language}
Metrics:
- Complexity: {complexity}/100
- Lines: {loc}
- Issues detected: {issues}

Code (relevant section):
{code_snippet}

Provide:
1. A 2-sentence explanation of why this code is problematic
2. The specific code smell category (God Class, Feature Envy, Long Method, etc.)
3. A concrete refactoring suggestion with a brief code example
4. Estimated effort (quick fix / moderate / significant refactor)

Respond in JSON format:
{
  "explanation": "...",
  "smell_category": "...",
  "suggestion": "...",
  "code_example": "...",
  "effort": "quick_fix | moderate | significant"
}
```

### Component 2: Sentence-Transformers — Code Embeddings

**Why Sentence-Transformers:**
- Open-source (free, no API costs).
- `all-MiniLM-L6-v2` model is only 80MB, produces 384-dimension embeddings.
- Runs locally on the server — no external API dependency.
- Embeddings let us find **semantically similar** code (not just textual duplicates). Two functions that do the same thing with different variable names will have similar embeddings.

**How we use it:**
- Embed every function body in the repo.
- Store embeddings in pgvector.
- Query: "find functions similar to this one" -> cosine similarity search.
- Flag clusters of similar functions as "potential duplication."

### Component 3: Scikit-learn — Code Smell Classifier

**Why Scikit-learn over PyTorch/TensorFlow:**
- Our classification task is simple: given metrics (complexity, LOC, nesting, fan-in, fan-out), predict code smell type and severity.
- This is a tabular data problem. Random Forest / Gradient Boosting (scikit-learn) outperforms neural networks on tabular data.
- Scikit-learn is trivial to train, serialize, and load. No GPU needed.
- Model size: < 1MB. Inference: < 1ms per file.

**Training data:**
- Public datasets: "Code Smell Detection" datasets from ML research papers.
- Augmented with labeled examples from well-known open-source repos.

**Model pipeline:**
```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.ensemble import GradientBoostingClassifier

pipeline = Pipeline([
    ('scaler', StandardScaler()),
    ('classifier', GradientBoostingClassifier(
        n_estimators=200,
        max_depth=6
    ))
])
```

**Input features per file:**
- Cyclomatic complexity
- LOC
- Max nesting depth
- Number of function parameters (avg)
- Number of dependencies (fan-in + fan-out)
- Duplicate score

**Output:** Code smell type + confidence score

---

## LAYER 9: Visualization Engine

### What This Layer Does
Renders the interactive 3D "code galaxy" — the visual centerpiece of the product.

### Technology: Three.js + Custom Force-Directed Graph

**Why Three.js:**
- Industry standard for WebGL in the browser. Most tutorials, largest community.
- Works inside extension iframes and side panels.
- InstancedMesh allows rendering 100,000+ nodes at 60fps (critical for large repos).
- **Why not React Three Fiber:** R3F is React-specific. Our UI is SolidJS. Using R3F would mean running two different UI frameworks. Vanilla Three.js integrates cleanly with any framework.
- **Why not D3.js for 3D:** D3 is SVG-based. SVG cannot handle 10k+ nodes without severe frame drops. WebGL (Three.js) is hardware-accelerated.

**We DO use D3 for:** The force simulation algorithm (`d3-force-3d`). D3's force simulation is the best implementation available and works independently of its rendering — we compute positions with D3, render with Three.js.

### Visualization Mapping

| Code Property       | Visual Property        |
|---------------------|------------------------|
| File                | Sphere (node)          |
| Complexity score    | Sphere radius          |
| Issue severity      | Color (green -> red)   |
| Language            | Shape variant          |
| Dependency          | Line (edge)            |
| Module/directory    | Spatial cluster        |
| Fan-in (dependents) | Glow/brightness        |

### Rendering Pipeline

```
Graph Data (JSON)
    |
    v
d3-force-3d simulation
    | (compute x, y, z positions)
    v
Three.js Scene
    |-- InstancedMesh (nodes) -- one draw call for ALL spheres
    |-- LineSegments (edges) -- one draw call for ALL connections
    |-- Sprites (labels) -- file names on hover
    `-- Post-processing
        |-- Bloom (glow on high-severity nodes)
        `-- FXAA (antialiasing)
```

### Interaction Model

- **Orbit controls:** Rotate, zoom, pan the galaxy
- **Hover:** Highlight node + connected edges, show tooltip
- **Click:** Select node, load details in side panel, call AI explanation API
- **Search:** Type file name -> camera flies to that node
- **Filter:** Toggle by language, severity, module
- **Time travel (Phase 2):** Slider to see how code quality changed over commits

### Performance Targets

| Repo Size        | Target FPS | Node Count  |
|------------------|-----------|-------------|
| Small (< 100)    | 60 fps    | < 100       |
| Medium (< 1000)  | 60 fps    | < 1,000     |
| Large (< 10k)    | 30+ fps   | < 10,000    |
| Huge (> 10k)     | 30+ fps   | LOD/culling |

**LOD (Level of Detail):** For repos with 10k+ files, we cluster distant nodes into single "galaxy arm" representations and only expand to individual files when zoomed in.

---

## LAYER 10: Cross-Cutting Concerns

### Authentication
- **Phase 1 (MVP):** No auth. Public repos only. Rate limit by IP.
- **Phase 2:** GitHub OAuth. "Login with GitHub" to analyze private repos. We request `repo` scope, use the user's token to access their private repos.
- **Phase 3:** Team accounts with shared dashboards.

### Error Handling
- Extension <-> Backend: Exponential backoff retry (3 attempts).
- GitHub API rate limits: 5,000 req/hour with auth, 60 without. Cache aggressively.
- OpenAI API failures: Graceful degradation — show metrics without AI explanations.
- WASM load failures: Fall back to regex-based basic metrics.

### Security
- Extension CSP: Strict, no `eval()`, no inline scripts.
- Backend: All inputs validated via Pydantic. SQL injection impossible with SQLAlchemy ORM.
- No user code is ever executed — only parsed as text via Tree-sitter.
- API keys (OpenAI, GitHub) stored as server environment variables, never exposed to extension.

### Monitoring & Logging
- Backend: Structured JSON logging -> stdout (for Docker/Railway logs).
- Extension: `console.debug` in development, silent in production.
- Error tracking: Sentry (free tier for open source).

---

## Deployment Architecture

```
┌─────────────────────────────────────────────────────┐
│                   USERS                             │
│   Chrome Web Store / Firefox Add-ons / Edge Store   │
└───────────────────────┬─────────────────────────────┘
                        | HTTPS
                        v
┌─────────────────────────────────────────────────────┐
│              Cloudflare (CDN + DDoS protection)      │
└───────────────────────┬─────────────────────────────┘
                        |
                        v
┌─────────────────────────────────────────────────────┐
│              Railway / Render                        │
│  ┌──────────────┐  ┌────────────────────┐           │
│  │ FastAPI       │  │ ARQ Workers (x2)   │           │
│  │ (2 instances) │  │ (analysis jobs)    │           │
│  └──────┬───────┘  └────────┬───────────┘           │
│         |                   |                        │
│  ┌──────┴───────┐  ┌───────┴──────────┐            │
│  │ PostgreSQL    │  │ Redis            │            │
│  │ (managed)     │  │ (managed)        │            │
│  └──────────────┘  └──────────────────┘            │
└─────────────────────────────────────────────────────┘
```

**Why Railway/Render:**
- Free tier available for prototyping.
- Managed PostgreSQL and Redis included.
- Docker-based deployment (push to deploy).
- Auto-scaling available on paid tiers.
- **Why not AWS/GCP:** Overkill for this project. Railway/Render abstracts infrastructure so the team can focus on code, not DevOps.

---

## Team Division (4 Members)

### Member 1: Extension Architecture + GitHub Integration
**Owns:** Layers 1, 2, 4
- WXT project setup and cross-browser configuration
- Content script (GitHub DOM injection)
- Background service worker (message routing, orchestration)
- Extension storage and state management
- Build pipeline for all 4 browsers

**Learning path:** JavaScript/TypeScript fundamentals -> WXT docs -> Chrome Extension docs -> MutationObserver API

### Member 2: Frontend UI + Visualization
**Owns:** Layers 3, 9
- SolidJS popup and side panel UI
- Three.js 3D visualization engine
- D3 force simulation integration
- Interactive controls (hover, click, search, filter)
- UI design and responsive layout

**Learning path:** HTML/CSS/JS -> SolidJS tutorial -> Three.js Journey course -> d3-force-3d docs

### Member 3: Backend API + Data Layer
**Owns:** Layers 6, 7
- FastAPI server setup
- PostgreSQL schema and migrations
- Redis caching layer
- GitHub API integration (repo fetching)
- WebSocket progress streaming
- ARQ job queue setup
- Docker configuration
- Deployment to Railway/Render

**Learning path:** Python fundamentals -> FastAPI tutorial -> SQLAlchemy/PostgreSQL -> Docker basics -> Redis docs

### Member 4: Analysis Engine + AI/ML
**Owns:** Layers 5, 8
- Tree-sitter WASM setup for in-browser parsing
- Web Worker pool implementation
- All code metrics (complexity, LOC, duplication, dependencies)
- Scikit-learn classifier training
- Sentence-transformers embedding pipeline
- OpenAI GPT-4o prompt engineering
- NetworkX graph algorithms

**Learning path:** Python fundamentals -> Tree-sitter docs -> AST concepts -> scikit-learn tutorial -> OpenAI API docs

---

## Implementation Phases

### Phase 1: MVP (Weeks 1-4)
- Chrome extension only
- Public repos only
- JS/TS analysis only
- 2D graph (D3 force-directed, skip 3D for now)
- Basic metrics (complexity, LOC)
- No AI (just metrics display)

**Goal:** Prove the concept works end-to-end.

### Phase 2: Core Product (Weeks 5-8)
- 3D visualization (Three.js)
- All 5 languages (JS/TS, Python, Java, Go, Rust)
- AI explanations (OpenAI)
- Dependency graph
- Duplicate detection
- Side panel UI

**Goal:** Feature-complete product that is genuinely useful.

### Phase 3: Polish (Weeks 9-12)
- All browsers (Firefox, Edge, Safari)
- ML code smell classifier
- Semantic similarity
- Private repos (GitHub OAuth)
- Export report (PDF/PNG)
- Performance optimization (large repos)

**Goal:** Production-ready, publishable to browser extension stores.

---

## File/Folder Structure (Final)

```
archviz/
├── extension/                    # Browser extension (WXT)
│   ├── entrypoints/
│   │   ├── background.ts
│   │   ├── content.ts
│   │   ├── popup/
│   │   └── sidepanel/
│   ├── components/
│   ├── lib/
│   │   ├── analyzer/             # Web Workers + Tree-sitter WASM
│   │   ├── api/                  # Backend API client
│   │   ├── visualization/        # Three.js engine
│   │   └── stores/               # SolidJS stores (state)
│   ├── public/
│   │   └── wasm/                 # Tree-sitter WASM binaries
│   ├── wxt.config.ts
│   ├── tailwind.config.js
│   ├── tsconfig.json
│   └── package.json
│
├── backend/                      # Python FastAPI server
│   ├── app/
│   │   ├── main.py
│   │   ├── routers/
│   │   ├── services/
│   │   ├── models/
│   │   ├── workers/
│   │   └── core/
│   ├── ml/
│   │   ├── train_classifier.py
│   │   ├── models/               # Saved ML models
│   │   └── data/                 # Training data
│   ├── tests/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── pyproject.toml
│
├── shared/                       # Shared types/contracts
│   └── types/
│       ├── analysis.ts           # TypeScript interfaces
│       └── analysis.py           # Pydantic models (mirror)
│
├── docker-compose.yml
├── plan.md                       # This document
├── README.md
└── .github/
    └── workflows/
        └── ci.yml
```

---

## Tech Stack Summary

| Layer                  | Technology               | Why                                              |
|------------------------|--------------------------|--------------------------------------------------|
| Extension Framework    | WXT                      | Cross-browser, Vite, TypeScript, framework-agnostic |
| Content Script         | Vanilla TypeScript       | Tiny bundle, no framework collision with GitHub   |
| UI Framework           | SolidJS                  | 7KB bundle, true reactivity, no re-renders       |
| Styling                | TailwindCSS              | Scoped, no runtime overhead, consistent design   |
| Service Worker         | Plain TypeScript         | No DOM = no framework needed, lightweight        |
| In-Browser Parsing     | Tree-sitter (WASM)       | Industry standard, all 5 languages, near-native  |
| Parallel Processing    | Web Workers              | Off-main-thread, zero UI impact                  |
| Backend Framework      | FastAPI (Python)         | Async, auto-docs, Pydantic, best ML ecosystem    |
| Job Queue              | ARQ                      | Async-native, uses Redis (already have it)       |
| Database               | PostgreSQL + pgvector    | Relational + JSONB + vector embeddings in one DB |
| Cache/Pub-Sub          | Redis                    | Caching + job broker + real-time updates         |
| 3D Rendering           | Three.js                 | WebGL, InstancedMesh, 100k+ nodes at 60fps      |
| Graph Layout           | d3-force-3d              | Best force-directed algorithm, framework-agnostic|
| AI Explanations        | OpenAI GPT-4o            | Best code understanding, structured output       |
| Code Embeddings        | Sentence-Transformers    | Free, local, 80MB model, semantic similarity     |
| Code Smell Classifier  | Scikit-learn             | Tabular data winner, < 1MB model, < 1ms inference|
| Deployment (extension) | Chrome/Firefox/Edge/Safari stores | Direct to users                          |
| Deployment (backend)   | Railway or Render        | Free tier, managed DB/Redis, push-to-deploy      |
| CDN/Protection         | Cloudflare               | Free tier, DDoS protection, SSL                  |
| Error Tracking         | Sentry                   | Free for open source, covers extension + backend |
