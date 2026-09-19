# Chat API Flow (NeuronixAI)

## API Flow

1. Run the Spring Boot application.
2. `POST /api/v1/auth/login` → Get the Bearer Token.
3. `POST /api/v1/chat` → Send the first message.
4. The backend creates a new conversation (if `conversationId` is `null`) and stores the message.
5. `GET /api/v1/conversations` → Get the created conversation.
6. `GET /api/v1/conversations/{conversationId}/messages` → View its messages.

## Postman Request

**Method:** `POST`

**URL**

`http://localhost:8080/api/v1/chat`

**Authorization:** Bearer Token

**Headers**

`Content-Type: application/json`

### New Conversation

```json
{
  "conversationId": null,
  "message": "Hello AI"
}
```

### Existing Conversation

```json
{
  "conversationId": 1,
  "message": "Continue this conversation."
}
```

## How `conversationId` Works

- `conversationId = null` → Creates a new conversation.
- `conversationId = existing ID` → Adds the message to that conversation.
