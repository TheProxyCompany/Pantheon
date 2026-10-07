# NVIDIA Nemotron 3 Diarization

This is the September 23, 2026 eight-speaker model. Orchard runs the pinned
NVIDIA NeMo-Speech.cpp implementation with its actual AOSC speaker cache and
FIFO state, using the official SHA-256-verified Q8 GGUF checkpoint. It does
not substitute the older four-speaker Sortformer model.

Predictions are eight anonymous arrival-order speaker probabilities at 10 ms
resolution. The recommended default buffers 640 ms of audio (six 80 ms frames
plus two right-context frames), before compute time. Input may be the same
24 kHz microphone stream Moshi consumes; the native runtime resamples it.

A speaker channel is a session-local acoustic identity, not a person's name.
Only the caller's confirmed mapping should connect it to a Life Map entity.
Overlapping channels remain separate and probability/confidence is retained.

Sources:
- https://huggingface.co/nvidia/Nemotron-3-Diarization
- https://github.com/NVIDIA/NeMo-Speech.cpp

Runtime license: Apache-2.0. Model license: NVIDIA Open Model License
(OpenMDW 1.1), distributed with the model at the source above.
