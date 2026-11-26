# chiaki-vocoder training notes

For anyone fine-tuning or retraining this vocoder. For usage, see [README.md](README.md).

## Comparison with pc_nsf_hifigan

The pc_nsf_hifigan figures are from the OpenVPI release notes (`pc-nsf-hifigan-44.1k-hop512-128bin-2025.02`).

| | pc_nsf_hifigan 2025.02 | chiaki_vocoder 2026.11 |
|---|---|---|
| Architecture | PC-NSF-HiFiGAN (MiniNSF) | NSF-HiFiGAN (full NSF, 8 harmonics) |
| Generator parameters | Not stated | 14.2M |
| Training data | About 79 h | About 22 h (21 h of recordings plus GTA mels) |
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
| Recordings by the voicebank creator (cut into 2–12 s clips) | Provided by the copyright holder, not distributed | About 11 h |
| GTA: mels batch-rendered from those recordings by the Chiaki acoustic model | — | About 0.6 h |

About 22 h of unique audio in total. Smaller sets are listed 2–3 times in the training index, and low-note clips are oversampled in stage 8.

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
| 7 | Random pitch-up 1.0–1.9× (probability 0.5), 12k global steps | High notes (= 2026.11) |
| 8 | Pitch augmentation widened to 0.5–1.9× (pitch-down too), low-note clips oversampled, a per-harmonic level loss, 12k global steps from 2026.11 | Low notes (= 2026.11.1) |

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
aug_min: 0.5
aug_max: 1.9
```

AdamW (lr 1e-4, betas 0.8 / 0.99), batch size 10, 48-frame crops.

## Fine-tuning

`finetune/chiaki_vocoder_2026.11.1_GD.ckpt` loads into upstream SingingVocoders (commit a930663) with `configs/base_hifi.yaml`. Its `model_args` and mel settings already match, so only these need changing:

```yaml
finetune_enabled: true
finetune_ckpt_path: path/to/chiaki_vocoder_2026.11.1_GD.ckpt
f0_max: 1400        # upstream default is 1100
# optional, to keep the high range of this model
key_aug: true
aug_min: 0.5
aug_max: 1.9
```

The generator, MPD and MSD load as-is. The MRD and CQTD weights in the checkpoint are ignored, because upstream does not have them. Fine-tuning then uses upstream's losses only, so the hoarseness and the consonant resonance that stages 4–5 removed may come back on long runs. Keep runs short and compare checkpoints by ear.

The modified task for CQTD and the extra losses is not included in this repository.

## Evaluating vocoders

We compared checkpoints mainly by blind A/B listening: level-matched, A/B order randomized, "same" allowed, hidden repeats to check consistency, and the decision rule written down before listening. Close candidates were tested on DiffSinger acoustic-model output from voicebanks the vocoder never saw, because copy-synthesis of the training voice hides most differences.

We also tried many objective metrics. Each was scored against blind answers, first after the fact, then on new pages with its prediction frozen before listening. None could rank close candidates, but some catch gross problems:

| Metric | Matches past blind answers | Predicts new blind tests | Use it for |
|---|---|---|---|
| Mel L1, whole-song breath error, mel fine-structure correlation | Poor | — | Nothing |
| Band level vs the real recording (2–6 kHz) | — | Wrong: the "closer" model lost 15/15 | Nothing |
| Mel balance shape error (output mel vs input mel, smoothed) | 27/28 | 4/7 over two tests | Gross colour errors |
| Harmonic-to-noise ratio, H4–H11, vs a reference vocoder | Partly | About chance | Gross buzz or noise |
| SingMOS-Pro (SingMOS v1.1.2), per-clip difference | 24/33 on close pages, 31/43 off low notes | 4/7 as part of a frozen rule; ranked models opposite to the ear on one voicebank | Gross quality drops; best of the metrics, but still not a judge |
| Daniel–Weber roughness, 90th percentile, low notes only | 18/26 | 3/5 | Low-note roughness alarm |
| DIN sharpness | Chance or reversed | — | Nothing |
| Per-bin error on inhales; harmonic error at 700–950 Hz | Tracked listening results during stages 4–5 | — | Only within one run |

Absolute SingMOS scores varied from 1.1 to 4.9 with the content, so only differences on the same clip mean anything.

## Tips

- **Pick checkpoints by blind listening.** Objective metrics, SingMOS included, only catch gross problems (see above).
- **GAN checkpoints oscillate.** Save several per run and choose among them.
- **Blind-test in pairs** (A/B), level-matched, with "same" allowed.
- **Before writing a new loss, check that the difference is audible.** Splice by band (for example, swap only above 4 kHz) or by time (swap a single 0.3 s span).
- **MPD and MSD barely see above 7 kHz**, and MSD pushes the generator to add sub-60 Hz energy. Use CQTD with a 45 Hz high-pass. If you port CQTD yourself, check that its output is not almost all bias (a scaling bug).
- **Weak high harmonics are not a data-size problem.** They are just as weak on the training set, so more data does not help.

## Known issues

- **Tone shift** sounds less natural than pc_nsf's (see above). A pc_aug experiment is planned.
- **Low male voices** (below about C3) sounded rougher than with pc_nsf in 2026.11, because the training data is mostly female. On a baritone DiffSinger voicebank, with the same mel through both vocoders, 2026.11's median HNR on held A2–C3 notes was 16 dB vs pc_nsf's 29 dB; above about A3 the two were equal. 2026.11.1 won a blind test against 2026.11 on a baritone voicebank 3 / 0 / 1.
- **Inhales and breath noise** sound less natural than pc_nsf's and follow the fine structure of the input mel less closely. A texture loss and mel-shaped noise on breath frames did not help. Noise that follows the mel more closely sounded worse, so mel fine-structure correlation is not a good target.

## Compute

| Where | GPU | Time | Cost |
|---|---|---|---|
| Local Apple Silicon machine | Apple GPU (MPS) | Most of stage 1 | — |
| Google Colab | Mostly L4 | About 20 GPU hours | 100 compute units |
| AWS g5.xlarge | A10G 24 GB | 209 GPU hours (129 Spot, 80 on-demand) | About US$180 |

About 230 cloud GPU hours in total, not counting the local machine. This includes failed experiments, parameter tuning, GTA rendering and the evaluation renders for the blind tests. Rerunning the recipe above takes much less.
