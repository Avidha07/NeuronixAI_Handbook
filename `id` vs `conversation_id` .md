# `id` vs `conversation_id` (Quick Note)

## What is the difference?

| Column | Meaning |
|--------|---------|
| `id` | Unique ID of a **message** (Primary Key) |
| `conversation_id` | ID of the **conversation** that the message belongs to (Foreign Key) |

### Example

| id | conversation_id | role |
|----|-----------------|------|
| 7 | 4 | USER |
| 8 | 4 | ASSISTANT |

Both messages belong to **Conversation 4**.

## Why do we need `conversation_id`?

A conversation contains multiple messages.

```text
Conversation 4
 ├── Message 7
 └── Message 8
```

Without `conversation_id`, the database wouldn't know which messages belong to which chat.

## Which ID is used for Delete?

API:

```http
DELETE /api/v1/conversations/{conversationId}
```

Example:

```http
DELETE /api/v1/conversations/4
```

Use **`conversationId`**, not the message `id`.

## Easy Memory Trick

- **`id` → One Message**
- **`conversation_id` → One Chat (contains many messages)**
