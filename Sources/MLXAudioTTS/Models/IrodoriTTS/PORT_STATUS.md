# Irodori TTS Swift Port — Status & Handoff

Tracking: VoiceAIkit Linear HAL-219. Branch: `feature/irodori-tts` (fork: jonpol01/mlx-audio-swift).

## Goal
Port `mlx_audio/tts/models/irodori_tts` (Python, at `~/Develop/Github/mlx-audio`) into this
package so `TTS.loadModel(modelRepo: "mlx-community/Irodori-TTS-600M-v3-VoiceDesign-8bit")`
works. Irodori = Echo-TTS-family flow-matching JP TTS (Rectified Flow DiT over
Semantic-DACVAE-32dim latents, 48 kHz), v3 duration prediction + VoiceDesign captions.

## ✅ PORT COMPLETE — `swift build` clean (0 errors)

All files written + committed + pushed:
- [x] `IrodoriTTSConfig.swift` — full config.py mirror (DiT/Sampler/Model, resolved props).
- [x] `IrodoriTTSText.swift` — JP normalisation + HF-tokenizer encode + error types.
- [x] `IrodoriDuration.swift` — 14-dim duration features (duration.py).
- [x] `IrodoriDiT.swift` — DiT backbone: joint attention, text/reference/caption encoders,
      diffusion blocks, **v3 duration predictor** (folded in from model.py). 961 ln.
- [x] `IrodoriTTSSampling.swift` — rectified-flow Euler CFG sampler (sampling.py). All three
      modes: independent (dual up to 4× + single-context 2×/3×), joint, alternating. Sway
      schedule, speaker-KV cache + rollback, temporal score rescale.
- [x] `IrodoriTTSModel.swift` — loader + generate pipeline (irodori_tts.py). fromPretrained
      downloads model + `dacvae/` subdir + llm-jp tokenizer; quantizes 8-bit DiT layers;
      reuses **MLXAudioCodecs.DACVAE** for decode (no codec port needed); SpeechGenerationModel.
- [x] Registration in `TTSModel.swift`: `case "irodori_tts"` + `inferModelType` contains("irodori").

## Key facts confirmed during port
- Target model is **dual-context** (use_caption_condition AND use_speaker_condition = true),
  duration predictor ON, config default cfg mode = "independent", seq 750.
- **DACVAE codec already existed** in the package (Sources/MLXAudioCodecs/DACVAE) with
  `decode(_:chunkSize:)` + `fromModelDirectory` → reused directly. Loaded from `dacvae/` subdir
  (fp weights, NOT quantized; only DiT Linear layers with `.scales` get quantized).
- Mobile memory: with duration predictor ON + sentence chunking, sequences are short
  (~125 frames / 5 s), so the 24 GB figure (seq 750) does NOT apply. Keep "independent"
  (best quality, model's trained default). max_seconds caps runaway length.
- Tokenizer (`llm-jp/llm-jp-3-150m`) is a SEPARATE repo — downloaded + AutoTokenizer.from.
- `voice` protocol param = VoiceDesign caption (default: 落ち着いた自然な声で、はっきりと読み上げてください。).

## Verification status
- ✅ `swift build` clean.
- ✅ **Weight-key structure verified against the real checkpoint** (fetched safetensors
  header from HF; all 98 unique normalized paths map to Swift modules: joint attention
  +dual adaLN +SwiGLU MLP in main blocks; EchoEncoderTransformerBlock for text/speaker/
  caption encoders; duration predictor incl. null_speaker/null_caption ParameterInfo;
  in_proj/out_proj/out_norm/cond_module). Load via update(verify:.noUnusedKeys) should pass.
- ❌ **Mac smoke test BLOCKED** — `swift run` can't load the macOS `default.metallib`
  (MLX inits Metal eagerly at stream creation; CPU-forcing doesn't bypass it; SwiftPM CLI
  build doesn't produce the macOS metallib on this machine). NOT a port bug — affects ANY
  mlx-audio model via CLI. Numerical correctness must be verified on-device (Xcode builds
  Metal correctly).

## Remaining (integration)
2. **VoiceAIkit wiring**: point project.yml mlx-audio-swift at this fork+branch (or local path);
   add catalog entry id `mlx-community/Irodori-TTS-600M-v3-VoiceDesign-8bit#mlxaudio`
   (the "#mlxaudio" routing marker already works in MarvisTTSManager); JP mapping —
   pass language: nil, voice: caption (single-language model, takes JP text directly).
3. **On-device JP test** + whisper-verify (whisper venv at /tmp/whisper-venv).

## If runtime errors appear (debugging pointers)
- Weight key mismatch → the DiT module structure (IrodoriDiT.swift @ModuleInfo keys) must match
  the sanitized PyTorch keys. Python `Model.sanitize` is the ground truth for key remapping.
- Shape errors in attention → check IrodoriJointAttention RoPE/head-split vs Python model.py.
- Garbled audio but no crash → likely DACVAE latent transpose or the latent_dim/patch handling;
  verify (1,T,latentDim)→(1,latentDim,T) before decode and chunkSize=50.
