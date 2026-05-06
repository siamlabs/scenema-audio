# Scenema Audio

**Zero-shot expressive voice cloning and speech generation.**

Every existing text-to-speech system converts words into sound, but none of them perform. Speech that merely pronounces words correctly is functionally useless for filmmaking, audiobooks, or any context where the emotional delivery carries as much meaning as the words themselves. Scenema Audio generates speech with intention, pacing, breath control, and emotional arcs that shift within a single generation, all from a text prompt that describes not just what to say but how to say it.

Built on an audio diffusion transformer extracted from [LTX 2.3](https://github.com/Lightricks/LTX-2)'s 22B parameter audiovisual model, it learned how people actually sound in real scenes: angry, laughing, whispering, crying, singing, exhausted, terrified.

## Quick Start

### Docker (Recommended)

```bash
docker pull scenemaai/scenema-audio:latest

docker run --gpus all -p 8210:8210 \
  -e HF_TOKEN=your_huggingface_token \
  scenemaai/scenema-audio:latest
```

### Generate Audio

```bash
curl -X POST http://localhost:8210/generate \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "<speak voice=\"A warm, clear male voice with a slight British accent. Measured, thoughtful pacing.\" gender=\"male\">The old lighthouse had stood on the cliff for over a century, its beam cutting through the fog like a blade of light.</speak>",
    "seed": 42
  }' \
  --output output.wav
```

### Voice Design (Preview a Voice)

```bash
curl -X POST http://localhost:8210/generate \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "<speak voice=\"A young woman with a smoky jazz-singer quality. Low register, intimate.\" gender=\"female\">The city never really sleeps. It just closes its eyes and pretends for a while.</speak>",
    "mode": "voice_design"
  }' \
  --output voice_preview.wav
```

### Zero-Shot Voice Cloning

Provide a few seconds of reference audio. The model generates expressive speech from the prompt, then transfers the reference voice's identity onto the performance. The reference does not need to contain any particular emotion, because the emotion comes from the generation stage.

```bash
curl -X POST http://localhost:8210/generate \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "<speak voice=\"Gravelly male voice, fast talking, rough.\" gender=\"male\"><action>He completely loses it, shouting</action>What the fuck are you waiting for?!</speak>",
    "reference_voice_url": "https://example.com/calm-reference.wav",
    "seed": 42
  }' \
  --output cloned_angry.wav
```

Any voice can perform any emotion, even if that voice has never been recorded in that emotional state. The reference only provides identity. The performance comes from the prompt.

## Prompt Format

```xml
<speak voice="VOICE_DESCRIPTION" gender="male|female"
       scene="OPTIONAL_SCENE" language="OPTIONAL_LANG_CODE"
       shot="closeup|wide|scene">
  <action>Performance direction.</action>
  Speech text here.
  <sound>Environmental audio event.</sound>
  More speech.
</speak>
```

### Attributes

| Attribute | Required | Default | Description |
|-----------|----------|---------|-------------|
| `voice` | Yes | | Detailed voice description. Drives vocal quality, emotion, accent, age, timbre, delivery style. The more specific and theatrical, the better. |
| `gender` | Yes | | `"male"` or `"female"`. Controls pronoun assignment in the compiled prompt sent to the diffusion model. |
| `scene` | No | | Environmental context. Conditions the ambient audio environment around the speech (rain, office hum, crowd noise). |
| `language` | No | `"en"` | Language code. The model supports major world languages with native-sounding output. |
| `shot` | No | `"closeup"` | Controls SFX prominence. `"closeup"`: speech-focused, SFX minimal. `"wide"`: environment + speech. `"scene"`: maximum environmental audio, SFX reinforced. |

### Child Elements

| Element | Description |
|---------|-------------|
| Text nodes | The actual speech content. Write natural prose. |
| `<action>` | Performance directions that shape HOW the speech is delivered. Not spoken aloud. Stage directions for the diffusion model: emotional shifts, physical delivery, pacing cues, breath control. |
| `<sound>` | Environmental audio events generated alongside the speech. Thunder cracks, doors slamming, rain starting. Only effective in `wide` or `scene` shot modes. |

### Voice Description

The `voice` attribute is the primary control for the entire output. Be specific and theatrical:

```xml
<!-- Weak -->
<speak voice="A man speaking" gender="male">...</speak>

<!-- Strong -->
<speak voice="Male, mid 60s. Deep baritone with gravel. Slight Southern American inflection.
Worn but warm. Nostalgic, firelight cadence. The voice of someone who has seen too much
and chosen kindness anyway." gender="male" scene="Fireside, night, crickets">...</speak>
```

### Action Tags

Action tags are the primary tool for controlling emotional performance. Place them between speech segments to direct delivery shifts:

```xml
<speak voice="Middle-aged man, warm but weathered." gender="male">
  <action>Calm, almost casual. Staring at his hands.</action>
  I used to think I had all the time in the world.
  <action>Voice tightens. Swallows. Fighting to stay composed.</action>
  Then one Tuesday morning, the doctor said three words that changed everything.
  <action>Long pause. Deep breath. When he speaks again, his voice is raw but steady.</action>
  And I realized... I hadn't called my son in six months.
  <action>Voice breaks on the last word. Clears throat. Forces a half-laugh.</action>
  Funny how that works, isn't it?
</speak>
```

Describe what the speaker is DOING and FEELING, not what the audio should sound like. Combine physical and emotional cues for richer performance.

## API Reference

### POST /generate

#### Request Body

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `prompt` | string | **required** | `<speak>` XML string. See Prompt Format above. |
| `mode` | string | `"generate"` | `"generate"` for full pipeline with chunking. `"voice_design"` for a single 15-second voice sample (no chunking, useful for previewing a voice description). |
| `reference_voice_url` | string | `null` | URL to reference audio (WAV or MP3) for zero-shot voice cloning. A few seconds of clean speech is sufficient. The reference provides identity only; emotional performance comes from the prompt. |
| `background_sfx` | bool | `false` | Keep generated environmental sound effects in the output. When `false`, non-vocal audio is stripped via MelBandRoFormer vocal separation. Set to `true` when using `shot="scene"` or `shot="wide"` with `<sound>` tags. |
| `validate` | bool | `true` | Enable Whisper speech validation. Each generated chunk is transcribed by faster-whisper and compared against expected text. If word match ratio falls below the threshold, the chunk is regenerated with extended duration and a new seed (up to 3 retries), keeping the best result. Adds <1s per chunk on GPU. Disable for faster generation when prompt reliability is not critical. |
| `seed` | int | `-1` | Generation seed. `-1` for random. Fixed seeds produce deterministic output for the same prompt and configuration. |
| `pace` | float | `1.5` | Duration allocation multiplier. Higher values give the model more time per chunk, resulting in slower, more deliberate speech. Lower values produce faster speech. The default 1.5x accounts for LTX's naturally slower speaking pace compared to real-time speech. |
| `min_match_ratio` | float | `0.90` | Whisper validation threshold. Minimum word match ratio (0.0 to 1.0) between generated audio transcription and expected text. Only used when `validate` is `true`. Lower values accept more pronunciation variance. |
| `skip_vc` | bool | `false` | Skip voice conversion (SeedVC) post-processing entirely. When `true`, no voice identity transfer or cross-chunk voice consistency normalization is applied. Useful for single-chunk generations where the voice description alone is sufficient. |
| `vc_steps` | int | `25` | SeedVC diffusion steps. More steps produce higher-quality voice identity transfer at the cost of processing time. Range: 10-50. |
| `vc_cfg_rate` | float | `0.5` | SeedVC classifier-free guidance rate. Controls how strongly the target voice identity is applied. Higher values produce stronger identity transfer but may reduce naturalness. Range: 0.0-1.0. |

#### Response

Returns WAV audio (48kHz stereo) directly in the response body with `Content-Type: audio/wav`.

### POST /generate (Async Mode)

For production deployments with webhook callbacks and S3/R2 upload:

```json
{
  "job_id": "my-job-123",
  "webhook_url": "https://your-server.com/callback",
  "upload_url": "https://s3-presigned-upload-url...",
  "input": {
    "prompt": "<speak voice=\"...\" gender=\"male\">...</speak>",
    "seed": 42,
    "validate": true,
    "pace": 1.5
  }
}
```

Returns immediately with `{"job_id": "my-job-123", "status": "processing"}`. When complete, uploads the WAV to the presigned URL and calls the webhook with:

```json
{
  "job_id": "my-job-123",
  "status": "succeeded",
  "metadata": {
    "duration_s": 12.4,
    "sample_rate": 48000,
    "processing_ms": 8200,
    "mode": "generate",
    "seed": 42,
    "has_reference_voice": false
  }
}
```

## Capabilities

### Emotional Acting

Emotional state shifts within a single generation. Action tags function as stage directions at specific points in the script.

```xml
<speak voice="A man on the edge. Explosive rage. Italian-American inflection."
       gender="male" scene="A dimly lit office, late at night">
  <action>He stands up slowly, voice dangerously low</action>
  You come into my house, you eat my food, and then you got the nerve
  to tell me how to run my business.
  <action>Voice rising, finger pointing</action>
  I built this thing from nothing while you were sitting on your ass.
</speak>
```

### Singing

```xml
<speak voice="A soulful female alto singing with raw emotion.
Blues-jazz phrasing, slight vibrato on sustained notes."
gender="female">
  <action>Soft piano intro, she takes a breath.</action>
  I heard love was a losing game, played it once and lost the same.
</speak>
```

### Child Voices

```xml
<speak voice="A six-year-old girl, bright and excited, speaking fast
with breathless enthusiasm. Slight lisp on S sounds."
gender="female">
  Mommy look! There is a rainbow and it goes all the way across the whole sky!
</speak>
```

### Scene-Aware Audio (Voice + Environment)

Set `shot="scene"` and `background_sfx: true` to generate speech with environmental audio in the same diffusion pass.

```xml
<speak voice="Male, mid 40s. Weathered. Urgent, projecting over wind."
       gender="male" scene="Open dock in a thunderstorm, heavy rain"
       shot="scene">
  <sound>Heavy rain and wind howling</sound>
  <action>He shouts over the storm</action>
  Get the lines! She is pulling loose!
  <sound>Thunder cracks overhead</sound>
  Move! I said move!
</speak>
```

### Multilingual

The model supports major world languages with native fluency. Set the `language` attribute and write the voice description to match.

```xml
<speak voice="Female, mid 70s. Soft alto. Native French speaker, Parisian accent.
Warm like wool blankets. Unhurried." gender="female"
scene="Cozy bedroom, lamplight" language="fr">
  <action>Elle s'assied au bord du lit</action>
  Alors, mon petit. Tu veux que je te raconte l'histoire du renard
  qui a trompé la lune?
</speak>
```

### Long-Form Narration

Text is automatically split at sentence boundaries using [Kokoro](https://github.com/hexgrad/kokoro) phoneme-level duration estimation. Voice identity is maintained across chunks via A2V latent conditioning.

```xml
<speak voice="An elderly storyteller with a weathered knowing voice.
Deep baritone, slow deliberate pacing."
gender="male">
  Many years later, as he faced the firing squad, Colonel Aureliano Buendia
  was to remember that distant afternoon when his father took him to discover ice.
  At that time Macondo was a village of twenty adobe houses, built on the bank
  of a river of clear water that ran along a bed of polished stones, which were
  white and enormous, like prehistoric eggs.
</speak>
```

## Hardware Requirements

### Minimum: 24 GB VRAM (RTX 4090, RTX A5000)

INT8 audio transformer + NF4 Gemma quantization. Everything fits on GPU.

```bash
docker run --gpus all -p 8210:8210 \
  -e AUDIO_CKPT=/app/models/scenema-audio-transformer-int8.safetensors \
  -e GEMMA_QUANTIZE=nf4 \
  scenemaai/scenema-audio:latest
```

### Recommended: 48 GB VRAM (A6000 Ada, A40, L40S)

Full bf16 precision, all models resident on GPU. Best quality, fastest generation.

```bash
docker run --gpus all -p 8210:8210 \
  scenemaai/scenema-audio:latest
```

### VRAM Configurations

| Configuration | VRAM Used | Gemma Encode | Notes |
|--------------|-----------|-------------|-------|
| INT8 audio + NF4 Gemma | ~18 GB | 0.2s/chunk* | Best for 24 GB cards |
| bf16 audio + bf16 Gemma (streaming) | ~16 GB GPU + 24 GB RAM | 7.5s/chunk | Low VRAM, slow encode |
| bf16 audio + bf16 Gemma (resident) | ~40 GB | 0.2s/chunk* | Best quality, needs 48 GB |

*With [SageAttention 2.2.0](https://github.com/thu-ml/SageAttention) installed. Without SA2, GPU-resident Gemma encode is ~5.5s/chunk.

## Performance

Benchmarked on NVIDIA RTX 4090, 100-word passage (~55 seconds of audio, 4 chunks):

| Configuration | Total Time | Real-Time Factor |
|--------------|-----------|-----------------|
| bf16 + bf16 (CPU streaming) | 83s | 0.66x |
| INT8 + bf16 (CPU streaming) | 66s | 0.83x |
| INT8 + NF4 (all GPU) | 35s | 1.57x |
| INT8 + NF4 + SageAttention 2 | 35s | 1.57x |

## Pipeline Architecture

```
XML prompt (voice description + scene + stage directions + text)
  |
  v
[Text Splitting] -----------> Sentence boundaries via Kokoro, ~15s max per segment
  |
  v
[Gemma 3 12B Encode] -------> Text conditioning (per segment)
  |
  v
[8-Step Diffusion] ---------> Audio latent generation
  |                            Voice continuity via A2V latent conditioning between segments
  v
[Audio Decode] --------------> Waveform
  |
  v
[MelBandRoFormer] ----------> Vocal separation (strips SFX unless background_sfx=true)
  |
  v
[SeedVC] -------------------> Voice identity transfer (when reference_voice_url provided
  |                            or multi-chunk for cross-chunk consistency)
  v
Output WAV (48kHz stereo)
```

### Key Design Decisions

**Kokoro for duration estimation.** Kokoro TTS (82M params, CPU) provides phoneme-level duration estimates. The chunker splits text at sentence boundaries when accumulated Kokoro estimates exceed 15 seconds (with a configurable `pace` multiplier for LTX's naturally slower speaking pace). No word counting.

**15-second chunk cap.** The model was trained on 20-second clips, but quality degrades (repetition, pronunciation failure) beyond ~15 seconds. The 15s cap ensures consistent quality.

**Voice continuity across segments.** The tail of each segment's audio is encoded and used as a voice reference for the next segment. This maintains consistent voice identity across arbitrarily long outputs without requiring a separate voice embedding model.

**Zero-shot voice cloning.** A2V latent conditioning gets about 60% of the way to matching a reference voice. SeedVC post-processing brings it to full identity transfer. No training, no enrollment, no voice database.

**Emotion and identity are independent controls.** The voice description drives the emotional performance. The reference audio drives the voice identity. For maximum emotional range with a cloned voice, use a strong character archetype in the voice description and let the reference audio handle identity.

**INT8 quantization.** Per-channel INT8 reduces the transformer from 9.8 GB to 4.9 GB with no measurable quality difference, enabling generation on consumer GPUs.

## Model Checkpoints

Hosted on HuggingFace: [ScenemaAI/scenema-audio](https://huggingface.co/ScenemaAI/scenema-audio)

| File | Size | Description |
|------|------|-------------|
| `scenema-audio-transformer.safetensors` | 9.8 GB | Audio diffusion transformer (bf16) |
| `scenema-audio-transformer-int8.safetensors` | 4.9 GB | Audio diffusion transformer (INT8, identical quality) |
| `scenema-audio-pipeline.safetensors` | 6.7 GB | Audio VAE decoder + vocoder + text projection |
| `scenema-audio-vae-encoder.safetensors` | 42.7 MB | Audio VAE encoder for reference voice encoding |

### How the Checkpoint Was Made

The audio transformer was extracted from the full LTX 2.3 22B audiovisual model by:

1. Loading the distilled checkpoint
2. Fusing the audio LoRA weights into the base model
3. Stripping all video-specific layers (video attention, video feed-forward, video-to-audio cross-attention)
4. Replacing stripped layers with `nn.Identity()` placeholders
5. Saving the remaining audio-only weights

The pipeline checkpoint contains the audio VAE decoder, vocoder, text embedding projection, and embeddings processor connectors extracted from the full 46 GB distilled pipeline.

## Building from Source

```bash
git clone https://github.com/ScenemaAI/scenema-audio.git
cd scenema-audio

docker build \
  --build-arg HF_TOKEN=your_huggingface_token \
  -t scenema-audio:latest .

docker run --gpus all -p 8210:8210 scenema-audio:latest
```

### Docker Build Arguments

| Arg | Default | Description |
|-----|---------|-------------|
| `HF_TOKEN` | required | HuggingFace token with Gemma 3 access |

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `AUDIO_CKPT` | `/app/models/scenema-audio-transformer.safetensors` | Path to audio transformer checkpoint |
| `PIPELINE_CKPT` | `/app/models/scenema-audio-pipeline.safetensors` | Path to pipeline checkpoint |
| `GEMMA_ROOT` | `/app/models/gemma-3-12b-it` | Path to Gemma 3 12B model directory |
| `GEMMA_QUANTIZE` | (empty) | Set to `nf4` for NF4 Gemma quantization |
| `GPU_PORT` | `8210` | HTTP service port |

## Limitations

- **Pronunciation**: The model occasionally garbles complex multi-syllable words and proper nouns. Spelling out difficult words phonetically can help.
- **15-second generation window**: Each audio segment is limited to ~15 seconds. Longer text is automatically split, but very long single sentences may be divided at suboptimal points.
- **Emotional range with voice cloning**: Voice cloning optimizes for identity accuracy, which can reduce the extremes of emotional delivery. For maximum expressiveness, use a strong emotional archetype in the voice description.
- **Multilingual pronunciation**: When a character switches languages mid-speech, the model may apply the primary language's phonetics to the foreign words. Use separate requests per language.
- **Generation speed**: Each 15-second segment takes 3-8 seconds depending on hardware. Audio is returned as a complete file, not streamed.
- **Gemma 3 12B is gated**: Requires accepting Google's terms of use and a HuggingFace token with access.
- **Reference audio quality sensitivity**: Low-quality references (compressed MP3, background noise) significantly degrade output. Use clean reference audio or rely on the voice description alone with SeedVC as a post-processing step.

## Acknowledgments

- [LTX-2](https://github.com/Lightricks/LTX-2) by Lightricks for the base audiovisual model
- [Gemma 3](https://ai.google.dev/gemma) by Google for the text encoder
- [SeedVC](https://github.com/Plachtaa/seed-vc) by Plachta for voice refinement
- [Kokoro](https://github.com/hexgrad/kokoro) by hexgrad for duration estimation
- [SageAttention](https://github.com/thu-ml/SageAttention) for attention acceleration

## License

MIT License. See [LICENSE](LICENSE) for details.
