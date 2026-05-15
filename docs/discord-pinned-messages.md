# Discord #scenema-audio — Pinned Messages

Pin these in order (first message = top of pins).

---

## Message 1: Welcome

**Welcome to #scenema-audio**

Scenema Audio is our open source expressive speech generation model. It generates speech with emotional acting, singing, crying, laughing, and environmental audio from a text prompt that describes not just what to say but how to say it. Built on an audio diffusion transformer extracted from LTX 2.3.

Scenema Audio is one component of the Scenema AI filmmaking platform, where it runs in production alongside other speech engines including Gemini Flash 3 TTS. The platform handles full video and film production end to end. The audio model is a small piece of that stack that we decided to open source.

This channel is for questions, feedback, and discussion about the open source model. For questions about the full Scenema AI platform, use #general.

**Links**
- GitHub: https://github.com/ScenemaAI/scenema-audio
- HuggingFace: https://huggingface.co/ScenemaAI/scenema-audio
- Demos and samples: https://scenema.ai/audio

---

## Message 2: Try it free

**Try it without installing anything**

Sign up for free at https://scenema.ai and tell the director agent you want to generate voiceover. It will handle the rest. There are also instructions on the website.

Voice design is available on the free tier. Voice cloning is a paid feature (Starter or Creator subscription).

---

## Message 3: Self-host

**Run it yourself**

Clone the repo to any cloud GPU VM (Vast.ai, RunPod, Lambda, etc.) and run `docker compose up`. That bootstraps the entire pipeline including model downloads. The built-in Gradio web UI supports all model features: generation, voice design, voice cloning, action tags, scene-aware SFX, and multilingual output.

Any NVIDIA GPU with 16 GB+ VRAM works. The default config uses INT8 + NF4 quantization and automatically offloads models on smaller cards. 24 GB+ keeps everything resident for faster generation.

See the GitHub README for SSH port forwarding instructions to access the UI from your local browser.

MIT license for the code. Model weights are under the LTX Community License.
