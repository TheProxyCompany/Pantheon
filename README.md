# orchard-models

Model profiles for [Orchard](https://github.com/TheProxyCompany): chat templates and control tokens for LLM inference on Apple Silicon.

## Architecture

This repository is the single source of truth for model-specific formatting across all Orchard SDKs. Each SDK includes this repo as a git submodule:

```
orchard-py/orchard/formatter/profiles/     → orchard-models
orchard-rs/profiles/                        → orchard-models
=```

When you add a new model profile here, all SDKs gain support after updating their submodule.

## Structure

Each model family has its own directory containing two files:

```
{model_family}/
├── control_tokens.json
└── chat_template.jinja
```

See `_example/` for a fully documented reference implementation.

## Adding a New Model

1. **Copy the example directory**
   ```bash
   cp -r _example/ {model_family}/
   ```

2. **Edit `control_tokens.json`**

   Fill in the model's special tokens. Leave fields empty (`""`) when the model doesn't use that feature.

   | Field | Required | Description |
   |-------|----------|-------------|
   | `template_type` | Yes | Model family identifier (e.g., `"llama"`, `"gemma"`) |
   | `begin_of_text` | Yes | BOS token |
   | `end_of_message` | Yes | End of turn/message token |
   | `end_of_sequence` | Yes | EOS token |
   | `start_image_token` | No | Image placeholder start (vision models) |
   | `end_image_token` | No | Image placeholder end (vision models) |
   | `thinking_start_token` | No | Reasoning mode start token |
   | `thinking_end_token` | No | Reasoning mode end token |
   | `tool_calling.*` | No | Tool/function calling delimiters |
   | `roles.*` | Yes | Role-specific start/end tags |

3. **Edit `chat_template.jinja`**

   Customize the Jinja2 template for the model's expected prompt format. The template receives all control tokens as context variables.

4. **Submit a PR**

5. **After merge**, update the submodule in each SDK

## control_tokens.json Reference

```json
{
    "template_type": "llama",

    "begin_of_text": "<|begin_of_text|>",
    "end_of_message": "<|eom_id|>",
    "end_of_sequence": "<|eot_id|>",

    "start_image_token": "",
    "end_image_token": "",

    "thinking_start_token": "",
    "thinking_end_token": "",

    "tool_calling": {
        "call_start": "",
        "call_end": "",
        "section_start": "",
        "section_end": "",
        "name_separator": ""
    },

    "roles": {
        "system": {
            "role_name": "system",
            "role_start_tag": "<|start_header_id|>",
            "role_end_tag": "<|end_header_id|>\n\n"
        },
        "user": { ... },
        "agent": { ... },
        "tool": { ... }
    }
}
```

### Tool Calling Tokens

For models with function/tool calling support, configure the `tool_calling` section:

| Field | Description | Example (DeepSeek) |
|-------|-------------|-------------------|
| `call_start` | Start of individual tool call | `<<\|tool_call_begin\|>>` |
| `call_end` | End of individual tool call | `<<\|tool_call_end\|>>` |
| `section_start` | Wraps all tool calls (optional) | `<<\|tool_calls_begin\|>>` |
| `section_end` | End of section wrapper | `<<\|tool_calls_end\|>>` |
| `name_separator` | Between function name and args | `<<\|tool_sep\|>>` |

Leave all fields empty for models without structured tool calling.

## SDK Integration

| SDK | Template Engine | Submodule Path |
|-----|-----------------|----------------|
| orchard-py | Jinja2 | `orchard/formatter/profiles/` |
| orchard-rs | minijinja | `profiles/` |

## License

Apache-2.0
