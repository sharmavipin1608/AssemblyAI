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

The entire configuration of your project workspace — agent personas and structural flow logic — is stored relationally.

```sql
-- 1. Project Container
CREATE TABLE projects (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    description TEXT,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 2. Project-Level API Keys / LLM Configuration
CREATE TABLE project_configs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID REFERENCES projects(id) ON DELETE CASCADE,
    provider            VARCHAR(50)  NOT NULL,  -- e.g. 'openai', 'anthropic'
    api_key_vault_sec_id VARCHAR(255),           -- Reference to an encrypted vault secret
    default_model       VARCHAR(100) NOT NULL
);

-- 3. Dynamic Agents
CREATE TABLE agents (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id       UUID REFERENCES projects(id) ON DELETE CASCADE,
    role_key         VARCHAR(100) NOT NULL,  -- e.g. 'researcher', 'shadow_writer'
    display_name     VARCHAR(255) NOT NULL,
    system_prompt_id VARCHAR(255) NOT NULL,  -- Versioned inside Langfuse
    model_override   VARCHAR(100) NULL,
    temperature      NUMERIC(2,1) DEFAULT 0.2
);

-- 4. Dynamic Execution Graph Topology (Routing Edges)
CREATE TABLE workflows (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id        UUID REFERENCES projects(id) ON DELETE CASCADE,
    source_node       VARCHAR(100) NOT NULL,  -- Agent role_key or 'START'
    target_node       VARCHAR(100) NOT NULL,  -- Agent role_key or 'END'
    routing_condition VARCHAR(100) NOT NULL DEFAULT 'always',  -- 'always', 'if_approved', 'if_gap_found'
    created_at        TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

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
| **Database** | PostgreSQL / Supabase | Relational schema fits the project/agent/workflow topology; Supabase adds a real-time layer and easy auth |

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
