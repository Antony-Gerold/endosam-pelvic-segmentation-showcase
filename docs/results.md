# Results

## Headline

| Method | Uterus | Ovary | Endo |
|---|---|---|---|
| **MedSAM decoder fine-tuned** | **0.857** | **0.783** | **0.744** |
| MedSAM oracle box | 0.842 | 0.744 | 0.696 |
| MedSAM simulated box | 0.805 | 0.683 | 0.641 |
| Binary U-Net + FTL | — | 0.298 | 0.556† |
| Multi-class U-Net | 0.656 | 0.000 | 0.000 |
| RAovSeg (published) | — | 0.290 | — |
| nnU-Net (published) | — | 0.272 | — |
| Inter-rater (humans) | 0.605 | 0.345 | — |

† 0.556 when detected; 0.389 averaged across all slices including non-detection.

## The data-fix discovery

| State | MedSAM Uterus Dice |
|---|---|
| Before alignment fix | 0.356 |
| After alignment fix | **0.842** |
| **Δ** | **+0.486** |

Larger than any architectural change in the project. **52 % of D1 subjects** had spatial misalignment from multi-sequence MRI acquisition.

## Class-imbalance ablation (RAovSeg's published finding, confirmed by our baselines)

| Configuration | Ovary Dice |
|---|---|
| RAovSeg full (slice classifier + segmenter) | 0.290 |
| RAovSeg without slice classifier | 0.013 |
| Multi-class U-Net (no slice filter) | 0.000 |
| Binary U-Net + FTL (implicit slice filter via training set) | 0.298 |

The slice classifier is doing 95 % of the work. Binary models capture this benefit implicitly by training only on organ-containing slices.

## MedSAM regime comparison

| Regime | Uterus | Ovary | Endo | Realism |
|---|---|---|---|---|
| Oracle box | 0.842 | 0.744 | 0.696 | Unrealistic (needs GT bbox) |
| Simulated noisy box | 0.805 | 0.683 | 0.641 | Realistic — models radiologist drawing |
| Decoder fine-tuned + oracle box | 0.857 | 0.783 | 0.744 | Unrealistic |
| Auto-prompt (U-Net → MedSAM) | < binary | < binary | < binary | Realistic |

The **simulated noisy-box** result is the most clinically defensible comparison against RAovSeg.

## Per-organ analysis

### Uterus

- Largest organ, most consistent boundaries.
- Inter-rater Dice 0.605 — *all top methods exceed inter-rater*, meaning model predictions are at human level.
- MedSAM decoder-FT 0.857 is approaching the theoretical ceiling for this dataset.

### Ovary

- 0.3 – 2.5 % of image pixels.
- Inter-rater Dice 0.345 — extreme human disagreement.
- Binary U-Net 0.298 matches the published RAovSeg baseline.
- MedSAM with a usable prompt jumps to 0.683 (simulated) or 0.783 (oracle) — **more than 2× the inter-rater ceiling**, suggesting MedSAM is generalising better than individual radiologists do on this annotation task.

### Endometrioma

- Smallest, hardest, and previously unattempted on UT-EndoMRI.
- Multi-class baseline: 0.000. Attention U-Net: 0.225.
- Binary model: **0.556 when detected** (≈ 70 % detection rate, overall mean 0.389).
- MedSAM decoder-FT: 0.744 — best result, but requires oracle box.

## Inter-rater context

Corrected inter-rater Dice on D1 (8 cross-sequence comparison pairs excluded):

| Organ | Inter-rater Dice ± std |
|---|---|
| Uterus | 0.605 ± 0.386 |
| Ovary | 0.345 ± 0.377 |

Comparison to published inter-rater Dice on other tasks:

| Organ / task | Inter-rater Dice |
|---|---|
| Liver | 0.92 |
| Kidney | 0.87 |
| Prostate | 0.83 |
| Pancreas | 0.78 |
| Uterus (this work) | 0.61 |
| Ovary in endometriosis MRI | **0.345** |

Caveat: cross-task inter-rater comparisons are approximate — different datasets, different protocols.

## Limitations summary

- **CPU-only training** → 15-epoch cap; no nnU-Net comparison; no 3D volumetric pipelines.
- **2D slice processing** — no volumetric context.
- **Simulated noisy boxes** are formula-derived, not from real radiologist annotations.
- **Decoder fine-tuning still needs oracle boxes at test time** — auto-prompting chain documented but bottlenecked by upstream U-Net.
- **Endometrioma small test set** — would benefit from cross-validation.
- **Dice only.** Surgical planning would also need Hausdorff distance.
