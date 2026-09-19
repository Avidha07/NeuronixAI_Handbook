# ConversationController – Quick API Guide

**Base URL:** `http://localhost:8080/api/v1`

## Step 1: Login (Get JWT Token)

**POST** `/auth/login`

**Body**

```json
{
  "email": "your-email@example.com",
  "password": "your-password"
}
```

**Response**

```json
{
  "accessToken": "eyJhbGciOi..."
}
```

> Copy the `accessToken` and use it as **Bearer Token** for all APIs below.

---

## APIs

### 1. Get All Conversations

**GET** `/conversations`

**Authorization:** `Bearer <accessToken>`

**Response:** `200 OK`

Returns all conversations of the logged-in user.

---

### 2. Get Conversation Messages

**GET** `/conversations/{conversationId}/messages`

Example:

```http
GET /conversations/4/messages
```

**Authorization:** `Bearer <accessToken>`

**Response:** `200 OK`

Returns all messages of Conversation **4**.

---

### 3. Delete Conversation

**DELETE** `/conversations/{conversationId}`

Example:

```http
DELETE /conversations/4
```

**Authorization:** `Bearer <accessToken>`

**Success:** `200 OK`

```text
deletion successful
```

If the conversation doesn't exist, it returns **404 Not Found**.

---

## Quick Flow

```text
Login
  ↓
Copy accessToken
  ↓
Authorization → Bearer Token
  ↓
GET /conversations
GET /conversations/{id}/messages
DELETE /conversations/{id}
```
