# chiaki-vocoder

> **The main home of this project is Hugging Face: [huggingface.co/chiakic/chiaki-vocoder](https://huggingface.co/chiakic/chiaki-vocoder).** Each version is published there first. This repository mirrors the docs, and the [Releases](https://github.com/chiakich/chiaki-vocoder/releases/latest) page has the OpenUtau package.

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Hugging Face](https://img.shields.io/badge/%F0%9F%A4%97%20Hugging%20Face-chiaki--vocoder-yellow)](https://huggingface.co/chiakic/chiaki-vocoder)
[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/A0A21UAIV9)

An NSF-HiFiGAN vocoder (mel + f0 to waveform) for singing voice synthesis, trained from scratch. It uses the same interface as the OpenVPI 44.1 kHz, 128-bin community vocoders, so it works as a drop-in replacement in DiffSinger and OpenUtau. The weights are Apache-2.0: free for commercial use, fine-tuning and redistribution.

Current version: **2026.11.1**

## Why

Many of the vocoders shared in the singing-synthesis community are licensed for non-commercial use only, and few are fully open. I wanted to contribute one that anyone can use, fine-tune and build on, and maybe someone will turn it into a better general-purpose vocoder.

Most of the training data is my own female voice, so other voicebanks, especially male or very different voices, may not sound as good.

## Quick start (OpenUtau)

1. Download `chiaki_vocoder_2026.11.1.oudep` from the [latest release](https://github.com/chiakich/chiaki-vocoder/releases/latest).
2. Drag it into the OpenUtau window and confirm.
3. In the voicebank's `dsconfig.yaml`, set `vocoder: chiaki_vocoder_2026.11.1`.

To bundle it with a voicebank instead, put `chiaki_vocoder_2026.11.1.onnx` and [`vocoder.yaml`](vocoder.yaml) in the voicebank's `dsvocoder/` folder and set `vocoder: dsvocoder`.

Tone shift (SHFC) works but sounds less natural than with pc_nsf_hifigan; keep it above −7 semitones.

## Downloads

| File | Where | Use |
|---|---|---|
| `chiaki_vocoder_2026.11.1.oudep` | Releases | OpenUtau dependency package |
| `chiaki_vocoder_2026.11.1.onnx` | Releases, Hugging Face | Inference (OpenUtau / DiffSinger ONNX) |
| `chiaki_vocoder_2026.11.1.ckpt` | Releases, Hugging Face | Generator weights, `{'generator': state_dict}` |
| `chiaki_vocoder_2026.11.1_GD.ckpt` | Releases, Hugging Face (`finetune/`) | Generator and discriminators, for fine-tuning with SingingVocoders |
| [`vocoder.yaml`](vocoder.yaml), [`config.json`](config.json), [`finetune/training_config.yaml`](finetune/training_config.yaml) | This repo | OpenUtau config, generator hyperparameters, final-stage training config |

Checksums are in [`SHA256SUMS`](SHA256SUMS). Older versions are kept as tags on Hugging Face and as releases here.

## Interface

- Inputs: `mel` float32 [1, T, 128] (natural-log magnitude) and `f0` float32 [1, T] in Hz. Output: `waveform` float32 [1, T × 512].
- Mel: 44.1 kHz, hop 512, window and FFT 2048, 128 bins, 40–16000 Hz, slaney.

## Quality

Blind A/B tests on DiffSinger output (level-matched, order randomized, "same" allowed):

- 2026.11.1 vs 2026.11: low male voice 3 / 0 / 1; DiffSinger songs 3 / 3 / 2; tone shift 3 / 2 / 1.
- 2026.11 vs pc_nsf_hifigan 2025.02: full OpenUtau song 8 / 3 / 1; high lines 4 / 0; tone shift 1 / 11.

Details, the training recipe, compute (about 230 cloud GPU hours), and how we evaluated vocoders, including which objective metrics did and did not agree with the ear, are in [NOTES.md](NOTES.md).

## Contributing and support

Contributions are welcome: singing recordings help the most, but training code or GPU time also helps a lot. Please open an issue here or contact me through Hugging Face.

This is a one-person project. I recorded most of the training data myself, singing until my voice went hoarse, and paid for the cloud GPUs myself. It is free under Apache-2.0 and will stay that way. If it helps your work, please consider supporting future versions on [Ko-fi](https://ko-fi.com/chiakic).

## License and credits

Weights: [Apache License 2.0](LICENSE). Suggested credit: "Chiaki-Vocoder by Chiaki.C, Apache-2.0".

Training data: VocalSet (Wilkins, Seetharaman, Wahl, Pardo; CC BY 4.0), NIT SONG070 F001 (Nagoya Institute of Technology; CC BY 3.0), and song data by Chiaki.C. Code: openvpi/SingingVocoders (MIT); the CQT discriminator was adapted from Amphion (MIT). See [`NOTICE.txt`](NOTICE.txt).
