# Initial Assessment & Technology Options

_Researched 2026-11-16. Based on the idea and StreamVibe research brief in [todo.md](../todo.md)._

## Bottom line

The plan is feasible on one consumer GPU. The main risk is **StreamVibe**: it still isn't released, and the app doesn't need it to ship.

The user *deliberately* holds up one item at a time, so we don't need a learned model to guess intent from a continuous video feed. A simple **event gate** can do that job:

- **Item present and still:** detect when an item is in frame and the image has stopped changing (frame differencing plus a sharpness check such as Laplacian variance), then capture the 1–3 best frames.
- **End of speech:** use the voice activity detector to know when the user stops talking, then attach the transcript.
- **Clarifying questions:** have the vision model return structured output with fields like `confidence` and `needs_clarification`.

Build the gate behind a clean interface. If StreamVibe or something similar ships and proves useful, it can be swapped in later.

## StreamVibe status (as of 2026-11-16)

1. **Released?** No. The [project page](https://stream-vibe.github.io/) shows "Code *soon*" and "Hugging Face *soon*", plus the line "Under internal approval, will release in 7 days." Its [Hugging Face papers page](https://huggingface.co/papers/2610.34662) has no linked models, datasets or Spaces. The expected ~Nov 9 release has passed and nothing has appeared.
2. **Hardware:** Not stated. The paper only compares against Qwen3-VL-8B and Qwen3.5-9B, and a chart labels "Qwen3.5-9B (base)". If that is the base model, it would fit in 24 GB, but that is an inference, not a stated fact.
3. **License:** None stated for code or models. The paper itself is under the standard arXiv license.
4. **Input:** A causal stream at 1 FPS: the model only sees past and current frames. With no released code, live webcam/RTSP support is unknown. The "harness" handles memory and state; the model itself decides when to respond.
5. **Alternatives:** see [Streaming / proactive alternatives](#streaming--proactive-alternatives-to-streamvibe).

## Technology options by component

| Component | Current pick | Alternatives / notes |
|---|---|---|
| **Voice activity detection** | Silero VAD (v6 line; latest reportedly 6.2.x) | No reason to change. Tiny, fast, runs on CPU. |
| **Speech-to-text** | [Parakeet-unified-en-0.6b](https://huggingface.co/nvidia/parakeet-unified-en-0.6b) | Good choice. Runs through NeMo (inference only), configurable 160–2080 ms latency. For short utterances already cut by the VAD, offline mode is fine and streaming isn't needed. For very low-latency streaming, NVIDIA points to `nemotron-speech-streaming-en-0.6b`. A community GGUF port exists if you want to avoid NeMo. Whisper-turbo via faster-whisper is a familiar fallback. |
| **Vision model** | "qwen3.8 w/ vision" | The only open Qwen3.8 vision model is [Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B) (~28B dense, released Aug 2026). It fits on a 24 GB card only quantized ([Unsloth GGUF/NVFP4](https://unsloth.ai/docs/models/qwen3.8)), and the KV cache gets tight. [Qwen3.5](https://ollama.com/library/qwen3.5) has vision in every size from 0.8B to 122B. **Suggestion:** Qwen3.5-9B for a fast first pass, Qwen3.8-27B for hard cases. |
| **Model serving** | — | Ollama (easiest), llama.cpp, vLLM (better throughput), LM Studio. |
| **Web search fallback** | — | Self-hosted SearXNG, or a search API such as Brave or Tavily. Reverse image search is the hard part; a practical workaround is searching on text the model reads (brand names, model numbers, barcodes). |
| **Inventory backend** | — | Build our own (SQLite plus embeddings for bin matching), or use [Homebox](https://selfhosting.sh/compare/grocy-vs-homebox/) as the store: it already models items, locations, labels and QR codes, and has a REST API. Grocy fits groceries and consumables better. |
| **Bin decision** | — | Compare the new item's embedding to existing bin contents, then have the model choose between the top few candidates. |

## Streaming / proactive alternatives to StreamVibe

The field is crowded in 2026, but most projects are research code that works on video files. Licenses and weight availability haven't been checked for most of them.

- **[OneStreamer](https://github.com/MCG-NJU/OneStreamer)** (Oct 2026): Apache-2.0 code, built on Qwen3-VL-4B, ~10 GB GPU memory. Weights listed as "public access pending"; takes video files only, so webcam capture would be on us.
- **[Proact-VL](https://arxiv.org/html/2603.03447)** (Microsoft): a learned head that decides when to speak, aimed at live companions.
- **STRIDE, ThinkStream, SyncPro, [LiveStarPro](https://arxiv.org/html/2606.17798v1) (8B):** all decide when to respond on a stream. See [Awesome-VLM-Streaming-Video](https://github.com/ydyhello/Awesome-VLM-Streaming-Video).
- **Foresight:** training-free, wraps a frozen Qwen3-VL-8B. Easiest to experiment with.
- **MiniCPM-o 4.5 / Qwen3.8-Omni-Flash-Realtime (Qwen-Live-Harness):** "omni" models that take audio and video together in real time. One of these could replace VAD + STT + vision model in a single model. Worth investigating, but it's unconfirmed whether the Qwen weights are open or API-only.

## Open questions

- What GPU / how much VRAM? (24 GB assumed.)
- Desktop webcam only, or should a phone work as the camera too?
- Must it work fully offline? That decides whether web search is allowed.
- How big will the inventory get, and do bins already have labels or QR codes?

## Sources

- [arXiv 2610.34662](https://arxiv.org/html/2610.34662) · [StreamVibe project page](https://stream-vibe.github.io/) · [HF papers](https://huggingface.co/papers/2610.34662) · [hyper.ai](https://hyper.ai/en/papers/2610.34662) · [AI Weekly](https://aiweekly.co/alerts/google-streamvibe-benchmarks-video-agents-on-evolving-intent)
- [Parakeet card](https://huggingface.co/nvidia/parakeet-unified-en-0.6b) · [Parakeet discussion](https://huggingface.co/nvidia/parakeet-unified-en-0.6b/discussions/2)
- [Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B) · [Runpod on Qwen3.8](https://www.runpod.io/blog/qwen3-8-27b-on-runpod-the-theory-behind-frontier-class-agentic-coding-that-fits-on-a-single-24gb-worker) · [Ollama qwen3.5](https://ollama.com/library/qwen3.5)
- [Cekura on Silero VAD](https://www.cekura.ai/blogs/what-is-silero-vad)
- [OneStreamer](https://github.com/MCG-NJU/OneStreamer) · [Proact-VL](https://arxiv.org/html/2603.03447) · [LiveStarPro](https://arxiv.org/html/2606.17798v1) · [ROMA](https://arxiv.org/html/2601.10323v1) · [Awesome-VLM-Streaming-Video](https://github.com/ydyhello/Awesome-VLM-Streaming-Video)
- [Grocy vs Homebox](https://selfhosting.sh/compare/grocy-vs-homebox/)
