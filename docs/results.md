# Results

All numbers are from the final report ([PDF](../report/EndoSAM_FinalReport.pdf)).

## Alignment correction

| | Before | After |
|---|---|---|
| Usable subjects | 59 | 96 |
| MedSAM oracle (uterus) | 0.356 | 0.842 |
| Inter-rater (uterus) | 0.327 | 0.605 |

Going from 0.356 to 0.842 for uterus just by loading the right image was bigger than any model change in the study.

## MedSAM (Dice)

| Prompt | Uterus | Ovary | Endo |
|---|---|---|---|
| Oracle (304 slices) | 0.842 | 0.744 | 0.696 |
| Simulated manual (304 slices) | 0.805 | 0.683 | 0.641 |
| **Decoder fine-tuning (234 slices)** | **0.857** | **0.783** | **0.744** |

Zero-shot with oracle boxes already works well. Simulated manual boxes drop only 4 to 8 percent. Decoder fine-tuning gives the best numbers, improving over the zero-shot oracle by +0.015 (uterus), +0.039 (ovary), and +0.048 (endometrioma).

## U-Net (Dice, test)

| Method | Uterus | Ovary | Endo |
|---|---|---|---|
| Baseline U-Net | 0.656 | 0.000 | 0.000 |
| Attention U-Net | 0.557 | 0.000 | 0.225 |
| Binary Ovary | n/a | 0.298 | n/a |
| Binary Endo | n/a | n/a | 0.556* |

\* When detected. Overall mean: 0.389 (70 test slices).

Attention did worse for uterus on test (0.557 vs 0.656), probably from the extra SCSE parameters overfitting the small training set, but helped endometrioma appear at 0.225. The binary ovary model (0.298) matches the published RAovSeg (0.290) with a simpler setup: one model instead of a slice classifier plus a segmenter. The binary endometrioma overall mean of 0.389 reflects a detection rate of about 70%.

## Auto-prompting (Dice)

| Box source | Uterus | Ovary | Endo |
|---|---|---|---|
| Multi-class U-Net | 0.392 | 0.093 | 0.275 |
| Binary models | n/a | 0.266 | 0.362 |

Localization is the bottleneck. Multi-class U-Net finds ovary in 3 of 83 slices; binary models find it in 63 of 83, but the resulting 0.266 is far below the 0.783 from oracle boxes, because the decoder was trained with perfect boxes and does not handle rough ones well.

## Complete comparison (Dice)

| Method | Uterus | Ovary | Endo | Type |
|---|---|---|---|---|
| RAovSeg [5] | n/a | 0.290 | n/a | Published |
| nnU-Net [5] | n/a | 0.272 | n/a | Published |
| Inter-rater | 0.605 | 0.345 | n/a | Measured |
| Baseline U-Net | 0.656 | 0.000 | 0.000 | Test |
| Attention U-Net | 0.557 | 0.000 | 0.225 | Test |
| Binary Ovary | n/a | 0.298 | n/a | Test |
| Binary Endo | n/a | n/a | 0.556 | Test |
| MedSAM Oracle | 0.842 | 0.744 | 0.696 | 304 sl. |
| MedSAM Simulated | 0.805 | 0.683 | 0.641 | 304 sl. |
| MedSAM Dec. FT | 0.857 | 0.783 | 0.744 | 234 sl. |
| Auto-box (U-Net) | 0.392 | 0.093 | 0.275 | Test |
| Auto-box (Binary) | n/a | 0.266 | 0.362 | Test |

## Contributions stated in the report

1. Identification and correction of a mask alignment problem affecting more than half the dataset.
2. The first evaluation of MedSAM on pelvic endometriosis MRI.
3. Binary organ-specific models that match the published baseline with a simpler pipeline.
4. The first endometrioma segmentation result.

## Limitations

- All training on CPU, which limited U-Net to 15 epochs and ruled out nnU-Net.
- 2D slices only; no 3D volume information.
- The simulated bounding box uses a derived formula, not actual radiologist annotations, though it is grounded in measured inter-rater variability.
- The decoder still needs oracle boxes at test time.
- Dice only; no boundary metrics such as Hausdorff distance.
- The binary endometrioma result is from a small test set and would be more reliable with cross-validation.

## Future work

Training with perturbed boxes [16], a learnable prompt module [14], MedSAM-2 [17] for 3D context, nnU-Net [9] on GPU, and MedSAM-U [15] for uncertainty-based prompt refinement.

Bracketed numbers refer to [references.md](references.md).
