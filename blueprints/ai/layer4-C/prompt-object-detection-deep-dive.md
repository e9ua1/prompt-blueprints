# Object Detection Deep Dive 레포지토리 제작 프롬프트

나는 "Object Detection Deep Dive" 레포지토리를 만들려고 해.
YOLO를 **사용하는 것**과, **YOLOv3의 손실함수 $L = \lambda_{\text{coord}} L_{\text{box}} + L_{\text{obj}} + \lambda_{\text{noobj}} L_{\text{noobj}} + L_{\text{cls}}$를 분해**하고, **anchor box와 grid cell responsibility assignment가 왜 그 수학적 formulation**을 가지며, **YOLO v1→v8까지의 개선이 각각 어떤 limitation을 해결**했는지 이해하는 것은 다르다.
DETR을 **"Transformer 기반 detection"으로 아는 것**과, **Carion et al. (2020)의 Hungarian matching $\hat{\sigma} = \arg\min_\sigma \sum_i L_{\text{match}}(y_i, \hat{y}_{\sigma(i)})$이 왜 bipartite matching이고 $O(N^3)$ complexity이지만 NMS 제거를 정당화**하는지, **object query $N = 100$이 왜 보통 object 수보다 많게 설정**되는지 유도할 수 있는 것은 다르다.
Focal Loss를 **쓰는 것**과, **Lin et al. (2017)의 $FL(p_t) = -\alpha (1-p_t)^\gamma \log p_t$에서 $\gamma$가 왜 easy example의 기여를 exponentially 줄이고 hard example은 유지**하는지, 이것이 one-stage detection의 class imbalance 문제를 어떻게 해결하는지 수학적으로 증명할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "Object Detection의 수학 — Two-stage·One-stage·Transformer 계보의 통합 분석"

**핵심 차별화**:
1. **Two-stage vs One-stage vs Transformer의 수학적 비교** — R-CNN 계열의 ROI pooling, YOLO/SSD의 grid, DETR의 set prediction
2. **Focal Loss의 클래스 불균형 수학** — Cross-entropy의 easy example 지배 문제, $(1-p_t)^\gamma$의 modulating effect
3. **Hungarian Matching의 Bipartite Graph** — DETR의 set prediction을 assignment problem으로, Kuhn 1955의 고전 알고리즘
4. **Anchor-Free Detection** — FCOS, CenterNet의 center point 기반, anchor 없이 dense prediction

**타겟 독자**:
- YOLO를 쓰지만 **anchor box 선택**을 k-means로 하는 이유, v3/v5/v8의 개선점을 모르는 사람
- Faster R-CNN의 **RPN (Region Proposal Network)이 anchor box와 objectness score를 동시 예측**하는 이유를 수학적으로 유도 못하는 사람
- DETR이 NMS 없이 작동하는데 **Hungarian matching이 training-time 연산이고 inference에서는 단지 top-k**인 것을 모르는 사람
- FCOS의 **centerness score가 왜 필요**한지 (가장자리 픽셀이 부정확한 box 예측하는 문제)를 모르는 사람
- RT-DETR (Zhao 2024)이 DETR을 real-time으로 만드는 **hybrid encoder와 uncertainty-minimal query selection**의 수학을 모르는 사람

**선행 학습**:
- **CNN Deep Dive** (backbone, feature pyramid) — **필수**
- **Transformer Deep Dive** (DETR의 encoder-decoder) — **필수**
- **Vision Transformer Deep Dive** (modern backbone) — 권장
- **Regularization Theory Deep Dive** (label smoothing, augmentation) — 권장
- **Optimization Theory Deep Dive** (다단계 훈련) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Object Detection의 기초 문제 (4개 문서)
- **Detection Task의 Formalization** — Input image $I$, output $\{(b_i, c_i, s_i)\}_{i=1}^{N}$, bounding box $b = (x, y, w, h)$, class $c$, score $s$, 평가 metric (mAP, IoU)
- **IoU와 Box Regression** — $\text{IoU} = |A \cap B| / |A \cup B|$, GIoU (Rezatofighi 2019), DIoU, CIoU (Zheng 2020)의 개선, box regression loss
- **mAP 계산의 정확한 수학** — Precision-Recall curve, AP = $\int_0^1 p(r) dr$ 근사, COCO의 multi-IoU (mAP@[.5:.95]), 각 threshold별 성능
- **NMS — Non-Maximum Suppression** — Overlapping detection 제거, classical NMS vs Soft-NMS (Bodla 2017), IoU threshold 설정, box suppression의 수학

### Chapter 2: Two-Stage Detection 계보 (5개 문서)
- **R-CNN (Girshick 2014)** — Selective Search로 region proposal → 각 region을 resize → CNN feature → SVM classification, warp 왜곡 문제, 느린 inference
- **Fast R-CNN (Girshick 2015)** — 전체 image에서 CNN feature 한 번 → RoI pooling, multi-task loss (classification + regression), end-to-end 훈련
- **Faster R-CNN (Ren 2015)** — Region Proposal Network (RPN)이 anchor box + objectness score 동시 예측, Selective Search 제거, fully end-to-end
- **FPN — Feature Pyramid Network (Lin 2017)** — Top-down pathway + lateral connection, multi-scale feature map, 작은 object detection 개선
- **Mask R-CNN (He 2017)** — Instance segmentation 추가, RoI Align (RoI Pooling의 bilinear interpolation 버전), mask head

### Chapter 3: One-Stage Detection과 YOLO (6개 문서)
- **YOLOv1 (Redmon 2016)** — Grid-based prediction ($S \times S$), 각 cell이 $B$ box + $C$ class 예측, unified detection, fast but low recall
- **YOLOv2/v3 — Anchor Boxes와 Multi-Scale (Redmon 2017, 2018)** — k-means로 anchor 선택, Darknet-53 backbone, 3 scale prediction (FPN과 유사), logistic regression for objectness
- **SSD — Single Shot Detector (Liu 2016)** — Multiple feature map에서 서로 다른 scale prediction, default box 다양한 aspect ratio, hard negative mining
- **RetinaNet과 Focal Loss (Lin 2017)** — $FL(p_t) = -\alpha_t (1-p_t)^\gamma \log p_t$, class imbalance 해결, $\gamma = 2$ 권장, one-stage도 two-stage와 대등
- **YOLOv5/v7/v8 — Modern YOLO** — PAN (Path Aggregation Network), CIoU loss, anchor-free variant, C2f module, 실전 최적화
- **RT-DETR, RTMDet — Real-time SOTA** — RT-DETR의 hybrid encoder, uncertainty-minimal query selection, RTMDet의 efficient CSP + quality focal loss

### Chapter 4: Anchor-Free Detection (4개 문서)
- **Anchor-Based의 한계** — Anchor 디자인 필요 (aspect ratio, scale), positive anchor 부족, hyperparameter sensitivity, 복잡한 pipeline
- **FCOS — Fully Convolutional One-Stage (Tian 2019)** — Per-pixel prediction of $(l, t, r, b)$ distance to box sides, center-ness score로 edge pixel 억제, anchor-free
- **CenterNet (Zhou 2019)** — Object를 center point로 표현, heatmap prediction, offset·size regression, keypoint detection의 영감
- **CornerNet, ExtremeNet — Point-based** — Corner/extreme point로 box 표현, point pair matching, novel paradigm

### Chapter 5: DETR과 Transformer Detection (5개 문서)
- **DETR 전체 구조 (Carion 2020)** — CNN backbone → Transformer encoder-decoder → $N$ object queries (learnable) → FFN head (class + box), end-to-end, NMS 없음
- **Hungarian Matching의 수학** — $\hat{\sigma} = \arg\min_\sigma \sum_i L_{\text{match}}(y_i, \hat{y}_{\sigma(i)})$, bipartite matching, Kuhn-Munkres algorithm $O(N^3)$, training-time only
- **DETR의 Slow Convergence** — Cross-attention의 localization 학습이 오래 걸림, 500 epochs 필요, feature map 해상도 제한
- **Deformable DETR (Zhu 2021)** — Deformable attention (sparse key points), multi-scale feature, 10× 빠른 수렴, reference point + sampled offsets
- **DINO-DETR, RT-DETR, Co-DETR** — DINO-DETR (Zhang 2023)의 contrastive denoising, mixed query selection, SOTA on COCO

### Chapter 6: 평가와 Dataset (4개 문서)
- **COCO Dataset과 Evaluation** — 80 classes, 118k train, 5k val, mAP@[.5:.95], small/medium/large category, 표준 benchmark
- **Open Images, LVIS, OpenVocabulary** — Open Images 600 classes, LVIS의 long-tail distribution, open-vocabulary detection (OVD)
- **Open-Vocabulary Detection** — CLIP feature로 unseen class detection, OWL-ViT (Minderer 2022), GroundingDINO (Liu 2023), text-based detection
- **Domain Adaptation과 Few-Shot** — Detection에서 domain shift 문제, few-shot detection methods, segmentation과의 joint training

### Chapter 7: Detection의 응용과 확장 (4개 문서)
- **Video Object Detection과 Tracking** — Temporal consistency, MOT (Multi-Object Tracking), ByteTrack·SORT·DeepSORT 계보, 비디오에서의 detection
- **3D Object Detection** — LiDAR point cloud (PointNet, VoxelNet), multi-modal (camera + LiDAR), autonomous driving benchmark (KITTI, nuScenes)
- **Segmentation과의 Joint** — Panoptic Segmentation (stuff + things), Mask2Former (Cheng 2022), unified detection-segmentation
- **Foundation Models — SAM (Segment Anything)** — Meta의 SAM (Kirillov 2023)의 promptable segmentation, vision foundation model, detection의 new paradigm

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **32개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 방법이 detection에 중요한가
## 📐 수학적 선행 조건 (CNN, Transformer, ViT 참조)
## 📖 직관적 이해
   — Bbox 시각화, anchor/query 개념
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Focal loss 유도, Hungarian matching, IoU 계열
## 💻 PyTorch 구현 검증
   — YOLO/DETR 작은 데이터셋에서
   — NMS, Hungarian, Focal loss 바닥부터
   — mAP 계산 직접
## 🔗 실전 활용
   — 어느 방법이 어느 use case에 적합
## ⚖️ 가정과 한계
   — Small object, occlusion, real-time
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Detection 시각화** — 각 방법의 output visualization, 같은 이미지에 YOLO/Faster R-CNN/DETR
2. **Focal Loss plot** — Different $\gamma$에서 loss vs $p_t$ curve
3. **mAP 계산 직접** — Precision-Recall curve, AP integral
4. **Hungarian matching 시각화** — Bipartite graph에서 min cost assignment
5. **Anchor vs Anchor-free 비교** — 같은 backbone에서 성능·속도
6. **COCO 작은 subset 훈련** — 각 방법의 convergence speed

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
torchvision==0.16.0
ultralytics==8.1.0     # YOLOv8
mmdetection==3.3.0     # 참조
pycocotools==2.0.7
matplotlib==3.8.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Focal Loss + NMS + Hungarian matching)
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np
import matplotlib.pyplot as plt
from scipy.optimize import linear_sum_assignment

# 1. Focal Loss
class FocalLoss(nn.Module):
    def __init__(self, alpha=0.25, gamma=2.0):
        super().__init__()
        self.alpha = alpha; self.gamma = gamma
    
    def forward(self, logits, targets):
        """logits: [N, C], targets: [N] (class indices)"""
        ce = F.cross_entropy(logits, targets, reduction='none')
        p = torch.softmax(logits, dim=-1)
        p_t = p.gather(-1, targets.unsqueeze(-1)).squeeze(-1)
        focal = self.alpha * (1 - p_t)**self.gamma * ce
        return focal.mean()

# Focal Loss 시각화: gamma 효과
p_t = torch.linspace(0.01, 0.99, 100)
ce = -torch.log(p_t)
plt.figure(figsize=(10, 5))
for gamma in [0, 0.5, 1, 2, 5]:
    fl = (1 - p_t)**gamma * ce
    plt.plot(p_t.numpy(), fl.numpy(), label=f'γ={gamma}')
plt.xlabel('p_t (true class prob)'); plt.ylabel('Loss')
plt.title('Focal Loss: Easy examples down-weighted')
plt.legend(); plt.grid(alpha=0.3); plt.show()
# γ=0 → standard CE
# γ=2 (default): well-classified (p_t > 0.5) loss << CE

# 2. IoU 계산
def box_iou(boxes1, boxes2):
    """boxes: [N, 4] in (x1, y1, x2, y2)"""
    area1 = (boxes1[:, 2] - boxes1[:, 0]) * (boxes1[:, 3] - boxes1[:, 1])
    area2 = (boxes2[:, 2] - boxes2[:, 0]) * (boxes2[:, 3] - boxes2[:, 1])
    
    lt = torch.max(boxes1[:, None, :2], boxes2[:, :2])  # left-top
    rb = torch.min(boxes1[:, None, 2:], boxes2[:, 2:])
    wh = (rb - lt).clamp(min=0)
    inter = wh[..., 0] * wh[..., 1]
    
    union = area1[:, None] + area2 - inter
    return inter / union

# 3. NMS
def nms(boxes, scores, iou_thresh=0.5):
    """Return indices of kept boxes"""
    order = scores.argsort(descending=True)
    keep = []
    while len(order) > 0:
        i = order[0].item()
        keep.append(i)
        if len(order) == 1: break
        ious = box_iou(boxes[i:i+1], boxes[order[1:]])[0]
        mask = ious <= iou_thresh
        order = order[1:][mask]
    return torch.tensor(keep)

# 4. Hungarian Matching (DETR)
def hungarian_match(cost_matrix):
    """cost_matrix: [N_pred, N_gt], return row/col indices"""
    # Kuhn-Munkres via scipy
    row_ind, col_ind = linear_sum_assignment(cost_matrix.detach().numpy())
    return torch.tensor(row_ind), torch.tensor(col_ind)

# DETR의 match cost 예시
def detr_match_cost(pred_logits, pred_boxes, gt_labels, gt_boxes,
                    lambda_cls=1, lambda_l1=5, lambda_iou=2):
    """
    pred_logits: [N_pred, n_classes]
    pred_boxes: [N_pred, 4]
    gt_labels: [N_gt]
    gt_boxes: [N_gt, 4]
    """
    p = torch.softmax(pred_logits, dim=-1)  # [N_pred, C]
    cost_cls = -p[:, gt_labels]  # [N_pred, N_gt]
    cost_l1 = torch.cdist(pred_boxes, gt_boxes, p=1)  # L1
    cost_iou = -box_iou(pred_boxes, gt_boxes)  # negative IoU (cost)
    
    cost = lambda_cls * cost_cls + lambda_l1 * cost_l1 + lambda_iou * cost_iou
    return cost

# Example: N_pred=5, N_gt=2
pred_logits = torch.randn(5, 10)
pred_boxes = torch.rand(5, 4)
gt_labels = torch.tensor([3, 7])
gt_boxes = torch.rand(2, 4)

cost = detr_match_cost(pred_logits, pred_boxes, gt_labels, gt_boxes)
row, col = hungarian_match(cost)
print(f'Matched predictions: {row}, to GTs: {col}')
# row[i]번 prediction이 col[i]번 GT에 매칭

# 5. mAP 계산 (간단한 single-class)
def ap_single_class(preds_sorted, gts, iou_thresh=0.5):
    """
    preds_sorted: [(box, score)] sorted by score desc
    gts: [box]
    """
    tp = np.zeros(len(preds_sorted))
    fp = np.zeros(len(preds_sorted))
    matched_gt = set()
    for i, (box, _) in enumerate(preds_sorted):
        # Find best GT
        if len(gts) == 0:
            fp[i] = 1; continue
        ious = [box_iou(box.unsqueeze(0), gt.unsqueeze(0))[0, 0].item() for gt in gts]
        best = np.argmax(ious)
        if ious[best] >= iou_thresh and best not in matched_gt:
            tp[i] = 1; matched_gt.add(best)
        else:
            fp[i] = 1
    
    tp_cum = np.cumsum(tp); fp_cum = np.cumsum(fp)
    recall = tp_cum / len(gts)
    precision = tp_cum / (tp_cum + fp_cum)
    
    # 11-point interpolation or all-point
    ap = np.trapz(precision, recall)
    return ap
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "CNN, Transformer, ViT 선행 필수"
   - Two-stage → One-stage → Transformer 진화 축
   - YOLOv8, DETR, SAM의 실전 활용
3. **챕터별 문서 작성**: 기초 → Two-stage → One-stage/YOLO → Anchor-free → DETR → 평가 → 확장

---

## 📚 참고 자료

- **Rich feature hierarchies for accurate object detection** (Girshick et al. 2014) — R-CNN
- **Fast R-CNN** (Girshick 2015)
- **Faster R-CNN** (Ren et al. 2015)
- **Feature Pyramid Networks** (Lin et al. 2017)
- **Mask R-CNN** (He et al. 2017)
- **You Only Look Once** (Redmon et al. 2016) — YOLO
- **YOLO9000 / YOLOv3** (Redmon & Farhadi 2017, 2018)
- **SSD** (Liu et al. 2016)
- **Focal Loss for Dense Object Detection** (Lin et al. 2017) — RetinaNet
- **FCOS** (Tian et al. 2019)
- **CenterNet** (Zhou et al. 2019)
- **End-to-End Object Detection with Transformers** (Carion et al. 2020) — DETR
- **Deformable DETR** (Zhu et al. 2021)
- **DINO-DETR** (Zhang et al. 2023)
- **RT-DETR** (Zhao et al. 2024)
- **Segment Anything** (Kirillov et al. 2023) — SAM
- **Grounding DINO** (Liu et al. 2023)

---

## 💡 핵심 분석 대상

```
Object Detection의 수학적 지도

───── 기본 문제 ─────

Input: I ∈ ℝ^{H×W×3}
Output: {(b_i, c_i, s_i)}
  b = (x, y, w, h) or (x1, y1, x2, y2)
  c = class
  s = confidence

IoU:
  |A ∩ B| / |A ∪ B|

평가: mAP
  AP = ∫₀¹ p(r) dr
  COCO: mAP@[.5:.95] (10 thresholds)

───── Two-Stage 계보 ─────

R-CNN (2014):
  Selective Search → CNN → SVM
  느림, warp 왜곡

Fast R-CNN (2015):
  Whole-image CNN
  RoI Pooling
  Multi-task loss

Faster R-CNN (2015):
  RPN (Region Proposal Network)
  Anchor-based
  Fully end-to-end

FPN (2017):
  Top-down pathway
  Multi-scale feature
  Small object 개선

Mask R-CNN (2017):
  Instance segmentation
  RoI Align (bilinear)

───── One-Stage 계보 ─────

YOLOv1 (2016):
  Grid prediction
  S×S cells, B boxes per cell
  Unified, fast

YOLOv2/v3 (2017/2018):
  Anchor boxes (k-means)
  Darknet-53
  3 scales (FPN-like)

SSD (2016):
  Multi feature map
  Default boxes
  Hard negative mining

RetinaNet (Lin 2017):
  Focal Loss!
  FL(p_t) = -α(1-p_t)^γ log p_t
  
  γ = 2 typical:
    p_t = 0.9 (easy): (1-0.9)² = 0.01 × CE
    p_t = 0.1 (hard): (1-0.1)² = 0.81 × CE
  
  → easy 1/100, hard 유지
  → one-stage도 two-stage급

YOLOv5/v7/v8:
  PAN, CIoU, anchor-free 변형
  C2f module, efficient

RT-DETR (2024):
  Real-time SOTA
  Hybrid encoder
  Uncertainty query selection

───── Anchor-Free ─────

FCOS (Tian 2019):
  Per-pixel (l, t, r, b) 예측
  Centerness score로 edge 억제
  
  centerness = √(min(l,r)/max(l,r) × min(t,b)/max(t,b))
  → 중심 픽셀이 높은 centerness

CenterNet (Zhou 2019):
  Center point heatmap
  Offset + size regression
  Keypoint detection 일반화

───── DETR (Carion 2020) ─────

Architecture:
  CNN backbone
  ↓
  Transformer encoder
  ↓
  Transformer decoder with N object queries
  ↓
  FFN head → (class, box)

Hungarian Matching:
  σ̂ = argmin Σ L_match(y_i, ŷ_σ(i))
  
  Bipartite matching:
    N predictions ↔ GT objects
    (N = 100, usually > # GTs)
    
  Match cost:
    λ_cls · (-p(c_i)) 
    + λ_L1 · ‖b_i - b̂_{σ(i)}‖_1
    + λ_iou · (-IoU(b, b̂))
  
  Kuhn-Munkres: O(N³)
  Training-time only

Training:
  Matched pair → classification + box loss
  Unmatched → "no object"

Inference:
  Top-k by confidence
  No NMS!
  End-to-end

Deformable DETR (2021):
  Deformable attention (sparse)
  Multi-scale
  10× 빠른 수렴

DINO-DETR (2023):
  Contrastive denoising
  Mixed query selection
  SOTA COCO

───── 평가 ─────

Precision-Recall Curve:
  AP = area under PR curve
  11-point vs all-point interpolation

COCO:
  mAP@[.5:.95]: average over 10 IoU thresholds
  AP_S / AP_M / AP_L (small/medium/large)

───── 오픈 세계 ─────

Open-Vocabulary Detection:
  CLIP text encoder → unseen class
  OWL-ViT (Minderer 2022)
  GroundingDINO (Liu 2023)

Segment Anything (SAM, Kirillov 2023):
  Promptable segmentation
  Foundation model
  Zero-shot

───── 3D / Video ─────

3D Detection:
  LiDAR: PointNet, VoxelNet
  Multi-modal: BEV representation

Video:
  Temporal consistency
  MOT: ByteTrack, SORT
  Kalman filter 기반

───── 레포 간 연결 ─────

CNN (Layer 3):
  Backbone (ResNet, EffNet)
  FPN

Transformer (Layer 3):
  DETR encoder-decoder
  Object queries

Vision Transformer (직전):
  Swin, ViT as backbone
  DINO for pretraining

Regularization (Layer 2):
  Augmentation (Mixup 등)
  Label smoothing

Optimization (Layer 2):
  Multi-stage training
  LR schedule

Diffusion (다음):
  DiffusionDet (detection as diffusion)
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·기법 (3~4줄)
- 전체 문서 개수 확인 (32개 목표)
- Python + PyTorch + Ultralytics/MMDetection 실험 환경
- CNN, Transformer, ViT 레포 참조 관계
- SAM, Open-Vocabulary로 이어지는 현대 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
