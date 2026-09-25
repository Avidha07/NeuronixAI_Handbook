# Fix: Delete Conversation Returns `404 Not Found` Instead of `401 Unauthorized`

## Overview

This document explains how the **Delete Conversation API** was fixed in the NeuronixAI backend.

### Expected Behavior

- First DELETE request → `200 OK` (Conversation deleted successfully)
- Second DELETE request for the same conversation → `404 Not Found` ("Conversation not found")

Earlier, the second request was returning `401 Unauthorized`, which was incorrect because the user was already authenticated.

---

# Problem Statement

Suppose a conversation exists.

```text
Conversation ID = 10
```

### First Request

```http
DELETE /api/v1/conversations/10
```

Response:

```http
200 OK
```

```json
"Deletion successful"
```

The conversation is now removed from the database.

### Second Request

The same API is called again.

```http
DELETE /api/v1/conversations/10
```

Since Conversation 10 no longer exists, the expected response should be:

```http
404 NOT FOUND
```

```json
"Conversation not found"
```

### Earlier Response

Instead of `404`, the API returned:

```http
401 Unauthorized
```

This was incorrect because the user's authentication was valid.

---

# What We Were Doing Earlier

Earlier, the service used `ResponseStatusException`.

```java
Conversation conversation = conversationRepository
        .findByIdAndUser(conversationId, user)
        .orElseThrow(() ->
                new ResponseStatusException(
                        HttpStatus.NOT_FOUND,
                        "Conversation not found"
                )
        );
```

### How this works

- Database searches for the conversation.
- If found, delete it.
- If not found, throw `ResponseStatusException`.

Flow:

```text
Request
   ↓
Controller
   ↓
Service
   ↓
Database
   ↓
Conversation not found
   ↓
ResponseStatusException(404)
```

Although this looks correct, the final response became `401` in our application.

---

# Why Was It Returning 401?

There are two different responsibilities in the backend.

| Responsibility | Status Code |
|---------------|------------|
| Authentication problem | 401 |
| Resource doesn't exist | 404 |

In our case:

- JWT token was valid.
- User was authenticated.
- The conversation simply did not exist.

The request flow looked like this.

```text
DELETE Request
      ↓
JWT Authentication
      ↓
Controller
      ↓
Service
      ↓
Conversation doesn't exist
      ↓
ResponseStatusException(404)
      ↓
Error handling
      ↓
401 Unauthorized (incorrect response)
```

The business problem was **Conversation Not Found**, not authentication.

---

# The Solution

Instead of using a generic exception, we created our own custom exception.

## Step 1: Create `ConversationNotFoundException`

**File**

```
exception/ConversationNotFoundException.java
```

```java
package com.neuronix.exception;

public class ConversationNotFoundException extends RuntimeException {

    public ConversationNotFoundException(String message) {
        super(message);
    }
}
```

### Why?

This exception clearly represents one specific problem.

Instead of saying:

> "Some HTTP error happened"

we now say:

> "Conversation was not found."

This makes the code cleaner and easier to maintain.

---

# Step 2: Update ConversationService

Old:

```java
.orElseThrow(() ->
        new ResponseStatusException(
                HttpStatus.NOT_FOUND,
                "Conversation not found"
        )
);
```

New:

```java
Conversation conversation = conversationRepository
        .findByIdAndUser(conversationId, user)
        .orElseThrow(() ->
                new ConversationNotFoundException(
                        "Conversation not found"
                )
        );

conversationRepository.delete(conversation);

return "Deletion successful";
```

### What changed?

Now the service throws a **business exception**, not an HTTP exception.

The service only focuses on business logic.

---

# Step 3: Handle the Exception Globally

**File**

```
exception/GlobalExceptionHandler.java
```

```java
package com.neuronix.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(InvalidCredentialsException.class)
    public Map<String, String> handleInvalidCredentials(
            InvalidCredentialsException ex) {

        return Map.of(
                "message",
                ex.getMessage()
        );
    }

    @ExceptionHandler(ConversationNotFoundException.class)
    public ResponseEntity<String> handleConversationNotFound(
            ConversationNotFoundException ex) {

        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(ex.getMessage());
    }
}
```

---

# What is `@RestControllerAdvice`?

Think of it as a **reception desk for errors**.

Whenever any controller throws a specific exception, this class catches it and decides what response should be sent.

Example:

```text
ConversationNotFoundException
        ↓
GlobalExceptionHandler
        ↓
HTTP 404
```

Instead of writing error handling in every controller, everything is managed in one place.

---

# Complete Request Flow

## First Delete

```text
Client
   ↓
JWT Authentication (Valid)
   ↓
Controller
   ↓
ConversationService
   ↓
Database
   ↓
Conversation Found
   ↓
Delete Conversation
   ↓
200 OK
```

---

## Second Delete

```text
Client
   ↓
JWT Authentication (Valid)
   ↓
Controller
   ↓
ConversationService
   ↓
Database
   ↓
Conversation Not Found
   ↓
ConversationNotFoundException
   ↓
GlobalExceptionHandler
   ↓
404 NOT FOUND
```

---

## Invalid JWT

```text
Client
   ↓
JWT Invalid
   ↓
Spring Security
   ↓
401 UNAUTHORIZED
```

Notice the difference.

- Invalid authentication → `401`
- Missing conversation → `404`

This is the correct REST API behavior.

---

# Understanding `orElseThrow()`

```java
conversationRepository
    .findByIdAndUser(conversationId, user)
    .orElseThrow(() ->
        new ConversationNotFoundException(
            "Conversation not found"
        )
    );
```

In simple English:

1. Search for the conversation.
2. If it exists, return it.
3. Otherwise, throw `ConversationNotFoundException`.

Think of it like asking a librarian for Book #10.

| Situation | Response |
|-----------|----------|
| Book exists | Give the book |
| Book doesn't exist | "Book not found." |

---

# Why Custom Exceptions Are Better

Imagine future errors.

```text
UserNotFoundException
ConversationNotFoundException
MessageNotFoundException
EmailAlreadyExistsException
InvalidCredentialsException
```

Instead of handling these separately inside every controller, the `GlobalExceptionHandler` manages them all.

```text
                 GlobalExceptionHandler
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
UserNotFound   ConversationNotFound   InvalidCredentials
      │                │                    │
     404              404                 401
```

This follows clean architecture principles by keeping:

- Business logic inside services.
- Error handling inside one centralized class.
- Controllers simple.

---

# Before vs After

| Feature | Earlier | After |
|----------|---------|--------|
| Exception Type | `ResponseStatusException` | `ConversationNotFoundException` |
| Error Handling | Generic | Centralized |
| Handler | Implicit | `GlobalExceptionHandler` |
| Second DELETE Response | `401 Unauthorized` | `404 Not Found` |
| Business Meaning | Incorrect | Correct |

---

# Final Result

### First Request

```http
DELETE /api/v1/conversations/10
```

Response:

```http
200 OK
```

```json
"Deletion successful"
```

### Second Request

```http
DELETE /api/v1/conversations/10
```

Response:

```http
404 NOT FOUND
```

```json
"Conversation not found"
```

---

# Key Learning

The most important concept is:

```text
Service
   ↓
Throw a specific exception
   ↓
GlobalExceptionHandler
   ↓
Convert it into the correct HTTP response
   ↓
Client/Postman
```

For this API:

```text
Conversation doesn't exist
        ↓
ConversationNotFoundException
        ↓
GlobalExceptionHandler
        ↓
404 NOT FOUND
```

This makes the API communicate the real problem to the client while keeping the backend code clean, reusable, and easy to maintain.
