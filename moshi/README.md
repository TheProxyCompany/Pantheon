# Moshi

Orchard runs Kyutai Moshi's temporal transformer, depth transformer and Mimi codec
through the pinned official Rust implementation on Candle Metal. This profile
is consumed by both Orchard clients. It is an audio architecture with stateful
24 kHz mono PCM sessions, not a text chat template.

The `duplex.checkpoints` revisions fix the three model assets used together.
Moshi generates its own speech and time-aligned text. That text must never be
presented as a transcript of the microphone. A separate local language model
can control the speech stream through the session's `speak` command while the
microphone stream continues. A new interruption epoch invalidates queued audio.

Code: https://github.com/kyutai-labs/moshi (MIT/Apache-2.0).
Weights: https://huggingface.co/kyutai/moshiko-candle-q8 (CC-BY-4.0).
Credit: Alexandre Défossez, Laurent Mazaré, Manu Orsini, Amélie Royer,
Patrick Pérez, Hervé Jégou, Edouard Grave and Neil Zeghidour, Kyutai (2024).
