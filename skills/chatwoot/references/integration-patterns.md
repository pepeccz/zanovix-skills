# Chatwoot Integration Patterns

Patterns extracted from a production Chatwoot integration. These are reusable across projects.

## 1. Config Settings Pattern

Use Pydantic Settings to centralize all Chatwoot configuration. Never use `os.getenv()` directly.

```python
from pydantic import Field
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    # Application API
    CHATWOOT_API_URL: str = Field(default="https://app.chatwoot.com")
    CHATWOOT_API_TOKEN: str = Field(default="")
    CHATWOOT_ACCOUNT_ID: str = Field(default="")
    CHATWOOT_INBOX_ID: str = Field(default="")

    # Platform API (self-hosted only)
    CHATWOOT_PLATFORM_TOKEN: str = Field(default="")

    # Webhook
    CHATWOOT_WEBHOOK_TOKEN: str = Field(default="")

    # Optional
    CHATWOOT_STORAGE_DOMAIN: str = Field(default="")
    CHATWOOT_TEAM_GROUP_ID: int | None = Field(default=None)
    CHATWOOT_IMAGE_SEND_DELAY_SECONDS: float = Field(default=5.0, ge=0.0)

    class Config:
        env_file = ".env"

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

**Why separate tokens**: The Application API token comes from a user's Profile Settings. The Platform API token comes from the Super Admin Console's Platform Apps section. They have different scopes and are generated differently.

## 2. ChatwootClient Class Pattern

Centralize all Chatwoot API calls in a single client class. Instantiate per-request (it reads settings on init).

```python
class ChatwootClient:
    def __init__(self):
        settings = get_settings()
        self.api_url = settings.CHATWOOT_API_URL.rstrip("/")
        self.api_token = settings.CHATWOOT_API_TOKEN
        self.platform_token = settings.CHATWOOT_PLATFORM_TOKEN
        self.account_id = settings.CHATWOOT_ACCOUNT_ID
        self.inbox_id = settings.CHATWOOT_INBOX_ID

        # Application API headers
        self.headers = {
            "api_access_token": self.api_token,
            "Content-Type": "application/json",
        }

    @property
    def platform_headers(self) -> dict[str, str]:
        """Platform API uses a different token but same header name."""
        return {
            "api_access_token": self.platform_token,
            "Content-Type": "application/json",
        }
```

**Why `@property` for `platform_headers`**: The platform token might be empty (SaaS deployments). Generating headers on access makes guard checks cleaner. Also, if you ever need to rotate tokens at runtime, the property pattern supports it.

## 3. Retry Decorator Pattern

Use `tenacity` for retries with exponential backoff. The critical rule: NEVER retry 422 responses.

```python
from tenacity import retry, retry_if_exception_type, stop_after_attempt, wait_exponential
import httpx

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10),
    retry=retry_if_exception_type(httpx.HTTPError),
    reraise=True,
)
async def some_api_call(self, ...):
    async with httpx.AsyncClient() as client:
        try:
            response = await client.post(url, json=payload, headers=self.headers, timeout=10.0)
            response.raise_for_status()
            return response.json()
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 422:
                # Permanent validation error -- log and return, do NOT retry
                logger.warning(f"Chatwoot 422: {e.response.text}")
                return None
            raise  # Transient errors bubble up to tenacity for retry
```

**Why no retry on 422**: A 422 means the data is invalid (duplicate email, missing required field, etc.). Retrying the exact same request will always fail. Log the response body to diagnose which field Chatwoot rejected.

**Retry timing**: With `multiplier=1, min=2, max=10`, retries happen at ~2s, ~4s, ~8s (capped at 10s). Three attempts total.

## 4. Fire-and-Forget Sync Pattern

Chatwoot sync should NEVER block the primary operation. Use `asyncio.create_task()` after the DB commit succeeds.

```python
import asyncio

# In your API route handler:
async def create_user(data: UserCreate, session: AsyncSession):
    # 1. Primary operation -- always succeeds regardless of Chatwoot
    user = User(**data.dict())
    session.add(user)
    await session.commit()

    # 2. Best-effort Chatwoot sync -- fire and forget
    if user.role == "agent" and user.email:
        asyncio.create_task(
            sync_agent_to_chatwoot(
                admin_user_id=user.id,
                action="create",
                display_name=user.display_name,
                email=user.email,
                password=data.password,  # Plaintext, in-memory only
            )
        )

    return user
```

**Why fire-and-forget**: If Chatwoot is down, the user creation should still succeed. The sync can be retried later via a manual resync endpoint.

**Critical**: The password is passed in-memory from the request to the sync task. It is NEVER stored in the database. After the sync task completes, the password is garbage collected.

## 5. DB Writeback Pattern (Separate Session)

When a fire-and-forget sync task needs to write back to the DB (e.g., storing the Chatwoot ID), it must use its own session. The original request's session is already closed.

```python
async def sync_agent_to_chatwoot(admin_user_id: uuid.UUID, action: str, **kwargs):
    """Best-effort sync -- never raises."""
    try:
        chatwoot = ChatwootClient()

        if action == "create":
            # Step 1: Create platform user
            platform_user = await chatwoot.create_platform_user(
                name=kwargs["display_name"],
                email=kwargs["email"],
                password=kwargs["password"],
            )
            if not platform_user:
                return False

            # Step 2: Link to account
            agent = await chatwoot.link_user_to_account(
                name=kwargs["display_name"],
                email=kwargs["email"],
            )

            # Step 3: Write IDs back to DB with a SEPARATE session
            from database.connection import get_async_session  # Lazy import
            from database.models import AdminUser

            async with get_async_session() as session:
                admin_user = await session.get(AdminUser, admin_user_id)
                if admin_user:
                    admin_user.chatwoot_user_id = platform_user["id"]
                    if agent:
                        admin_user.chatwoot_agent_id = agent["id"]
                    await session.commit()

            return True

    except Exception as e:
        logger.warning(f"Chatwoot sync failed: {e}", exc_info=True)
        return False
```

**Why lazy imports**: The sync module imports `database.connection` and `database.models` inside the function body, not at module level. This prevents circular imports since the sync module is often imported by both the API layer and the agent layer.

**Why separate session**: The original API request's session is scoped to the request lifecycle. By the time the `create_task` coroutine runs, that session may already be closed. The sync task creates its own session with `get_async_session()` (an async context manager).

## 6. Dual ID Storage Pattern

When using both Platform API and Application API, you need TWO Chatwoot IDs per entity:

```python
class AdminUser(Base):
    __tablename__ = "admin_users"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    email: Mapped[str | None]

    # Platform API ID -- for user lifecycle (update password, delete)
    chatwoot_user_id: Mapped[int | None] = mapped_column(default=None)

    # Application API ID -- for operational actions (assign conversation)
    chatwoot_agent_id: Mapped[int | None] = mapped_column(default=None)
```

**Why two IDs**:
- `chatwoot_user_id`: Returned by `POST /platform/api/v1/users`. Used for `PATCH/DELETE /platform/api/v1/users/{id}`. This is the platform-level identity.
- `chatwoot_agent_id`: Returned by `POST /api/v1/accounts/{account_id}/agents`. Used for `POST .../assignments {"assignee_id": agent_id}`. This is the account-level identity.

They are different numbers pointing to different Chatwoot internal objects.

## 7. Resync Endpoint Pattern

Always provide a manual resync endpoint for when fire-and-forget sync fails or Chatwoot was down during initial sync.

```python
@router.post("/admin-users/{user_id}/chatwoot-sync")
async def resync_admin_user_chatwoot(
    user_id: uuid.UUID,
    password: str | None = Body(None),  # Optional -- needed for create
    current_user = Depends(require_role("admin")),
):
    result = await resync_agent_to_chatwoot(user_id, password=password)

    if not result["success"]:
        raise HTTPException(
            status_code=400 if result["error"] else 502,
            detail={
                "error": result["error"],
                "action": result["action"],
                "chatwoot_agent_id": result["chatwoot_agent_id"],
                "chatwoot_user_id": result["chatwoot_user_id"],
            },
        )

    return {
        "message": f"Chatwoot sync successful (action: {result['action']})",
        "chatwoot_agent_id": result["chatwoot_agent_id"],
        "chatwoot_user_id": result["chatwoot_user_id"],
    }
```

The resync function determines the action automatically:
- Has `chatwoot_user_id` --> update
- Has neither ID --> create (requires password)
- Has `chatwoot_agent_id` but no `chatwoot_user_id` --> legacy link, cannot manage via Platform API

**Why password is optional**: For updates, the password is not needed (only name/email sync). For creates, the Platform API requires a password, so the admin must provide it.

## 8. Contact Sync Pattern (Simpler)

For syncing customer contacts (not agents), the pattern is simpler -- no Platform API involved:

```python
async def sync_contact_to_chatwoot(contact, save_contact_id: bool = False) -> bool:
    """Best-effort sync of contact data to Chatwoot."""
    chatwoot = ChatwootClient()
    chatwoot_id = contact.chatwoot_contact_id

    # Discover contact by phone if not stored
    if not chatwoot_id:
        found = await chatwoot.find_contact_by_phone(contact.phone)
        if found:
            chatwoot_id = found["id"]
            if save_contact_id:
                contact.chatwoot_contact_id = chatwoot_id  # Caller commits

    if not chatwoot_id:
        return False  # No matching Chatwoot contact

    # Sync fields — adapt to your model's fields
    await chatwoot.update_contact(
        contact_id=chatwoot_id,
        name=contact.full_name or None,
        email=contact.email or None,  # Never send empty string
        custom_attributes={"role": contact.role},  # Your custom attributes
    )
    return True
```

**Key differences from agent sync**:
- Uses Application API only (no Platform API)
- Contact is found by phone number, not created
- No password involved
- `save_contact_id=True` allows the sync to discover and store the Chatwoot contact ID
- Adapt the synced fields and `custom_attributes` to your domain model

## 9. Best-Effort Error Handling Philosophy

Every sync function follows the same pattern:

1. **Never raise exceptions** -- catch everything at the top level
2. **Return `bool`** -- `True` for success, `False` for any failure
3. **Log warnings, not errors** -- Chatwoot being down is not a system error, it is an expected degradation
4. **Include `exc_info=True`** -- stack traces help debug rare issues
5. **Structured logging** -- always include entity IDs as `extra` fields

```python
async def sync_something(...) -> bool:
    try:
        # ... sync logic ...
        return True
    except Exception as e:
        logger.warning(
            f"Failed to sync {entity_id}: {e}",
            extra={"entity_id": str(entity_id)},
            exc_info=True,
        )
        return False
```

## 10. Password Sync Pattern

When your app manages user passwords and needs to keep them in sync with Chatwoot:

1. **Capture plaintext** in the API request handler (before hashing)
2. **Pass to sync task** in-memory via `asyncio.create_task()`
3. **Forward to Platform API** in the sync function
4. **Never store** the plaintext password anywhere

```python
# In route handler:
hashed = bcrypt.hash(data.password)
admin_user.password_hash = hashed
await session.commit()

# Fire-and-forget -- password passed in memory
asyncio.create_task(
    sync_agent_to_chatwoot(
        admin_user_id=admin_user.id,
        action="update",
        password=data.password,  # Plaintext, in-memory only
        chatwoot_user_id=admin_user.chatwoot_user_id,
    )
)
```

The password exists in memory only during the request lifecycle and the sync task's execution. After both complete, it is garbage collected. No persistence, no logs.

## 11. Chatwoot Agent Listing for Admin Panel

To show Chatwoot agents in your admin panel (for manual linking), combine Chatwoot's agent list with your DB to mark which ones are already linked:

```python
@router.get("/chatwoot-agents")
async def list_chatwoot_agents(session: AsyncSession = Depends(get_session)):
    chatwoot = ChatwootClient()
    agents = await chatwoot.list_agents()

    # Get already-linked agent IDs from DB
    result = await session.execute(
        select(AdminUser.chatwoot_agent_id).where(
            AdminUser.chatwoot_agent_id.is_not(None)
        )
    )
    linked_ids = {row[0] for row in result.all()}

    return [
        {**agent, "linked": agent["id"] in linked_ids}
        for agent in agents
    ]
```

This lets admins see which Chatwoot agents are already linked to app users and which are available for linking.
