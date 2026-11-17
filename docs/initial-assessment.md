# Initial Assessment & Technology Options

_Researched 2026-11-16. Based on the idea and StreamVibe research brief in [todo.md](../todo.md). Revised the same day after review; the [decisions](#decisions-after-review-2026-11-16) section replaces the original "bottom line"._

## Decisions after review (2026-11-16)

### v1 in one paragraph

A **phone web app** (a PWA, which can be added to the home screen) with a shutter button and hold-to-talk, talking to a **Python base station** on the LAN. The base station runs the agent queue, an OpenAI-compatible LLM client, DuckDuckGo web search, reverse image search only when needed, and Parakeet speech-to-text. Bins are predefined and the LLM picks the best match. Follow-up questions go to a "Needs attention" inbox. Storage is SQLite, and every raw capture is kept. StreamVibe is **not** a dependency.

### Capture

- **The user triggers every capture.** A button press, or later a spoken "snap." Automatically segmenting a continuous video of a busy room (frame differencing on a whole room) is not realistic and has no known precedent in real projects.
  - Correction to the original draft: "auto-capture when the item is held still" does have precedent (document-scanner apps, ID verification, barcode scanners), but only because the scene is controlled. It isn't suitable as the main method here.
- **Best-frame selection behind the trigger.** Keep a ring buffer of the last ~1–2 seconds of frames. On trigger, score them for sharpness (Laplacian variance) and keep the best 1–3. This is how phone camera "best shot" burst modes work, and it covers the delay between saying "snap" and the trigger firing.
- **Bulk video mode (phase 3).** This is the user's audio-first idea:
  1. Transcribe the audio.
  2. Have the LLM split the transcript into items, each with timestamps and a label.
  3. For each item's video segment, sample frames and run an open-vocabulary detector (YOLO-World, Grounding DINO or OWLv2).
  4. Score frames on detection confidence, image similarity to the label (CLIP/SigLIP) and sharpness, then keep the top frames.

  _(The precedents and model names in this section come from general knowledge; they weren't re-verified in this session.)_

### Interaction model: capture now, process in the background

| Mode | How it works | When |
|---|---|---|
| **A. Shutter + hold-to-talk** | Each item is its photos plus an optional voice note. The user does all the splitting. Items go into a processing queue. | **v1** |
| **B. Always-on mic + "snap" wake word** | The mic records continuously (AIRI-style). "Snap" takes the photo. Speech between snap N and snap N+1 belongs to item N, and the LLM tidies up the edges. | Phase 2, PC |
| **C. Passive streaming (StreamVibe-style)** | The model watches and decides on its own when to capture or speak. | Research track |

- **Follow-up questions go to a "Needs attention" inbox.** The agent marks an unclear item and writes a question, and the user answers whenever they like, by typing or by voice. Nothing blocks capture. Spoken questions are an option in mode B.
- **Rule:** ask only when the answer changes which bin the item goes in or what it's called.

### Platform: base station plus a thin client

- **Server (Python, e.g. FastAPI)** runs the database, photo storage, job queue, agent, LLM client, speech-to-text and chromedriver. A desktop process is needed anyway for chromedriver and the agent. It can run on the PC or on the LAN server next to the LLM.
- **Client: a mobile web page served by the server.** It uses the phone's camera and mic through the browser, so no APK is needed, and it can be tested in a desktop browser. The same page works on PC with a webcam.
  - Phone browsers only allow camera and mic access over HTTPS (except on localhost), so the LAN setup needs mkcert, a self-signed certificate, or Tailscale's HTTPS.
- **Native Android only later**, and only if a browser can't do what's needed (e.g. listening in the background).

### LLM, search and speech

- **LLM:** one OpenAI-compatible client, with the endpoint as a setting: Qwen3.8 Flash-Next on Strata (LAN server, the default), llama.cpp, or a cloud frontier model. Big models are assumed to run off-box, and the local GPU (24 GB) is used only for speech-to-text and any vision helpers. A fully offline fallback is just another endpoint with web search turned off.
- **Structured output:** every schema starts with a `reasoning` field before the answer fields (`name`, `tags`, `bin`, `confidence`, ...).
- **Speech:** reuse the pipeline from the user's modified AIRI (Silero VAD, a speech/non-speech classifier, Parakeet, end-of-speech detection). It has a known memory leak to fix. Details are pending a write-up from the Claude instance that manages that fork.
- **Web search:** free DuckDuckGo search, as in text-generation-webui (probably the `ddgs` package; to confirm against the AIRI integration).
- **Reverse image search:** the user's existing chromedriver-driven Google search, not SerpAPI.
  - Run it only when the LLM's confidence is low, and skip it for easy items.
  - Cache results by image hash.
  - One request at a time, with long backoff.
  - CAPTCHAs become a "Needs attention" item for the user to solve; the software never tries to get past one.
  - This automates Google outside its terms of service, which is accepted for personal-scale use.

### Bins

- **v1:** predefined bins, each with a title and a one-line description. The LLM picks the best match, and "suggest a new bin" is an allowed answer. No embeddings are needed for placement.
- **Later: bootstrap.** After a photo sweep of a sample of items, the LLM proposes an initial set of bins that splits them roughly evenly. The user approves or edits it.
- **Later: rebalance.** When a bin grows past a threshold, the LLM proposes sub-bins and which items move where. The user approves, then physically moves the items.
- Embeddings are for **search** ("where's my soldering iron?"), not placement.

### Scale

- Target: thousands to tens of thousands of items (eventually, a whole household). v1 stays simple.
- SQLite + FTS5 (full-text search) + `sqlite-vec` (embeddings); photos on disk.
- **Raw captures (photos + audio) are kept permanently**, so the whole inventory can be re-processed as models improve.

### Phases

1. **v1:** mode A, a phone/PC web client, the base-station queue, predefined bins, the "Needs attention" inbox.
2. **Phase 2:** mode B on PC (AIRI-derived always-on speech + "snap" wake word), spoken follow-up questions, bin bootstrap and rebalance.
3. **Phase 3:** bulk video mode (audio-first segmentation + open-vocabulary detection), and evaluating streaming/proactive models (mode C).

---

_The research below is from the original assessment and is kept for reference._

## Original bottom line (superseded)

The original draft proposed an automatic **event gate**: frame differencing plus a sharpness check to detect a still item, the voice detector to mark the end of speech, and structured `confidence`/`needs_clarification` output for clarifying questions. Review replaced this with user-triggered capture (see [Capture](#capture)). StreamVibe is still unreleased and not needed for v1.

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

Resolved in review:

- **GPU:** 24 GB on the PC, but large models run off-box (LAN server or cloud).
- **Camera:** a phone web client first; the same page works with a PC webcam.
- **Offline:** online is the intended use. A fully offline fallback is supported, but it's lower quality and web search is off.
- **Inventory size:** eventually thousands to tens of thousands of items.

Still open:

- Do bins already have labels or QR codes?
- Should the base station run on the PC or on the LAN server?
- Details of the AIRI speech pipeline (pending a write-up from the Claude instance that manages that fork).

## Sources

- [arXiv 2610.34662](https://arxiv.org/html/2610.34662) · [StreamVibe project page](https://stream-vibe.github.io/) · [HF papers](https://huggingface.co/papers/2610.34662) · [hyper.ai](https://hyper.ai/en/papers/2610.34662) · [AI Weekly](https://aiweekly.co/alerts/google-streamvibe-benchmarks-video-agents-on-evolving-intent)
- [Parakeet card](https://huggingface.co/nvidia/parakeet-unified-en-0.6b) · [Parakeet discussion](https://huggingface.co/nvidia/parakeet-unified-en-0.6b/discussions/2)
- [Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B) · [Runpod on Qwen3.8](https://www.runpod.io/blog/qwen3-8-27b-on-runpod-the-theory-behind-frontier-class-agentic-coding-that-fits-on-a-single-24gb-worker) · [Ollama qwen3.5](https://ollama.com/library/qwen3.5)
- [Cekura on Silero VAD](https://www.cekura.ai/blogs/what-is-silero-vad)
- [OneStreamer](https://github.com/MCG-NJU/OneStreamer) · [Proact-VL](https://arxiv.org/html/2603.03447) · [LiveStarPro](https://arxiv.org/html/2606.17798v1) · [ROMA](https://arxiv.org/html/2601.10323v1) · [Awesome-VLM-Streaming-Video](https://github.com/ydyhello/Awesome-VLM-Streaming-Video)
- [Grocy vs Homebox](https://selfhosting.sh/compare/grocy-vs-homebox/)
