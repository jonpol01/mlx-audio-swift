# Irodori TTS Swift Port — Status & Handoff

Tracking: VoiceAIkit Linear HAL-219. Branch: `feature/irodori-tts`.

## Goal
Port `mlx_audio/tts/models/irodori_tts` (Python, at `~/Develop/Github/mlx-audio`) into this
package so `TTS.loadModel(modelRepo: "mlx-community/Irodori-TTS-600M-v3-VoiceDesign-8bit")`
works. Irodori = Echo-TTS-family flow-matching JP TTS (Rectified Flow DiT over
Semantic-DACVAE-32dim latents, 48 kHz). Mirror the Swift EchoTTS structure
(`Sources/MLXAudioTTS/Models/EchoTTS/`) and apply Irodori's deltas from the Python.

## Done (committed)
- [x] `IrodoriTTSConfig.swift` — full mirror of config.py (DiT/Sampler/Model configs,
      resolved-property logic, decodeIfPresent defaults). Commit 6a9bd88.
- [x] `IrodoriDuration.swift` — 14-dim duration features (duration.py). Commit: see log.
- [x] `IrodoriTTSText.swift` — JP normalisation (regex map, width folding, kana widening,
      bracket/punct stripping) + `irodoriEncodeText` (HF tokenizer, manual BOS, right-pad)
      + `IrodoriTTSError`. Commit c0c0a2d.

## Remaining (in dependency order)
1. **IrodoriDiT.swift** — from Python `model.py` (1529 ln). Template: `EchoDiT.swift`.
   Deltas vs Echo: caption-condition encoder branch (VoiceDesign), dual conditioning
   (speaker AND caption in v3-VoiceDesign), adaln rank 192, mlp ratios from config,
   per-branch mlp_ratio resolution. Check RoPE/positional details against Python — do
   NOT assume Echo's.
2. **IrodoriTTSAudio.swift** — Semantic-DACVAE-Japanese-32dim DECODER. Weights ship in
   the model repo under `dacvae/` (verified: dacvae/config.json + dacvae/model.safetensors
   429 MB). Python model.py contains the decoder modules + weight namespacing — port decode
   path only (no encode needed unless ref-audio cloning is kept; encode IS needed for
   refAudio voice-clone conditioning → check Python irodori_tts.py how ref audio → latents).
3. **IrodoriTTSSampling.swift** — from sampling.py (623 ln). Template: EchoTTSSampling.swift.
   Deltas: caption CFG scale (third guidance branch), cfg_guidance_mode "independent" vs
   "alternating" (MUST implement alternating — it's the mobile path), sway sampling
   t-schedule (v3), speaker_kv cache options, duration integration.
4. **IrodoriDuration.swift** — from duration.py (156 ln). v3 duration predictor
   (token_sum_adarn_zero_no_aux arch; predicts output frames from text+ref+caption).
5. **IrodoriTTSModel.swift** — from irodori_tts.py (474 ln). Template: EchoTTSModel.swift.
   - `fromPretrained`: download repo (HubClient pattern — see KokoroMultilingualProcessor
     `ensureNeuralModelDownloaded` / other models' loaders), load config.json, weights +
     `dacvae/` subdir weights, sanitize keys per Python, quantization handling (8bit repos
     have quantization config — follow how other models in this package apply
     `quantize(model:)` from config).
   - TOKENIZER: NOT in the model repo. Download `llm-jp/llm-jp-3-150m` tokenizer files
     separately via HubClient (matching: ["tokenizer*", "*.model", "special_tokens*"]) and
     `AutoTokenizer.from(modelFolder:)` (import Tokenizers). Same for caption tokenizer
     (caption_tokenizer_repo_resolved — likely same repo).
   - Conform to `SpeechGenerationModel` exactly like EchoTTSModel (sampleRate from config;
     generate(text:voice:refAudio:refText:language:generationParameters:) → MLXArray).
   - Conditioning mapping: refAudio → voice-clone; `voice` param = VoiceDesign caption
     (default when nil: "落ち着いた自然な声で、はっきりと読み上げてください。").
   - MOBILE DEFAULTS: override sampler at init → sequenceLength 300, cfgGuidanceMode
     "alternating" (Python defaults 750/"independent" need ~24 GB; 300/alternating ≈ 2 GB).
6. **Registration** — `Sources/MLXAudioTTS/TTSModel.swift`: add `case "irodori_tts":` in the
   model-type switch (mirror the kokoro case at ~line 189) + `inferModelType` returns
   "irodori_tts" when repo name contains "irodori" (~line 253 area; note existing checks use
   underscored names — repo names are hyphened, so add an explicit contains("irodori")).
7. **Build**: `swift build 2>&1 | tail -30` until clean. Toolchain Swift 6.2+ (manifest).
8. **VoiceAIkit wiring** (separate repo `~/Develop/Github/VoiceAIkit`): point project.yml
   mlx-audio-swift at this local path or the jonpol01 fork+branch; add catalog entry
   id `mlx-community/Irodori-TTS-600M-v3-VoiceDesign-8bit#mlxaudio` (the "#mlxaudio" routing
   marker already works); in `MarvisTTSManager.speak` the JP language mapping for irodori:
   model takes Japanese text directly (no language param needed — single-language model;
   pass language: nil, voice: caption).

## Gotchas discovered
- Echo's Swift files are the closest idiom (Module/@ModuleInfo, no force-unwraps).
- Python `cfg_guidance_mode="independent"` batches 3 CFG branches → 3x memory. The
  "alternating" mode alternates guidance per step — read sampling.py carefully.
- Repo tree: `model.safetensors` (829 MB, 8bit DiT) + `dacvae/model.safetensors` (429 MB,
  fp DACVAE — likely NOT quantized; load separately, do not quantize).
- max ~12 s audio at sequenceLength 300 — fine; VoiceAIkit already sentence-chunks.
