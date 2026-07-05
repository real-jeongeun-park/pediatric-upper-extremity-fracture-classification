# Dual-view Supervised Contrastive Learning for Classification of Pediatric Upper Extremity Fractures

This repository contains the implementation associated with the paper **"Dual-view Supervised Contrastive Learning for Classification of Pediatric Upper Extremity Fractures."**

**Authors:** Yumin Shin¹†, Jeongeun Park²†, Daesung Kang¹*</br>
¹ School of Bio-Health Convergence, Sungshin Women's University</br>
² School of AI Convergence, Sungshin Women's University

## Overview

Pediatric fractures are among the most common childhood injuries, but the immaturity of a child's skeleton makes radiographic interpretation difficult, leading to frequent diagnostic errors in clinical practice. Upper extremity fractures account for a large share of pediatric fractures, and the ulna in particular is a common site of missed diagnoses. Diagnostic accuracy also varies substantially with reader experience, with a large gap in miss rates between specialists and residents.

This project proposes a **dual-view classification model** based on **Supervised Contrastive Learning (SupCon)** that leverages both anteroposterior (AP) and lateral X-ray projections to accurately classify pediatric upper extremity fractures. The model uses a two-stage framework: (1) supervised contrastive pretraining to learn robust feature representations, followed by (2) fine-tuning of the entire network for the downstream classification task.


## Method

### Model Architecture

- Each patient's AP and lateral projection images are passed through a **shared-weight encoder** to extract per-view feature vectors.
- The two extracted features are fused via either **cross-attention** or **simple concatenation**.
- The fused representation is passed through a **multi-layer perceptron (MLP)** to predict a single fracture class.

<img width="1380" height="1098" alt="image01" src="https://github.com/user-attachments/assets/a67a8698-b7b3-42ec-95c7-6b67d8b8b41f" />

### Training Strategy: Pretraining + Fine-tuning

**Stage 1 — Pretraining:**
- A ResNet-34 backbone pretrained on ImageNet is used as the shared encoder.
- Each view is encoded into a 512-dim feature vector, then projected through a projection head into a 128-dim embedding space and L2-normalized.
- Samples belonging to the same class or acquired from the same patient are treated as **positive pairs**; all others are **negative pairs**.
- The model is trained with the Supervised Contrastive Loss (SupCon), enabling richer and more discriminative representation learning than standard self-supervised approaches.

**Stage 2 — Fine-tuning:**
- The pretrained encoder weights are transferred to the classifier.
- The two view-specific feature vectors are fused (cross-attention or concatenation) and passed through an MLP to predict the final fracture class.
- The entire network (encoder + classifier) is fine-tuned end-to-end using cross-entropy loss.

## Dataset

- **PediURF**: a dataset of 10,530 pediatric upper extremity X-ray images collected from Shenzhen Children's Hospital, with one AP and one lateral projection per patient.
- **Classes** (by fracture location):
  - Proximal ulna and radius fractures — 528 cases
  - Midshaft ulna and radius fractures — 1,319 cases
  - Distal ulna and radius fractures — 3,374 cases
  - Class ratio ≈ 1 : 2.3 : 5.9, reflecting real-world clinical fracture frequency
- **Split**: patient-level train / validation / test split of 64 : 16 : 20 to prevent data leakage and ensure generalization.

### Preprocessing & Augmentation

- All images resized to 224×224.
- Random horizontal/vertical flips applied with 50% probability.
- Color jitter (brightness, contrast, etc.) applied.
- Normalization using ImageNet mean/std statistics.
- The same random augmentation parameters are applied to both AP and lateral views to preserve spatial correspondence.

## Experimental Setup

- **Baselines:** ResNet-34, BYOL, SimCLR, compared against the proposed SupCon-based model.
- **Pretraining loss:** SupCon loss with temperature τ = 0.07.
- **Fine-tuning loss:** Cross-entropy.
- **Optimizer:** Adam, with Cosine Annealing learning-rate scheduling.
- **Batch size:** 64.
- Contrastive models (SupCon, BYOL, SimCLR) pretrained for 200 epochs with encoder learning rate 1e-3.
- Fine-tuning performed for 50 epochs per configuration, sweeping encoder/classifier learning-rate pairs: (1e-6, 1e-5), (1e-5, 1e-4), (1e-4, 1e-3); the best combination was selected per model.
- **Metrics:** Accuracy (Acc), Precision (Pre), Sensitivity (Sen), F1-score (F1).

## Results

| Model | Fusion | Acc | Pre | Sen | F1 |
|---|---|---|---|---|---|
| ResNet-34 | Concatenation | 0.923 | 0.926 | 0.923 | 0.924 |
| ResNet-34 | Cross-attention | 0.926 | 0.926 | 0.926 | 0.926 |
| BYOL | Concatenation | 0.915 | 0.918 | 0.915 | 0.915 |
| BYOL | Cross-attention | 0.855 | 0.855 | 0.855 | 0.855 |
| SimCLR | Concatenation | 0.922 | 0.922 | 0.922 | 0.922 |
| SimCLR | Cross-attention | 0.924 | 0.925 | 0.924 | 0.924 |
| SupCon | Concatenation | 0.929 | 0.929 | 0.929 | 0.929 |
| **SupCon** | **Cross-attention** | **0.930** | **0.930** | **0.930** | **0.930** |

**Key findings:**
- The **SupCon cross-attention model** achieved the best performance across all metrics, with F1-score and precision both reaching **0.930**, a 0.4% improvement over the ResNet-34 baseline.
- SupCon consistently achieved top performance under both fusion strategies.
- BYOL's performance dropped sharply with cross-attention fusion, likely because BYOL's pretraining encourages high embedding similarity between the two views, limiting the benefit of cross-view information extraction.
- SimCLR outperformed BYOL regardless of fusion method, likely because SimCLR's use of negative pairs (unlike BYOL, which uses only positive pairs) helps it capture subtler inter-class differences between fracture sites.
- t-SNE visualizations show that SupCon embeddings form well-separated, distinct clusters per class, while SimCLR shows partial clustering with class overlap, and BYOL embeddings show almost no class separation — visually confirming that supervised contrastive learning produces more discriminative representations than self-supervised alternatives.
- With a sensitivity of 0.930, the proposed model suppresses the fracture miss rate to as low as approximately **7.0%**, compared to miss rates of up to 34% reported for non-specialist readers in prior literature.

## Conclusion

The proposed SupCon-based dual-view model achieved the best performance across all four evaluation metrics, demonstrating strong potential as a clinical decision-support tool to help non-specialists in pediatric emergency settings make more reliable fracture diagnoses.

**Limitations & Future Work:** This study used only a single ResNet-34 backbone and a fixed batch size of 64 during pretraining. Future work will explore additional backbones (e.g., ResNet-50, EfficientNet-B1) to validate the generality of the proposed training strategy, along with hyperparameter optimization (e.g., batch size) to further improve learning efficiency and diagnostic accuracy.

## References

1. Kamaci S. et al., "Epidemiology of Pediatric Fractures and Effect of Socioeconomic Status on Fracture Incidence in Türkiye: A Nationwide Analysis of 2 Million Fractures," *Journal of Pediatric Orthopaedics*, vol. 45, 2025.
2. Tang S. et al., "A Comprehensive X-ray Dataset for Pediatric Ulna and Radius Fractures Analysis," *Scientific Data*, vol. 13, 2026.
3. Lempesis V. et al., "Time trends in pediatric fracture incidence in Sweden during the period 1950–2006," *Acta Orthopaedica*, vol. 88, 2017.
4. Smith J. E., Tse S., Barrowman N., Bilal A., "Missed fractures on radiographs in a pediatric emergency department," *Canadian Journal of Emergency Medicine*, vol. 18, 2016.
5. Thammaroj T. et al., "Accuracy and Determinants of Radiographic Diagnosis in Pediatric Hand Fractures," *Cureus*, vol. 17, 2025.
6. Khosla P. et al., "Supervised Contrastive Learning," *Advances in Neural Information Processing Systems*, 2020.
7. Chen K., Zhuang D., Chang J. M., "SuperCon: Supervised contrastive learning for imbalanced skin lesion classification," *arXiv:2202.05685*, 2022.
8. Janisch M. et al., "Pediatric radius torus fractures in x-rays—how computer vision could render lateral projections obsolete," *Frontiers in Pediatrics*, vol. 10, 2022.
9. Grill J. et al., "Bootstrap your own latent - a new approach to self-supervised learning," *Advances in Neural Information Processing Systems*, 2020.
10. Chen T. et al., "A simple framework for contrastive learning of visual representations," *International Conference on Machine Learning*, 2020.

## Citation

If you use this work, please cite:

```
Shin Y, Park J, Kang D. "Dual-view Supervised Contrastive Learning for Classification of Pediatric Upper Extremity Fractures."
```
