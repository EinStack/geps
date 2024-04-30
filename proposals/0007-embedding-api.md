---
GEP: 0007
Title: Unified Embedding API
Discussion:
Implementation:
---

# Unified Embedding API

## Abstract

This GEP discusses the unified embeddings api and its interactions with embedding model providers.

## Motivation

Many LLM applications have a semantic search feature within their architecture. To support these applications on Glide, a unified embedding api is necessary.
This will allow applications to embed chat requests as part of a retrieval augmented generation (RAG) application flow.

### Requirements

- R1: Handles all provider specific logic
- R2: Easily maintained
- R3: API schemas must unify common request params (e.g. dimensions)
- R4: API routes must unify common embedding endpoints/API

## Design

```yaml
routes:
    chat: /v1/language/{pool-id}/chat/ 
    transcribers: /v1/speech/transcribers/{pool-id}/  
    speech-synthesizer:  /v1/speech/synthesizers/{pool-id}/ 
    multi-modal: /v1/multi/{pool-id}/multimodal/ 
    embedding: /v1/embeddings/{pool-id}/embed/ 
```

#### User Request Schema for Embedding Route

```yaml
{
 "message": "Where was it played?",
 "dimensions": 1536
}
```

#### Response Schema for Embedding Route
```yaml
{
  "provider": "cohere",
  "model": "embed-multilingual-v3.0",
  "provider_response": {
    "embedding": [
        0.0023064255,
        -0.009327292,
        ....
        -0.0028842222,
      ],
    "token_count": {
      "prompt_tokens": 9,
      "total_tokens": 9
    }
  }
}
```

## Alternatives Considered

[TBU, what other solutions were considered and why they were rejected]

## Future Work

- Could we abstract away the entire RAG architecture? A single endpoint that takes a chat message -> embeds -> text semantic search -> LLM -> response
