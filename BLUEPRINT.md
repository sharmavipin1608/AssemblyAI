# Assembly AI — System Blueprint

---

## 1. High-Level Architecture

```
+-------------------------------------------------------------+
|                        Unified UI                           |
|      (React / React Flow Canvas / Chat & State View)        |
+------------------------------+------------------------------+
                               |
                   API (REST / WebSockets)
                               |
+------------------------------v------------------------------+
|                    Dynamic Backend Engine                   |
|                  (FastAPI / Python Runtime)                 |
+------------------------------+------------------------------+
                               |
             +-----------------+-----------------+
             |                                   |
+------------v--------------+     +--------------v-----------+
|    Orchestrator Engine    |     |    Metadata & State DB   |
|  (Compiles graphs         |     |  (PostgreSQL / Supabase) |
|   dynamically via         |     |  Stores: Specs, Logs,    |
|   LangGraph / Code)       |     |  Runs                    |
+------------+--------------+     +--------------------------+
             |
+------------v----------------------------------------------------------+
|                  Observability & Prompts Layer (Langfuse)             |
|  - Manages prompt version history                                     |
|  - Tracks LLM tokens, costs, loops, and deep-nested tool calls        |
+-----------------------------------------------------------------------+
```

---

## 2. Database Schema

### Definition Layer — what a project looks like

```sql
-- 1. Project Container
CREATE TABLE projects (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    description TEXT,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 2. Project-Level LLM Configuration
--    api_key_encrypted stores a Fernet-encrypted ciphertext.
--    The master encryption key lives as an environment variable (ASSEMBLY_SECRET_KEY).
CREATE TABLE project_configs (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id        UUID REFERENCES projects(id) ON DELETE CASCADE,
    provider          VARCHAR(50)  NOT NULL,   -- 'openai' | 'anthropic' | 'groq'
    api_key_encrypted TEXT         NOT NULL,
    default_model     VARCHAR(100) NOT NULL
);

-- 3. Dynamic Agents
CREATE TABLE agents (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id       UUID REFERENCES projects(id) ON DELETE CASCADE,
    role_key         VARCHAR(100) NOT NULL,   -- e.g. 'researcher', 'shadow_writer'
    display_name     VARCHAR(255) NOT NULL,
    system_prompt_id VARCHAR(255) NOT NULL,   -- Prompt name/version managed in Langfuse
    model_override   VARCHAR(100) NULL,
    temperature      NUMERIC(2,1) DEFAULT 0.2
);

-- 4. Built-in Tools (v1: web_search, code_executor, file_reader, calculator)
CREATE TABLE tools (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id   UUID REFERENCES projects(id) ON DELETE CASCADE,
    tool_key     VARCHAR(100) NOT NULL,   -- matches a registered built-in key
    display_name VARCHAR(255) NOT NULL,
    description  TEXT         NOT NULL,  -- shown to the LLM at runtime
    config       JSONB        NOT NULL DEFAULT '{}',  -- tool-specific overrides
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 5. Agent <-> Tool Assignment (many-to-many)
CREATE TABLE agent_tools (
    agent_id UUID REFERENCES agents(id) ON DELETE CASCADE,
    tool_id  UUID REFERENCES tools(id)  ON DELETE CASCADE,
    PRIMARY KEY (agent_id, tool_id)
);

-- 6. Execution Graph Topology (Routing Edges)
CREATE TABLE workflows (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id        UUID REFERENCES projects(id) ON DELETE CASCADE,
    source_node       VARCHAR(100) NOT NULL,   -- agent role_key or 'START'
    target_node       VARCHAR(100) NOT NULL,   -- agent role_key or 'END'
    routing_condition VARCHAR(100) NOT NULL DEFAULT 'always',  -- 'always' | 'if_approved' | 'if_gap_found'
    created_at        TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Execution Layer — what happened when it ran

```sql
-- 7. Runs — one row per graph execution
CREATE TABLE runs (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id        UUID REFERENCES projects(id) ON DELETE CASCADE,
    status            VARCHAR(20)  NOT NULL DEFAULT 'queued',  -- queued | running | completed | failed
    user_input        TEXT         NOT NULL,
    final_state       JSONB,         -- full ProjectState snapshot at completion
    total_loops       INT          NOT NULL DEFAULT 0,
    langfuse_trace_id VARCHAR(255),  -- top-level trace ID for cross-referencing in Langfuse
    error_message     TEXT,          -- populated on failure
    started_at        TIMESTAMP,
    completed_at      TIMESTAMP,
    created_at        TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 8. Run Node Logs — one row per node firing per run
CREATE TABLE run_node_logs (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id           UUID REFERENCES runs(id) ON DELETE CASCADE,
    node_role_key    VARCHAR(100) NOT NULL,
    sequence_order   INT          NOT NULL,   -- global firing order within the run
    loop_index       INT          NOT NULL DEFAULT 0,  -- which self-correction iteration
    input_snapshot   JSONB,        -- state entering the node
    output_snapshot  JSONB,        -- state delta produced by the node
    langfuse_span_id VARCHAR(255), -- span ID for this node in Langfuse
    started_at       TIMESTAMP,
    completed_at     TIMESTAMP
);
```

### SQLAlchemy Models (Python source of truth)

Alembic diffs these models against the live DB to auto-generate migration files. The raw SQL above is reference only.

```python
import uuid
from sqlalchemy import Column, String, Text, Numeric, Integer, ForeignKey, TIMESTAMP
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import DeclarativeBase, relationship
from sqlalchemy.sql import func

class Base(DeclarativeBase):
    pass

class Project(Base):
    __tablename__ = "projects"
    id          = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name        = Column(String(255), nullable=False)
    description = Column(Text)
    created_at  = Column(TIMESTAMP, server_default=func.now())

class ProjectConfig(Base):
    __tablename__ = "project_configs"
    id                = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    project_id        = Column(UUID(as_uuid=True), ForeignKey("projects.id", ondelete="CASCADE"), nullable=False)
    provider          = Column(String(50),  nullable=False)
    api_key_encrypted = Column(Text,        nullable=False)
    default_model     = Column(String(100), nullable=False)

class Agent(Base):
    __tablename__ = "agents"
    id               = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    project_id       = Column(UUID(as_uuid=True), ForeignKey("projects.id", ondelete="CASCADE"), nullable=False)
    role_key         = Column(String(100), nullable=False)
    display_name     = Column(String(255), nullable=False)
    system_prompt_id = Column(String(255), nullable=False)
    model_override   = Column(String(100))
    temperature      = Column(Numeric(2, 1), default=0.2)

class Tool(Base):
    __tablename__ = "tools"
    id           = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    project_id   = Column(UUID(as_uuid=True), ForeignKey("projects.id", ondelete="CASCADE"), nullable=False)
    tool_key     = Column(String(100), nullable=False)
    display_name = Column(String(255), nullable=False)
    description  = Column(Text,        nullable=False)
    config       = Column(JSONB,        nullable=False, server_default="{}")
    created_at   = Column(TIMESTAMP,   server_default=func.now())

class AgentTool(Base):
    __tablename__ = "agent_tools"
    agent_id = Column(UUID(as_uuid=True), ForeignKey("agents.id", ondelete="CASCADE"), primary_key=True)
    tool_id  = Column(UUID(as_uuid=True), ForeignKey("tools.id",  ondelete="CASCADE"), primary_key=True)

class Workflow(Base):
    __tablename__ = "workflows"
    id                = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    project_id        = Column(UUID(as_uuid=True), ForeignKey("projects.id", ondelete="CASCADE"), nullable=False)
    source_node       = Column(String(100), nullable=False)
    target_node       = Column(String(100), nullable=False)
    routing_condition = Column(String(100), nullable=False, default="always")
    created_at        = Column(TIMESTAMP,   server_default=func.now())

class Run(Base):
    __tablename__ = "runs"
    id                = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    project_id        = Column(UUID(as_uuid=True), ForeignKey("projects.id", ondelete="CASCADE"), nullable=False)
    status            = Column(String(20),  nullable=False, default="queued")
    user_input        = Column(Text,        nullable=False)
    final_state       = Column(JSONB)
    total_loops       = Column(Integer,     nullable=False, default=0)
    langfuse_trace_id = Column(String(255))
    error_message     = Column(Text)
    started_at        = Column(TIMESTAMP)
    completed_at      = Column(TIMESTAMP)
    created_at        = Column(TIMESTAMP,   server_default=func.now())

class RunNodeLog(Base):
    __tablename__ = "run_node_logs"
    id               = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    run_id           = Column(UUID(as_uuid=True), ForeignKey("runs.id", ondelete="CASCADE"), nullable=False)
    node_role_key    = Column(String(100), nullable=False)
    sequence_order   = Column(Integer,     nullable=False)
    loop_index       = Column(Integer,     nullable=False, default=0)
    input_snapshot   = Column(JSONB)
    output_snapshot  = Column(JSONB)
    langfuse_span_id = Column(String(255))
    started_at       = Column(TIMESTAMP)
    completed_at     = Column(TIMESTAMP)
```

### Migration Workflow (Alembic)

```
# One-time setup
alembic init alembic

# After any model change
alembic revision --autogenerate -m "describe the change"
alembic upgrade head

# Roll back one step
alembic downgrade -1
```

Migration files live in `alembic/versions/` and are committed to git — same discipline as Flyway versioned scripts, just Python-generated.

---

## 3. Engine Layer

The engine is completely decoupled from FastAPI — it has no HTTP concerns. It receives a `project_id` and a `user_input`, compiles a graph, runs it, and streams state events back to the caller.

### 3a. Generic State Shape

A single typed state is shared across all projects. Agents write their outputs into `shared_context` under their own key. This keeps LangGraph happy with a typed schema while remaining fully generic across different pipeline topologies.

```python
from typing import Dict, Any, List, Optional
from typing_extensions import TypedDict

MAX_LOOPS = 10  # hard cap — configurable per-project in a future roadmap item

class ProjectState(TypedDict):
    run_id:            str
    project_id:        str
    user_input:        str
    shared_context:    Dict[str, Any]   # agents read/write here (e.g. shared_context["research_notes"])
    checker_feedback:  Dict[str, Any]   # structured output from evaluator nodes
    execution_history: List[str]        # append-only log — one entry per node firing
    current_loops:     int
    final_output:      Optional[str]    # populated by the terminal node
```

### 3b. Graph Compiler

Reads the DB topology and compiles a `StateGraph` at runtime. Every project gets its own compiled graph instance per run.

```python
from langgraph.graph import StateGraph, START, END

def compile_dynamic_project_graph(project_id: str) -> StateGraph:
    agents_list = db.get_agents_for_project(project_id)
    edges_list  = db.get_workflows_for_project(project_id)

    graph = StateGraph(ProjectState)

    for agent in agents_list:
        node_fn = create_generic_agent_node(agent)
        graph.add_node(agent.role_key, node_fn)

    for edge in edges_list:
        if edge.routing_condition == "always":
            graph.add_edge(edge.source_node, edge.target_node)
        else:
            router_fn, routing_map = create_conditional_router(edge, agents_list)
            graph.add_conditional_edges(edge.source_node, router_fn, routing_map)

    return graph.compile()
```

### 3c. Generic Node Runner

Every agent node runs the same function. The full state is passed in — agents use what they need based on their system prompt. Tool calls follow the standard ReAct pattern: the LLM decides if/when to call a tool, the runner executes it, the result is fed back, and the loop continues until the LLM produces a final text response.

```python
def create_generic_agent_node(agent):
    async def node_fn(state: ProjectState) -> dict:
        # 1. Fetch versioned system prompt from Langfuse
        system_prompt = langfuse.get_prompt(agent.system_prompt_id)

        # 2. Resolve LLM — provider-agnostic via LangChain wrappers
        llm = get_llm_for_project(agent)  # returns ChatOpenAI or ChatAnthropic

        # 3. Bind tools if this agent has any assigned
        tools = get_tools_for_agent(agent.id)  # fetches from agent_tools + tool registry
        if tools:
            llm = llm.bind_tools(tools)

        # 4. Build messages — full state serialised as context
        messages = build_messages(system_prompt, state)

        # 5. ReAct loop: call LLM → execute tool calls → repeat until final response
        response = await run_react_loop(llm, messages, tools)

        # 6. Return state delta
        return {
            "shared_context":    {**state["shared_context"], agent.role_key: response.content},
            "execution_history": state["execution_history"] + [f"{agent.role_key}: {response.content[:100]}..."],
            "current_loops":     state["current_loops"] + (1 if is_evaluator(agent) else 0),
        }

    return node_fn
```

### 3d. LLM Provider Abstraction

`llm.py` instantiates the correct LangChain chat model based on the project's `provider` field. The node runner never references a specific provider.

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

def get_llm_for_project(agent) -> BaseChatModel:
    config   = db.get_project_config(agent.project_id)
    api_key  = decrypt(config.api_key_encrypted)
    model    = agent.model_override or config.default_model
    temp     = float(agent.temperature)

    if config.provider == "openai":
        return ChatOpenAI(api_key=api_key, model=model, temperature=temp)
    if config.provider == "anthropic":
        return ChatAnthropic(api_key=api_key, model=model, temperature=temp)
    if config.provider == "groq":
        return ChatOpenAI(api_key=api_key, model=model, temperature=temp,
                          base_url="https://api.groq.com/openai/v1")
    raise ValueError(f"Unknown provider: {config.provider}")
```

### 3e. Conditional Router

The router evaluates `checker_feedback` from the state and returns the next node key. The routing map (which condition maps to which node) is derived from the edges stored in the DB.

```python
def create_conditional_router(edge, agents_list):
    routing_map = {agent.role_key: agent.role_key for agent in agents_list}
    routing_map["END"] = END

    def router_fn(state: ProjectState) -> str:
        if state["current_loops"] >= MAX_LOOPS:
            return "END"
        feedback = state.get("checker_feedback", {})
        if feedback.get("approved"):
            return "END"
        return feedback.get("flaw_location", "END")

    return router_fn, routing_map
```

### 3f. Tool Registry

Built-in tools are registered at startup. Each tool is a plain async Python function that takes a string input and returns a string result. The registry maps `tool_key` → callable for the node runner to look up.

```python
# engine/tools/registry.py
TOOL_REGISTRY: Dict[str, Callable] = {}

def register(tool_key: str):
    def decorator(fn):
        TOOL_REGISTRY[tool_key] = fn
        return fn
    return decorator

def get_tools_for_agent(agent_id: str) -> List[BaseTool]:
    assigned = db.get_tools_for_agent(agent_id)
    return [wrap_as_langchain_tool(TOOL_REGISTRY[t.tool_key], t) for t in assigned]
```

```python
# engine/tools/web_search.py
@register("web_search")
async def web_search(query: str) -> str:
    ...

# engine/tools/code_executor.py
@register("code_executor")
async def code_executor(code: str) -> str:
    ...

# engine/tools/file_reader.py
@register("file_reader")
async def file_reader(path: str) -> str:
    ...

# engine/tools/calculator.py
@register("calculator")
async def calculator(expression: str) -> str:
    ...
```

---

## 4. Self-Correction Feedback Loop

The Project Writer workflow relies on conditional edge routing driven by evaluation grading fields.

```
       [ START ]
           |
           v
    +--------------+
    |  Researcher  | <------------------------+
    +------+-------+                          | (if research gaps found)
           |                                  |
           v                                  |
    +--------------+                          |
    | ShadowWriter | <------------+           |
    +------+-------+              |           |
           |                      |           |
           v                      | (if writing flaws found)
    +--------------+              |           |
    |   Checker    |--------------+-----------+
    +------+-------+
           |
           | (approved)
           v
    +--------------+
    |  Publisher   |
    +------+-------+
           |
         [ END ]
```

### Node Responsibilities

| Node | Input | Output |
|------|-------|--------|
| **Researcher** | `user_input` | Updates `research_notes` |
| **ShadowWriter** | `research_notes` | Builds `draft_content` |
| **Checker** | `draft_content`, `user_input`, `research_notes` | Structured `checker_feedback` |
| **Publisher** | Approved `draft_content` | Final output |

### Checker Feedback Payload

```json
{
  "approved": false,
  "flaw_location": "researcher",
  "comments": "The draft lacks analysis on the depth parameters specified in the original request."
}
```

The conditional router reads `ProjectState["checker_feedback"]["flaw_location"]`. If it points to `researcher`, the state is redirected back to the Researcher node with the comments appended to `execution_history`, triggering a self-healing loop.

---

## 5. Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Backend Runtime** | FastAPI (Python) | Native async I/O, SSE streaming, clean integration with agent libraries |
| **Orchestration** | LangGraph | Built for cyclic state propagation and loops |
| **DB Migrations** | SQLAlchemy + Alembic | ORM models as source of truth; auto-generated versioned migration files |
| **Observability** | Langfuse (self-hosted, Docker) | Visual trace explorer; tracks latency, token costs, and exact execution paths |
| **Frontend** | React + Tailwind CSS + React Flow | React Flow enables a visual canvas where users wire agent cards into workflow graphs |
| **Database** | PostgreSQL (Docker) | Self-contained, portable between local and Oracle Free Tier |

---

## 6. FastAPI Layer

### Project Structure

```
assembly_ai/
├── main.py                    # App entry — creates FastAPI instance, mounts routers, CORS
├── core/
│   ├── config.py              # Pydantic-settings: DB URL, ASSEMBLY_SECRET_KEY, env flags
│   ├── database.py            # Async SQLAlchemy engine + session factory (AsyncSession)
│   └── security.py            # Fernet encrypt() / decrypt() helpers for API keys
├── models/                    # SQLAlchemy ORM models — talks to the DB
│   └── db.py
├── schemas/                   # Pydantic request/response shapes — talks to the API client
│   ├── project.py             # ProjectCreate, ProjectResponse
│   ├── agent.py               # AgentCreate, AgentUpdate, AgentResponse
│   ├── tool.py                # ToolCreate, ToolResponse
│   ├── workflow.py            # WorkflowCreate, WorkflowResponse
│   └── run.py                 # RunCreate, RunResponse, RunDetail (includes node logs)
├── api/
│   └── v1/
│       ├── projects.py
│       ├── agents.py
│       ├── tools.py
│       ├── workflows.py
│       └── runs.py            # Includes SSE streaming endpoint
├── engine/                    # LangGraph orchestration — no HTTP concerns here
│   ├── compiler.py            # compile_dynamic_project_graph()
│   ├── nodes.py               # create_generic_agent_node()
│   ├── conditional.py         # create_conditional_router()
│   └── tools/
│       ├── registry.py        # Maps tool_key → callable
│       ├── web_search.py
│       ├── code_executor.py
│       ├── file_reader.py
│       └── calculator.py
└── services/
    ├── llm.py                 # Provider abstraction: OpenAI / Anthropic / Groq
    └── langfuse.py            # Langfuse client wrapper
```

> **`models/` vs `schemas/` rule:** `models/` defines what the database looks like. `schemas/` defines what the API accepts and returns. They are always kept separate. A response schema like `ProjectConfigResponse` returns `provider` and `default_model` but never `api_key_encrypted`.

### API Endpoints

All resources are nested under `/api/v1/projects/{id}/` because everything is project-scoped.

```
# Projects
POST   /api/v1/projects
GET    /api/v1/projects
GET    /api/v1/projects/{id}
DELETE /api/v1/projects/{id}

# LLM Config (API key stored encrypted, never returned in responses)
POST   /api/v1/projects/{id}/config
GET    /api/v1/projects/{id}/config

# Agents
POST   /api/v1/projects/{id}/agents
GET    /api/v1/projects/{id}/agents
PUT    /api/v1/projects/{id}/agents/{agent_id}
DELETE /api/v1/projects/{id}/agents/{agent_id}

# Tool assignment to agents
POST   /api/v1/projects/{id}/agents/{agent_id}/tools/{tool_id}
DELETE /api/v1/projects/{id}/agents/{agent_id}/tools/{tool_id}

# Tools
POST   /api/v1/projects/{id}/tools
GET    /api/v1/projects/{id}/tools
PUT    /api/v1/projects/{id}/tools/{tool_id}
DELETE /api/v1/projects/{id}/tools/{tool_id}

# Workflow edges
POST   /api/v1/projects/{id}/workflows
GET    /api/v1/projects/{id}/workflows
DELETE /api/v1/projects/{id}/workflows/{workflow_id}

# Runs
POST   /api/v1/projects/{id}/runs             # triggers execution, returns run_id immediately
GET    /api/v1/projects/{id}/runs             # list all runs for a project
GET    /api/v1/projects/{id}/runs/{run_id}    # full run detail + all node logs (post-completion review)
GET    /api/v1/projects/{id}/runs/{run_id}/stream   # SSE: live node-by-node output while running
```

### Streaming Pattern (SSE)

The run is triggered via `POST /runs` which returns the `run_id` immediately. The client then opens an SSE connection to `/runs/{run_id}/stream` and receives one event per node completion:

```json
event: node_complete
data: {"node": "researcher", "loop_index": 0, "sequence_order": 1, "output": "..."}

event: node_complete
data: {"node": "shadow_writer", "loop_index": 0, "sequence_order": 2, "output": "..."}

event: run_complete
data: {"status": "completed", "total_loops": 1}
```

Human review happens after the run — `GET /runs/{run_id}` returns the full state including all node logs and the final output. No mid-run interruption in v1.

---

## 7. Development Phasing

### Phase 1 — Hardcoded Core (Week 1)
Build the Project Writer agent as a standalone Python script. Use hardcoded LangGraph nodes and in-memory state. Goal: get the self-correction loop working locally — the Checker should reject the draft at least once before the Publisher fires.

### Phase 2 — Schema Extraction (Week 2)
Extract all hardcoded configs into a `project_writer_spec.json` file. Refactor the engine to be config-driven: it reads the JSON and constructs the agent loop from that spec alone. Wire in **Langfuse** at this stage to capture trace metrics and prompt versions.

### Phase 3 — Platform Layer (Weeks 3–4)
Migrate the JSON spec into PostgreSQL. Wrap the engine in a **FastAPI** server, apply the full schema via Alembic, and expose the v1 endpoints. Build a minimal frontend that lets you define agents and prompts, connect them visually via React Flow, and trigger runs with live SSE streaming output.

---

## 8. Future Roadmap

These are intentionally out of scope for v1 to keep the initial build tangible and shippable.

| Item | Description |
|------|-------------|
| **Auth / multi-user** | API key or JWT protection on all endpoints; per-user project isolation |
| **Human-in-the-loop (mid-run)** | Allow a run to pause at a designated node and wait for human input before continuing — requires WebSockets |
| **Workflow versioning** | Snapshot graph topology at run time so old runs remain replayable after the workflow is edited |
| **Custom HTTP tool endpoints** | Let users register their own API endpoints as agent tools, configured via URL + headers + payload schema |
| **Agent memory** | Persist per-agent context across runs for long-running or stateful workflows |
| **Per-project max_loops config** | Store loop cap in the DB per project instead of the global `MAX_LOOPS = 10` constant |
| **Per-agent input field mapping** | Let each agent declare which state fields it reads instead of receiving full state — reduces LLM noise on large states |
| **Per-project state schema** | Define a custom typed state shape per project in the DB for maximum flexibility across radically different pipeline types |
| **Parallel agent execution** | Run N agents simultaneously against the same goal and compare outputs — foundation for Agent Forge integration |
