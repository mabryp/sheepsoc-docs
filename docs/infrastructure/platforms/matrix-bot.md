# Matrix Bot (sheepsoc-bot)

**Purpose:** E2EE Matrix bot that bridges Element chat rooms to OpenWebUI RAG and Ollama LLM inference.

| Key | Value |
|---|---|
| Added | 2026-04-22 |
| Status | Running — `matrix-bot.service` |
| Bot Matrix ID | `@sheepsoc-bot:matrix.pmabry.com` |
| Homeserver | `matrix.pmabry.com` (Synapse) |
| Default model | `gemma3:12b` via Ollama |
| Conda env | `matrixbot` — at `~/infrastructure/miniconda3/envs/matrixbot/` |

## Overview

The Matrix bot, named **sheepsoc-bot**, is a systemd-managed Python service that joins Matrix rooms and responds to @mentions by routing the user's question through OpenWebUI's chat completions API. OpenWebUI in turn performs an Elasticsearch RAG lookup and sends the retrieved context along with the question to an Ollama LLM for a final answer.

This gives any user in an authorized Element room a conversational interface to the same knowledge bases available in OpenWebUI — without needing to open a browser. Different rooms can be mapped to different OpenWebUI Knowledge Base collections, so a room dedicated to one topic area always queries only that collection.

## Key Facts

| Key | Value |
|---|---|
| systemd unit | `matrix-bot.service` — enabled at boot, runs as `pmabry` |
| Matrix library | `matrix-nio[e2e]` — full E2EE support |
| OpenWebUI API | `http://localhost:8080/api/chat/completions` — JWT auth, bot account has admin role |
| OpenWebUI user | `sheepsoc@pmabry.com` — must remain admin in OpenWebUI |

## How It Works

When a message arrives in a room the bot has joined, it checks whether the message contains an @mention of the bot (by display name `sheepsoc-bot` or by full MXID `@sheepsoc-bot:matrix.pmabry.com`). If a match is found, it:

1. Looks up the room's entry in `config.yaml` to find the associated OpenWebUI Knowledge Base collection ID (or `null` for plain LLM with no RAG context).
2. Sends the user's message to `http://localhost:8080/api/chat/completions` with the collection ID included in the request payload.
3. OpenWebUI retrieves relevant document chunks from Elasticsearch using `nomic-embed-text` (768-dimensional cosine similarity), injects them as context, and queries Ollama (`gemma3:12b` by default) for a response.
4. The bot posts OpenWebUI's response back into the Matrix room.

All communication between the bot and Matrix is end-to-end encrypted. The bot manages its own E2EE device keys, stored persistently in the crypto store directory.

## Room Configuration

Each room the bot participates in is configured via a named entry in `~/repositories/matrix-bot/config.yaml` under the `rooms:` key. The key for each entry is the Matrix internal room ID (not the human-readable alias).

```yaml
# Example entry in config.yaml
rooms:
  "!exampleRoomId:matrix.pmabry.com":
    collection_id: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    mention_only: true
```

| Field | Description |
|---|---|
| `collection_id` | UUID of the OpenWebUI Knowledge Base to use for RAG — set to `null` to disable RAG and use plain LLM |
| `mention_only` | When `true`, the bot responds only when @sheepsoc-bot appears in the message body. Recommended for all rooms. |

The bot auto-accepts room invites. A room does not need to be in `config.yaml` for the bot to join — but if it is not listed, the bot will use plain LLM inference with no RAG context.

## Config Schema Reference

Every field recognized by `bot.py` is documented below. The config is loaded once at startup from `~/repositories/matrix-bot/config.yaml`; changes require a service restart to take effect.

### matrix section

| Field | Type | Description |
|---|---|---|
| `homeserver` | string (URL) | Full HTTPS URL of the Matrix homeserver. Example: `https://matrix.pmabry.com` |
| `username` | string (MXID) | Fully-qualified Matrix user ID of the bot account. Example: `@sheepsoc-bot:matrix.pmabry.com`. The localpart is also used as the bot's display name. |
| `password` | string | Bot account password. Used only on the very first run to obtain an access token. After that, `session.json` is used for authentication. |
| `device_name` | string | Friendly label for this Matrix device registration, visible in Element's device list. |
| `store_path` | string (path) | Directory where matrix-nio stores E2EE cryptographic state. Default: `./crypto_store`. |
| `session_file` | string (path) | Path to the JSON file that persists the Matrix access token between restarts. Default: `./session.json`. |

### openwebui section

| Field | Type | Description |
|---|---|---|
| `url` | string (URL) | Base URL of the OpenWebUI instance. Example: `http://localhost:8080` |
| `email` | string | Email address of the OpenWebUI account the bot authenticates as. Must be an **admin** account. Currently: `sheepsoc@pmabry.com` |
| `password` | string | Password for the OpenWebUI account. Used to obtain a JWT via `/api/v1/auths/signin`. The JWT is managed automatically and refreshed before expiry. |
| `model` | string | Ollama model identifier passed in every chat completions request. Default: `gemma3:12b`. |

### rooms section

Each key under `rooms:` is a Matrix internal room ID (format: `!opaque_string:matrix.pmabry.com`). The value is a mapping with the following fields:

| Field | Type | Description |
|---|---|---|
| `name` | string | Human-readable label for this room. Used only in log output. Has no functional effect. |
| `collection_id` | string (UUID) or null | UUID of the OpenWebUI Knowledge Base collection to use for RAG retrieval. Set to `null` to skip RAG. Obtain the UUID from OpenWebUI: Workspace → Knowledge → click the collection → copy the UUID from the browser URL. |
| `mention_only` | boolean | When `true`, the bot only responds when the message body contains the bot's MXID, localpart, or formatted HTML mention. Recommended: leave as `true`. |

!!! note "Unlisted Rooms"
    A room does not need to appear in `config.yaml` for the bot to join it — the bot auto-accepts all invites. However, unlisted rooms receive no RAG context and behave as if `mention_only: true` and `collection_id: null`.

## Add a New Room (SOP)

Follow these steps to connect a new Matrix room to a specific Knowledge Base collection.

1. **Invite the bot.** In Element, open the room, open the member panel, and invite `@sheepsoc-bot:matrix.pmabry.com`. The bot will auto-accept.
2. **Get the internal room ID.** In Element: Room Settings → Advanced → "Internal room ID". It looks like `!AbcXyz123:matrix.pmabry.com`.
3. **Get the Knowledge Base collection ID.** In OpenWebUI: Workspace → Knowledge → click the collection → copy the UUID from the browser URL bar.
4. **Add the entry to config.yaml.**
    ```bash
    pmabry@sheepsoc:~$ vim ~/repositories/matrix-bot/config.yaml
    ```
    Add a block under `rooms:` as shown in the Room configuration section above.
5. **Restart the bot.**
    ```bash
    pmabry@sheepsoc:~$ sudo systemctl restart matrix-bot
    ```

## Files and Directories

| Path | Purpose |
|---|---|
| `~/repositories/matrix-bot/bot.py` | Main bot process — Matrix event loop, message handling, OpenWebUI API calls |
| `~/repositories/matrix-bot/config.yaml` | Homeserver credentials, bot account details, room → collection mappings |
| `~/repositories/matrix-bot/session.json` | Matrix access token — **do not delete** (forces re-registration, breaks E2EE for past messages) |
| `~/repositories/matrix-bot/crypto_store/` | E2EE device keys — **do not delete** (same consequence as session.json deletion) |
| `~/infrastructure/miniconda3/envs/matrixbot/` | Conda environment with `matrix-nio[e2e]` and its dependencies |

!!! danger "Do Not Delete"
    Deleting `session.json` or the `crypto_store/` directory will force the bot to register as a new Matrix device. Other room members will see security warnings ("new unverified device") and past E2EE messages will become unreadable from that device. If either file goes missing, you will need to verify the bot's new device in Element before normal E2EE operation resumes.

## Start / Stop / Restart

```bash
# Check status
pmabry@sheepsoc:~$ systemctl status matrix-bot.service

# Restart (required after editing config.yaml)
pmabry@sheepsoc:~$ sudo systemctl restart matrix-bot.service

# Stop
pmabry@sheepsoc:~$ sudo systemctl stop matrix-bot.service

# Start
pmabry@sheepsoc:~$ sudo systemctl start matrix-bot.service
```

## Reading Logs

```bash
# Most recent 50 lines
pmabry@sheepsoc:~$ journalctl -u matrix-bot.service -n 50

# Follow live output
pmabry@sheepsoc:~$ journalctl -u matrix-bot.service -f

# Since last restart
pmabry@sheepsoc:~$ journalctl -u matrix-bot.service --since "10 min ago"
```

## Code Architecture

The bot is a single-file Python asyncio application at `~/repositories/matrix-bot/bot.py`. This section describes each major component in enough detail to maintain or debug the bot without re-reading the source.

### OpenWebUIClient Class

`OpenWebUIClient` encapsulates all HTTP communication with the OpenWebUI API. It maintains a single `aiohttp.ClientSession` for the lifetime of the process.

**JWT authentication and auto-refresh:** On initialization, no token is held. The `authenticate()` coroutine POSTs credentials to `/api/v1/auths/signin` and stores the returned JWT. The token expiry is computed with a 60-second pre-expiry buffer — the token is treated as expired one minute before the server-side expiry. If the server does not return an `expires_at` field, a default of one hour from now is used.

**chat() method and 401 retry logic:** `chat(message, collection_id)` checks whether a valid token exists before proceeding. The method POSTs to `/api/chat/completions` with the message, model, and (if provided) the Knowledge Base collection ID. The request timeout is 120 seconds to accommodate long LLM inference times. If the server returns HTTP 401, the method calls `authenticate()` inline and immediately retries once with the new token.

### Session Persistence

matrix-nio's `AsyncClient.login()` registers a new Olm device with the homeserver every time it is called. To avoid device proliferation, `bot.py` uses the following logic at startup:

1. If `session.json` exists: call `client.restore_login()` with the saved `user_id`, `device_id`, and `access_token`. The bot resumes as the same device.
2. If `session.json` does not exist: call `client.login()` with the password, then write the resulting credentials to `session.json`. This is a one-time first-run path.

!!! danger "Critical"
    Deleting `session.json` forces the bot through the first-run path again, which registers an entirely new Olm device. The new device cannot decrypt any Megolm session keys that were shared with the old device. All previously exchanged E2EE messages will be permanently unreadable. Do not delete `session.json` unless you are intentionally re-registering the bot.

### Mention Detection

When `mention_only: true` is set for a room, the bot only processes a message if any one of the following three conditions is true:

1. The plain-text message body contains the bot's full MXID: `@sheepsoc-bot:matrix.pmabry.com`
2. The plain-text message body contains the bot's localpart: `sheepsoc-bot`
3. The HTML-formatted message body contains the bot's full MXID

Condition 2 exists because Element formats @mention pills using the user's display name, not the localpart. When the display name is set to the localpart, the plain-text body of a mention reads as `"sheepsoc-bot: hello"`. The localpart check catches this case.

### Event Callbacks

| Callback Name | Trigger Event | Behavior |
|---|---|---|
| `on_message` | `RoomMessageText` | Ignores messages from the bot itself. If `mention_only` is set and the message does not mention the bot, ignores the message. Otherwise strips mention tokens, calls `owui.chat()`, and posts the response back to the room. The `ignore_unverified_devices=True` flag is passed so the bot does not refuse to send if there are unverified devices in the room. |
| `on_invite` | `InviteMemberEvent` | Automatically accepts all room invites. There is no allowlist — any user who can invite on the homeserver can add the bot to a room. |
| `on_undecrypted` | `MegolmEvent` | Logs a warning when an encrypted message cannot be decrypted. Diagnostic only. Common causes: E2EE session not yet established, or `session.json` / `crypto_store/` was deleted. |

## Dependencies

### Services

- [OpenWebUI & RAG](openwebui-rag.md) — the bot routes every @mention through OpenWebUI's chat completions API (`/api/chat/completions`); OpenWebUI must be running for the bot to answer
- [Ollama](../services.md) — provides the LLM inference that OpenWebUI calls on behalf of the bot; default model is `gemma3:12b`
- [Elasticsearch & ELSER](elasticsearch-elser.md) — OpenWebUI depends on Elasticsearch as its vector store; if ES is down, RAG context retrieval will fail and bot answers will lack document context
- [Conda](conda.md) — the `matrixbot` conda environment (Python 3.11) is the runtime for `matrix-bot.service`

### System Packages (apt)

`libolm` is the C library that implements the Olm/Megolm E2EE protocol. matrix-nio's E2EE support is a Python binding to this library. These must be installed on the host OS before the Python packages can be installed.

```bash
pmabry@sheepsoc:~$ sudo apt install libolm3 libolm-dev
```

### Conda Environment

The bot runs in the `matrixbot` conda environment (Python 3.11). To recreate it from scratch if needed:

```bash
# Create the environment with Python 3.11
pmabry@sheepsoc:~$ conda create -n matrixbot python=3.11

# Activate it (this switches your shell into the matrixbot environment)
pmabry@sheepsoc:~$ conda activate matrixbot

# Install the required Python packages
(matrixbot) pmabry@sheepsoc:~$ pip install "matrix-nio[e2e]" aiohttp pyyaml
```

The `[e2e]` extras selector on `matrix-nio` is mandatory — it installs the Python bindings for `libolm` that enable E2EE. Installing plain `matrix-nio` without the extras will produce a bot that cannot decrypt or send encrypted messages.

### Python Package Summary

| Package | Purpose |
|---|---|
| `matrix-nio[e2e]` | Async Matrix client library with full E2EE support (Olm/Megolm). The `[e2e]` extras are required. |
| `aiohttp` | Async HTTP client used by `OpenWebUIClient` for all calls to the OpenWebUI API. |
| `pyyaml` | YAML parser used to load `config.yaml` at startup. |

## Troubleshooting

| Symptom | Check / Fix |
|---|---|
| Bot does not respond at all | Check service is running: `systemctl status matrix-bot.service`. Check logs: `journalctl -u matrix-bot.service -n 50`. Verify the homeserver `matrix.pmabry.com` is reachable. |
| Bot stops decrypting messages | Confirm `~/repositories/matrix-bot/session.json` and `~/repositories/matrix-bot/crypto_store/` still exist and are non-empty. If either is missing, the bot has lost its device identity — stop the service and restore from backup if possible. |
| OpenWebUI auth fails (HTTP 401/403) | The bot manages its JWT automatically — a stale token is not the cause. Check: (1) `sheepsoc@pmabry.com` must retain admin role in OpenWebUI (Admin Panel → Users); user role cannot use the chat API. (2) Confirm the password in `config.yaml` under `openwebui.password` is correct. |
| Bot responds in unencrypted rooms but not encrypted ones | The `matrixbot` conda environment must have `matrix-nio[e2e]` installed (not plain `matrix-nio`). Check: `~/infrastructure/miniconda3/envs/matrixbot/bin/pip show matrix-nio` — look for `python-olm` in the dependencies. |
| RAG returns no context (answers seem generic) | Confirm the room's `collection_id` in `config.yaml` is the correct UUID. Open OpenWebUI → Workspace → Knowledge and cross-check. Also verify Elasticsearch is up: `curl -u elastic:<password> http://localhost:9200/_cluster/health`. |
| Bot joined a room but is under wrong ID in config | The room ID in `config.yaml` must be the internal ID starting with `!`, not the human-readable alias. Retrieve the correct ID from Element: Room Settings → Advanced → Internal room ID. |

## Known Issues and Lessons Learned

### Device Proliferation on Every Restart

**Symptom:** The bot's Matrix account accumulated a new Olm device entry after every service restart. Room members saw repeated "new unverified device" security warnings.

**Cause:** The initial implementation called `client.login()` unconditionally at startup. Every call to `login()` registers a brand-new device with the homeserver.

**Fix:** Session persistence via `session.json`. On all restarts after the first, the bot calls `client.restore_login()` with the saved credentials instead of `login()`.

### @mention Detection Not Triggering

**Symptom:** Messages with an @mention in Element did not cause the bot to respond, even though the bot was in the room and the service was running.

**Cause:** Element formats @mention pills using the user's display name, not their Matrix localpart. If the bot's display name had not been set correctly, the plain-text body of a mention would not match the MXID or localpart checks.

**Fix:** Two changes were made together: (1) the bot calls `client.set_displayname(bot_localpart)` at startup to ensure the display name is always `sheepsoc-bot`; (2) the mention check was expanded to also match `bot_localpart` in the plain message body.

### OlmUnverifiedDeviceError on First Send After Setup

**Symptom:** The bot would receive messages and call OpenWebUI successfully but fail to send the response back to the room, logging an `OlmUnverifiedDeviceError`.

**Cause:** During initial debugging, several abandoned device registrations had accumulated on the bot's account. These phantom devices were still listed by the homeserver as participants in the E2EE room sessions. matrix-nio refused to encrypt for unverified devices by default.

**Fix:** Abandoned device registrations were deleted via the Matrix API, and `ignore_unverified_devices=True` was added to every `client.room_send()` call.

### OpenWebUI Model List Empty for Non-Admin Accounts

**Symptom:** The bot authenticated to OpenWebUI successfully but the chat completions endpoint returned no models or empty responses.

**Cause:** The `sheepsoc@pmabry.com` account had been inadvertently demoted from admin to user role in OpenWebUI. The user role cannot see the model list and cannot use the chat API effectively.

**Fix:** The account must retain **admin** role in OpenWebUI. To verify or restore this, log in to OpenWebUI as an admin, go to Admin Panel → Users, and confirm the role column shows "admin" for `sheepsoc@pmabry.com`.

!!! warning "Do Not Demote"
    Do not change the `sheepsoc@pmabry.com` OpenWebUI account to user role. The user role cannot list models or use the chat completions API. The bot will silently fail to retrieve answers if the account is demoted.

## Caveats and Warnings

!!! note "JWT Auth"
    The bot authenticates to OpenWebUI using a JWT obtained at startup and refreshed automatically before expiry. The JWT itself is not stored in `config.yaml` — only the email and password are. Persistent 401 errors indicate a wrong password or a demoted account role, not a stale token.

!!! note "Model Note"
    The default model is `gemma3:12b`. To change it, update the model setting in `config.yaml` and restart the service. The model must be present in Ollama — verify with `ollama list`.

!!! note "Invite Behavior"
    The bot auto-accepts all room invites. It does not enforce an allowlist of rooms by default. Any user who can invite on the homeserver could add the bot to a room. If this is a concern, add an invite filter to `bot.py`.

## See Also

- [Knowledge Bases](knowledge-bases.md) — catalog of all OpenWebUI Knowledge Base UUIDs; use these to configure room → collection mappings in `config.yaml`
- [Services](../services.md) — service catalog entry for `matrix-bot.service`
- [Shutdown & Startup](../runbooks/shutdown-startup.md) — includes the Matrix bot in the planned shutdown and post-boot verification sequences
