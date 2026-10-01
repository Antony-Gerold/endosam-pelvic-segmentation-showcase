# Methodology

Everything here follows the final report ([PDF](../report/EndoSAM_FinalReport.pdf)).

## 1. Dataset: UT-EndoMRI

UT-EndoMRI [5] is the first public benchmark for this task: 124 subjects from two hospitals.

| Cohort | Source | Subjects | Sequences | Raters |
|---|---|---|---|---|
| D1 | Memorial Hermann, 15 sites, 9 scanners | 51 | T2, T1FS | 3 per structure |
| D2 | Texas Children's | 73 | T2, T2FS, T1FS, T1 | 1 |

Each structure (uterus, ovary, endometrioma/cyst) is a separate binary NIfTI file. Data is split 70/15/15 by subject with seed 42. Metric: Dice Similarity Coefficient. Boundary metrics such as Hausdorff distance were not computed.

## 2. Mask alignment problem and fix

The first MedSAM run gave 0.356 Dice for uterus. MRI scanners capture multiple sequences of the same patient in one session; T2 might be 560 by 473 pixels while T1FS is 700 by 569. The radiologist annotated on whichever sequence showed the organ best, but a standard loader takes the first image file it finds, so a 700 by 569 mask ends up on a 560 by 473 image.

**Fix:** check the width and height of each mask file and find the image file with matching dimensions. This is done per mask, because different structures in the same subject were sometimes annotated on different sequences. After the fix, 96 of 124 subjects are usable versus 59 before, and MedSAM uterus Dice went from 0.356 to 0.842 (+0.486).

## 3. Inter-rater agreement

D1 has 3 raters per structure, so pairwise Dice was computed for all combinations. 8 of 35 comparisons had raters who annotated on different sequences, which gives artificially low Dice because of the alignment issue. After excluding them:

- Uterus: 0.605 ± 0.386
- Ovary: 0.345 ± 0.377

The ovary value is lower than liver (0.92), kidney (0.87), prostate (0.83), and pancreas (0.78). Radiologists themselves struggle with ovary boundaries in endometriosis.

## 4. Preprocessing

Images are clipped to the 1st and 99th percentile of foreground pixels and scaled to 0-1. U-Net input is 256 by 256 with 3 channels (target slice plus one above and one below). MedSAM input is 1024 by 1024, as required by the ViT-B encoder.

## 5. MedSAM

MedSAM [6] takes an image and a bounding-box prompt and outputs a mask. It has an image encoder (ViT-B, 90M parameters), a prompt encoder (6K), and a mask decoder (4M). The official Zenodo weights were used; HuggingFace community weights gave much worse results (0.30-0.50 versus 0.85+).

- **Oracle box:** tightest rectangle around the ground truth plus a 5-10 pixel margin.
- **Simulated manual box:** noise scaled by inter-rater agreement, σ = d_bbox × 0.15 × (1 − D_inter-rater). Five noisy boxes per slice, predictions averaged.
- **Decoder fine-tuning:** encoder and prompt encoder frozen, only the decoder trained, to avoid overfitting on about 80 representative samples. Encoder embeddings pre-computed once. Loss: BCE + Dice. AdamW, learning rate 1e-4, weight decay 1e-4, gradient clipping at 1.0, 10 epochs. Training data: 1110 slices from 99 subjects. Test: 234 slices from 25 held-out subjects.

## 6. U-Net

- **Baseline and Attention:** EfficientNet-B0 [19] encoder (5.3M parameters) with a U-Net decoder, 4-class softmax output. The attention version adds SCSE modules [18] after each decoder block. Cross-entropy loss, Adam at LR 1e-3, 15 epochs on CPU.
- **Binary models:** multi-class U-Net reached Dice near zero for ovary and endometrioma, because softmax competition with 95%+ background pixels leaves no gradient for minority classes. Separate 2-class U-Nets were trained per organ, using only slices that contain the target organ, with Focal Tversky Loss:

$$TI = \frac{TP}{TP + 0.8 \cdot FN + 0.2 \cdot FP}, \qquad L_{FTL} = 0.7 \cdot (1 - TI)^{1.33} + 0.3 \cdot L_{CE}$$

  The 0.8 weight on false negatives means missing an organ pixel costs 4 times more than a false alarm. LR 3e-4, after a first try at 1e-3 collapsed at epoch 7.

## 7. Auto-prompting

To make segmentation fully automatic, bounding boxes were extracted from multi-class U-Net and binary model predictions and fed to the MedSAM decoder.

Bracketed numbers refer to [references.md](references.md).
