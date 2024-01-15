---
GEP: 5
Title: Fallback & Routing
Discussion: Link
Implementation: Link
---

# Fallback & Routing

## Abstract

In this GEP, we talk about fallback mechanism, the key strategy to cope with model provider availability issues, 
and load balancing which is a somewhat related topic.

## Motivation

Model provider APIs are external resources that **could be unavailable** from time to time.
When your applications use those API in the business-critical workflows, this becomes a problem that could cause revenue loss.

Model API could be unavailable in the following ways:
- API returns `50X` errors that indicate internal service issues
- API returns `429` errors indicating rate or quota limits. API is healthy in this case, but given credentials are not going to work for specified period of time
- API could answer for too long a.k.a. **timeout**. Increase in timeout rate means availability issues on the provider side.

At the same time, routing requests to a healthy model provider could be just one of requirements
the application may have to pick the next model to serve the request. Other considerations may be helpful, too, like:
- **Weighted Routing (a.k.a Weighted Round Robin or WRR)** e.g. let 30% of requests be processed by the Cohere model, 20% by Anthropic, and the remaining 50% by OpenAI.
- **Priority-based Routing** e.g. OpenAI should serve all requests if available. If not, then Anthropic, and finally, Google Bard
- **Latency-based Routing** e.g. the next request should be served by the fastest available model in the pool. This is useful for real-time applications where latency is important.
- **Cost-based Routing** e.g. the next request should be served by the cheapest available model in the pool
- **Performance-based Routing** e.g. the next request should be served by the model that performs the best for my request (see GEP003).
- etc.

WRR could be useful to perform **A/B testing** of model pool setup to find the best model/params/prompt combination for a given task.

### Requirements

- R1: We MUST fallback to a healthy model if any in a pool doing our best to fulfill routing strategy specified in configs
- R2: Fallback MUST be as quick as possible. No redundant wasteful retrying or waiting

## Design

The off-the-shelf strategy to tackle the availability issues is redundancy.
So we have some backup models to talk to in case the main one is down.
The mechanism of switching models in case of failures is called fallbacking.

For most serious production applications that have real users, downtime is not an option. Hence, fallbacking should be
an intrinsic feature of Glide routing despite routing strategy.

There may be exceptions to this dictated by the model nature.
For example, embedding API may require one model to be used for all embeddings produced.

Based on this, we group all models we support into types (see GEP0002):

- language models
- embeddings
- speech transcribers
- speech synthesizers
- etc.

Each model type is represented as a list of model pools that could serve the incoming request uniformly.
Each model pool is a list of models (along with their params & operational info like API key) and routing strategy that defines
how incoming requests should be served.

The model pool ID or name is exposed to clients and they provide IDs during requests (see GEP0002).
This is how Glide understands which one should serve the concrete request.

### Model Routing & Fallbacking

Glide leverages fallback models on the first fail and do adaptive health tracking (via token bucket). 
Moving right to the next model serves as a retry for the first one while we spend as little time as possible waiting. 

Then, the whole service instance is going to be aware about number of fails that happen to a certain model 
and as soon as that number goes over the roof, 
the model is going to be "disqualified" to serve requests for some time. 
This means that as soon as the model is dead we pull it out of the pool, so the next requests are going to be served by healthy models only. 
This strategy should lead to much better latency on failover.

![Routing & Fallbacks](./imgs/glide-fallback-logic.png)

### Routing Strategies

#### Priority

Route tracking to the first healthy model in the order in which the user defined models in the router setup.
So if the first model is healthy it's going to handle all requests until it's unavailable. 
Then, the second model will handle all requests, and so on.

If the first model is available again, all traffic will go to it.

#### Round Robin

Iterate through model list in cycle. 
If the model is not available, it will iterate again until it finds a healthy model or make a full cycle with no luck.

### Weighted Round Robin

Send each model traffic proportional to the model weight.

If a model become unavailable, it's taken out of consideration completely until it's available again.
So the traffic split was:
- model A: 8
- model B: 1
- model C 1

And the model A becomes unavailable, then model B & C will handle an equal portion of all traffic.

### Least Latency

Picks a healthy model with the least average latency over time. 
If the least latency model becomes unhealthy, it will pick the second the best, etc.

Since we don't know the true distribution of model latencies,
and it's likely a moving value, we do some attempts to estimate it and keep updated over time.

The algorithm consists of two stages:
- Warm up: Before considering model latencies we may want to collect more than one sample to make better decisions.
To learn about latencies, we route requests to all "cold" models in round-robin manner
- Least Latency selection: Once all models are warmed, we pick one with the least latency

All latency probes are summarized by moving average algorithm. 
Instead of just giving all samples the same weight, we use Exponentially Weighted Moving Average that deprioritizes old latency samples.

Additionally, we should update our latency stats as response latency is a dynamic distribution,
we cannot simply stick to the fastest model discovered on the warmup stage (as we could overlook
other model latencies that might have improved over time).
For that, we introduced expiration time after which the model receives a request
even if it was not the fastest to respond.

### Lang Router Config

The config structure was already covered in [GEP0001](0001-gep.md), so in the GEP, let's just focus on the
model pool config specifically:

```YAML
# some other configs...
routers:
  language:
    - id: my-pool
      enabled: true # # bool, true by default
      strategy: priority # round_robin, weighted_round_robin, priority, least_latency, priority, etc.
      models:
        - id: openai-boring
          enabled: true # bool, true by default
          error_budget: "30/s" # supported time units: ms, s, m, h
          latency:
            decay: 0.06
            warmup_samples: 3
            update_interval: "30s"
          openai: # anthropic, azureopenai, gemini, other providers we support
            model: gpt-3.5-turbo
            api_key: ${env:OPENAI_API_KEY}
            default_params:
              temperature: 0
```

Notes:
- The bare minimal setup must include one pool with at least one model.
  We don't require to set up more than one model to simplify gateway setup for users who just want to try it or experiment.
  That said, we may want to show a warning that such a pool better have some redundancy to be more resilient to provider outages.
- Each pool ID should have a unique name across other pools of this type (e.g. language pools)
- Each model ID should be unique for the pool it's defined in
- Pool and model definition include the "enabled" field to simplify disabling pools/models without a need to remove their definitions from the file

### Router API

This could be useful to have an ability to inspect the current pool setup in case Glide users don't have access to the underlying gateway configuration (it's likely that gateway is deployed by the team that doesn't do LLM work directly, but rather helps with operations/engineering).

In that case, Glide provides the following API (see GEP0002, for other API):

```
GET /v1/language/
# This will return all enabled pools like:
[
  {
    "id": "mypool",
    "strategy": "priority",
    "models": [
      {
        "id": "openai-boring",
        "openai": {
          "model": "gpt-3.5-turbo",
          "default_params": {
            "temperature": 0
          }
        }
      }
    ]
  }
]
```

API keys and any other sensitive data is going to be redacted in this output.

### Testing

The real providers are costly to be used for E2E testing which is useful thing 
to have implemented for Glide to ensure as few regressions as possible. 

In addition to that, real providers may not be convenient to use to check for abnormal conditions like rate limiting.

Hence, we need a fake provider that will help to emulate needed conditions for us:

- In-memory provider that is used for unit/integration tests
- A client and a simple server that would generate needed faulty conditions in E2E tests according to a given config. Let's call it `FaultyProvider`.

The FaultyProvider could be useful in other cases that resemble E2E scenario like 
- load testing e.g. sending a lot of requests through Glide to measure its performance
- or resiliency testing e.g. send a lot of requests through Glide to a pool with a few FaultyProvider configured to fail differently and measure how many requests end up failing

## Model Pool Use Cases

The model pool abstraction seems like a powerful idea that allows to do these things (not an exhaustive list):

- Switch clients to a new pool with different setup in a backward-compatible manner (e.g. keeps the old pool in place, add a new one, switch the app to the new pool, remove the old one)
- Serve several applications with different needs/use cases without a need to deploy another Glide cluster

### References

- https:github.com/zehuamama/balancer/
- https:github.com/yyyar/gobetween/tree/master/src/balance
- https:github.com/mr-karan/balance/blob/main/balance.go
- https:github.com/liangwt/wrr/blob/master/scheduler.go
- https:github.com/hedzr/lb/blob/master/wrr/wrr.go

## Future Work

- Cost- and performance-based routing is out of scope for Glide MVP, but could be brought on later on
- Streaming model responses & model that accept streaming input (e.g. STT models) are out of scope right now, but will be considered in the future 
