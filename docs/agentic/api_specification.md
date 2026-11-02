# Occupify Agentic (AI & Job Matching) — API Specification

For business and system error codes, refer to [error_code.md](error_code.md).

> [!NOTE]
> **Base Port & Gateway Routing:**
> - Direct Service Port: `8186` (Internal FastAPI runtime)
> - Gateway Port: `8180` (External ingress entry point)
> - Gateway Routes: `/ai/**` (Streaming pass-through without timeout)
> - Database Port: PostgreSQL `5535` (`occupify_agentic` with `pgvector` HNSW index), RabbitMQ `5672`
> - Architecture: FastAPI (Python 3.13) + FastStream AMQP Broker + Qwen/BGE-M3 Dense Embeddings + GLM AI Agents

---

## Standard Response Envelopes

Endpoints returning standard REST data adhere to the Occupify response envelope:

### Single Data Response (`SingleResponse<T>`)
```json
{
  "statusCode": 200,
  "message": "Operation completed successfully",
  "data": { ... }
}
```

### Resource Created Response (`CreatedResponse<T>`)
```json
{
  "statusCode": 201,
  "message": "Resource created successfully",
  "data": { ... }
}
```

### Error Response (`ErrorResponse`)
```json
{
  "statusCode": 422,
  "errorCode": "PII_REMOVAL_FAILED",
  "message": "Failed to remove personal data from your message. Nothing was sent.",
  "error": {
    "code": "PII_REMOVAL_FAILED",
    "message": "Failed to remove personal data from your message. Nothing was sent."
  }
}
```

---

## 1. Chat & Conversational Agent (`/ai/chat`)

### 1.1 Stream Chat Message (Server-Sent Events)
- **Endpoint:** `POST /ai/chat`
- **Description:** Sends a user prompt or CV context to the conversational agent. Streams back incremental token chunks, tool execution metadata, and token usage via Server-Sent Events (SSE).
- **Authorization:** Authenticated (`X-User-Id` header injected by Gateway or Bearer JWT)
- **Headers:** `Content-Type: application/json`, `Accept: text/event-stream`

#### Request Body
| Field | Type | Required | Constraints | Description |
| :--- | :--- | :--- | :--- | :--- |
| `message` | String | Yes | Max 20,000 chars | User input message or query |
| `context` | String | No | Max 20,000 chars | Optional background context or pasted CV text |
| `fileIds` | Array[String] | No | Max 3 file IDs | IDs of previously uploaded PDF CV documents |
| `sessionId` | String (UUID) | No | Valid UUID | Existing chat session ID. Omit to initialize a new session |

#### Request Body Example
```json
{
  "message": "Please search for Backend Engineer jobs matching my resume skills.",
  "context": "Prefer remote or hybrid roles with monthly salaries > 20,000,000 VND.",
  "fileIds": ["c1234567-89ab-cdef-0123-456789abcdef"],
  "sessionId": "a0000000-0000-0000-0000-000000000001"
}
```

#### SSE Stream Events Protocol (`text/event-stream`)
The endpoint streams chunked blocks separated by `\n\n`:

##### 1. Event: `meta`
Sent at connection start containing session and attachment context:
```http
event: meta
data: {"sessionId": "a0000000-0000-0000-0000-000000000001", "newSession": false, "messageId": "msg-001", "attachments": [{"id": "c1234567-89ab-cdef-0123-456789abcdef", "name": "Resume.pdf", "size": 154200, "chars": 3450}]}
```

##### 2. Event: `delta`
Incrementally streams generated text tokens:
```http
event: delta
data: {"text": "I analyzed your resume and found "}

event: delta
data: {"text": "3 matching Backend Engineer positions: ..."}
```

##### 3. Event: `done`
Emitted upon successful completion containing full response metadata and token consumption stats:
```http
event: done
data: {"sessionId": "a0000000-0000-0000-0000-000000000001", "messageId": "msg-001", "model": "glm-5.3-flashx", "usage": {"tokensIn": 3200, "tokensOut": 450, "tokensReasoning": 12, "tokensTotal": 3662, "latencyMs": 1420}}
```

##### 4. Event: `error`
Emitted if an unhandled LLM or pipeline failure occurs:
```http
event: error
data: {"code": "LLM_UNAVAILABLE", "message": "Failed to reach the AI model. Please try again."}
```

---

### 1.2 Upload CV Document (PDF)
- **Endpoint:** `POST /ai/chat/upload`
- **Description:** Uploads a PDF document (e.g., CV / Resume) for text extraction and PII pre-sanitization before chatting.
- **Authorization:** Authenticated (`X-User-Id`)
- **Headers:** `Content-Type: multipart/form-data`

#### Multipart Form Fields
| Field | Type | Required | Constraints | Description |
| :--- | :--- | :--- | :--- | :--- |
| `file` | Binary (PDF) | Yes | Max 10MB, max 20 pages, PDF format | The resume/CV PDF document |

#### Response Example (`200 OK`)
```json
{
  "statusCode": 200,
  "message": "File uploaded and parsed successfully",
  "data": {
    "fileId": "c1234567-89ab-cdef-0123-456789abcdef",
    "filename": "Tien_Nguyen_Resume.pdf",
    "size": 245800,
    "pages": 2,
    "chars": 4210,
    "contentType": "application/pdf"
  }
}
```

---

### 1.3 List Chat Sessions
- **Endpoint:** `GET /ai/chat/sessions`
- **Description:** Returns all historical chat sessions belonging to the authenticated user.
- **Authorization:** Authenticated (`X-User-Id`)

#### Response Example (`200 OK`)
```json
{
  "statusCode": 200,
  "message": "Sessions retrieved successfully",
  "data": [
    {
      "id": "a0000000-0000-0000-0000-000000000001",
      "title": "Backend Developer Job Search",
      "createdAt": "2026-11-02T14:20:00Z",
      "updatedAt": "2026-11-02T14:25:30Z",
      "messageCount": 6
    }
  ]
}
```

---

### 1.4 Get Chat Session Messages
- **Endpoint:** `GET /ai/chat/sessions/{sessionId}/messages`
- **Description:** Retrieves the complete ordered conversation history for a specific session.
- **Authorization:** Authenticated (`X-User-Id`)

#### Path Parameters
| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `sessionId` | String (UUID) | Yes | Chat session UUID |

#### Response Example (`200 OK`)
```json
{
  "statusCode": 200,
  "message": "Messages retrieved successfully",
  "data": [
    {
      "id": "msg-001",
      "role": "user",
      "content": "Find me React Native jobs in HCMC.",
      "createdAt": "2026-11-02T14:20:05Z",
      "attachments": []
    },
    {
      "id": "msg-002",
      "role": "assistant",
      "content": "Here are 2 active React Native mobile positions currently recruiting...",
      "createdAt": "2026-11-02T14:20:12Z",
      "model": "glm-5.3-flashx"
    }
  ]
}
```

---

### 1.5 Delete Chat Session
- **Endpoint:** `DELETE /ai/chat/sessions/{sessionId}`
- **Description:** Deletes a chat session and all associated message history.
- **Authorization:** Authenticated (`X-User-Id`)

#### Response Example (`200 OK`)
```json
{
  "statusCode": 200,
  "message": "Session deleted successfully"
}
```

---

## 2. Observability & Developer Tools (`/ai/dev`)

> [!NOTE]
> `/ai/dev` endpoints are enabled in `dev`, `local`, and `staging` environments to inspect privacy gate filters and latency telemetry.

### 2.1 Live Execution Log Stream (SSE)
- **Endpoint:** `GET /ai/dev/logs`
- **Description:** Real-time Server-Sent Events stream emitting execution events (`tool.call`, `tool.result`, `pii.cleaned`, `job_match.stage`, `llm.response`, `run.end`).
- **Authorization:** Whitelisted for developer inspection.
- **Format:** `text/event-stream`

### 2.2 Get Recent Structured Logs
- **Endpoint:** `GET /ai/dev/logs/recent`
- **Description:** Returns the last N structured log objects from the in-memory circular buffer.
- **Query Parameters:**
  - `limit`: Integer (Default: `200`, Max: `1000`)
  - `sessionId`: String (UUID, optional filter)

#### Response Example (`200 OK`)
```json
{
  "statusCode": 200,
  "data": [
    {
      "ts": "2026-11-02T16:03:35.989Z",
      "level": "INFO",
      "event": "pii.cleaned",
      "source": "project",
      "counts": {"PHONE": 1, "EMAIL": 1}
    },
    {
      "ts": "2026-11-02T16:03:39.360Z",
      "level": "INFO",
      "event": "job_sync.project_upserted",
      "project_id": "e2e-project-live-01",
      "jobs_n": 1
    }
  ]
}
```

---

## 3. System Health & Readiness Checks

### 3.1 Service Liveness
- **Endpoint:** `GET /health`
- **Description:** Standard container orchestrator liveness probe.
- **Response:** `{"status": "UP"}` (`200 OK`)

### 3.2 Service Readiness
- **Endpoint:** `GET /ready`
- **Description:** Verifies database connectivity (PostgreSQL pgvector), AMQP broker state, sentence-transformer model loading, and vector index health.
- **Response Example (`200 OK` or `503 Service Unavailable`):**
```json
{
  "status": "UP",
  "checks": {
    "database": "UP",
    "broker": "UP",
    "index": "ok",
    "models_loaded": true
  }
}
```

---

## 4. Internal AMQP Topology & RPC Contracts

### 4.1 Downstream Core RPC Calls (`occupify.agent.rpc`)
Direct Exchange: `occupify.agent.rpc`

| Tool / Routing Key | Caller | Target Queue | Request Payload | Response Schema | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `core.project.search` | `occupify-agentic` | `occupify.core.rpc.project.search` | `{"keyword": str, "page": int, "size": int, "sort": str}` | `PageResponse<AgentProjectResponse>` | Batch sync & fallback candidate search |
| `core.project.get` | `occupify-agentic` | `occupify.core.rpc.project.get` | `{"projectId": str}` | `SingleResponse<AgentProjectResponse>` | Fetches fresh project and job fields for re-embedding |

### 4.2 Event-Driven Sync Subscriptions (`occupify.core.exchange`)
Topic Exchange: `occupify.core.exchange`

| Topic Routing Key | Bound Queue | Trigger Event | Consumer Action |
| :--- | :--- | :--- | :--- |
| `project.created` | `occupify.agentic.job_sync` | `ProjectCreatedDomainEvent` | Fetches project via `core.project.get`, sanitizes text, computes BGE-M3 embeddings, and upserts jobs into `job_index`. |
| `project.updated` | `occupify.agentic.job_sync` | `ProjectUpdatedDomainEvent` | Re-fetches project, re-sanitizes, re-embeds, and atomically updates `job_index`. |
| `project.closed` | `occupify.agentic.job_sync` | `ProjectClosedDomainEvent` | Deletes all active jobs associated with the closed `projectId` from `job_index`. |
| `job.status.changed` | `occupify.agentic.job_item_sync` (bind: `job.#`) | `JobStatusChangedDomainEvent` | If `newStatus == "FILLED"` or `"CLOSED"`, immediately deletes the specific `jobId` from search index. |
