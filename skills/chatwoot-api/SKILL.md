---
name: chatwoot-api
description: Implement Chatwoot API integrations for customer service systems. Use this skill when working with Chatwoot webhooks, contacts, conversations, agents, messages, labels, teams, or any Chatwoot API endpoint. Also use when implementing user/agent sync between an application and Chatwoot, managing Chatwoot platform users, or building admin panels that manage Chatwoot agents. Covers both Application API and Platform API with self-hosted gotchas.
---

# Chatwoot Integration Skill

## Overview

Chatwoot exposes **two distinct APIs** with different auth mechanisms, base paths, and capabilities.

| API | Auth Header | Base Path | Token Source | Use For |
|-----|-------------|-----------|--------------|---------|
| **Application API** | `api_access_token: <token>` | `/api/v1/accounts/{account_id}/...` | Profile Settings (user/admin token) | Contacts, conversations, messages, labels, agents, teams |
| **Platform API** | `api_access_token: <token>` | `/platform/api/v1/...` | Super Admin Console (Platform App token) | User lifecycle (create/update/delete), SSO. **Self-hosted only.** |

Both APIs use the same header name (`api_access_token`) but with **different tokens**. Keep them separate in config.

## Critical Gotchas

These are hard-won lessons from production. Read them BEFORE writing any Chatwoot integration code.

### 1. Auth Header Is NOT `Authorization: Bearer`

Chatwoot uses a custom header for both APIs:

```
api_access_token: <your_token>
```

**NOT** `Authorization: Bearer <token>`. The official docs sometimes show Bearer format, but self-hosted Chatwoot 4.x requires the `api_access_token` header. If you get 401s, check the header name first.

### 2. Platform API `/users/{id}/account_users` Does NOT Exist in 4.x

The Chatwoot docs mention a Platform API endpoint to link a user to an account:

```
POST /platform/api/v1/users/{id}/account_users
```

**This endpoint does not exist in Chatwoot 4.x.** It returns 404.

**Workaround**: Use the Application API to create an agent with the same email:

```
POST /api/v1/accounts/{account_id}/agents
{"name": "Agent Name", "email": "same@email.com", "role": "agent"}
```

Chatwoot internally matches the platform user and the agent by email. This is the hybrid flow: Platform API creates the user (with password), Application API links to account.

### 3. `account_id` Must Be `int` in JSON Payloads

If your `account_id` is stored as a string (common with env vars), cast it to `int` before sending in JSON payloads. Some endpoints silently fail or return 422 with string account IDs.

### 4. Never Retry 422 Responses

A 422 from Chatwoot means **permanent validation error** (duplicate email, invalid field, etc.). Retrying will never succeed. Catch `HTTPStatusError`, check for 422, log it, and return `False` -- do NOT let it bubble into your retry decorator.

### 5. Contact/Agent Sync Must Be Fire-and-Forget

Sync to Chatwoot should NEVER block the primary operation. If a user is created in your DB, that succeeds regardless of whether Chatwoot is reachable. Use `asyncio.create_task()` after the DB commit.

### 6. Docker: `restart` Does NOT Reload `.env`

`docker compose restart <service>` does NOT re-read `.env` files. You need `docker compose down && docker compose up -d`. For code changes to the Chatwoot image itself, you also need `docker compose build`.

### 7. Image Uploads Require Multipart, Not JSON

Chatwoot does NOT support `external_url` in attachments for sending images. You must:
1. Download the image yourself
2. Upload as `multipart/form-data` with `attachments[]` field
3. Use headers WITHOUT `Content-Type: application/json` (let httpx set the multipart boundary)

### 8. Labels API Replaces, Not Appends

`POST .../conversations/{id}/labels` **replaces** all labels. To add labels without removing existing ones, first GET the conversation, merge the label sets, then POST the union.

### 9. WhatsApp Image Ordering Requires Delays

When sending multiple images via WhatsApp through Chatwoot, add a configurable delay (e.g., 3-5s) between sends. Without delays, WhatsApp may deliver them out of order.

## Quick Start

### 1. Config Settings

Use Pydantic Settings for all Chatwoot configuration:

```python
from pydantic import Field
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    CHATWOOT_API_URL: str = Field(default="https://app.chatwoot.com")
    CHATWOOT_API_TOKEN: str = Field(default="")           # Application API token
    CHATWOOT_ACCOUNT_ID: str = Field(default="")          # Cast to int for JSON payloads
    CHATWOOT_INBOX_ID: str = Field(default="")
    CHATWOOT_PLATFORM_TOKEN: str = Field(default="")      # Platform API token (self-hosted only)
    CHATWOOT_WEBHOOK_TOKEN: str = Field(default="")       # Webhook URL auth
    CHATWOOT_STORAGE_DOMAIN: str = Field(default="")      # For active_storage URLs
    CHATWOOT_IMAGE_SEND_DELAY_SECONDS: float = Field(default=5.0, ge=0.0)

    class Config:
        env_file = ".env"
```

### 2. Client Class Pattern

```python
import httpx
from tenacity import retry, retry_if_exception_type, stop_after_attempt, wait_exponential

class ChatwootClient:
    def __init__(self):
        settings = get_settings()
        self.api_url = settings.CHATWOOT_API_URL.rstrip("/")
        self.api_token = settings.CHATWOOT_API_TOKEN
        self.platform_token = settings.CHATWOOT_PLATFORM_TOKEN
        self.account_id = settings.CHATWOOT_ACCOUNT_ID

        # Application API headers
        self.headers = {
            "api_access_token": self.api_token,
            "Content-Type": "application/json",
        }

    @property
    def platform_headers(self) -> dict[str, str]:
        """Separate headers for Platform API -- different token."""
        return {
            "api_access_token": self.platform_token,
            "Content-Type": "application/json",
        }
```

### 3. Basic Usage

```python
client = ChatwootClient()

# Find contact
contact = await client.find_contact_by_phone("+34612345678")

# Send message
await client.send_message(
    customer_phone="+34612345678",
    message="Hello from the system",
    conversation_id=42,
)

# Create platform user + link to account (hybrid flow)
user = await client.create_platform_user(name="Agent", email="agent@co.com", password="secret")
agent = await client.link_user_to_account(name="Agent", email="agent@co.com")
```

## Reference Files

Read these for detailed endpoint documentation and integration patterns:

| File | When to Read |
|------|-------------|
| `references/application-api.md` | Working with contacts, conversations, messages, labels, agents, teams |
| `references/platform-api.md` | Managing platform users (create/update/delete), SSO, self-hosted admin |
| `references/integration-patterns.md` | Building sync services, fire-and-forget patterns, retry logic, dual ID storage |
