---
Initial Idea:

Inventory app:
 - Show each item on camera, Optional verbal description, Optional target bin
 - AI identifies item, Web search if necessary, Integrate verbal description, Generate tags, Decide target bin if user hasn't
 - Silero Voice Activity Detection, NVIDIA Parakeet Unified EN 0.6B STT, qwen3.8 w/ vision, 

---
**StreamVibe — follow-up research brief**

**What it is:** StreamVibe, from "Proactive Video Agents for Evolving User Intent" (arXiv:2610.34662, Oct 2026, Google-led). Three components: a benchmark, the StreamVibe-VL streaming model, and a hierarchical memory harness. It watches a continuous video stream, infers the user's implicit intent as it evolves, and decides when/whether to intervene — selective frame attention rather than whole-feed ingestion.

**What we know:** As of Nov 1 2026, paper and project page only; no public code, weights, or API. A community roundup listed code/models as expected around Nov 9.

**What I need now:**
1. Has code, weights, or an API been released as of today (Nov 16 2026)? Direct links to any repo, HuggingFace, or hosted endpoint.
2. Hardware requirements — can it run on a single consumer GPU (~24 GB VRAM), or is it server-class only?
3. License and terms for local/commercial use.
4. Actual input: does it attach to a live webcam/RTSP stream, or is it offline-video-only in the current release?
5. Any related open-source projects that solve the same "select which frames matter from a live feed" problem — alternatives if StreamVibe itself isn't usable.

**Downstream goal:** I'm building an inventory app. User shows items to a webcam one at a time; the system identifies each item, generates tags, decides storage bin placement, and occasionally asks clarifying questions. StreamVibe was planned as the proactive layer sitting in front of a vision LLM, deciding which frames to pass through and when to prompt the user. A workable alternative to StreamVibe for that specific role would also be welcome.

---
