# Why `conversationId` Starts from 4 (Neon PostgreSQL)

Even if all conversations are deleted, PostgreSQL does **not** reset the ID sequence automatically.

Example:

| Action | ID |
|--------|----|
| First conversation | 1 |
| Second conversation | 2 |
| Third conversation | 3 |
| Delete all conversations | Table becomes empty |
| Create new conversation | **4** |

## Why?

- `DELETE` removes rows only.
- The sequence (`SERIAL`/`BIGSERIAL`) remembers the last used ID.
- IDs are meant to be **unique**, not necessarily continuous.

## Reset IDs (Testing Only)

If the table is empty, reset the sequence:

```sql
ALTER SEQUENCE conversation_id_seq RESTART WITH 1;
```

Or reset both data and IDs together:

```sql
TRUNCATE TABLE conversation RESTART IDENTITY;
```

## Quick Note

- `DELETE` → Removes rows, sequence stays the same.
- `TRUNCATE ... RESTART IDENTITY` → Removes rows and resets IDs.
