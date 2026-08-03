Since your conversations live as LangGraph threads in Mongo already, the cleanest approach is to **not touch the checkpointer collection at all** — keep favorites/folders as pure metadata in a separate collection keyed by `thread_id`. That keeps LangGraph's state management untouched and gives you a clean read/write path for UI features.

**Schema**
```python
# collection: conversation_metadata
{
    "_id": ObjectId(...),
    "thread_id": "abc-123",          # FK to LangGraph checkpoint thread
    "user_id": "u_9982",
    "title": "Q3 Client Onboarding RAG",   # auto-generated or user-renamed
    "favorite": False,
    "folder_id": "folder_456",       # nullable — null = uncategorized
    "tags": ["infomax", "urgent"],
    "created_at": "...",
    "last_active_at": "...",
    "pinned_order": None             # for manual favorite ordering
}

# collection: folders
{
    "_id": "folder_456",
    "user_id": "u_9982",
    "name": "Active Clients",
    "parent_folder_id": None,        # supports nesting if you want it
    "created_at": "..."
}
```

**Why a separate collection instead of stuffing this into checkpoint state**
- The checkpointer's job is agent *execution* state (messages, todos, VFS pointers) — mixing UI concerns into it means every favorite/rename touches your LangGraph persistence layer and risks colliding with checkpoint writes.
- This mirrors the separation you're already using with CompositeBackend (scratch vs. durable) — same principle: keep orchestration state and application/UI state on separate lanes.

**API shape (FastAPI)**
```python
POST   /conversations/{thread_id}/favorite      # toggle
POST   /folders                                  # create folder
PATCH  /conversations/{thread_id}                # move to folder, rename, tag
GET    /conversations?folder_id=456&favorite=true&user_id=u_9982
```

**Auto-title generation**
Since you already run a Research/Planner agent for todo decomposition, reuse the same LLM call pattern: on first user message (or thread completion), fire a cheap async call to summarize the thread into a 4-6 word title, write it to `conversation_metadata.title`. Don't block the main chat response on this — do it as a background task (`asyncio.create_task` or a lightweight queue job) so it doesn't add latency to the user-facing turn.

**Listing/search performance at your scale**
For tens of thousands of users each with potentially hundreds of threads:
- Index `conversation_metadata` on `(user_id, folder_id)` and `(user_id, favorite)` — these are your two hot-path queries (folder view, favorites view).
- Don't join against the checkpoint collection for list views — the metadata collection alone should be enough to render the sidebar. Only pull from the checkpointer when a specific thread is opened.

**One gotcha**: if you ever delete a thread's checkpoint (cleanup job, TTL expiry), make sure you cascade-delete or orphan-flag the corresponding `conversation_metadata` doc — otherwise your UI shows favorited conversations that 404 when opened.

Want me to sketch the actual FastAPI router + Pydantic models for this, or the Mongo aggregation pipeline for the folder-view query?



Since your conversations live as LangGraph threads in Mongo already, the cleanest approach is to **not touch the checkpointer collection at all** — keep favorites/folders as pure metadata in a separate collection keyed by `thread_id`. That keeps LangGraph's state management untouched and gives you a clean read/write path for UI features.

**Schema**
```python
# collection: conversation_metadata
{
    "_id": ObjectId(...),
    "thread_id": "abc-123",          # FK to LangGraph checkpoint thread
    "user_id": "u_9982",
    "title": "Q3 Client Onboarding RAG",   # auto-generated or user-renamed
    "favorite": False,
    "folder_id": "folder_456",       # nullable — null = uncategorized
    "tags": ["infomax", "urgent"],
    "created_at": "...",
    "last_active_at": "...",
    "pinned_order": None             # for manual favorite ordering
}

# collection: folders
{
    "_id": "folder_456",
    "user_id": "u_9982",
    "name": "Active Clients",
    "parent_folder_id": None,        # supports nesting if you want it
    "created_at": "..."
}
```

**Why a separate collection instead of stuffing this into checkpoint state**
- The checkpointer's job is agent *execution* state (messages, todos, VFS pointers) — mixing UI concerns into it means every favorite/rename touches your LangGraph persistence layer and risks colliding with checkpoint writes.
- This mirrors the separation you're already using with CompositeBackend (scratch vs. durable) — same principle: keep orchestration state and application/UI state on separate lanes.

**API shape (FastAPI)**
```python
POST   /conversations/{thread_id}/favorite      # toggle
POST   /folders                                  # create folder
PATCH  /conversations/{thread_id}                # move to folder, rename, tag
GET    /conversations?folder_id=456&favorite=true&user_id=u_9982
```

**Auto-title generation**
Since you already run a Research/Planner agent for todo decomposition, reuse the same LLM call pattern: on first user message (or thread completion), fire a cheap async call to summarize the thread into a 4-6 word title, write it to `conversation_metadata.title`. Don't block the main chat response on this — do it as a background task (`asyncio.create_task` or a lightweight queue job) so it doesn't add latency to the user-facing turn.

**Listing/search performance at your scale**
For tens of thousands of users each with potentially hundreds of threads:
- Index `conversation_metadata` on `(user_id, folder_id)` and `(user_id, favorite)` — these are your two hot-path queries (folder view, favorites view).
- Don't join against the checkpoint collection for list views — the metadata collection alone should be enough to render the sidebar. Only pull from the checkpointer when a specific thread is opened.

**One gotcha**: if you ever delete a thread's checkpoint (cleanup job, TTL expiry), make sure you cascade-delete or orphan-flag the corresponding `conversation_metadata` doc — otherwise your UI shows favorited conversations that 404 when opened.

Want me to sketch the actual FastAPI router + Pydantic models for this, or the Mongo aggregation pipeline for the folder-view query?


