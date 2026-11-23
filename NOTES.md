# chiaki-vocoder training notes

For anyone fine-tuning or retraining this vocoder. For usage, see [README.md](README.md).

## Comparison with pc_nsf_hifigan

The pc_nsf_hifigan figures are from the OpenVPI release notes (`pc-nsf-hifigan-44.1k-hop512-128bin-2025.02`).

| | pc_nsf_hifigan 2025.02 | chiaki_vocoder 2026.11 |
|---|---|---|
| Architecture | PC-NSF-HiFiGAN (MiniNSF) | NSF-HiFiGAN (full NSF, 8 harmonics) |
| Generator parameters | Not stated | 14.2M |
| Training data | About 79 h | About 16 h of recordings plus GTA mels, about 36 h in the index |
| Training steps | 40k + 108k (unit not stated) | 488k global steps from scratch, then staged fine-tuning |
| Interface | 44.1 kHz, hop 512, win/FFT 2048, 128 bins, 40–16000 Hz | Same, drop-in replacement |
| Pitch range | E2–D#7 | Training f0 65–1400 Hz (about C2–F6), plus pitch augmentation up to 1.9×; higher is untested |
| Tone shift (mel/f0 mismatch) | ±12 semitones | Usable but less natural than pc_nsf; keep negative shifts above −7 semitones |
| Weight license | CC BY-NC-SA 4.0 | Apache License 2.0 |

SingingVocoders counts one global step for the generator and one for the discriminators, so the number of real updates is half the global step count.

### Blind listening tests

DiffSinger acoustic-model output, level-matched, A/B order randomized, "same" allowed.

| Material | chiaki_vocoder / pc_nsf / same |
|---|---|
| Test song (3 voicebanks), soft lines | 2 / 2 / – |
| Test song, high lines | 2 / 2 / – |
| Test song, 8 chorus lines | 4 / 2 / 2 |
| Full song exported from OpenUtau | 8 / 3 / 1 |
| High lines of the same song | 4 / 0 / – |
| Tone shift −4, −7, +7 (4 pairs each) | 1 / 11 / – |

### Tone shift

- Only OpenUtau's tone shift curve (SHFC) and the vocoder pitch curve make the mel and f0 disagree. Normal transposition does not.
- This model's pitch augmentation resamples the audio and recomputes the mel, so its training mel and f0 always matched. It is not pc_aug.
- When f0 is 7 or more semitones above the mel's pitch (tone shift −7 or lower), HNR drops to 2–15 dB and some clips stay at the mel's pitch. Shifting f0 down is fine to −12 semitones.
- Measured on 26 validation clips with the mel fixed and f0 shifted.

## Data

| Source | License | Length |
|---|---|---|
| VocalSet | CC BY 4.0 | About 8.7 h |
| NIT SONG070 F001 | CC BY 3.0 | About 0.5 h |
| Recordings by the voicebank creator (cut into 2–12 s clips) | Provided by the copyright holder, not distributed | About 7 h |
| GTA: mels rendered from those recordings by the Chiaki acoustic model | — | — |

No non-commercial or unclear-license data was used.

## Training recipe

openvpi/SingingVocoders (commit a930663), with no pretrained weights. CQTD and the final-stage losses need a modified task (`training/nsf_HiFigan_cqt_task.py`) that is not in upstream or in this repository.

| Stage | Change | Purpose |
|---|---|---|
| 1 | Train from scratch: full NSF (8 harmonics), MPD + MSD, 224k+ global steps | Base model. mini_nsf gives a buzzing jitter; do not use it |
| 2 | Add MRD, then STFT loss | Remove the jitter |
| 3 | Add the creator's recordings | Target voice |
| 4 | Add CQTD and the harmonic-gap loss; high-pass the discriminator input at 45 Hz | Remove hoarseness on unseen voices |
| 5 | Lower the harmonic-gap weight on unvoiced frames to 0.3; add the band-bias loss | Harmonic buzz and a 2 kHz resonance on consonants |
| 6 | Add GTA mels | Match acoustic-model mels |
| 7 | Random pitch-up 1.0–1.9× (probability 0.5), 12k global steps | High notes |

Stage 1 barely improves after 224k. After stage 5, long runs bring the 2 kHz resonance back, so keep the later stages short.

Final-stage overrides (the full config is `finetune/training_config.yaml`):

```yaml
use_stftloss: true
lab_aux_stftloss: 10
use_cqtd: true
d_highpass_hz: 45
lab_harm_gap: 5
harm_gap_uv_weight: 0.3
lab_band_bias: 5
key_aug: true
key_aug_prob: 0.5
aug_min: 1.0
aug_max: 1.9
```

AdamW (lr 1e-4, betas 0.8 / 0.99), batch size 10, 48-frame crops.

## Fine-tuning

`finetune/chiaki_vocoder_2026.11_GD.ckpt` loads into upstream SingingVocoders (commit a930663) with `configs/base_hifi.yaml`. Its `model_args` and mel settings already match, so only these need changing:

```yaml
finetune_enabled: true
finetune_ckpt_path: path/to/chiaki_vocoder_2026.11_GD.ckpt
f0_max: 1400        # upstream default is 1100
# optional, to keep the high range of this model
key_aug: true
aug_min: 1.0
aug_max: 1.9
```

The generator, MPD and MSD load as-is. The MRD and CQTD weights in the checkpoint are ignored, because upstream does not have them. Fine-tuning then uses upstream's losses only, so the hoarseness and the consonant resonance that stages 4–5 removed may come back on long runs. Keep runs short and compare checkpoints by ear.

The modified task for CQTD and the extra losses is not included in this repository.

## Tips

- **Pick checkpoints by blind listening.** Most objective metrics disagree with the ear, including whole-song breath error, band level bias, mel fine-structure correlation and mel L1. Only two tracked listening results: per-bin error on inhales, and harmonic error at 700–950 Hz.
- **GAN checkpoints oscillate.** Save several per run and choose among them.
- **Blind-test in pairs** (A/B), level-matched, with "same" allowed.
- **Before writing a new loss, check that the difference is audible.** Splice by band (for example, swap only above 4 kHz) or by time (swap a single 0.3 s span).
- **MPD and MSD barely see above 7 kHz**, and MSD pushes the generator to add sub-60 Hz energy. Use CQTD with a 45 Hz high-pass. If you port CQTD yourself, check that its output is not almost all bias (a scaling bug).
- **Weak high harmonics are not a data-size problem.** They are just as weak on the training set, so more data does not help.

## Known issues

- **Tone shift** sounds less natural than pc_nsf's (see above). A pc_aug experiment is planned.
- **Low male voices** (below about C3) sound rougher than with pc_nsf, because the training data is mostly female. On a baritone DiffSinger voicebank, with the same mel through both vocoders, median HNR on held A2–C3 notes was 16 dB vs pc_nsf's 29 dB. Above about A3 the two were equal. Work on this is in progress.
- **Inhales and breath noise** sound less natural than pc_nsf's and follow the fine structure of the input mel less closely. A texture loss and mel-shaped noise on breath frames did not help. Noise that follows the mel more closely sounded worse, so mel fine-structure correlation is not a good target.

## Cost

- Stage 1 ran mostly on a local Apple Silicon machine.
- The rest took about 200 GPU hours on AWS g5.xlarge (one A10G 24 GB), about US$200, including failed experiments and parameter tuning. Rerunning the recipe above takes much less.
