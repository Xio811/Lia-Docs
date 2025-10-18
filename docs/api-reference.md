# LIA API Reference

## Overview

The LIA API provides RESTful endpoints for interacting with the chatbot system. All endpoints use JSON for request and response payloads.

## Base URL

```
Production: https://api.lia-chatbot.com/v1
Staging: https://staging-api.lia-chatbot.com/v1
Development: http://localhost:8000/v1
```

## Authentication

All API requests require authentication using Bearer tokens.

### Request Header

```http
Authorization: Bearer <your_access_token>
Content-Type: application/json
```

### Obtaining Access Token

**Endpoint**: `POST /auth/token`

**Request Body**:
```json
{
  "client_id": "your_client_id",
  "client_secret": "your_client_secret",
  "grant_type": "client_credentials"
}
```

**Response**:
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

## Core Endpoints

### 1. Send Message

Send a message to the chatbot and receive a response.

**Endpoint**: `POST /chat/message`

**Request Body**:
```json
{
  "session_id": "unique-session-id",
  "user_id": "user-123",
  "message": "Hello, can you help me?",
  "context": {
    "channel": "web",
    "language": "en"
  }
}
```

**Response**:
```json
{
  "response_id": "resp-789",
  "session_id": "unique-session-id",
  "message": "Hello! I'd be happy to help you. What do you need assistance with?",
  "intent": "greeting",
  "confidence": 0.95,
  "entities": [],
  "suggestions": [
    "I need help with my account",
    "I have a question about pricing",
    "I want to learn more about features"
  ],
  "timestamp": "2025-10-18T23:47:00Z"
}
```

**Status Codes**:
- `200 OK`: Successful response
- `400 Bad Request`: Invalid input
- `401 Unauthorized`: Authentication failed
- `429 Too Many Requests`: Rate limit exceeded
- `500 Internal Server Error`: Server error

---

### 2. Create Session

Initialize a new chat session.

**Endpoint**: `POST /chat/session`

**Request Body**:
```json
{
  "user_id": "user-123",
  "metadata": {
    "channel": "web",
    "device": "desktop",
    "browser": "Chrome"
  }
}
```

**Response**:
```json
{
  "session_id": "sess-456",
  "created_at": "2025-10-18T23:47:00Z",
  "expires_at": "2025-10-19T23:47:00Z"
}
```

---

### 3. Get Session History

Retrieve conversation history for a session.

**Endpoint**: `GET /chat/session/{session_id}/history`

**Query Parameters**:
- `limit` (optional): Number of messages to retrieve (default: 50, max: 100)
- `offset` (optional): Pagination offset (default: 0)

**Response**:
```json
{
  "session_id": "sess-456",
  "messages": [
    {
      "id": "msg-001",
      "role": "user",
      "content": "Hello",
      "timestamp": "2025-10-18T23:45:00Z"
    },
    {
      "id": "msg-002",
      "role": "assistant",
      "content": "Hi! How can I help you?",
      "timestamp": "2025-10-18T23:45:01Z"
    }
  ],
  "total_count": 2,
  "has_more": false
}
```

---

### 4. End Session

Close a chat session.

**Endpoint**: `DELETE /chat/session/{session_id}`

**Response**:
```json
{
  "session_id": "sess-456",
  "status": "closed",
  "closed_at": "2025-10-18T23:47:00Z"
}
```

---

### 5. Get User Context

Retrieve stored context for a user.

**Endpoint**: `GET /users/{user_id}/context`

**Response**:
```json
{
  "user_id": "user-123",
  "preferences": {
    "language": "en",
    "timezone": "America/New_York",
    "notification_enabled": true
  },
  "profile": {
    "name": "John Doe",
    "email": "john@example.com"
  },
  "context_data": {
    "last_topic": "account_management",
    "frequent_queries": ["billing", "features"]
  }
}
```

---

### 6. Update User Context

Update user context and preferences.

**Endpoint**: `PATCH /users/{user_id}/context`

**Request Body**:
```json
{
  "preferences": {
    "language": "es",
    "timezone": "Europe/Madrid"
  },
  "context_data": {
    "custom_field": "custom_value"
  }
}
```

**Response**:
```json
{
  "user_id": "user-123",
  "updated": true,
  "timestamp": "2025-10-18T23:47:00Z"
}
```

---

### 7. Analyze Intent

Analyze text to extract intent and entities without sending a full chat message.

**Endpoint**: `POST /nlp/analyze`

**Request Body**:
```json
{
  "text": "I want to book a flight to Paris next week",
  "language": "en"
}
```

**Response**:
```json
{
  "intent": "book_flight",
  "confidence": 0.92,
  "entities": [
    {
      "type": "destination",
      "value": "Paris",
      "start": 26,
      "end": 31
    },
    {
      "type": "time",
      "value": "next week",
      "start": 32,
      "end": 41
    }
  ],
  "sentiment": {
    "polarity": "neutral",
    "score": 0.0
  }
}
```

---

### 8. Get Available Intents

Retrieve list of all configured intents.

**Endpoint**: `GET /nlp/intents`

**Response**:
```json
{
  "intents": [
    {
      "name": "greeting",
      "description": "User greeting messages",
      "examples": ["hello", "hi", "good morning"]
    },
    {
      "name": "book_flight",
      "description": "Flight booking requests",
      "examples": ["book a flight", "I want to fly to"]
    }
  ],
  "total": 2
}
```

---

### 9. Feedback

Submit feedback on a chatbot response.

**Endpoint**: `POST /chat/feedback`

**Request Body**:
```json
{
  "response_id": "resp-789",
  "rating": 5,
  "helpful": true,
  "comment": "Very helpful response!"
}
```

**Response**:
```json
{
  "feedback_id": "fb-001",
  "received": true,
  "timestamp": "2025-10-18T23:47:00Z"
}
```

---

### 10. Health Check

Check API health and status.

**Endpoint**: `GET /health`

**Response**:
```json
{
  "status": "healthy",
  "version": "1.0.0",
  "timestamp": "2025-10-18T23:47:00Z",
  "services": {
    "database": "operational",
    "nlp_engine": "operational",
    "cache": "operational"
  }
}
```

## Rate Limiting

API requests are rate-limited to prevent abuse:

- **Free Tier**: 100 requests per minute
- **Basic Tier**: 1,000 requests per minute
- **Premium Tier**: 10,000 requests per minute

**Rate Limit Headers**:
```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 995
X-RateLimit-Reset: 1697752020
```

## Error Handling

### Error Response Format

```json
{
  "error": {
    "code": "INVALID_INPUT",
    "message": "The request body is invalid",
    "details": {
      "field": "message",
      "issue": "Field is required"
    },
    "request_id": "req-xyz123"
  }
}
```

### Common Error Codes

| Code | Description |
|------|-------------|
| `AUTHENTICATION_FAILED` | Invalid or expired token |
| `INVALID_INPUT` | Request validation failed |
| `RESOURCE_NOT_FOUND` | Requested resource doesn't exist |
| `RATE_LIMIT_EXCEEDED` | Too many requests |
| `INTERNAL_ERROR` | Server-side error |
| `SERVICE_UNAVAILABLE` | Service temporarily unavailable |

## Webhooks

LIA can send webhooks to your endpoints for specific events.

### Webhook Configuration

**Endpoint**: `POST /webhooks`

**Request Body**:
```json
{
  "url": "https://your-domain.com/webhook",
  "events": ["message.sent", "session.ended"],
  "secret": "your-webhook-secret"
}
```

### Webhook Payload

```json
{
  "event": "message.sent",
  "timestamp": "2025-10-18T23:47:00Z",
  "data": {
    "session_id": "sess-456",
    "message_id": "msg-789",
    "user_id": "user-123",
    "content": "Hello"
  }
}
```

## SDK Support

Official SDKs available:

- **JavaScript/TypeScript**: `npm install lia-chatbot-sdk`
- **Python**: `pip install lia-chatbot`
- **Java**: Maven/Gradle support
- **Go**: `go get github.com/lia/go-sdk`

## API Versioning

API versions are specified in the URL path. Current version: `v1`

Breaking changes will result in a new version number. All versions are supported for at least 12 months after deprecation notice.

## Support

For API support:
- Documentation: https://docs.lia-chatbot.com
- Email: api-support@lia-chatbot.com
- Status Page: https://status.lia-chatbot.com

## Related Documents

- [Architecture](./architecture.md)
- [Security](./security.md)
- [NLP Configuration](./nlp-configuration.md)
