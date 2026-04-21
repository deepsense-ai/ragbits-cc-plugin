# New Auth Checklist

Quality gates for adding authentication to the generated chat application.

## Dependency

Auth ships with `ragbits-chat`. No extra is required beyond `"ragbits-chat"` in `pyproject.toml`.

## Imports

```python
from ragbits.chat.auth import ListAuthenticationBackend
from ragbits.chat.auth.session_store import InMemorySessionStore
```

Richer options live in `ragbits.chat.auth.backends` — `MultiAuthenticationBackend`, `OAuth2AuthenticationBackend` (Discord, Google, etc.). Surface them when the user asks, but keep the scaffolded default simple.

## Backend Factory

Generate a `get_auth_backend()` helper that returns a `ListAuthenticationBackend` with two seed users — one admin, one regular. The template is deliberately minimal; real deployments should swap it for an OAuth backend or an external user store.

```python
def get_auth_backend() -> ListAuthenticationBackend:
    """Return a simple list-based auth backend with test credentials."""
    users = [
        {
            "user_id": "1",
            "username": "admin",
            "password": "admin123",
            "email": "admin@example.com",
            "full_name": "Admin User",
            "roles": ["admin"],
        },
        {
            "user_id": "2",
            "username": "user",
            "password": "user123",
            "email": "user@example.com",
            "full_name": "Regular User",
            "roles": ["user"],
        },
    ]
    return ListAuthenticationBackend(users, session_store=InMemorySessionStore())
```

The `InMemorySessionStore` loses sessions on process restart. Swap for a persistent session store (Redis, database) before going to production.

## Wiring into `RagbitsAPI`

Script entrypoint:

```python
if __name__ == "__main__":
    from ragbits.chat.api import RagbitsAPI

    RagbitsAPI(Chat, auth_backend=get_auth_backend()).run()
```

CLI entrypoint (the preferred launcher):

```bash
ragbits api run {app_name_snake}.app:Chat --auth {app_name_snake}.app:get_auth_backend
```

## Accessing the User in `chat()`

Authenticated sessions expose the user via `ChatContext`:

```python
async def chat(self, message: str, history: ChatFormat, context: ChatContext):
    user = context.user  # type from ragbits.chat.auth
    # user.username, user.roles, user.email, ...
```

Use this for role-gated replies, per-user context, or personalization. Avoid exposing sensitive user fields back through the LLM prompt — pass only what the answer needs.

## README Notes

Include in the generated README:

- That auth is enabled
- The two seed accounts (`admin`/`admin123`, `user`/`user123`) with a **warning** that they are for local testing only
- How to launch with and without the `--auth` flag
- Where to customize the user list or swap to OAuth

## Validation Checklist

- [ ] `get_auth_backend()` is defined before the `__main__` block
- [ ] `RagbitsAPI(...)` receives `auth_backend=get_auth_backend()` (the call, not the function reference)
- [ ] `InMemorySessionStore` is instantiated, not passed as a class
- [ ] README flags the seed credentials as demo-only
- [ ] `pyproject.toml` lists `ragbits-chat`
- [ ] If roles are used to gate behavior in `chat()`, the README documents which roles unlock which features
