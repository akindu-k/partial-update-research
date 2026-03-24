# Race Condition Handling

Beanie's change tracking uses **optimistic locking** via the **revision system** to handle race conditions:

## The Problem

Two concurrent instances could modify the same document:

```python
# Process A
doc_a = await MyDocument.get(id="123")
doc_a.name = "Alice"

# Process B  
doc_b = await MyDocument.get(id="123")  
doc_b.name = "Bob"

# Both call save_changes() - conflict!
```

## The Solution: Revision ID

When enabled, each document tracks a `revision_id`:

```python
use_revision_id = self.get_settings().use_revision

if use_revision_id and not ignore_revision:
    find_query["revision_id"] = self.revision_id  # ← Include in WHERE clause
    
if use_revision_id:
    new_revision_id = uuid4()
    arguments.append(SetRevisionId(new_revision_id))  # ← Increment on save
```

## How It Works

1. **Each load** captures the document's current `revision_id`
2. **MongoDB update** filters by both `_id` AND `revision_id`:
   ```python
   db.collection.update_one(
       {"_id": "123", "revision_id": "abc-def"},  # must match BOTH
       {"$set": {...}, "$set": {"revision_id": "new-uuid"}}
   )
   ```
3. **If another process saved first**, the `revision_id` no longer matches → **0 documents updated**
4. **Conflict detection**:
   ```python
   try:
       result = await self.find_one(find_query).update(...)
   except DuplicateKeyError:
       if use_revision_id and not ignore_revision:
           raise  # Revision conflict!
       raise
   ```

## Instance-Level State Safety

Each concurrent instance has **its own `_saved_state`**:

```python
# Process A's memory
_saved_state = {"name": "Original"}

# Process B's memory (separate instance)
_saved_state = {"name": "Original"}  # Independent snapshot
```

No shared memory issues between concurrent requests.

## Limitations & Trade-offs

| Aspect | Behavior |
|--------|----------|
| **What's protected** | Prevents conflicting writes; detects stale data |
| **Failed updates** | Raises `DuplicateKeyError` – caller must handle retry logic |
| **Last-write-wins** | Without revision checking (`ignore_revision=True`), later write overwrites earlier |
| **Lost updates** | Possible if `ignore_revision=True` is used |

## Example: Handling Conflicts

```python
try:
    await doc.save_changes()
except DuplicateKeyError:
    # Reload and retry
    doc = await MyDocument.get(id=doc.id)
    doc.name = "New value"
    await doc.save_changes()
```

**Note**: This is **optimistic, not pessimistic** locking — no database locks are held. Conflicts are detected at commit time rather than preventing concurrent access.