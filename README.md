# Pantheon

Model profiles for [Orchard](https://github.com/TheProxyCompany): chat templates, control tokens, and capability declarations for LLM inference on Apple Silicon.

## Why This Exists

Every model has its own prompt format, special tokens, and behavioral quirks. Hugging Face standardized the basics — `tokenizer_config.json` for special tokens, `chat_template` as inline Jinja for prompt format. But HF has no way to declare what a model can *do*. There's no capability manifest. No structured representation of tool calling format. No declaration of whether a model supports thinking, vision, or structured output. It's all baked into Jinja as imperative code, or left implicit.

This repo fills that gap. Each model profile contains model data plus its prompt
template:

- **`control_tokens.json`** — Special tokens that delimit turns and sequences (BOS, EOS, role tags).
- **`capabilities.yaml`** — What the model can do and how to activate it (thinking, tool calling, vision, etc.).
- **`generation.yaml`** — Model generation defaults and recommended sampling profiles.
- **`chat_template.jinja`** — The prompt assembly template for the model.
- **`templates/*.jinja`** — Optional prompt assembly variants when one model version needs multiple templates.

Control tokens are the atoms. Capabilities declare behaviors. Generation declares
sampling policy. Template variants assemble them into prompts.

## Structure

```
{model_family}/
├── control_tokens.json     # Special tokens — BOS, EOS, role tags
├── capabilities.yaml       # What the model can do and how to activate it
├── generation.yaml         # Default, recommended, and deterministic sampling configs
├── chat_template.jinja     # Default prompt format
└── templates/              # Optional, only for multiple prompt formats
    └── reasoning.jinja
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
| `json` | `{"name": "fn", "parameters": {...}}` | Llama 3.x, Granite, Gemma, most models |
| `python` | `tool_name.call(query="...")` | Llama 3.1 (built-in tools) |
| `hermes` | JSON object in XML tags | Hermes 2/3, Qwen 2.5, Granite 4.0 |
| `pythonic` | `[fn(arg="val")]` | Llama 4, OLMo 3 |
| `xml` | Full XML structure | Qwen3-Coder |

Each format can declare its own `tokens` for start/end delimiters. When a model wasn't trained on tool calling (`native: false`), the engine uses constrained generation to enforce the format — no training required.

## generation.yaml

A declarative manifest of the model's generation settings. Keep sampling policy out of `capabilities.yaml`.

| Field | Type | Description |
|-------|------|-------------|
| `tokens` | dict | Optional BOS, PAD, and EOS token ids from the model generation config |
| `default` | dict | Default generation settings from the model's generation config |
| `recommended` | dict | Recommended quality/headline-eval sampling settings |
| `deterministic` | dict | Engineering lane for greedy, reproducible perf/comparison runs |

Common sampling keys are `temperature`, `top_p`, `top_k`, `min_p`, `presence_penalty`, `frequency_penalty`, and `repetition_penalty`.

## Adding a New Model

1. **Copy the example directory**
   ```bash
   cp -r _example/ {model_family}/
   ```

2. **Edit `control_tokens.json`**

   Define the model's special tokens — the delimiters that mark turns and sequences.

   | Field | Required | Description |
   |-------|----------|-------------|
   | `template_type` | No | Template variant, usually `default`; use named variants only with `templates/*.jinja` |
   | `model_types` | No | Model type strings that should load this profile |
   | `begin_of_text` | Yes | Beginning-of-sequence token |
   | `end_of_message` | Yes | End-of-turn token |
   | `end_of_sequence` | Yes | End-of-sequence token |
   | `roles.*` | Yes | Role-specific start/end tags and role names |

3. **Edit `capabilities.yaml`**

   Declare the model's capabilities using the schema described above.

4. **Edit `generation.yaml`**

   Declare model generation defaults, recommended quality settings, and deterministic engineering settings.

5. **Edit `chat_template.jinja`**

   Customize the Jinja2 template for the model's prompt format. Add named
   variants such as `templates/reasoning.jinja` only when one model version
   needs multiple prompt formats. The template receives:
   - All `control_tokens.json` fields as variables (`begin_of_text`, `roles`, etc.)
   - All `capabilities.yaml` data as the `capabilities` object (e.g., `capabilities.thinking.tokens.start`)
   - Runtime variables: `interactions`, `add_generation_prompt`, `prefill`, `reasoning`, `task`, `tools`

6. **Submit a PR**

7. **After merge**, update the submodule in each SDK

## Earlier turns: what every chat template must do

The engine reuses work between turns through a prefix cache keyed by token ids, so the
next prompt has to start with the last prompt followed by what the model generated. Two
rules make that hold. `orchard-rs` checks both for every profile
(`test_a_replayed_reply_extends_the_prompt_it_answered` and
`test_reasoning_stays_in_the_conversation_and_earlier_turns_do_not_move`), and `orchard-py`
runs the same checks under Jinja2.

1. **Define the generation prompt once**, as a macro that takes the thinking flag, and use
   it both for the turn being asked for and for earlier turns.
2. **A replayed turn.** When an assistant message has `generated`, the client is sending
   the exact token ids the model produced. Render the generation prompt for
   `message.generated_thinking`, then `message.generated`, then the turn end, and nothing
   else: no content, no tool calls, they are inside the ids. Everything before it must
   render as it did when that turn was the one being asked for.
3. **A turn that arrives as text** keeps its reasoning on every later turn, in the model's
   own format. Read `reasoning_content`, then `reasoning`, then `thinking` when it is a
   string. Never look at later messages to decide how an earlier one renders: the
   conversation so far must be a prefix of the longer conversation.

A model that switches thinking at the head of the prompt (Gemma 4 writes `<|think|>` into
the system turn) re-renders the head in the next request's mode; the replayed turn still
keeps the generation prompt it answered. See `nemotron_h/chat_template.jinja` for the
shortest complete example.

## Images and audio: what the template has to write itself

A Hugging Face chat template usually writes one placeholder per image or audio clip and
leaves the rest to the model's *processor*, which replaces the placeholder with the soft
tokens and whatever the model was trained to see around them. Orchard has no processor
step. The client finds the placeholder in the rendered prompt, cuts the text there, and
sends the media as its own layout segment; the engine tokenizes each piece of text on its
own and puts the soft tokens in between. So everything the processor would add besides
the soft tokens has to be text in the template:

| Model | The processor expands the placeholder to | The template writes |
|-------|------------------------------------------|---------------------|
| Gemma 4 | `<\|image>` + soft tokens + `<image\|>` | `<\|image><\|image\|><image\|>` (the middle token is the placeholder the client removes) |
| Gemma 3 | `\n\n<start_of_image>` + soft tokens + `<end_of_image>\n\n` | the same text without the soft tokens (`<start_of_image>` is the placeholder and stays) |

When porting a template, read the processor's `replace_image_token` /
`replace_audio_token` (or its equivalent) as well as the Jinja, and compare token ids with
`processor.apply_chat_template(..., tokenize=True)` for a prompt with one and with two
images.

Declare a placeholder that must not reach the engine as text under the capability's
`placeholders` map (`vision.placeholders.image`, `audio.placeholders.audio`), or as
`vision.tokens.placeholder` for images. The clients only remove what is declared there;
an undeclared placeholder stays in the prompt text while the layout counts the text
without it, and the engine refuses the request.

## SDK Integration

| SDK | Template Engine | Submodule Path |
|-----|-----------------|----------------|
| orchard-py | Jinja2 | `orchard/formatter/profiles/` |
| orchard-rs | minijinja | `profiles/` |

## License

Apache-2.0
