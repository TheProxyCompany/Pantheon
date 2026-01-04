# orchard-models

Model profiles for [Orchard](https://github.com/TheProxyCompany): chat templates and control tokens for LLM inference on Apple Silicon.

## Architecture

This repository is the single source of truth for model-specific formatting across all Orchard SDKs. Each SDK includes this repo as a git submodule:

```
orchard-py/orchard/formatter/profiles/     → orchard-models
orchard-rs/profiles/                        → orchard-models
orchard-swift/Resources/profiles/           → orchard-models
```

When you add a new model profile here, all SDKs gain support after updating their submodule.

## Structure

Each model family has its own directory containing two files:

```
{model_family}/
├── control_tokens.json
└── chat_template.jinja
```

### control_tokens.json

Defines model-specific special tokens:

```json
{
    "template_type": "llama",
    "begin_of_text": "<|begin_of_text|>",
    "end_of_sequence": "<|eot_id|>",
    "roles": {
        "user": {
            "role_name": "user",
            "role_start_tag": "<|start_header_id|>",
            "role_end_tag": "<|end_header_id|>\n\n"
        },
        "agent": { ... },
        "system": { ... }
    }
}
```

### chat_template.jinja

Jinja2 template for formatting conversations. Receives control tokens as context and renders the full prompt string.

## Adding a new model

1. Create a directory named after the model family (e.g., `mistral/`)
2. Add `control_tokens.json` with the model's special tokens
3. Add `chat_template.jinja` with the conversation formatting template
4. Submit a PR
5. After merge, update the submodule in each SDK

## SDK Integration

| SDK | Template Engine | Submodule Path |
|-----|-----------------|----------------|
| orchard-py | Jinja2 | `orchard/formatter/profiles/` |
| orchard-rs | minijinja | `profiles/` |
| orchard-swift | Stencil (planned) | `Resources/profiles/` |

## License

Apache-2.0
