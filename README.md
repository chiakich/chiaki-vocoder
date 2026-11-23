---
license: apache-2.0
library_name: onnx
tags:
  - vocoder
  - neural-vocoder
  - nsf-hifigan
  - hifigan
  - audio
  - singing-voice-synthesis
  - diffsinger
  - openutau
---

# chiaki-vocoder

Current version: 2026.11

An NSF-HiFiGAN vocoder (mel + f0 to waveform), made for the Chiaki voicebank and trained from scratch. It uses the same mel settings and interface as the OpenVPI 44.1 kHz, 128-bin community vocoders, so it should work with anything that produces those mels and an f0 curve. Tested with DiffSinger / OpenUtau; other uses are untested. The weights are released under the Apache License 2.0: they can be used commercially, fine-tuned and redistributed, as long as the license and NOTICE.txt are kept. Training details are in [NOTES.md](NOTES.md).

## Files

| File | Use |
|---|---|
| `chiaki_vocoder_2026.11.onnx` | Inference (OpenUtau / DiffSinger ONNX) |
| `vocoder.yaml` | OpenUtau vocoder config |
| `chiaki_vocoder_2026.11.ckpt` | Generator weights, `{'generator': state_dict}` |
| `config.json` | Generator hyperparameters |
| `finetune/chiaki_vocoder_2026.11_GD.ckpt` | Generator and discriminators, `{'state_dict': ...}`, for fine-tuning with SingingVocoders (`finetune_ckpt_path`) |
| `finetune/training_config.yaml` | The training config of the final stage |
| `LICENSE` | Apache License 2.0 |
| `NOTICE.txt` | Credits and license details |

Checksums are in `SHA256SUMS`.

## Interface

- Inputs: `mel` float32 [1, T, 128] (natural-log magnitude) and `f0` float32 [1, T] in Hz. Output: `waveform` float32 [1, T × 512].
- Mel settings: 44.1 kHz, hop 512, window and FFT 2048, 128 bins, 40–16000 Hz, slaney.
- The acoustic model must use the same mel settings.
- Trained mostly on singing, and mostly on female voices. Speech is untested.

## Using it in OpenUtau

Either way works:

- **As a dependency:** drag `chiaki_vocoder_2026.11.oudep` (from the [GitHub release](https://github.com/chiakich/chiaki-vocoder/releases/latest)) into the OpenUtau window and confirm. Then set `vocoder: chiaki_vocoder_2026.11` in the voicebank's `dsconfig.yaml`.
- **Bundled with a voicebank:** put `chiaki_vocoder_2026.11.onnx` and `vocoder.yaml` in the voicebank's `dsvocoder/` folder, and set `vocoder: dsvocoder` in `dsconfig.yaml`.

Tone shift (SHFC) works, but sounds less natural than with pc_nsf_hifigan, because chiaki_vocoder was trained without pc_aug. Keep it above −7 semitones. A vocoder pitch curve is not recommended.

## Quality

Blind A/B listening tests against OpenVPI pc_nsf_hifigan 2025.02, on DiffSinger acoustic-model output:

- Test song, 3 voicebanks:
  - soft lines 2/2
  - high lines 2/2
  - chorus lines: chiaki_vocoder 4, pc_nsf 2, same 2
- A full song exported from OpenUtau: chiaki_vocoder 8, pc_nsf 3, same 1. On high lines chiaki_vocoder won 4/0.
- Tone shift at −4, −7 and +7 semitones: chiaki_vocoder 1, pc_nsf 11.
- Known weaknesses:
  - Tone shift sounds less natural than pc_nsf's. A pc_aug experiment is planned.
  - Some breath noises and inhales sound less natural than pc_nsf's.
  - Low male voices (below about C3) sound rougher than with pc_nsf, because the training data is mostly female. Work on this is in progress.

## Training

- **Code:** openvpi/SingingVocoders (MIT, commit a930663), with no pretrained weights. The multi-scale sub-band CQT discriminator was adapted from Amphion (MIT).
- **Architecture:** full NSF source (8 harmonics), HiFiGAN generator with 14.2M parameters. Discriminators: MPD, MSD, MRD and MS-SB-CQTD.
- **Data:** VocalSet, NIT SONG070 and the voicebank creator's own recordings, about 36 h including acoustic-model renders.

The training recipe and fine-tuning instructions are in [NOTES.md](NOTES.md).

## Credits and license

The weights are licensed under the [Apache License 2.0](LICENSE). Suggested credit: "chiaki-vocoder 2026.11 by Chiaki.C (Chiaki voicebank), Apache-2.0".

Training data and sources:
- VocalSet, Julia Wilkins, Prem Seetharaman, Alison Wahl, Bryan Pardo. CC BY 4.0.
- NIT SONG070 F001, Nagoya Institute of Technology. CC BY 3.0.
- Song Data by Chiaki.C

See `NOTICE.txt` for details.
