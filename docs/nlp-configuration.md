# NLP Configuration Guide

## Overview

This document describes the Natural Language Processing (NLP) configuration for the LIA chatbot system, including intent definitions, entity recognition, model training, and optimization strategies.

## NLP Pipeline Architecture

```
Input Text
    ↓
Text Preprocessing
    ↓
Language Detection
    ↓
Tokenization
    ↓
Intent Classification
    ↓
Entity Extraction
    ↓
Sentiment Analysis
    ↓
Output (Intent + Entities + Sentiment)
```

## Configuration Files

### Main Configuration

**File**: `config/nlp/main.yaml`

```yaml
nlp:
  version: "1.0"
  language_support:
    - en
    - es
    - fr
    - de
  default_language: en
  
  models:
    intent_classifier:
      type: transformer
      model_name: bert-base-multilingual-cased
      confidence_threshold: 0.7
      
    entity_extractor:
      type: spacy
      model_name: en_core_web_lg
      custom_entities: true
      
    sentiment_analyzer:
      type: vader
      threshold:
        positive: 0.5
        negative: -0.5

  preprocessing:
    lowercase: true
    remove_punctuation: false
    remove_stopwords: false
    lemmatization: true
    
  features:
    max_sequence_length: 512
    embedding_dimension: 768
```

## Intent Configuration

### Intent Definition Structure

**File**: `config/nlp/intents.yaml`

```yaml
intents:
  - name: greeting
    description: "User greets the chatbot"
    examples:
      - "hello"
      - "hi"
      - "hey"
      - "good morning"
      - "good evening"
      - "what's up"
    responses:
      - "Hello! How can I help you today?"
      - "Hi there! What can I do for you?"
      - "Hey! How may I assist you?"
    priority: high
    
  - name: goodbye
    description: "User ends the conversation"
    examples:
      - "goodbye"
      - "bye"
      - "see you later"
      - "thanks, that's all"
    responses:
      - "Goodbye! Have a great day!"
      - "See you later!"
    priority: high
    end_session: true
    
  - name: help
    description: "User requests assistance"
    examples:
      - "I need help"
      - "can you help me"
      - "I'm stuck"
      - "what can you do"
    responses:
      - "I'm here to help! I can assist you with account management, technical support, and general inquiries. What do you need?"
    priority: high
    
  - name: account_login
    description: "User wants to login to account"
    examples:
      - "I want to login"
      - "how do I sign in"
      - "login to my account"
      - "access my account"
    entities:
      - email
      - username
    responses:
      - "I can help you with login. You can access your account at our login page."
    priority: medium
    
  - name: technical_support
    description: "User needs technical assistance"
    examples:
      - "I have a technical problem"
      - "something is not working"
      - "I found a bug"
      - "error message"
    entities:
      - error_code
      - product_name
    responses:
      - "I'll help you resolve this technical issue. Can you provide more details?"
    priority: high
    escalate_to_human: true
    
  - name: billing_inquiry
    description: "Questions about billing and payments"
    examples:
      - "how much does it cost"
      - "what's the price"
      - "billing question"
      - "invoice"
    entities:
      - amount
      - date
    responses:
      - "I can help with billing questions. What would you like to know?"
    priority: medium
    
  - name: product_information
    description: "Inquiries about products or features"
    examples:
      - "tell me about your products"
      - "what features do you have"
      - "product information"
    entities:
      - product_name
      - feature_name
    responses:
      - "I'd be happy to tell you about our products. What specifically are you interested in?"
    priority: low
    
  - name: fallback
    description: "Triggered when intent cannot be determined"
    examples: []
    responses:
      - "I'm not sure I understand. Could you rephrase that?"
      - "I didn't quite get that. Can you try asking in a different way?"
    priority: lowest
    confidence_threshold: 0.3
```

## Entity Configuration

### Entity Types

**File**: `config/nlp/entities.yaml`

```yaml
entities:
  - name: email
    type: pattern
    pattern: "\\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}\\b"
    examples:
      - "user@example.com"
      - "john.doe@company.org"
      
  - name: phone_number
    type: pattern
    pattern: "\\+?\\d{1,3}?[-.\\s]?\\(?\\d{1,4}?\\)?[-.\\s]?\\d{1,4}[-.\\s]?\\d{1,9}"
    examples:
      - "+1-555-123-4567"
      - "555-1234"
      
  - name: date
    type: duckling
    dimension: time
    examples:
      - "tomorrow"
      - "next week"
      - "December 25th"
      
  - name: amount
    type: duckling
    dimension: amount-of-money
    examples:
      - "$100"
      - "50 euros"
      
  - name: product_name
    type: lookup
    values:
      - "Pro Plan"
      - "Basic Plan"
      - "Enterprise Plan"
      - "Premium Features"
    fuzzy_matching: true
    
  - name: username
    type: custom
    model: trained_ner_model
    
  - name: error_code
    type: pattern
    pattern: "ERR-\\d{3,4}"
    examples:
      - "ERR-404"
      - "ERR-500"
      
  - name: order_id
    type: pattern
    pattern: "ORD-[A-Z0-9]{8}"
    examples:
      - "ORD-ABC12345"
```

## Model Training

### Training Data Structure

**File**: `training_data/training.json`

```json
{
  "rasa_nlu_data": {
    "common_examples": [
      {
        "text": "hello",
        "intent": "greeting",
        "entities": []
      },
      {
        "text": "I want to login with email user@example.com",
        "intent": "account_login",
        "entities": [
          {
            "start": 29,
            "end": 46,
            "value": "user@example.com",
            "entity": "email"
          }
        ]
      }
    ],
    "regex_features": [],
    "lookup_tables": [],
    "entity_synonyms": []
  }
}
```

### Training Configuration

**File**: `config/nlp/training_config.yaml`

```yaml
training:
  pipeline:
    - name: "WhitespaceTokenizer"
    - name: "RegexFeaturizer"
    - name: "LexicalSyntacticFeaturizer"
    - name: "CountVectorsFeaturizer"
    - name: "CountVectorsFeaturizer"
      analyzer: "char_wb"
      min_ngram: 1
      max_ngram: 4
    - name: "DIETClassifier"
      epochs: 100
      batch_size: 64
      learning_rate: 0.001
    - name: "EntitySynonymMapper"
    - name: "ResponseSelector"
      epochs: 100
  
  parameters:
    batch_size: 64
    epochs: 100
    validation_split: 0.2
    early_stopping:
      enabled: true
      patience: 10
      monitor: val_loss
    
  evaluation:
    metrics:
      - accuracy
      - precision
      - recall
      - f1_score
    cross_validation_folds: 5
```

## Language Models

### Supported Models

#### 1. BERT-based Models

```yaml
bert_config:
  model_name: "bert-base-multilingual-cased"
  hidden_size: 768
  num_attention_heads: 12
  num_hidden_layers: 12
  max_position_embeddings: 512
  fine_tuning:
    learning_rate: 2e-5
    epochs: 3
    batch_size: 32
```

#### 2. GPT-based Models

```yaml
gpt_config:
  model_name: "gpt-3.5-turbo"
  temperature: 0.7
  max_tokens: 150
  top_p: 0.9
  frequency_penalty: 0.0
  presence_penalty: 0.0
```

#### 3. Custom Models

```yaml
custom_model:
  architecture: "lstm"
  layers:
    - type: embedding
      dimension: 300
      trainable: true
    - type: lstm
      units: 128
      dropout: 0.2
    - type: dense
      units: 64
      activation: relu
    - type: dense
      units: num_intents
      activation: softmax
```

## Confidence Thresholds

```yaml
confidence:
  intent_classification:
    high_confidence: 0.85
    medium_confidence: 0.70
    low_confidence: 0.50
    fallback_threshold: 0.30
    
  entity_extraction:
    minimum_confidence: 0.60
    
  actions:
    high_confidence:
      action: "respond_directly"
    medium_confidence:
      action: "ask_clarification"
    low_confidence:
      action: "use_fallback"
    below_threshold:
      action: "escalate_to_human"
```

## Text Preprocessing

### Preprocessing Pipeline

```yaml
preprocessing:
  steps:
    - name: "lowercase"
      enabled: true
      
    - name: "remove_urls"
      enabled: true
      replacement: "<URL>"
      
    - name: "remove_emails"
      enabled: false
      replacement: "<EMAIL>"
      
    - name: "remove_special_chars"
      enabled: true
      keep: [".", "?", "!"]
      
    - name: "normalize_whitespace"
      enabled: true
      
    - name: "expand_contractions"
      enabled: true
      language: en
      
    - name: "spelling_correction"
      enabled: false
      max_distance: 2
      
    - name: "lemmatization"
      enabled: true
      pos_tags: ["NOUN", "VERB", "ADJ"]
      
    - name: "stopword_removal"
      enabled: false
      custom_stopwords: []
```

## Sentiment Analysis

```yaml
sentiment:
  analyzer: "vader"
  
  thresholds:
    very_positive: 0.7
    positive: 0.3
    neutral: 0.0
    negative: -0.3
    very_negative: -0.7
    
  response_adaptation:
    enabled: true
    adjust_tone: true
    
  scoring:
    compound_score: true
    individual_scores:
      - positive
      - negative
      - neutral
```

## Context Handling

```yaml
context:
  enabled: true
  
  context_variables:
    - name: "user_name"
      type: string
      persistence: session
      
    - name: "last_intent"
      type: string
      persistence: session
      
    - name: "entities_collected"
      type: list
      persistence: session
      
    - name: "conversation_topic"
      type: string
      persistence: user
      
  context_lifetime:
    session_context: 3600  # seconds
    user_context: 2592000  # 30 days
    
  context_reset:
    on_new_session: false
    on_explicit_request: true
    on_topic_change: false
```

## Multi-Language Support

```yaml
multilingual:
  enabled: true
  
  languages:
    - code: en
      name: English
      models:
        intent: bert-base-cased
        entity: en_core_web_lg
      
    - code: es
      name: Spanish
      models:
        intent: bert-base-multilingual-cased
        entity: es_core_news_lg
      
    - code: fr
      name: French
      models:
        intent: bert-base-multilingual-cased
        entity: fr_core_news_lg
  
  auto_detect: true
  fallback_language: en
  
  translation:
    enabled: true
    service: google_translate
    cache_translations: true
```

## Performance Optimization

```yaml
optimization:
  caching:
    enabled: true
    cache_predictions: true
    cache_ttl: 3600
    
  batching:
    enabled: true
    batch_size: 32
    max_wait_time: 100  # milliseconds
    
  model_quantization:
    enabled: false
    precision: int8
    
  gpu_acceleration:
    enabled: true
    device: cuda:0
    
  model_pruning:
    enabled: false
    sparsity: 0.3
```

## Evaluation Metrics

```yaml
evaluation:
  metrics:
    intent_classification:
      - accuracy
      - precision
      - recall
      - f1_score
      - confusion_matrix
      
    entity_extraction:
      - precision
      - recall
      - f1_score
      
    response_quality:
      - relevance_score
      - user_satisfaction
      
  benchmarks:
    intent_accuracy_target: 0.90
    entity_f1_target: 0.85
    response_time_target: 200  # milliseconds
```

## Testing and Validation

### Test Cases

**File**: `tests/nlp/test_cases.yaml`

```yaml
test_cases:
  - input: "hello there"
    expected_intent: greeting
    expected_confidence: ">0.8"
    expected_entities: []
    
  - input: "I need help with my order ORD-ABC12345"
    expected_intent: technical_support
    expected_entities:
      - type: order_id
        value: "ORD-ABC12345"
    
  - input: "what's the price"
    expected_intent: billing_inquiry
    expected_confidence: ">0.7"
```

## Deployment

```yaml
deployment:
  model_registry:
    type: mlflow
    url: https://mlflow.lia-chatbot.com
    
  version_control:
    enabled: true
    strategy: semantic_versioning
    
  a_b_testing:
    enabled: true
    traffic_split:
      model_a: 0.9
      model_b: 0.1
    
  rollback:
    auto_rollback: true
    error_threshold: 0.05
    monitoring_window: 3600  # seconds
```

## Monitoring and Logging

```yaml
monitoring:
  metrics:
    - intent_distribution
    - confidence_scores
    - entity_extraction_rate
    - fallback_rate
    - response_time
    
  logging:
    level: INFO
    log_predictions: true
    log_confidence_scores: true
    log_entities: true
    
  alerts:
    - condition: "fallback_rate > 0.2"
      action: notify_team
      severity: warning
    - condition: "avg_confidence < 0.6"
      action: trigger_retraining
      severity: critical
```

## Best Practices

1. **Intent Design**
   - Keep intents specific and non-overlapping
   - Provide at least 10-15 examples per intent
   - Regularly review and update intent examples

2. **Entity Extraction**
   - Use appropriate entity types (pattern, lookup, ML-based)
   - Maintain lookup tables for domain-specific entities
   - Test entity extraction with varied input formats

3. **Model Training**
   - Use balanced training data
   - Implement cross-validation
   - Monitor for overfitting

4. **Performance**
   - Cache frequent predictions
   - Use batch processing for multiple requests
   - Optimize model size for production

5. **Maintenance**
   - Regular model retraining with new data
   - Monitor model performance metrics
   - Version control for models and configurations

## Related Documents

- [Architecture](./architecture.md)
- [API Reference](./api-reference.md)
- [Maintenance Guide](./maintenance-guide.md)
