# Moshi

The Rust Orchard client resolves this profile and sends stateful 24 kHz mono PCM
sessions to PIE. PIE owns the native C++ Moshi temporal/depth transformers, Mimi
codec, ARC reference encoder and Carbon streams. It is an audio architecture,
not a text chat template. The current native implementation accepts the pinned
MoshiRAG safetensors checkpoint; the base Q8 descriptors remain available as asset
metadata but are rejected by the native Rust client until implemented.

The `duplex.checkpoints` revisions fix the three model assets used together.
Moshi generates its own speech and time-aligned text. That text must never be
presented as a transcript of the microphone. A separate local language model
can ground the speech stream with asynchronous reference updates while the
microphone stream continues. Forced speech is available only when the engine's
ready event advertises it. A new interruption epoch invalidates queued audio.

Code: https://github.com/kyutai-labs/moshi (MIT/Apache-2.0).
Weights: https://huggingface.co/kyutai/moshiko-candle-q8 (CC-BY-4.0).
Credit: Alexandre Défossez, Laurent Mazaré, Manu Orsini, Amélie Royer,
Patrick Pérez, Hervé Jégou, Edouard Grave and Neil Zeghidour, Kyutai (2024).
