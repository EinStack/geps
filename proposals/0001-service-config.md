---
MEP: 1
Title: Service Config
Discussion: N/A
Implementation: https://github.com/modelgateway/glide/pull/37
---

# Service Config

## Abstract

Glide is going to have many configs that control both functional and non-functional aspects of the service work. 
This GEP defines how that config is going to be set, structured and maintained across the service lifetime.

## Motivation

Glide is a gateway that provides a bunch of configurations to set, manage, and maintain. 
This ensures flexibility needed to meet various needs of our end users.

For example:

- **Telemetry Configs**: Log Output Format, Min Log Level, OTEL Configs
- **HTTP Server Configs**: Is enabled?, IP and Port to bind, max body size, CORS, etc.
- **SSL/TLS Configs:** Ciphers, Cert Paths, etc.
- **LLM Provider Configs**: API Keys, Default Parameters, Timeouts, Connection Pool Configs, etc.
- **Routing and Load Balancing Configs**: Balancing Strategy, Routing Weights, Provider Pool Definitions, etc.
- **Caching Configs**: Redis Connection Configs, TTLs, etc.
- **gRPC Server Configs**: Is enabled?, IP and Port to bind, etc.
- etc.

The configuration may be versioned via git to track and audit all changes. 
At the same time, some configs are sensitive data (notably, LLM API Keys) 
and it must not be provided directly in the config (otherwise, they would be leaked in git).

As any other config, users should be able to easily apply new changes to the config. 
The API keys may need to be rotated from time to time.

Finally, the more configs we expose, the harder may become to understand the overall config
and find the right setting to adjust. So the config structure is an important thing here to eliminate this fatigue.

### Requirements

- R1: Config format must be readable
- R2: Config format must allow comments
- R3: Config format must be `git diff`-able and versioned
- R4: Config must be easy to understand and reason about
- R5: Config must allow to include sensitive values from environment variables
- R6: Config may allow to include sensitive values from other plain files
- R7: Config should be easy to change and operate on like "hitless" reloads 

## Design

There are basically a few sources of configurations possible:
- via Glide CLI params & args
- via environment variables
- via a separate config file
- via a combination of CLI, environment variables, and/or config file

Setting complex nested configs via CLI or environment variables is usually a tedious error-prone process.
So Glide will try to leverage the file as config approach as long as possible.

### The Config Source

Glide config is represented as a YAML file (R1, R2, R3). 
YAML is a human-friendly version of JSON that supports comments and some useful magic like (including environment variables).
Go has good support for the YAML format.

### The Config Structure

The config structure uses Glide's internal logical structure that we are going to communicate in Glide docs (R4).
So learning more about how Glide works should give enough context to understand the config. 
The opposite should be true as well, so knowing how the config looks will give some understanding about Glide's overall design.
This is a route that the OTEL Collector config takes.

Glide should allow you to pull values of configs from environment variables (see the config example below).
This helps to keep sensitive information (R5) out of the config file that avoids leakages.

### Config Ops

Glide should be able to automatically detect changes to the config (especially, our domain-related configs like routing) and
reload its internal representation together with the related state (R7). 
In Kubernetes, configs are going to be represented as a configmap mounted into Glide pods. 
On configmap change, the mounted file changes as well. So we need to watch for the file changes there. 
Outside of Kubernetes, we may support config reloading on SIGHUP signals.

### Config Example

The most minimalistic possible config setup would look like this:

```yaml
routes:
  language:
    - id: openai-pool
      providers:
        - id: openai-boring
          openai:
              model: gpt-3.5-turbo
              api_key: ${env:OPENAI_API_KEY}
              default_params:
                temperature: 0
```

Here is a rich config example to get better sense of the config structure:

```yaml
telemetry:
  logging:
    level: info
    encoding: console # console, json
  # other configs

api:
  http:
    listen_addr: 0.0.0.0:7685
    max_body_size: "2Mi"
    tls:
      ca_path:
      cert_path:
    # other configs

  grpc:
    listen_addr: 0.0.0.0:7686
    tls:
      ca_path:
      cert_path:
    # other configs

routes:
  language:
    - id: openai-pool
      strategy: priority # round-robin, weighted-round-robin, priority, least-latency, priority, etc.
      models:
        - id: openai-boring
          openai: # anthropic, azureopenai, gemini, other providers we support
            model: gpt-3.5-turbo
            api_key: ${env:OPENAI_API_KEY}
            default_params:
              temperature: 0

        - id: anthropic-boring
          anthropic:
            model: claude-2
            apiKey: ${env:ANTHROPIC_API_KEY}
            default_params: # set the default request params
              temperature: 0

    - id: latency-critical-pool
      strategy: least-latency
      models:
        - id: openai-boring
          timeout_ms: 200
          openai:
            model: gpt-3.5-turbo
            api_key: ${env:OPENAI_API_KEY}
         
        - id: anthropic-boring
          timeout_ms: 200
          anthropic:
            api_key: ${env:ANTHROPIC_API_KEY}

    - id: anthropic
      strategy: weighted-round-robin
      models:
        - id: openai
          weight: 30
          openai:
            api_key: ${env:OPENAI_API_KEY}
        - id: anthropic
          weight: 70
          anthropic:
            api_key: ${env:ANTHROPIC_API_KEY}

    - id: ab-test1
      strategy: weighted-round-robin
      models:
        - id: openai-gpt4
          weight: 50
          openai:
            model: gpt-4
            api_key: ${env:OPENAI_API_KEY}
            default_params:
              temperature: 0.7

        - id: openai-chatgpt
          weight: 50
          openai:
            model: gpt-3.5-turbo
            api_key: ${env:OPENAI_API_KEY}
            default_params:
              temperature: 0.7

        # we want to use OpenAI only in this A/B test, but that's bad resiliency wise
        # so add Anthropic just in case OpenAI is down
        - id: anthropic-fallback
          weight: 0
          anthropic:
            model: claude-2
            api_key: ${env:ANTHROPIC_API_KEY}

# in case of speach-to-text models
#  transcribers:
#    - ...

# in case of text-to-speach models
#  synthesizers:
#    - ...

# in case of embeddings API
#  embeddings:
#    - ...
```

### References

- [OTEL Collector: Config Sample](https://opentelemetry.io/docs/collector/configuration/)
- [MLFlow AI Gateway: Configuration](https://mlflow.org/docs/latest/llms/gateway/index.html#ai-gateway-configuration)
- [etcd: Configuration Options](https://etcd.io/docs/v3.4/op-guide/configuration/)
- [etcd: Config Sample](https://github.com/etcd-io/etcd/blob/release-3.4/etcd.conf.yml.sample)
- [Portkey: Config Sample](https://docs.portkey.ai/portkey-docs/portkey-features/ai-gateway/load-balancing)
- [HashiCorp Vault: Configurations](https://developer.hashicorp.com/vault/docs/configuration)
- [Prometheus AlertManager: Config](https://prometheus.io/docs/alerting/latest/configuration/)
- [Kong: Config](https://github.com/Kong/kong/blob/master/kong.conf.default)

## Alternatives Considered

- **Using JSON as config format**. We could still use JSON as secondary config format (OTEL Collector does that), but it doesn't developer-friendly enough to be our primary config format (fails at R1, R2, and it's not too strong with R3)

## Future Work

- If our users want that, we could consider supporting more than one config formats like JSON or TOML.
- Pull configs from sources other than filesystem (like fetching it via HTTP). Needs some signals from community to figure out if this is useful
