# Pantheon

Model profiles for [Orchard](https://github.com/TheProxyCompany): chat templates, control tokens, and capability declarations for LLM inference on Apple Silicon.

## Why This Exists

Every model has its own prompt format, special tokens, and behavioral quirks. Hugging Face standardized the basics — `tokenizer_config.json` for special tokens, `chat_template` as inline Jinja for prompt format. But HF has no way to declare what a model can *do*. There's no capability manifest. No structured representation of tool calling format. No declaration of whether a model supports thinking, vision, or structured output. It's all baked into Jinja as imperative code, or left implicit.

This repo fills that gap. Each model profile is three files:

- **`control_tokens.json`** — Special tokens that delimit turns and sequences (BOS, EOS, role tags).
- **`capabilities.yaml`** — What the model can do and how to activate it (thinking, tool calling, vision, etc.).
- **`chat_template.jinja`** — Prompt assembly using tokens, capabilities, and messages.

Control tokens are the atoms. Capabilities declare behaviors. The chat template assembles them into a prompt.

## Architecture

This repository is the single source of truth for model-specific formatting across all Orchard SDKs. Each SDK includes this repo as a git submodule:

```
orchard-py/orchard/formatter/profiles/     → Pantheon
orchard-rs/profiles/                        → Pantheon
```

When you add a new model profile here, all SDKs gain support after updating their submodule.

## Structure

```
{model_family}/
├── control_tokens.json     # Special tokens — BOS, EOS, role tags
├── capabilities.yaml       # What the model can do and how to activate it
└── chat_template.jinja     # Prompt format using tokens, capabilities, and messages
```

See `_example/` for a fully documented reference implementation.

## capabilities.yaml

A declarative manifest of the model's capabilities. Each top-level key is a capability name, and each capability has typed fields:

| Field | Type | Description |
|-------|------|-------------|
| `native` | bool | Whether the model was trained on this capability |
| `tasks` | string[] | Task names this capability handles |
| `formats` | object[] | Output format variants. Each has `name` and optional `tokens` |
| `contains` | string[] | Child capabilities that activate within this one |
| `tokens` | dict | Special tokens that delimit this capability |
| `placeholders` | dict | Template strings substituted during prompt assembly |

### What `native` means

When `native: true`, the model was trained on this capability and knows how to use it. When `native: false`, Orchard grants the capability through constrained generation — the inference engine constrains the model's output to a valid structure (a tool call schema, a thinking format, etc.) and the model adapts. Both work. Native capabilities are higher quality; granted capabilities work with any instruction-tuned model.

This means a model that was never trained on tool calling can still make tool calls. A model with no thinking training can still produce structured reasoning. The capabilities file declares what's available and how it's delimited — the engine handles the rest.

### Example: Llama 3

```yaml
thinking:
  native: false
  tokens:
    start: "```thinking\n"
    end: "```\n"

tool_calling:
  native: true
  formats:
    - name: json
      tokens:
        start: "```json\n"
        end: "```\n"
    - name: python
      tokens:
        start: "<|python_tag|>"
        end: "<|eom_id|>"
```

Llama 3 was trained on tool calling (native) and supports two output formats — JSON and Python. Thinking is not native, but Orchard can grant it using markdown-fenced delimiters.

### Tool Calling Formats

Models use different formats for tool calls. The `formats` field declares which ones the model supports:

| Format | Payload | Models |
|--------|---------|--------|
| `json` | `{"name": "fn", "parameters": {...}}` | Llama 3.x, Phi-4, Granite, Gemma, most models |
| `python` | `tool_name.call(query="...")` | Llama 3.1 (built-in tools) |
| `hermes` | JSON object in XML tags | Hermes 2/3, Qwen 2.5, Granite 4.0 |
| `pythonic` | `[fn(arg="val")]` | Llama 4, OLMo 3 |
| `xml` | Full XML structure | Qwen3-Coder |

Each format can declare its own `tokens` for start/end delimiters. When a model wasn't trained on tool calling (`native: false`), the engine uses constrained generation to enforce the format — no training required.

## Adding a New Model

1. **Copy the example directory**
   ```bash
   cp -r _example/ {model_family}/
   ```

2. **Edit `control_tokens.json`**

   Define the model's special tokens — the delimiters that mark turns and sequences.

   | Field | Required | Description |
   |-------|----------|-------------|
   | `template_type` | Yes | Model family identifier (e.g., `"llama"`, `"gemma"`) |
   | `begin_of_text` | Yes | Beginning-of-sequence token |
   | `end_of_message` | Yes | End-of-turn token |
   | `end_of_sequence` | Yes | End-of-sequence token |
   | `roles.*` | Yes | Role-specific start/end tags and role names |

3. **Edit `capabilities.yaml`**

   Declare the model's capabilities using the schema described above.

4. **Edit `chat_template.jinja`**

   Customize the Jinja2 template for the model's prompt format. The template receives:
   - All `control_tokens.json` fields as variables (`begin_of_text`, `roles`, etc.)
   - All `capabilities.yaml` data as the `capabilities` object (e.g., `capabilities.thinking.tokens.start`)
   - Runtime variables: `interactions`, `add_generation_prompt`, `prefill`, `reasoning`, `task`, `tools`

5. **Submit a PR**

6. **After merge**, update the submodule in each SDK

## SDK Integration

| SDK | Template Engine | Submodule Path |
|-----|-----------------|----------------|
| orchard-py | Jinja2 | `orchard/formatter/profiles/` |
| orchard-rs | minijinja | `profiles/` |

## License

Apache-2.0
