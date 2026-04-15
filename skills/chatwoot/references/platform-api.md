# Chatwoot Platform API Reference

## Important: Self-Hosted Only

The Platform API is available ONLY on self-hosted Chatwoot instances. The Chatwoot SaaS (app.chatwoot.com) does not expose Platform API endpoints.

## Authentication

- **Header**: `api_access_token: <platform_token>`
- **Token source**: Super Admin Console > Platform Apps > Create App > Access Token
- **Base path**: `/platform/api/v1/...`

The header name is the same as the Application API (`api_access_token`), but the token value is different. Store them separately in your config:

```python
CHATWOOT_API_TOKEN = "..."       # Application API (Profile Settings)
CHATWOOT_PLATFORM_TOKEN = "..."  # Platform API (Super Admin Console)
```

### Headers

```python
@property
def platform_headers(self) -> dict[str, str]:
    return {
        "api_access_token": self.platform_token,
        "Content-Type": "application/json",
    }
```

## Endpoints

### Create Platform User

```
POST /platform/api/v1/users
{"name": "Agent Name", "email": "agent@company.com", "password": "SecurePass123"}
```

**Password is REQUIRED** for user creation. There is no way to create a user without a password via the Platform API.

Response:

```json
{"id": 42, "name": "Agent Name", "email": "agent@company.com", ...}
```

The `id` returned is the **platform user ID** -- store this for future update/delete operations.

```python
async def create_platform_user(self, name: str, email: str, password: str) -> dict | None:
    if not self.platform_token:
        logger.warning("CHATWOOT_PLATFORM_TOKEN not configured")
        return None

    async with httpx.AsyncClient() as client:
        try:
            response = await client.post(
                f"{self.api_url}/platform/api/v1/users",
                json={"name": name, "email": email, "password": password},
                headers=self.platform_headers,
                timeout=10.0,
            )
            response.raise_for_status()
            return response.json()
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 422:
                # Duplicate email, validation error -- do not retry
                logger.warning(f"422 creating user: {e.response.text}")
                return None
            raise
```

### Update Platform User

```
PATCH /platform/api/v1/users/{platform_user_id}
{"name": "New Name", "email": "new@email.com", "password": "NewPass123"}
```

All fields are optional. Only send the fields you want to change. Password can be updated this way -- useful for password sync between your app and Chatwoot.

Possible error codes:
- **422**: Validation error (e.g., email taken by another user)
- **404**: User not found

Both 422 and 404 should NOT be retried:

```python
except httpx.HTTPStatusError as e:
    if e.response.status_code in (422, 404):
        logger.warning(f"Platform API error ({e.response.status_code}): {e.response.text}")
        return False
    raise
```

### Delete Platform User

```
DELETE /platform/api/v1/users/{platform_user_id}
```

- Returns 200 on success
- Returns 404 if already deleted (treat as idempotent success or log and return `False`)

```python
except httpx.HTTPStatusError as e:
    if e.response.status_code == 404:
        logger.debug(f"User {platform_user_id} already deleted")
        return False
    raise
```

### Get Platform User

```
GET /platform/api/v1/users/{platform_user_id}
```

Returns user details. Useful for verifying sync state.

### SSO Login URL

```
GET /platform/api/v1/users/{platform_user_id}/login
```

Returns a URL that logs the user into Chatwoot without credentials. Useful for building "Open in Chatwoot" buttons in your admin panel.

---

## What Does NOT Work in Chatwoot 4.x

### `/users/{id}/account_users` -- Does NOT Exist

The Chatwoot API docs describe this endpoint:

```
POST /platform/api/v1/users/{id}/account_users
{"account_id": 1, "role": "agent"}
```

**This endpoint returns 404 in Chatwoot 4.x.** It may exist in newer versions, but as of v4.x it is not implemented.

### The Hybrid Workaround

To link a platform user to a Chatwoot account, use the Application API instead:

```
POST /api/v1/accounts/{account_id}/agents
{"name": "Same Name", "email": "same@email.com", "role": "agent"}
```

Chatwoot matches the agent to the platform user by email internally.

**The complete 3-step flow**:

1. **Platform API**: Create user with password
   ```
   POST /platform/api/v1/users
   {"name": "Agent", "email": "agent@co.com", "password": "secret"}
   → returns {id: 42}  (this is chatwoot_user_id)
   ```

2. **Application API**: Link to account by creating agent with same email
   ```
   POST /api/v1/accounts/{account_id}/agents
   {"name": "Agent", "email": "agent@co.com", "role": "agent"}
   → returns {id: 7}  (this is chatwoot_agent_id)
   ```

3. **Store both IDs**: You need both for different operations
   - `chatwoot_user_id` (42) -- for Platform API operations (update password, delete user)
   - `chatwoot_agent_id` (7) -- for Application API operations (assign conversation)

---

## Platform Token Guard

Always check that the platform token is configured before making Platform API calls. If it is empty, log a warning and return gracefully -- do not fail the primary operation:

```python
if not self.platform_token:
    logger.warning("CHATWOOT_PLATFORM_TOKEN not configured -- skipping")
    return None  # or False
```

This makes the Platform API integration optional. Projects using Chatwoot SaaS (no Platform API) will simply skip these operations.
