# Chatwoot Application API Reference

## Authentication

- **Header**: `api_access_token: <token>` (NOT `Authorization: Bearer`)
- **Token source**: Chatwoot UI > Profile Settings > Access Token (user or admin token)
- **Base path**: `/api/v1/accounts/{account_id}/...`

All examples use `httpx.AsyncClient` with Python type hints.

## Common Patterns

### Retry Decorator

Apply to all Application API calls except best-effort operations (team assignment, label removal):

```python
from tenacity import retry, retry_if_exception_type, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10),
    retry=retry_if_exception_type(httpx.HTTPError),
    reraise=True,
)
async def api_call(...):
    ...
```

### 422 Error Handling

Always catch 422 separately -- it is a permanent validation error that must NOT be retried:

```python
try:
    response.raise_for_status()
except httpx.HTTPStatusError as e:
    if e.response.status_code == 422:
        logger.warning(f"Validation error (422): {e.response.text}")
        return False  # or None -- never retry
    raise  # Let retry decorator handle transient errors
```

### httpx Client Pattern

Create a new `AsyncClient` per request (connection pooling is handled by httpx internally). Set explicit timeouts:

```python
async with httpx.AsyncClient() as client:
    response = await client.get(url, headers=self.headers, timeout=10.0)
    response.raise_for_status()
```

---

## Endpoints by Resource

### Contacts

#### Search Contact by Phone

```
GET /api/v1/accounts/{account_id}/contacts/search?q={phone}
```

Returns `{"payload": [contact, ...]}`. The payload is a list; take the first match.

```python
response = await client.get(
    f"{self.api_url}/api/v1/accounts/{self.account_id}/contacts/search",
    params={"q": phone},
    headers=self.headers,
    timeout=10.0,
)
payload = response.json().get("payload", [])
contact = payload[0] if payload else None
```

#### Create Contact

```
POST /api/v1/accounts/{account_id}/contacts
{"inbox_id": "...", "phone_number": "+34...", "name": "optional"}
```

Response: `{"payload": {"contact": {...}}}` -- note the nested structure.

```python
response = await client.post(
    f"{self.api_url}/api/v1/accounts/{self.account_id}/contacts",
    json={"inbox_id": self.inbox_id, "phone_number": phone, "name": name},
    headers=self.headers,
    timeout=10.0,
)
contact = response.json().get("payload", {}).get("contact", {})
```

#### Update Contact

```
PUT /api/v1/accounts/{account_id}/contacts/{contact_id}
{"name": "...", "email": "...", "custom_attributes": {"key": "value"}}
```

Only include fields you want to change. Omitted fields are NOT cleared.

**Important**: Never send an empty string for `email` -- Chatwoot may reject it with 422. Filter empty strings to `None` and omit from payload.

```python
payload = {}
if name is not None:
    payload["name"] = name
if email:  # Filters both None and ""
    payload["email"] = email
if custom_attributes is not None:
    payload["custom_attributes"] = custom_attributes
```

#### Get Contact (with Inboxes)

```
GET /api/v1/accounts/{account_id}/contacts/{contact_id}
```

Response includes `contact_inboxes` -- needed to get `source_id` for conversation creation.

```python
response = await client.get(
    f"{self.api_url}/api/v1/accounts/{self.account_id}/contacts/{contact_id}",
    headers=self.headers,
    timeout=10.0,
)
contact = response.json().get("payload", {})
source_id = contact["contact_inboxes"][0]["source_id"]
```

---

### Conversations

#### Create Conversation

```
POST /api/v1/accounts/{account_id}/conversations
{"source_id": "...", "inbox_id": 123, "contact_id": 456, "status": "open"}
```

The `source_id` comes from the contact's `contact_inboxes`. For WhatsApp, it is the phone number without `+`.

Response: `{"id": conversation_id, ...}`

#### Create Conversation with Template Message

Combine conversation creation and template sending in one call:

```python
payload = {
    "source_id": phone.lstrip("+"),
    "inbox_id": int(self.inbox_id),  # Must be int
    "contact_id": contact_id,
    "status": "open",
    "message": {
        "content": "Fallback text for non-WhatsApp",
        "template_params": {
            "name": template_name,
            "category": "UTILITY",  # or "MARKETING"
            "language": "es",
            "processed_params": {
                "body": {"1": "param1_value", "2": "param2_value"},
            },
        },
    },
}
```

#### Get Conversation

```
GET /api/v1/accounts/{account_id}/conversations/{conversation_id}
```

Returns conversation details including `labels`, `custom_attributes`, `meta`.

#### Update Custom Attributes

```
POST /api/v1/accounts/{account_id}/conversations/{conversation_id}/custom_attributes
{"custom_attributes": {"key": "value"}}
```

---

### Messages

#### Send Text Message

```
POST /api/v1/accounts/{account_id}/conversations/{conversation_id}/messages
{"content": "text", "message_type": "outgoing", "private": false}
```

#### Send Private Note (Internal)

Same endpoint, but with `"private": true`:

```python
{"content": "Internal note for agents", "message_type": "outgoing", "private": true}
```

Private notes are only visible to agents in the Chatwoot UI, never sent to the customer.

#### Send Template Message (to Existing Conversation)

```python
{
    "content": "Fallback text",
    "message_type": "outgoing",
    "template_params": {
        "name": "template_name",
        "category": "UTILITY",
        "language": "es",
        "processed_params": {
            "body": {"1": "value1"},
        },
    },
}
```

Template messages are required for WhatsApp messages outside the 24-hour conversation window.

#### Send Image (Multipart Upload)

Chatwoot does NOT support external URLs for image attachments. You must download and re-upload:

```python
# Step 1: Download image
img_response = await client.get(image_url, timeout=30.0)
img_response.raise_for_status()

filename = image_url.split("/")[-1] or "image.png"
content_type = img_response.headers.get("content-type", "image/png")

# Step 2: Upload as multipart -- DO NOT include Content-Type in headers
files = {"attachments[]": (filename, img_response.content, content_type)}
data = {"content": caption or "", "message_type": "outgoing", "private": "false"}
headers = {"api_access_token": self.api_token}  # No Content-Type header!

response = await client.post(
    f"{self.api_url}/api/v1/accounts/{self.account_id}/conversations/{conversation_id}/messages",
    data=data,
    files=files,
    headers=headers,
    timeout=30.0,
)
```

**Critical**: For multipart uploads, only include `api_access_token` in headers. Do NOT include `Content-Type: application/json` -- it breaks the multipart boundary.

#### Get Messages (Paginated)

```
GET /api/v1/accounts/{account_id}/conversations/{conversation_id}/messages?page=1&per_page=100
```

Response: `{"payload": [messages...], "meta": {"pages": total_pages}}`

Message types: `0` = incoming (from contact), `1` = outgoing, `2` = activity.

Filter image messages by checking `attachments[].file_type == "image"`.

---

### Agents

#### List Agents

```
GET /api/v1/accounts/{account_id}/agents
```

Returns flat array of agent objects (not wrapped in `payload`).

#### Create Agent (Link Platform User to Account)

```
POST /api/v1/accounts/{account_id}/agents
{"name": "Agent Name", "email": "agent@company.com", "role": "agent"}
```

This is the **critical workaround** for the missing Platform API `/account_users` endpoint. If you created a platform user with the same email, Chatwoot links them internally.

Returns the agent object with `id` (the `agent_id` for assignment operations).

---

### Labels

#### Set Labels (Replaces All)

```
POST /api/v1/accounts/{account_id}/conversations/{conversation_id}/labels
{"labels": ["label1", "label2", "label3"]}
```

**WARNING**: This replaces ALL labels. To add without removing:

```python
# 1. Get current labels
conversation = await self.get_conversation(conversation_id)
current = conversation.get("labels", [])

# 2. Merge
merged = list(set(current) | set(new_labels))

# 3. Set merged list
await client.post(..., json={"labels": merged})
```

To remove labels, compute the set difference and POST the result.

---

### Assignments

#### Assign to Team

```
POST /api/v1/accounts/{account_id}/conversations/{conversation_id}/assignments
{"team_id": 42}
```

Best-effort -- may fail if the bot token lacks assignment permissions. Do not retry.

#### Assign to Agent

```
POST /api/v1/accounts/{account_id}/conversations/{conversation_id}/assignments
{"assignee_id": agent_id}
```

Same endpoint, different payload key.

---

## Response Patterns

| Endpoint | Response Wrapper |
|----------|-----------------|
| Contact search | `{"payload": [contacts...]}` |
| Contact create | `{"payload": {"contact": {...}}}` |
| Contact get | `{"payload": {..., "contact_inboxes": [...]}}` |
| Conversation create | `{"id": ..., ...}` (flat) |
| Messages list | `{"payload": [messages...], "meta": {"pages": N}}` |
| Agents list | `[agents...]` (flat array, no wrapper) |
| Agent create | `{"id": ..., ...}` (flat) |

Each endpoint has its own wrapper convention. Always check the actual response structure.
