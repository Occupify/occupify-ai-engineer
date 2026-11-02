# Occupify Agentic Error Codes Specification

This document details all business, system, and LLM-related error codes thrown by `occupify-agentic`.
Errors are caught by FastAPI exception handlers and returned in the standardized Occupify `ErrorResponse` envelope:

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

## Error Codes Reference Table

| Error Code | HTTP Status | Exception Class | Description / Business Meaning | Triggering Scenario |
| :--- | :--- | :--- | :--- | :--- |
| **`UNAUTHORIZED`** | `401 UNAUTHORIZED` | `AppError` | **Missing / Invalid Authentication** | Request missing `X-User-Id` header (and local development bypass is disabled). |
| **`CHAT_SESSION_NOT_FOUND`** | `404 NOT_FOUND` | `SessionNotFoundError` | **Chat Session Not Found** | Thrown when querying or deleting a `sessionId` that does not exist or does not belong to the user. |
| **`CHAT_BUSY`** | `409 CONFLICT` | `ChatBusyError` | **Session Concurrency Conflict** | Thrown when a new message is sent to a chat session that is currently processing an ongoing stream. |
| **`FILE_INVALID`** | `400 BAD_REQUEST` | `FileInvalidError` | **Uploaded File Invalid** | Thrown when uploaded file is not a valid PDF, exceeds 10 MB, exceeds 20 pages, or is password-protected. |
| **`PII_REMOVAL_FAILED`** | `422 UNPROCESSABLE_ENTITY` | `PiiRemovalFailedError` | **Privacy Gate Failure** | Thrown when PII remnants (unmasked phone numbers, emails, addresses) survive after cleaning passes. Message is blocked from downstream AI. |
| **`CHAT_CONTEXT_FULL`** | `422 UNPROCESSABLE_ENTITY` | `ChatContextFullError` | **Context Window Exceeded** | The cumulative token count of the conversation history exceeds the model's maximum context window. |
| **`CHAT_CONTENT_FILTERED`** | `422 UNPROCESSABLE_ENTITY` | `ContentFilteredError` | **Content Policy Filtered** | The model provider's safety classifier flagged the prompt or generated response. |
| **`LLM_UNAVAILABLE`** | `502 BAD_GATEWAY` | `LlmUnavailableError` | **Model Provider Unreachable** | Upstream AI API (GLM/Zhipu) timed out, failed to respond, or returned an HTTP 5xx error. |
| **`JOB_MATCH_BUSY`** | `409 CONFLICT` | `JobMatchBusyError` | **Matching Run Concurrency Conflict** | A job matching pipeline run is already in progress for this user/session. |
| **`JOB_MATCH_NOT_FOUND`** | `404 NOT_FOUND` | `JobMatchNotFoundError` | **Matching Run Record Not Found** | Thrown when querying a `run_id` that does not exist in the database. |
| **`JOB_MATCH_CONTEXT_FULL`**| `422 UNPROCESSABLE_ENTITY` | `JobMatchContextFullError` | **CV Context Exceeded** | Candidate profile and CV document exceed token capacity for profile extraction. |
| **`RUN_BUDGET_EXCEEDED`** | `429 TOO_MANY_REQUESTS` | `BudgetExceeded` | **Token / Request Budget Exceeded** | The pipeline exceeded the allocated maximum request count or input/output token allowance for the run. |
| **`RPC_TIMEOUT`** | `504 GATEWAY_TIMEOUT` | `AppError` | **Downstream RPC Timeout** | AMQP request to Occupify Core (`core.project.search` / `core.project.get`) exceeded `RABBITMQ_RPC_TIMEOUT_S` (10s). |
| **`RPC_ERROR`** | `502 BAD_GATEWAY` | `AppError` | **Downstream RPC Communication Failure** | Failed to publish to or receive response from `occupify.agent.rpc` exchange. |

---

## Non-Blocking System Warning Codes

When pipeline steps experience non-fatal degradations, warnings are logged and returned in the run outcome metadata:

| Warning Code | Severity | Description | System Fallback Behavior |
| :--- | :--- | :--- | :--- |
| **`INDEX_UNAVAILABLE`** | Warning | PostgreSQL `job_index` is missing or unreachable. | Degrades to live `core.project.search` RPC queries instead of pgvector hybrid search. |
| **`RERANKER_UNAVAILABLE`** | Warning | Cross-encoder reranker model failed to load. | Degrades to weighted heuristic scoring (skill overlap + salary match) without neural reranking. |
| **`SKILL_MATCH_FALLBACK`** | Informational | LLM skill inference unavailable. | Falls back to deterministic regex and taxonomy dictionary matching. |
| **`ASSESSMENT_FALLBACK`** | Informational | Fit assessment subagent call timed out. | Returns candidate matches with rule-based fit metrics without deep textual rationale. |
