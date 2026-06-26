# Clinical Context: Engineering Decisions for Brain Tumor MRI

> Motaz Alqaoud, PhD — Biomedical Engineer

The gap between an ML benchmark and a useful clinical tool is mostly engineering, not model architecture. This document covers the decisions in this repo that come from clinical constraints — not from best-practices articles.

---

## 1. Voxel Spacing

MRI scanners do not produce standardized images. Two T1-weighted brain scans from different sites can have voxel sizes of 1×1×1 mm and 0.9×0.9×3 mm respectively — the same physical tumor looks geometrically different in each.

This matters for brain tumors specifically because tumor volume feeds into treatment decisions. A glioma measured at 3 cm³ vs 4.5 cm³ has different implications for radiation planning. If your model was trained on 1 mm isotropic data and runs on 3 mm slice-thickness acquisitions, its spatial measurements are off by a factor proportional to that spacing difference.

Resampling to isotropic spacing before any processing is not optional. It happens in `nifti_loader.py` before normalization or augmentation, so the model always operates in a consistent physical coordinate system.

---

## 2. Augmentation Has Anatomy Constraints

The brain is approximately bilaterally symmetric, which makes horizontal flipping look like a safe augmentation. It isn't.

Tumor location is a clinical feature. Left temporal lobe involvement affects language function. Right vs left frontal gliomas carry different surgical risk profiles. A model trained with random horizontal flips learns that tumor laterality is arbitrary — and it isn't, either for diagnosis or for treatment planning.

90° rotations are similarly problematic. MRI is acquired in defined orientations (axial, sagittal, coronal). A 90° rotation doesn't produce another valid brain MRI — it produces something a radiologist would never see in clinical practice. Models trained on these tend to fail when they encounter real protocol-conformant images from scanners they haven't seen.

What is safe: rotations within ±15°, small translations, mild intensity shifts within the T1 or T2 range. The augmentation in `transforms.py` stays within these constraints.

---

## 3. Loss Function Design for Class Imbalance

Brain tumor pixels make up roughly 0.5–3% of a typical MRI slice, depending on tumor type and slice position. Cross-entropy optimizes for pixel count, so it naturally learns to predict background. A model that does exactly that achieves 97%+ pixel accuracy and no clinical value. This is the default behavior of vanilla cross-entropy on medical segmentation tasks — not a hypothetical edge case.

Multi-class segmentation makes this worse. Background dominates, followed by normal brain parenchyma, then the tumor classes themselves — and within tumor classes, glioma is more common than pituitary, which is more common than meningioma in this dataset. A flat Dice weighting treats all classes equally; `WeightedDiceLoss` assigns higher weight to rarer classes to prevent them from collapsing during training.

The hybrid loss here combines three terms for three distinct failure modes:
- **Dice** — handles class imbalance at the segment level
- **Focal** — up-weights hard-to-classify pixels that Dice treats equally
- **Boundary** — penalizes imprecise tumor edges, which matter for volumetric accuracy

Per-class Dice is tracked separately during training because overall Dice can look acceptable while one class quietly collapses to zero.

---

## 4. Multi-Class Segmentation and WHO Grading

Glioma, meningioma, and pituitary adenoma are not interchangeable findings.

Gliomas are graded I–IV under WHO classification. Grade IV glioblastoma has a median survival of roughly 15 months with aggressive treatment. Grade I pilocytic astrocytomas are largely curable. Meningiomas are mostly benign (WHO grade I) and often managed with observation rather than intervention. Pituitary adenomas are frequently treated hormonally, without resection.

Collapsing these into a binary "tumor vs background" model produces something that detects mass but cannot contribute to clinical decision-making. Differentiating the tumor type — even with the uncertainty that a 2D imaging model carries — is the signal that makes segmentation clinically relevant.

This is reflected in the 4-class output (background + glioma + meningioma + pituitary) and why per-class Dice is the primary evaluation metric rather than mean Dice across all classes.

---

## 5. Reading in Three Planes

Neuroradiologists do not interpret a single axial slice in isolation. A finding that is ambiguous in the axial plane is often definitive in the sagittal or coronal view. Artifacts that mimic lesions in one plane commonly disappear in the others.

A model trained purely on axial slices and evaluated only on axial predictions is not fully validated — the other two planes are untested. The 3-plane viewer in `src/visualization/viewer.py` exists for this reason: any segmentation worth reporting should hold up in all three orientations before it influences a downstream decision.

---

## Further Reading

- [The Medical Segmentation Decathlon](http://medicaldecathlon.com) — multi-organ and tumor segmentation benchmarks
- [BraTS Challenge](https://www.synapse.org/brats) — the standard brain tumor segmentation benchmark dataset
- Ronneberger et al. (2015) — [U-Net: Convolutional Networks for Biomedical Image Segmentation](https://arxiv.org/abs/1505.04597)
- Menze et al. (2015) — [The Multimodal Brain Tumor Image Segmentation Benchmark (BraTS)](https://ieeexplore.ieee.org/document/6975210)

---

*Questions or corrections — open a GitHub issue or reach out on [LinkedIn](https://linkedin.com/in/motazalqaoud).*
