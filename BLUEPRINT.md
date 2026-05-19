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

## 3. Core Execution Pattern: Dynamic Graph Parsing

The backend reads the database topology and compiles a state machine at runtime.

### State Object Schema

```python
from typing import Dict, Any, List
from typing_extensions import TypedDict

class ProjectState(TypedDict):
    project_id:        str
    user_input:        str
    research_notes:    str
    draft_content:     str
    checker_feedback:  Dict[str, Any]
    current_loops:     int
    execution_history: List[str]
```

### Dynamic Graph Compilation

```python
from langgraph.graph import StateGraph, START, END

def compile_dynamic_project_graph(project_id: str):
    # 1. Fetch configurations from database
    agents_list = db.get_agents_for_project(project_id)
    edges_list  = db.get_workflows_for_project(project_id)

    workflow_graph = StateGraph(ProjectState)

    # 2. Dynamically add nodes from database Agent specs
    for agent in agents_list:
        # Generic node runner that pulls system prompts from Langfuse
        node_fn = create_generic_agent_node(agent)
        workflow_graph.add_node(agent.role_key, node_fn)

    # 3. Add edges and routing rules dynamically
    for edge in edges_list:
        if edge.routing_condition == "always":
            workflow_graph.add_edge(edge.source_node, edge.target_node)
        else:
            # Conditional routing (e.g. Evaluator / Checker logic)
            condition_router = create_conditional_router(edge)
            workflow_graph.add_conditional_edges(
                edge.source_node,
                condition_router,
                {
                    "retry_research": "researcher",
                    "retry_writer":   "shadow_writer",
                    "approve":        "publisher",
                }
            )

    return workflow_graph.compile()
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
| **Backend Runtime** | FastAPI (Python) | Native async I/O, SSE/WebSocket streaming, clean integration with agent libraries |
| **Orchestration** | LangGraph | Built for cyclic state propagation, loops, and human-in-the-loop interruption |
| **Observability** | Langfuse | Visual trace explorer for complex loops; tracks latency, token costs, and exact execution paths |
| **Frontend** | React + Tailwind CSS + React Flow | React Flow enables a visual canvas where users wire agent cards into workflow graphs |
| **Database** | PostgreSQL (Docker) | Self-contained, portable between local and Oracle Free Tier; same container config across environments |

---

## 6. Development Phasing

### Phase 1 — Hardcoded Core (Week 1)
Build the Project Writer agent as a standalone Python script. Use hardcoded LangGraph nodes and in-memory state. Goal: get the self-correction loop working locally — the Checker should reject the draft at least once before the Publisher fires.

### Phase 2 — Schema Extraction (Week 2)
Extract all hardcoded configs into a `project_writer_spec.json` file. Refactor the engine to be config-driven: it reads the JSON and constructs the agent loop from that spec alone. Wire in **Langfuse** at this stage to capture trace metrics and prompt versions.

### Phase 3 — Platform Layer (Weeks 3–4)
Migrate the JSON spec into PostgreSQL/Supabase. Wrap the engine in a **FastAPI** server and expose endpoints:
- `POST /api/projects/create`
- `POST /api/projects/{id}/run`
- `GET  /api/projects/{id}/runs/{run_id}/stream`

Build a minimal frontend that lets you define agents and prompts, connect them visually via React Flow, and trigger runs with live streaming output.

---

## 7. Future Roadmap

These are intentionally out of scope for v1 to keep the initial build tangible and shippable.

| Item | Description |
|------|-------------|
| **Workflow versioning** | Snapshot graph topology at run time so old runs remain replayable after the workflow is edited |
| **Custom HTTP tool endpoints** | Let users register their own API endpoints as agent tools, configured via URL + headers + payload schema |
| **Agent memory** | Persist per-agent context across runs for long-running or stateful workflows |
| **Multi-user / auth** | Per-user project isolation, role-based access |
