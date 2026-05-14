# Session Management Guide

Practical guide for adding persistent session management to FAST applications, enabling users to resume past conversations after page refreshes, logouts, or device switches.

By default, FAST sessions are ephemeral — a UUID is generated client-side and held in React state. Refreshing the page or logging out loses the session. This guide describes how to add a persistence layer so users can list, resume, and manage their past sessions.

---

## Overview

Session management in FAST involves three concerns:

1. **Session identity** — Generating and tracking session IDs that link a user to a conversation
2. **Session metadata persistence** — Storing session records (name, status, timestamps) so users can browse and resume past sessions
3. **Conversation history persistence** — Ensuring the actual messages are recoverable (handled by AgentCore Memory — see [MEMORY_INTEGRATION.md](./MEMORY_INTEGRATION.md))

This guide focuses on concerns 1 and 2. AgentCore Memory already handles concern 3 — when you pass a `runtimeSessionId`, AgentCore stores and retrieves conversation history automatically. The missing piece is a way for users to discover *which* sessions exist and select one to resume.

---

## Architecture Approaches

There are several valid approaches to session persistence. The right choice depends on your scale, complexity, and existing infrastructure. A few options to consider are shown below.

| Approach | Storage | Pros | Cons | Best For |
|----------|---------|------|------|----------|
| **DynamoDB + API Gateway** | Server-side | Survives device switches, shared across clients, queryable | Requires backend infrastructure | Production applications, multi-device users |
| **localStorage / IndexedDB** | Client-side | Zero infrastructure, instant | Lost on clear/device switch, no cross-device | Prototypes, single-device use cases |
| **AgentCore Memory listing** | Server-side (existing) | No new infrastructure | Limited query flexibility, no custom metadata | Simple apps where session list = memory sessions |
| **S3 + presigned URLs** | Server-side | Good for large session payloads | Higher latency for metadata queries | Sessions with large attachments or artifacts |

This guide details the **DynamoDB + API Gateway** approach, which is robust and follows the same pattern already used by the FAST Feedback API and therefore requires minimal infrastructure code changes.

---

## DynamoDB + API Gateway Approach

This approach adds:
- A DynamoDB table to store session metadata (keyed by userId + sessionId)
- API Gateway routes for session CRUD operations
- A Lambda function to handle the API logic
- Frontend service code to call the API and populate the sidebar

**Note:** The code snippets in this document represent one way to implement this approach and are meant to be customized, validated, and security reviewed for your application before use in production.

### Data Model

```
Table: {stack_name_base}-Sessions
├── Partition Key: userId (String)    — Cognito user sub
├── Sort Key: sessionId (String)      — UUID generated client-side
├── Attributes:
│   ├── name (String)                 — Display name (first message or user-provided)
│   ├── status (String)               — "active" | "completed" | "cancelled"
│   ├── createdAt (String)            — ISO 8601 timestamp
│   ├── updatedAt (String)            — ISO 8601 timestamp (last activity)
│   ├── messageCount (Number)         — Approximate message count
│   └── metadata (Map)               — Optional application-specific data
```

**Why userId as partition key?** Each user only queries their own sessions. This gives efficient `Query` operations for listing without needing a GSI. The `sessionId` sort key enables direct `GetItem` when you know both keys.

### API Design

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/sessions` | List all sessions for the authenticated user |
| `POST` | `/sessions` | Create a new session record |
| `GET` | `/sessions/{sessionId}` | Get a specific session's metadata |
| `PATCH` | `/sessions/{sessionId}` | Update session metadata (name, status) |
| `DELETE` | `/sessions/{sessionId}` | Delete a session record |

All endpoints are protected by the existing Cognito User Pools Authorizer. The `userId` is extracted from the JWT claims — users can only access their own sessions.

---

## Step 1: Add the DynamoDB Table (CDK)

Add the sessions table to `infra-cdk/lib/backend-stack.ts`, following the same pattern as the existing Feedback table:

```typescript
import * as dynamodb from "aws-cdk-lib/aws-dynamodb";

// Inside your backend stack class:
const sessionsTable = new dynamodb.Table(this, "SessionsTable", {
  tableName: `${config.stack_name_base}-Sessions`,
  partitionKey: { name: "userId", type: dynamodb.AttributeType.STRING },
  sortKey: { name: "sessionId", type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
  encryption: dynamodb.TableEncryption.AWS_MANAGED,
  pointInTimeRecovery: true,
});
```

**💡 Tip:** If you need to query sessions by `updatedAt` (e.g., "most recently active"), add a Local Secondary Index:

```typescript
sessionsTable.addLocalSecondaryIndex({
  indexName: "updatedAt-index",
  sortKey: { name: "updatedAt", type: dynamodb.AttributeType.STRING },
});
```

---

## Step 2: Add API Gateway Routes (CDK)

Add session routes to the existing REST API (or create a new one). This follows the same pattern as the Feedback API:

```typescript
// Reference the existing API or create routes on it
const sessionsResource = api.root.addResource("sessions");
const sessionByIdResource = sessionsResource.addResource("{sessionId}");

// List + Create
sessionsResource.addMethod("GET", sessionsLambdaIntegration, {
  authorizer: cognitoAuthorizer,
  authorizationType: apigateway.AuthorizationType.COGNITO,
});
sessionsResource.addMethod("POST", sessionsLambdaIntegration, {
  authorizer: cognitoAuthorizer,
  authorizationType: apigateway.AuthorizationType.COGNITO,
});

// Get, Update, Delete by ID
sessionByIdResource.addMethod("GET", sessionsLambdaIntegration, {
  authorizer: cognitoAuthorizer,
  authorizationType: apigateway.AuthorizationType.COGNITO,
});
sessionByIdResource.addMethod("PATCH", sessionsLambdaIntegration, {
  authorizer: cognitoAuthorizer,
  authorizationType: apigateway.AuthorizationType.COGNITO,
});
sessionByIdResource.addMethod("DELETE", sessionsLambdaIntegration, {
  authorizer: cognitoAuthorizer,
  authorizationType: apigateway.AuthorizationType.COGNITO,
});

// CORS
sessionsResource.addCorsPreflight({
  allowOrigins: [frontendUrl, "http://localhost:3000"],
  allowMethods: ["GET", "POST", "OPTIONS"],
  allowHeaders: ["Content-Type", "Authorization"],
});
sessionByIdResource.addCorsPreflight({
  allowOrigins: [frontendUrl, "http://localhost:3000"],
  allowMethods: ["GET", "PATCH", "DELETE", "OPTIONS"],
  allowHeaders: ["Content-Type", "Authorization"],
});
```

---

## Step 3: Create the Sessions Lambda

Create `infra-cdk/lambdas/sessions/index.py`. This follows the same Powertools pattern as the Feedback Lambda:

```python
"""Session management API handler.

Provides CRUD operations for user session metadata stored in DynamoDB.
All endpoints extract userId from the Cognito JWT claims.
"""

import os
from datetime import datetime, timezone

import boto3
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver, CORSConfig
from aws_lambda_powertools.logging.correlation_paths import API_GATEWAY_REST
from aws_lambda_powertools.utilities.typing import LambdaContext
from botocore.exceptions import ClientError

# Environment variables
TABLE_NAME = os.environ["SESSIONS_TABLE_NAME"]
CORS_ALLOWED_ORIGINS = os.environ.get("CORS_ALLOWED_ORIGINS", "*")

# Parse CORS origins - can be comma-separated list
cors_origins = [
    origin.strip() for origin in CORS_ALLOWED_ORIGINS.split(",") if origin.strip()
]
primary_origin = cors_origins[0] if cors_origins else "*"
extra_origins = cors_origins[1:] if len(cors_origins) > 1 else None

cors_config = CORSConfig(
    allow_origin=primary_origin,
    extra_origins=extra_origins,
    allow_headers=["Content-Type", "Authorization"],
    allow_credentials=True,
)

tracer = Tracer()
logger = Logger()
app = APIGatewayRestResolver(cors=cors_config)

dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table(TABLE_NAME)


def _get_user_id() -> str:
    """Extract userId from Cognito JWT claims in the request context.

    Returns:
        The Cognito user sub (unique user identifier).

    Raises:
        ValueError: If claims are missing from the request context.
    """
    request_context = app.current_event.request_context
    authorizer = request_context.authorizer
    claims = authorizer.get("claims", {}) if authorizer else {}

    if not claims:
        raise ValueError("Unauthorized: no claims in request context")

    return claims.get("sub")


@app.get("/sessions")
def list_sessions() -> dict:
    """List all sessions for the authenticated user.

    Returns:
        Dict with 'sessions' key containing list of session metadata,
        sorted by updatedAt descending (most recent first).
    """
    user_id = _get_user_id()

    response = table.query(
        KeyConditionExpression="userId = :uid",
        ExpressionAttributeValues={":uid": user_id},
        ScanIndexForward=False,  # Most recent first (by sort key)
    )

    sessions = response.get("Items", [])
    # Sort by updatedAt descending for display
    sessions.sort(key=lambda s: s.get("updatedAt", ""), reverse=True)

    return {"sessions": sessions}


@app.post("/sessions")
def create_session() -> dict:
    """Create a new session record.

    Expects JSON body with:
        - sessionId (str): Client-generated UUID
        - name (str, optional): Display name for the session

    Returns:
        The created session metadata.
    """
    user_id = _get_user_id()
    body = app.current_event.json_body

    session_id = body["sessionId"]
    now = datetime.now(timezone.utc).isoformat()

    item = {
        "userId": user_id,
        "sessionId": session_id,
        "name": body.get("name", "New conversation"),
        "status": "active",
        "createdAt": now,
        "updatedAt": now,
        "messageCount": 0,
        "metadata": body.get("metadata", {}),
    }

    table.put_item(Item=item)
    return {"session": item}


@app.get("/sessions/<sessionId>")
def get_session(sessionId: str) -> dict:
    """Get a specific session's metadata.

    Args:
        sessionId: The session UUID from the path parameter.

    Returns:
        The session metadata, or 404 if not found.
    """
    user_id = _get_user_id()

    response = table.get_item(Key={"userId": user_id, "sessionId": sessionId})
    item = response.get("Item")

    if not item:
        return {"error": "Session not found"}, 404

    return {"session": item}


@app.patch("/sessions/<sessionId>")
def update_session(sessionId: str) -> dict:
    """Update session metadata (name, status, messageCount).

    Args:
        sessionId: The session UUID from the path parameter.

    Returns:
        The updated session metadata.
    """
    user_id = _get_user_id()
    body = app.current_event.json_body
    now = datetime.now(timezone.utc).isoformat()

    update_expr_parts = ["#updatedAt = :now"]
    attr_names = {"#updatedAt": "updatedAt"}
    attr_values = {":now": now}

    for field in ("name", "status", "messageCount", "metadata"):
        if field in body:
            update_expr_parts.append(f"#{field} = :{field}")
            attr_names[f"#{field}"] = field
            attr_values[f":{field}"] = body[field]

    table.update_item(
        Key={"userId": user_id, "sessionId": sessionId},
        UpdateExpression="SET " + ", ".join(update_expr_parts),
        ExpressionAttributeNames=attr_names,
        ExpressionAttributeValues=attr_values,
    )

    return {"session": {"sessionId": sessionId, "updatedAt": now, **body}}


@app.delete("/sessions/<sessionId>")
def delete_session(sessionId: str) -> dict:
    """Delete a session record.

    Args:
        sessionId: The session UUID from the path parameter.

    Returns:
        Confirmation of deletion.
    """
    user_id = _get_user_id()

    table.delete_item(Key={"userId": user_id, "sessionId": sessionId})
    return {"deleted": sessionId}


@logger.inject_lambda_context(correlation_id_path=API_GATEWAY_REST)
def handler(event: dict, context: LambdaContext) -> dict:
    """Lambda entry point.

    Args:
        event: API Gateway proxy event.
        context: Lambda execution context.

    Returns:
        API Gateway proxy response.
    """
    return app.resolve(event, context)
```

---

## Step 4: Frontend Integration

### Session Service

Create a service layer following the existing `feedbackService.ts` pattern:

```typescript
// src/services/sessionService.ts

export interface SessionMetadata {
  sessionId: string;
  name: string;
  status: "active" | "completed" | "cancelled";
  createdAt: string;
  updatedAt: string;
  messageCount: number;
  metadata?: Record<string, unknown>;
}

const SESSION_API_URL = import.meta.env.VITE_SESSION_API_URL;

async function getAuthHeaders(): Promise<Headers> {
  // Use your existing auth utility to get the Cognito ID token
  const token = await getIdToken();
  return new Headers({
    "Content-Type": "application/json",
    Authorization: token,
  });
}

export async function listSessions(): Promise<SessionMetadata[]> {
  const response = await fetch(`${SESSION_API_URL}/sessions`, {
    headers: await getAuthHeaders(),
  });
  const data = await response.json();
  return data.sessions;
}

export async function createSession(
  sessionId: string,
  name?: string
): Promise<SessionMetadata> {
  const response = await fetch(`${SESSION_API_URL}/sessions`, {
    method: "POST",
    headers: await getAuthHeaders(),
    body: JSON.stringify({ sessionId, name }),
  });
  const data = await response.json();
  return data.session;
}

export async function updateSession(
  sessionId: string,
  updates: Partial<Pick<SessionMetadata, "name" | "status" | "messageCount">>
): Promise<void> {
  await fetch(`${SESSION_API_URL}/sessions/${sessionId}`, {
    method: "PATCH",
    headers: await getAuthHeaders(),
    body: JSON.stringify(updates),
  });
}

export async function deleteSession(sessionId: string): Promise<void> {
  await fetch(`${SESSION_API_URL}/sessions/${sessionId}`, {
    method: "DELETE",
    headers: await getAuthHeaders(),
  });
}
```

### Session Flow

The typical user flow becomes:

1. **New conversation** — Frontend generates `crypto.randomUUID()`, calls `POST /sessions` to persist the record, then invokes AgentCore with that `runtimeSessionId`
2. **Page load / login** — Frontend calls `GET /sessions` to populate the sidebar with past sessions
3. **Resume conversation** — User clicks a session in the sidebar. Frontend sets the `runtimeSessionId` to the selected session's ID. AgentCore Memory automatically loads the conversation history for that session.
4. **During conversation** — Frontend periodically calls `PATCH /sessions/{id}` to update `messageCount` and `updatedAt`
5. **Session naming** — After the first assistant response, auto-generate a session name (e.g., first few words of the user's message, or ask the LLM to generate a title)

### Session Resumption

When resuming a session, the key insight is that **AgentCore Memory already stores the conversation history** keyed by `runtimeSessionId`. You don't need to re-send past messages. Simply:

1. Set the `runtimeSessionId` to the existing session's ID
2. Send the new user message
3. AgentCore Memory loads the prior conversation context automatically

The session metadata table is only for *discovering* which sessions exist — the actual conversation state lives in AgentCore Memory.

---

## Step 5: Store the API URL in SSM

Follow the existing pattern of storing API URLs in SSM for cross-stack access:

```typescript
new ssm.StringParameter(this, "SessionsApiUrlParam", {
  parameterName: `/${config.stack_name_base}/sessions-api-url`,
  stringValue: api.url,
  description: "Sessions API Gateway URL",
});
```

The frontend reads this at build time (via Amplify environment variables) or at runtime from a config endpoint.

---

## Session Naming Strategies

Automatically naming sessions improves the user experience in the sidebar. Common approaches:

| Strategy | Implementation | Quality |
|----------|---------------|---------|
| **First message truncation** | `message.slice(0, 50)` | Low — often not descriptive |
| **LLM-generated title** | Ask the model to title the conversation after first exchange | High — but adds latency/cost |
| **Keyword extraction** | Pull key nouns from the first few messages | Medium — no extra LLM call |
| **User-provided** | Let users rename sessions in the sidebar | Highest — but requires user action |

A practical approach is to use first-message truncation as the default, then offer a rename option in the UI.

---

## Advanced: Long-Running Agent Sessions

For agents that run autonomously for extended periods (minutes to hours), session management becomes more complex. In addition to the basic CRUD above, you may need:

### Status Tracking

Extend the session metadata with richer status information:

```python
# Additional DynamoDB attributes for long-running sessions:
{
    "status": "running",          # active | running | completed | failed | cancelled
    "detail": "Iteration 5/20",   # Human-readable progress
    "startedAt": "...",           # When the agent started
    "completedAt": "...",         # When the agent finished (if terminal)
    "lastHeartbeat": "...",       # Last time the agent reported activity
}
```

### Polling for Progress

If your agent runs longer than the SSE connection timeout (typically 60–90 seconds on AgentCore), you'll need a polling-based architecture:

1. Agent writes status updates to DynamoDB during execution
2. Frontend polls `GET /sessions/{id}` every few seconds to display progress
3. Agent streams detailed output to S3 (JSONL format) for the frontend to consume

### Cancellation

Add a cancellation mechanism:
1. Frontend calls `PATCH /sessions/{id}` with `{"status": "cancelled"}`
2. Agent checks DynamoDB status before each tool call or iteration
3. If cancelled, agent stops gracefully and updates status to terminal

---

## Cost Considerations

| Resource | Cost | Notes |
|----------|------|-------|
| DynamoDB (on-demand) | ~$1.25 per million writes, ~$0.25 per million reads | Negligible for most applications |
| API Gateway | $3.50 per million requests | Shared with other API routes |
| Lambda | Free tier covers 1M requests/month | Minimal compute per request |

For a typical application with hundreds of users and thousands of sessions, the monthly cost of session management infrastructure is well under $1. For most GenAI applications, costs are dominated by token use.

---

## Alternative: Client-Side Only (localStorage)

For prototypes or single-device applications, you can skip the backend entirely and store session metadata in `localStorage`:

```typescript
const SESSIONS_KEY = "fast-sessions";

function saveSessions(sessions: SessionMetadata[]): void {
  localStorage.setItem(SESSIONS_KEY, JSON.stringify(sessions));
}

function loadSessions(): SessionMetadata[] {
  const raw = localStorage.getItem(SESSIONS_KEY);
  return raw ? JSON.parse(raw) : [];
}
```

**Limitations:**
- Lost when user clears browser data
- Not available on other devices
- No server-side validation or access control
- Storage limit (~5–10 MB depending on browser)

This is acceptable for demos and prototypes but not recommended for production applications where users expect to resume sessions across devices or after clearing their browser.

---

## Further Reading

- [FAST Memory Integration Guide](./MEMORY_INTEGRATION.md) — How AgentCore Memory stores conversation history
- [FAST Deployment Guide](./DEPLOYMENT.md) — Deploying CDK infrastructure changes
- [FAST Streaming Guide](./STREAMING.md) — Frontend-backend communication patterns
- [AWS Lambda Powertools](https://docs.powertools.aws.dev/lambda/python/latest/) — Lambda handler patterns used in the sessions Lambda
- [Amazon DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/) — Table design and query patterns
