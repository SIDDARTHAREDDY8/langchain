---
type: Factory
title: Chat Model Initialization with init_chat_model
description: Factory function for instantiating chat models from provider strings with unified configuration and runtime model switching.
tags: [chat-models, factory-pattern, initialization, model-parameters, configuration, provider-registry]
verified:
  - by: openwiki/0.5.0
    at: 2026-09-20T08:24:05.454Z
sources:
  - id: openwiki-source-c479d4fffee5cf62576699e4
    resource: repo://libs/langchain_v1/langchain/chat_models/base.py
generated: { by: "openwiki/0.5.0", at: "2026-09-20T08:24:05.454Z" }
---

## Overview

`init_chat_model` is a factory function that creates chat model instances from a unified interface. It centralizes model instantiation across all supported provider integrations (OpenAI, Anthropic, Bedrock, Google Vertex AI, etc.), handles parameter routing to provider-specific constructors, and supports runtime model configuration via LangChain's `Runnable` configuration system.

The factory accepts a **model name with optional provider prefix** (e.g., `"openai:gpt-4"`, `"anthropic:claude-opus-4-7"`), infers the provider when unspecified, retrieves the provider's integration package, and instantiates the corresponding chat model class with translated kwargs.

**Core responsibilities:**
- Accept and parse model identifiers with or without provider prefixes
- Infer providers from model name prefixes using heuristics
- Dynamically import provider integration packages and classes
- Route provider-specific kwargs to provider constructors
- Support both fixed (non-configurable) and runtime-configurable (switch model/provider at invoke time) initialization modes
- Route requests through LangSmith gateway when provider is `langsmith`

## Location

**File**: `repo://libs/langchain_v1/langchain/chat_models/base.py`

**Public export**: `repo://libs/langchain_v1/langchain/chat_models/__init__.py#L5`

## Function Signature

```python
def init_chat_model(
    model: str | None = None,
    *,
    model_provider: str | None = None,
    configurable_fields: Literal["any"] | list[str] | tuple[str, ...] | None = None,
    config_prefix: str | None = None,
    **kwargs: Any,
) -> BaseChatModel | _ConfigurableModel
```

**Parameters:**

- `model` (`str | None`): Model identifier, optionally with provider prefix (`"provider:model-name"`). If `None`, returns a configurable model that requires model name at runtime. Examples:
  - `"openai:gpt-5.5"` (explicit prefix)
  - `"gpt-5.5"` (inferred as OpenAI)
  - `None` (configurable at runtime)

- `model_provider` (`str | None`): Provider name as an alternative to prefix format. Used when provider is dynamic or needs to be independently configurable. Normalized to lowercase with underscores (e.g., `"azure-openai"` → `"azure_openai"`).

- `configurable_fields` (`Literal["any"] | list[str] | tuple[str, ...] | None`): 
  - `None`: No fields are configurable (fixed model, default if `model` is specified)
  - `"any"`: All parameters become configurable at runtime (⚠️ security: includes api_key, base_url)
  - `list[str] | tuple[str, ...]`: Specified parameter names (e.g., `("temperature", "max_tokens")`) are configurable
  - Defaults to `("model", "model_provider")` if `model` is `None`

- `config_prefix` (`str | None`): Optional namespace prefix for runtime config keys. Used when multiple configurable models exist in the same application. Config is accessed via `config["configurable"]["{config_prefix}_{param}"]`.

- `**kwargs`: Provider-specific parameters passed to the underlying chat model's constructor. Common parameters:
  - `temperature` (`float`): Randomness control (0–1 or provider-specific range)
  - `max_tokens` (`int`): Maximum output tokens
  - `timeout` (`float`): Request timeout in seconds
  - `max_retries` (`int`): Retry attempt limit
  - `base_url` (`str`): Custom API endpoint (OpenAI-compatible)
  - `rate_limiter` (`BaseRateLimiter`): Rate limiting instance
  - Provider-specific: `openai_api_key`, `anthropic_api_url`, `bedrock_region`, etc.

**Returns:**

- `BaseChatModel`: A fixed (non-configurable) chat model when `configurable_fields` is `None` (the default if `model` is specified)
- `_ConfigurableModel`: A wrapper runnable that defers model instantiation until `invoke`/`stream` is called with configuration, enabling runtime model selection

**Raises:**

- `TypeError`: If `model` is not a string (e.g., a model object is passed)
- `ValueError`: If provider cannot be inferred or is not supported
- `ImportError`: If the provider's integration package is not installed

## Model Name Parsing and Provider Inference

### Explicit Provider Prefix

If a colon (`:`) divides the model string and the prefix is a registered provider, it is extracted:

```python
init_chat_model("openai:gpt-5.5")     # provider='openai', model='gpt-5.5'
init_chat_model("anthropic:claude-opus-4-7")  # provider='anthropic', model='claude-opus-4-7'
```

### Bare Model Name with Inference

Without an explicit prefix, `_attempt_infer_model_provider` uses case-insensitive prefix matching:

| Model Prefix | Inferred Provider |
|---|---|
| `gpt-`, `o1`, `o3`, `chatgpt`, `text-davinci` | `openai` |
| `claude` | `anthropic` |
| `command` | `cohere` |
| `accounts/fireworks` | `fireworks` |
| `gemini` | `google_vertexai` (⚠️ deprecated default; changing to `google_genai` in next major release) |
| `amazon.`, `anthropic.`, `meta.` | `bedrock` |
| `mistral`, `mixtral` | `mistralai` |
| `deepseek` | `deepseek` |
| `grok` | `xai` |
| `sonar` | `perplexity` |
| `solar` | `upstage` |

**Example:**

```python
init_chat_model("gpt-4")  # inferred as openai:gpt-4
init_chat_model("claude-sonnet-4-5-20250929")  # inferred as anthropic
```

If inference fails and `model_provider` is not provided, a `ValueError` lists supported providers and suggests the documentation.

## Built-in Provider Registry

The `_BUILTIN_PROVIDERS` dictionary maps provider names to module paths, class names, and instantiation functions. Each entry is a tuple: `(module_path, class_name, creator_func)`.

**All Built-in Providers** (repo://libs/langchain_v1/langchain/chat_models/base.py#L56-L97):

| Provider | Package | Class | Module | Notes |
|---|---|---|---|---|
| `openai` | `langchain-openai` | `ChatOpenAI` | `langchain_openai` | |
| `anthropic` | `langchain-anthropic` | `ChatAnthropic` | `langchain_anthropic` | |
| `azure_openai` | `langchain-openai` | `AzureChatOpenAI` | `langchain_openai` | |
| `azure_ai` | `langchain-azure-ai` | `AzureAIOpenAIApiChatModel` | `langchain_azure_ai.chat_models` | Submodule import |
| `google_vertexai` | `langchain-google-vertexai` | `ChatVertexAI` | `langchain_google_vertexai` | |
| `google_genai` | `langchain-google-genai` | `ChatGoogleGenerativeAI` | `langchain_google_genai` | |
| `google_anthropic_vertex` | `langchain-google-vertexai` | `ChatAnthropicVertex` | `langchain_google_vertexai.model_garden` | Google's Anthropic model on Vertex |
| `anthropic_bedrock` | `langchain-aws` | `ChatAnthropicBedrock` | `langchain_aws` | Bedrock-hosted Anthropic |
| `bedrock` | `langchain-aws` | `ChatBedrock` | `langchain_aws` | Generic Bedrock models |
| `bedrock_converse` | `langchain-aws` | `ChatBedrockConverse` | `langchain_aws` | Bedrock Converse API |
| `cohere` | `langchain-cohere` | `ChatCohere` | `langchain_cohere` | |
| `deepseek` | `langchain-deepseek` | `ChatDeepSeek` | `langchain_deepseek` | |
| `fireworks` | `langchain-fireworks` | `ChatFireworks` | `langchain_fireworks` | |
| `groq` | `langchain-groq` | `ChatGroq` | `langchain_groq` | |
| `huggingface` | `langchain-huggingface` | `ChatHuggingFace` | `langchain_huggingface` | Uses `from_model_id()` |
| `ibm` | `langchain-ibm` | `ChatWatsonx` | `langchain_ibm` | Uses `model_id=` param |
| `litellm` | `langchain-litellm` | `ChatLiteLLM` | `langchain_litellm` | |
| `meta` | `langchain-meta` | `ChatMetaModel` | `langchain_meta` | |
| `mistralai` | `langchain-mistralai` | `ChatMistralAI` | `langchain_mistralai` | |
| `nvidia` | `langchain-nvidia-ai-endpoints` | `ChatNVIDIA` | `langchain_nvidia_ai_endpoints` | |
| `ollama` | `langchain-ollama` | `ChatOllama` | `langchain_ollama` | Fallback to `langchain_community` |
| `openrouter` | `langchain-openrouter` | `ChatOpenRouter` | `langchain_openrouter` | |
| `perplexity` | `langchain-perplexity` | `ChatPerplexity` | `langchain_perplexity` | |
| `together` | `langchain-together` | `ChatTogether` | `langchain_together` | |
| `upstage` | `langchain-upstage` | `ChatUpstage` | `langchain_upstage` | |
| `xai` | `langchain-xai` | `ChatXAI` | `langchain_xai` | |
| `baseten` | `langchain-baseten` | `ChatBaseten` | `langchain_baseten` | |
| `langsmith` | `langchain-openai` | `ChatOpenAI` | `langchain_openai` | Routes via LangSmith gateway |

**Design notes:**

- The registry contains 28 built-in providers. It is **not exhaustive**. Unlisted providers can still be used if their integration package is installed, but model name inference will not work; `model_provider` must be specified explicitly.
- Most entries use the standard `_call` creator function, which directly instantiates the class.
- Special creators: `huggingface` uses `from_model_id(model_id=...)`, `ibm` uses `model_id=...`, `langsmith` wraps instantiation with gateway configuration.

## Parameter Mapping and Creator Functions

The `_get_chat_model_creator` function retrieves the provider's creator function and returns a partially-applied callable:

```python
@functools.lru_cache(maxsize=len(_BUILTIN_PROVIDERS))
def _get_chat_model_creator(provider: str) -> Callable[..., BaseChatModel]:
    # Look up provider in registry
    pkg, class_name, creator_func = _BUILTIN_PROVIDERS[provider]
    # Import module and get class
    module = _import_module(pkg, class_name)
    cls = getattr(module, class_name)
    # Return partial with class bound
    return functools.partial(creator_func, cls=cls)
```

**Standard Creator (`_call`)**:

```python
def _call(cls: type[BaseChatModel], **kwargs: Any) -> BaseChatModel:
    return cls(**kwargs)
```

Forwards all kwargs directly to the provider's `__init__`.

**Special Creators**:

- **HuggingFace**: `lambda cls, model, **kwargs: cls.from_model_id(model_id=model, **kwargs)`
  - Uses the class method `from_model_id` instead of direct instantiation
  - `model` parameter becomes `model_id=`

- **IBM**: `lambda cls, model, **kwargs: cls(model_id=model, **kwargs)`
  - Maps `model` to `model_id=` parameter in constructor

- **LangSmith Gateway** (`_init_langsmith`):
  - Calls `_apply_gateway_config` to inject gateway credentials and base URL from environment (or `LANGSMITH_GATEWAY` URL)
  - Sets `use_responses_api=True` to enable compatibility with gateway
  - Falls back to `LANGSMITH_API_KEY` if `LANGSMITH_GATEWAY_API_KEY` not set

## Fixed Model Initialization (Non-Configurable)

When `model` is a non-None string and `configurable_fields` is `None` (the default), `init_chat_model` immediately invokes a helper function to parse the model identifier, look up the provider's creator function, and instantiate the chat model. The resulting `BaseChatModel` is immediately ready for `invoke()`, `stream()`, or other operations without requiring runtime configuration.

**Flow:**

```python
# Fixed (non-configurable) - returns ready-to-use BaseChatModel
model = init_chat_model("openai:gpt-5.5", temperature=0.7)
model.invoke("Hello")  # Works immediately
```

1. `_parse_model()` extracts provider and model name from the input string
2. `_get_chat_model_creator()` retrieves the provider's cached creator function
3. The creator function instantiates the class with merged kwargs
4. `BaseChatModel` instance is returned

## Runtime-Configurable Model Initialization

When `model` is `None` or `configurable_fields` is not `None`, `init_chat_model` returns a `_ConfigurableModel` instance instead of an actual chat model. This allows deferred, runtime-switchable initialization.

**Flow:**

```python
# Configurable - returns _ConfigurableModel
model = init_chat_model(configurable_fields=("model", "temperature"))

# Model is instantiated later with runtime config
model.invoke(
    "Hello",
    config={"configurable": {"model": "gpt-5.5", "temperature": 0.9}}
)
```

### _ConfigurableModel Structure and Lifecycle

`_ConfigurableModel` is a `Runnable` that wraps the actual model initialization. It implements the following lifecycle:

**Initialization** – stores configuration parameters and flags:
- `_default_config`: Dictionary of parameter defaults (e.g., `{"model": "gpt-5.5", "temperature": 0.7}`)
- `_configurable_fields`: Either `"any"` (all fields configurable) or a whitelist of field names
- `_config_prefix`: String prefix (e.g., `"model_a"`) for isolating config keys when multiple configurable models exist in the same application
- `_queued_declarative_operations`: List of method calls (e.g., `bind_tools`, `with_structured_output`) to apply after model instantiation

**Declarative Operations** – intercepted and queued:
- When `bind_tools()` or `with_structured_output()` is called on a `_ConfigurableModel`, `__getattr__` intercepts the call instead of delegating to the underlying model (which doesn't exist yet)
- The operation and its arguments are recorded in `_queued_declarative_operations`
- A new `_ConfigurableModel` is returned with the operation queued, enabling method chaining without mutation

**Runtime Invocation** – deferred model instantiation:
- On `invoke()`, `stream()`, or other execution methods, `_model(config)` is called to construct the actual model
- `_model_params(config)` extracts runtime overrides from `config["configurable"]`, applying the `_config_prefix` filter and `_configurable_fields` whitelist
- Overrides are merged over `_default_config`
- `_init_chat_model_helper()` instantiates the actual `BaseChatModel` using merged parameters
- Queued declarative operations are applied in order to the instantiated model
- The decorated model is delegated to for the actual invocation

**Example: Full Lifecycle**

```python
# 1. Create configurable model with default OpenAI
model = init_chat_model(
    "openai:gpt-5.5",
    configurable_fields=("model", "temperature"),
    config_prefix="my",
    temperature=0.5
)

# 2. Queue declarative operation
model_with_tools = model.bind_tools([tool1, tool2])
# model itself is unchanged; the queue is in model_with_tools

# 3. Invoke with runtime override
response = model_with_tools.invoke(
    "Hello",
    config={
        "configurable": {
            "my_model": "anthropic:claude-opus",
            "my_temperature": 0.8
        }
    }
)
# At invocation:
# - model_with_tools._model(config) extracts {"model": "anthropic:claude-opus", "temperature": 0.8}
# - Merges with defaults: {"model": "anthropic:claude-opus", "temperature": 0.8, ...}
# - Instantiates ChatAnthropic with merged params
# - Applies bind_tools to the instance
# - Returns the decorated model, which handles the actual invoke
```

### Configuration Key Mapping with config_prefix

When `config_prefix` is set, configuration keys are namespaced to avoid collisions. The prefix is automatically stripped during lookup:

```python
model = init_chat_model(
    "gpt-5.5",
    configurable_fields=("temperature", "max_tokens"),
    config_prefix="llm1"  # Becomes "llm1_" internally
)

model.invoke(
    "Hello",
    config={
        "configurable": {
            "llm1_temperature": 0.9,  # Stripped to "temperature"
            "llm1_max_tokens": 200,   # Stripped to "max_tokens"
        }
    }
)
```

### Whitelisting Configurable Fields

By default, `configurable_fields="any"` allows any field to be overridden at runtime, including security-sensitive ones like `api_key` and `base_url`. For production use, explicitly whitelist fields to prevent unintended runtime modification:

```python
# ✓ Safe: only temperature and max_tokens can change
model = init_chat_model(
    "gpt-5.5",
    configurable_fields=("temperature", "max_tokens")
)

# ✗ Dangerous: api_key can be redirected at runtime
model = init_chat_model(
    "gpt-5.5",
    configurable_fields="any"
)
```

## Module Import and Error Handling

The `_import_module()` function dynamically imports provider integration packages with helpful error messages:

```python
def _import_module(module: str, class_name: str) -> ModuleType:
    try:
        return importlib.import_module(module)
    except ImportError as e:
        pkg = module.split(".", maxsplit=1)[0].replace("_", "-")
        msg = f"Initializing {class_name} requires the {pkg} package. Please install it with `pip install {pkg}`"
        raise ImportError(msg) from e
```

**Fallback Logic:**

- For `ollama`, if `langchain_ollama` is not installed, it tries `langchain_community.chat_models` as a fallback for backward compatibility
- For other providers, a clear error message is raised immediately

## Model-to-Provider Inference

The `_attempt_infer_model_provider()` function uses case-insensitive prefix matching to infer the provider from a model name:

```python
def _attempt_infer_model_provider(model_name: str) -> str | None:
    model_lower = model_name.lower()
    
    # OpenAI models (including newer models and aliases)
    if any(model_lower.startswith(pre) for pre in ("gpt-", "o1", "o3", "chatgpt", "text-davinci")):
        return "openai"
    
    # Anthropic models
    if model_lower.startswith("claude"):
        return "anthropic"
    
    # ... (other patterns)
    
    return None  # Cannot infer
```

**Special Case – Gemini (google_vertexai vs google_genai):**

When a model name starts with `gemini`, `_attempt_infer_model_provider()` returns `"google_vertexai"` but emits a `DeprecationWarning`. Future major releases will change the default to `"google_genai"`. Users are encouraged to pass `model_provider='google_genai'` or use the prefix form `"google_genai:gemini-1.5"` to lock in the desired behavior.

## Batching Behavior

`_ConfigurableModel` implements `batch()`, `abatch()`, `batch_as_completed()`, and `abatch_as_completed()` with an optimization: if only one or zero configs are provided, it delegates directly to the underlying model's batch implementation. If multiple distinct configs are provided, it falls back to the base `Runnable.batch()` which parallelizes the invocations.

## Common Patterns

### 1. Fixed Model – Most Common Use Case

```python
from langchain.chat_models import init_chat_model

# Simple: infer OpenAI from model name
model = init_chat_model("gpt-5.5", temperature=0.7)
model.invoke("What is 2+2?")

# Explicit provider prefix
model = init_chat_model("anthropic:claude-opus", temperature=0.8)
response = model.invoke("Hello")

# With custom parameters
model = init_chat_model(
    "openai:gpt-5.5",
    temperature=0.5,
    max_tokens=100,
    timeout=30,
    api_key="sk-..."
)
```

### 2. Configurable Model – No Default

```python
# Defer all decisions to runtime
model = init_chat_model()

# Later, with configuration
response = model.invoke(
    "Question",
    config={
        "configurable": {
            "model": "gpt-5.5",
            "model_provider": "openai"
        }
    }
)
```

### 3. Configurable Model – With Default

```python
# Start with OpenAI, but allow runtime switching
model = init_chat_model(
    "openai:gpt-5.5",
    configurable_fields="any",
    temperature=0.7
)

# Use default
model.invoke("Question 1")

# Switch provider and adjust temperature
model.invoke(
    "Question 2",
    config={
        "configurable": {
            "model": "claude-opus",
            "model_provider": "anthropic",
            "temperature": 0.9
        }
    }
)
```

### 4. Isolated Configurable Models with config_prefix

```python
# Two independent configurable models in the same chain
summarizer = init_chat_model(
    configurable_fields=("model", "temperature"),
    config_prefix="summarizer",
    temperature=0.3
)

translator = init_chat_model(
    configurable_fields=("model", "temperature"),
    config_prefix="translator",
    temperature=0.5
)

chain = (
    {"text": lambda x: x}
    | {"summary": summarizer, "translation": translator}
)

chain.invoke(
    "Long document",
    config={
        "configurable": {
            "summarizer_model": "gpt-5.5",
            "translator_model": "claude-opus"
        }
    }
)
```

### 5. Binding Tools to Configurable Models

```python
from pydantic import BaseModel, Field

class WeatherTool(BaseModel):
    """Get weather for a location"""
    location: str = Field(..., description="City, state")

model = init_chat_model(
    configurable_fields=("model", "model_provider")
)

# Chain operations together
model_with_tools = model.bind_tools([WeatherTool])

# Invoke with configuration
response = model_with_tools.invoke(
    "What's the weather in NYC?",
    config={"configurable": {"model": "gpt-5.5"}}
)
```

## Related Topics

- [Chat Models](/openwiki/chat-models.md) – Base classes and model concepts
- [Runnables](/openwiki/runnables.md) – Configuration and invocation patterns
- [Quickstart](/openwiki/quickstart.md) – Getting started with LangChain
