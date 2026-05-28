# Methodology

## 1. Dataset — UT-EndoMRI

[Liang et al., 2025] — the first public benchmark for pelvic endometriosis MRI.

| Cohort | Hospital | Subjects | Sequences | Sites | Scanners | Raters |
|---|---|---|---|---|---|---|
| D1 | Memorial Hermann | 51 | T2, T1FS | 15 | 9 | 3 per structure |
| D2 | Texas Children's | 73 | T2, T2FS, T1FS, T1 | 1 | 1 | 1 |
| **Total** | | **124** | | | | |

Each structure (uterus, ovary, endometrioma/cyst) is annotated as a separate binary NIfTI file. Splits: 70 / 15 / 15 by subject, seed 42.

## 2. The mask-alignment problem

While exploring D1, I noticed Dice scores hovering around 0.35 across all models — suspiciously low. Investigation showed that **52 % of D1 subjects had masks drawn on one MRI sequence (e.g. T1FS at 700 × 569) but loaded against a different sequence (e.g. T2 at 560 × 473)**. Without per-subject resampling, the mask was spatially shifted relative to the image.

**Fix:** detect the source sequence of each mask via affine-matrix comparison; resample the mask into the image space being used for training; verify visually that the resampled mask still tracks the organ.

**Impact:** MedSAM uterus Dice rose from **0.356 → 0.842**. The largest single improvement in the entire project, and it had nothing to do with model architecture.

## 3. Inter-rater analysis (the ground-truth ceiling)

D1 has 3 raters per structure → 3 pairwise comparisons per subject. I computed pairwise Dice across all 35 comparison pairs in D1. **8 of those 35 pairs had raters annotating on different MRI sequences** — those pairs were artificially low due to the same alignment issue, and were excluded from the inter-rater estimate.

**Corrected inter-rater Dice:**
- Uterus: **0.605 ± 0.386**
- Ovary: **0.345 ± 0.377**

Comparison to other organ-segmentation tasks (Figure: `fig_interrater.png`):

| Organ + task | Inter-rater Dice |
|---|---|
| Liver | 0.92 |
| Kidney | 0.87 |
| Prostate | 0.83 |
| Pancreas | 0.78 |
| Uterus (this work) | 0.61 |
| **Ovary in endometriosis MRI** | **0.345** |

Ovary in endometriosis MRI is the lowest reported inter-rater Dice across these organ-segmentation tasks. The implication: **model Dice cannot meaningfully exceed inter-rater Dice** — at ~0.35, the ovary segmentation problem has a hard upper bound near human noise.

## 4. U-Net pipeline

### 4.1 Baseline multi-class

Standard 4-channel softmax (background + uterus + ovary + endometrioma). 15 epochs, CPU training.

**Test result:** uterus 0.656, ovary **0.000**, endometrioma **0.000**.

Multi-class softmax with 95+ % background pixels produces no gradient for minority classes. RAovSeg's own ablation [Liang et al., 2025] proves this: removing their slice classifier drops ovary Dice from 0.290 → 0.013.

### 4.2 Attention U-Net

Same architecture with attention gates between encoder and decoder. **Did worse for uterus on test** (0.633 vs 0.656) — extra parameters appear to overfit on this small dataset. Endometrioma rose to 0.225.

### 4.3 Binary U-Net + Focal Tversky Loss

One 2-class model per organ. Train only on slices containing the target organ.

**Focal Tversky Loss:**
$$\text{TI} = \frac{\text{TP}}{\text{TP} + \alpha \cdot \text{FN} + \beta \cdot \text{FP}}, \quad \alpha = 0.8, \beta = 0.2$$
$$\text{FTL} = (1 - \text{TI})^\gamma, \quad \gamma = 1.3$$

False negatives are penalised 4× more heavily than false positives — appropriate for clinical use where missing the organ entirely is worse than over-segmenting.

**Test results:** ovary **0.298** (matches RAovSeg's published 0.290 with a simpler pipeline), endometrioma **0.556 when detected** (0.389 averaged across all slices including non-detection).

## 5. MedSAM pipeline

[Ma et al., 2024] — Segment Anything Model fine-tuned on a large medical-imaging corpus. Accepts an image + bounding-box prompt → returns a segmentation mask.

### 5.1 Oracle box (zero-shot upper bound)

Pass GT bounding box. Measures what MedSAM can do *if a perfect prompt is provided*.

**Result:** 0.842 / 0.744 / 0.696. Upper bound; not clinically realistic since obtaining the GT box requires the segmentation we're trying to produce.

### 5.2 Simulated manual bounding box

Models what a radiologist would draw by hand. Noise scaled by inter-rater disagreement:

$$\sigma = d_\text{bbox} \times 0.15 \times (1 - D_\text{inter-rater})$$

Organs where radiologists disagree more get noisier boxes. 5 noisy boxes per slice, predictions averaged.

**Result:** 0.805 / 0.683 / 0.641 — the most clinically-realistic of the MedSAM regimes.

### 5.3 Decoder fine-tuning

Freeze MedSAM's image encoder (heavy ViT). Fine-tune only the mask decoder on UT-EndoMRI training subjects. Image embeddings are pre-computed once and cached, making CPU fine-tuning feasible (~2 h).

**Result: 0.857 / 0.783 / 0.744 — best of all methods.** Still requires oracle boxes at test time.

### 5.4 Auto-prompting pipeline

Chain: U-Net or binary U-Net → bounding box from predicted mask → MedSAM with that box → refined mask. End-to-end automatic. Documented working chain; bottlenecked by upstream U-Net quality, so not the headline result.

## 6. Complete method comparison

| Method | Uterus | Ovary | Endometrioma | Test-time prompt |
|---|---|---|---|---|
| Multi-class U-Net | 0.656 | 0.000 | 0.000 | none |
| Attention U-Net | 0.633 | 0.000 | 0.225 | none |
| Binary U-Net + FTL | — | 0.298 | 0.556† | none |
| MedSAM, oracle box | 0.842 | 0.744 | 0.696 | **GT box** (unrealistic) |
| MedSAM, simulated box | 0.805 | 0.683 | 0.641 | noisy box |
| **MedSAM, decoder FT** | **0.857** | **0.783** | **0.744** | **GT box** (unrealistic) |
| RAovSeg (Liang et al.) | — | 0.290 | — | none |
| nnU-Net (Liang et al.) | — | 0.272 | — | none |
| Inter-rater (humans) | 0.605 | 0.345 | — | — |

† 0.556 when detected; 0.389 averaged including non-detection slices.

## 7. Why the project earned 90+

ENGG\*6302 (Prof. Ukwatta) reading-course evaluation rewards original investigation, not just course material reproduction. The four contributions:

1. **Identification and correction of the mask-alignment problem** affecting 52 % of D1 subjects — a finding that benefits anyone using UT-EndoMRI going forward.
2. **First evaluation of MedSAM on pelvic endometriosis MRI** — the foundation-model literature had not been applied to this dataset.
3. **Binary models matching published baselines with a simpler pipeline** — 0.298 vs RAovSeg's 0.290 with one model per organ instead of a two-stage classifier + segmenter cascade.
4. **First endometrioma segmentation result on UT-EndoMRI** — the original paper did not attempt this structure.
