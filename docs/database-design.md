# Database Design Documentation

## Overview

This document describes the database architecture and schema design for the LIA chatbot system. The system uses a combination of relational and NoSQL databases to handle different data requirements efficiently.

## Database Architecture

### Database Stack

```
┌─────────────────────────────────────────────┐
│         Application Layer                    │
└──────────────┬──────────────────────────────┘
               │
    ┌──────────┴──────────┐
    │                     │
    ▼                     ▼
┌─────────┐         ┌─────────┐
│PostgreSQL│        │  Redis   │
│  (Primary│        │ (Cache/  │
│   RDBMS) │        │ Sessions)│
└─────────┘         └─────────┘
    │                     │
    ▼                     ▼
┌─────────┐         ┌─────────┐
│ MongoDB  │        │Elasticsearch│
│(Documents│        │  (Search)   │
│/Context) │        │             │
└─────────┘         └─────────────┘
```

## PostgreSQL Schema

### Primary Relational Database

Used for structured data requiring ACID properties.

#### Users Table

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(255) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    full_name VARCHAR(255),
    phone_number VARCHAR(50),
    language_preference VARCHAR(10) DEFAULT 'en',
    timezone VARCHAR(50) DEFAULT 'UTC',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    last_active TIMESTAMP WITH TIME ZONE,
    is_active BOOLEAN DEFAULT TRUE,
    deleted_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT valid_email CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$')
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_active ON users(is_active) WHERE is_active = TRUE;
```

#### Sessions Table

```sql
CREATE TABLE sessions (
    session_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id),
    channel VARCHAR(50) NOT NULL, -- web, mobile, api
    device_info JSONB,
    ip_address INET,
    started_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    ended_at TIMESTAMP WITH TIME ZONE,
    last_activity TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(20) DEFAULT 'active', -- active, ended, expired
    metadata JSONB,
    
    CONSTRAINT valid_status CHECK (status IN ('active', 'ended', 'expired'))
);

CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_status ON sessions(status);
CREATE INDEX idx_sessions_started_at ON sessions(started_at DESC);
```

#### Messages Table

```sql
CREATE TABLE messages (
    message_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES sessions(session_id),
    user_id UUID NOT NULL REFERENCES users(user_id),
    role VARCHAR(20) NOT NULL, -- user, assistant, system
    content TEXT NOT NULL,
    intent VARCHAR(100),
    confidence_score DECIMAL(3,2),
    entities JSONB,
    sentiment VARCHAR(20),
    sentiment_score DECIMAL(3,2),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    response_time_ms INTEGER,
    metadata JSONB,
    
    CONSTRAINT valid_role CHECK (role IN ('user', 'assistant', 'system')),
    CONSTRAINT valid_confidence CHECK (confidence_score >= 0 AND confidence_score <= 1)
);

CREATE INDEX idx_messages_session_id ON messages(session_id);
CREATE INDEX idx_messages_user_id ON messages(user_id);
CREATE INDEX idx_messages_created_at ON messages(created_at DESC);
CREATE INDEX idx_messages_intent ON messages(intent);
CREATE INDEX idx_messages_gin_entities ON messages USING gin(entities);
```

#### User Preferences Table

```sql
CREATE TABLE user_preferences (
    preference_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id),
    preference_key VARCHAR(100) NOT NULL,
    preference_value TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(user_id, preference_key)
);

CREATE INDEX idx_user_preferences_user_id ON user_preferences(user_id);
CREATE INDEX idx_user_preferences_key ON user_preferences(preference_key);
```

#### Intents Table

```sql
CREATE TABLE intents (
    intent_id SERIAL PRIMARY KEY,
    intent_name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    category VARCHAR(50),
    priority VARCHAR(20) DEFAULT 'medium',
    is_active BOOLEAN DEFAULT TRUE,
    requires_auth BOOLEAN DEFAULT FALSE,
    escalate_to_human BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT valid_priority CHECK (priority IN ('low', 'medium', 'high', 'critical'))
);

CREATE INDEX idx_intents_name ON intents(intent_name);
CREATE INDEX idx_intents_category ON intents(category);
```

#### Intent Examples Table

```sql
CREATE TABLE intent_examples (
    example_id SERIAL PRIMARY KEY,
    intent_id INTEGER NOT NULL REFERENCES intents(intent_id) ON DELETE CASCADE,
    example_text TEXT NOT NULL,
    language VARCHAR(10) DEFAULT 'en',
    is_validated BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_intent_examples_intent_id ON intent_examples(intent_id);
CREATE INDEX idx_intent_examples_language ON intent_examples(language);
```

#### Feedback Table

```sql
CREATE TABLE feedback (
    feedback_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id UUID REFERENCES messages(message_id),
    session_id UUID REFERENCES sessions(session_id),
    user_id UUID REFERENCES users(user_id),
    rating INTEGER,
    helpful BOOLEAN,
    comment TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT valid_rating CHECK (rating >= 1 AND rating <= 5)
);

CREATE INDEX idx_feedback_message_id ON feedback(message_id);
CREATE INDEX idx_feedback_rating ON feedback(rating);
CREATE INDEX idx_feedback_created_at ON feedback(created_at DESC);
```

#### Analytics Events Table

```sql
CREATE TABLE analytics_events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type VARCHAR(100) NOT NULL,
    user_id UUID REFERENCES users(user_id),
    session_id UUID REFERENCES sessions(session_id),
    event_data JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_analytics_event_type ON analytics_events(event_type);
CREATE INDEX idx_analytics_user_id ON analytics_events(user_id);
CREATE INDEX idx_analytics_created_at ON analytics_events(created_at DESC);
CREATE INDEX idx_analytics_gin_event_data ON analytics_events USING gin(event_data);
```

#### API Keys Table

```sql
CREATE TABLE api_keys (
    key_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(user_id),
    key_hash VARCHAR(255) NOT NULL,
    key_name VARCHAR(100),
    permissions JSONB,
    rate_limit INTEGER DEFAULT 1000,
    is_active BOOLEAN DEFAULT TRUE,
    expires_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    last_used_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_api_keys_user_id ON api_keys(user_id);
CREATE INDEX idx_api_keys_active ON api_keys(is_active) WHERE is_active = TRUE;
```

## MongoDB Collections

### Conversation Contexts Collection

```javascript
db.createCollection("conversation_contexts", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["session_id", "user_id"],
            properties: {
                session_id: {
                    bsonType: "string",
                    description: "Session identifier"
                },
                user_id: {
                    bsonType: "string",
                    description: "User identifier"
                },
                context_data: {
                    bsonType: "object",
                    description: "Conversation context variables"
                },
                collected_entities: {
                    bsonType: "array",
                    description: "Entities collected during conversation"
                },
                current_topic: {
                    bsonType: "string",
                    description: "Current conversation topic"
                },
                dialog_state: {
                    bsonType: "string",
                    description: "Current state in dialog flow"
                },
                created_at: {
                    bsonType: "date"
                },
                updated_at: {
                    bsonType: "date"
                },
                expires_at: {
                    bsonType: "date"
                }
            }
        }
    }
});

db.conversation_contexts.createIndex({ "session_id": 1 }, { unique: true });
db.conversation_contexts.createIndex({ "user_id": 1 });
db.conversation_contexts.createIndex({ "expires_at": 1 }, { expireAfterSeconds: 0 });
```

### Knowledge Base Collection

```javascript
db.createCollection("knowledge_base", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["title", "content", "category"],
            properties: {
                title: {
                    bsonType: "string"
                },
                content: {
                    bsonType: "string"
                },
                category: {
                    bsonType: "string"
                },
                tags: {
                    bsonType: "array",
                    items: {
                        bsonType: "string"
                    }
                },
                language: {
                    bsonType: "string"
                },
                metadata: {
                    bsonType: "object"
                },
                embeddings: {
                    bsonType: "array",
                    description: "Vector embeddings for semantic search"
                },
                created_at: {
                    bsonType: "date"
                },
                updated_at: {
                    bsonType: "date"
                }
            }
        }
    }
});

db.knowledge_base.createIndex({ "category": 1 });
db.knowledge_base.createIndex({ "tags": 1 });
db.knowledge_base.createIndex({ "title": "text", "content": "text" });
```

### Conversation History Collection

```javascript
db.createCollection("conversation_history", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["session_id", "messages"],
            properties: {
                session_id: {
                    bsonType: "string"
                },
                user_id: {
                    bsonType: "string"
                },
                messages: {
                    bsonType: "array",
                    items: {
                        bsonType: "object",
                        required: ["role", "content", "timestamp"],
                        properties: {
                            role: {
                                enum: ["user", "assistant", "system"]
                            },
                            content: {
                                bsonType: "string"
                            },
                            intent: {
                                bsonType: "string"
                            },
                            entities: {
                                bsonType: "array"
                            },
                            timestamp: {
                                bsonType: "date"
                            }
                        }
                    }
                },
                summary: {
                    bsonType: "string"
                },
                created_at: {
                    bsonType: "date"
                },
                updated_at: {
                    bsonType: "date"
                }
            }
        }
    }
});

db.conversation_history.createIndex({ "session_id": 1 }, { unique: true });
db.conversation_history.createIndex({ "user_id": 1 });
db.conversation_history.createIndex({ "created_at": -1 });
```

## Redis Data Structures

### Session Cache

```
Key Pattern: session:{session_id}
Type: Hash
TTL: 3600 seconds (1 hour)

Fields:
- user_id: string
- status: string
- channel: string
- last_activity: timestamp
- context_data: json
```

### Rate Limiting

```
Key Pattern: ratelimit:{user_id}:{endpoint}
Type: String (counter)
TTL: 60 seconds

Tracks request counts per minute per user/endpoint
```

### Response Cache

```
Key Pattern: response:{intent}:{entity_hash}
Type: String (JSON)
TTL: 300 seconds (5 minutes)

Caches common responses to reduce processing
```

### User Presence

```
Key Pattern: presence:{user_id}
Type: String
TTL: 300 seconds (5 minutes)

Tracks active users
```

## Elasticsearch Indices

### Messages Index

```json
{
  "mappings": {
    "properties": {
      "message_id": { "type": "keyword" },
      "session_id": { "type": "keyword" },
      "user_id": { "type": "keyword" },
      "content": { 
        "type": "text",
        "analyzer": "standard"
      },
      "intent": { "type": "keyword" },
      "entities": { "type": "nested" },
      "timestamp": { "type": "date" },
      "sentiment": { "type": "keyword" },
      "confidence_score": { "type": "float" }
    }
  }
}
```

### Knowledge Base Index

```json
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "standard"
      },
      "content": {
        "type": "text",
        "analyzer": "standard"
      },
      "category": { "type": "keyword" },
      "tags": { "type": "keyword" },
      "embeddings": {
        "type": "dense_vector",
        "dims": 768
      },
      "timestamp": { "type": "date" }
    }
  }
}
```

## Data Relationships

```
users (1) ──── (N) sessions
sessions (1) ──── (N) messages
users (1) ──── (N) user_preferences
users (1) ──── (N) api_keys
intents (1) ──── (N) intent_examples
messages (1) ──── (1) feedback
```

## Backup and Recovery

### Backup Strategy

```yaml
backup:
  postgresql:
    method: pg_dump
    frequency: daily
    retention: 30 days
    incremental: true
    
  mongodb:
    method: mongodump
    frequency: daily
    retention: 30 days
    
  redis:
    method: rdb_snapshot
    frequency: hourly
    retention: 7 days
    
  elasticsearch:
    method: snapshot
    frequency: daily
    retention: 14 days
```

### Recovery Procedures

1. **Point-in-Time Recovery**: WAL archiving for PostgreSQL
2. **Replica Promotion**: Automated failover to read replicas
3. **Backup Restoration**: Tested monthly restoration procedures

## Performance Optimization

### Indexing Strategy

- Primary keys on all tables
- Foreign key indexes
- Composite indexes for common query patterns
- Partial indexes for filtered queries
- Full-text search indexes

### Query Optimization

```sql
-- Example: Efficient message retrieval with pagination
EXPLAIN ANALYZE
SELECT m.*, u.username
FROM messages m
JOIN users u ON m.user_id = u.user_id
WHERE m.session_id = $1
ORDER BY m.created_at DESC
LIMIT 50 OFFSET 0;
```

### Partitioning

```sql
-- Partition messages by month for better performance
CREATE TABLE messages_2025_10 PARTITION OF messages
FOR VALUES FROM ('2025-10-01') TO ('2025-11-01');
```

## Data Retention

```yaml
retention_policies:
  messages:
    active_data: 90 days
    archived_data: 2 years
    
  sessions:
    completed: 180 days
    
  analytics_events:
    detailed: 30 days
    aggregated: indefinite
    
  feedback:
    all: indefinite
```

## Security Considerations

- Encryption at rest for sensitive data
- Column-level encryption for PII
- Row-level security policies
- Audit logging for data access
- Regular security assessments

## Monitoring

```yaml
monitoring:
  metrics:
    - connection_pool_usage
    - query_performance
    - disk_usage
    - replication_lag
    - cache_hit_ratio
    
  alerts:
    - high_connection_count
    - slow_queries
    - replication_delay
    - disk_space_low
```

## Migration Strategy

```sql
-- Example migration using Flyway/Liquibase
-- V1__initial_schema.sql
-- V2__add_sentiment_analysis.sql
-- V3__add_analytics_events.sql
```

## Related Documents

- [Architecture](./architecture.md)
- [API Reference](./api-reference.md)
- [Security](./security.md)
- [Maintenance Guide](./maintenance-guide.md)
