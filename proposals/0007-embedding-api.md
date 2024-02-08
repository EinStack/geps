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

TBU

#### Response Schema for Embedding Route
TBU

## Alternatives Considered

[TBU, what other solutions were considered and why they were rejected]

## Future Work

- Could we abstract away the entire RAG architecture? A single endpoint that takes a chat message -> embeds -> text semantic search -> LLM -> response
