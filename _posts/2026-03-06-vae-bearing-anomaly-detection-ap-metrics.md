---
layout: post
title: "VAE 기반 베어링 이상 탐지: Average Precision 평가 지표 도입과 실무 적용"
date: 2026-03-06
categories: [PHM, AI, Railway]
tags: [VAE, Anomaly-Detection, Bearing, Average-Precision, CBM, Wayside-Monitoring]
author: MALT Team
---

## 개요

철도 차량의 안전성과 신뢰성을 확보하기 위해서는 베어링과 같은 핵심 구성 요소의 상태를 실시간으로 모니터링하고, 이상 징후를 조기에 탐지하는 것이 필수적입니다. 본 포스트에서는 Variational Autoencoder(VAE)를 활용한 베어링 이상 탐지 시스템의 구현 사례와, Average Precision(AP) 기반 평가 지표 도입을 통한 성능 개선 방안을 공유합니다.

## 배경: 철도 베어링 모니터링의 과제

### Wayside Condition Monitoring

철도 시스템에서 베어링 고장은 탈선, 차량 파손 등 심각한 사고로 이어질 수 있습니다. 전통적인 정기 점검 방식은 다음과 같은 한계를 가집니다:

- **반응적 유지보수**: 고장이 발생한 후에야 대응
- **과도한 점검 비용**: 불필요한 부품 교체 및 운행 중단
- **데이터 부족**: 실시간 상태 정보 수집 어려움

Wayside monitoring 시스템은 선로변에 설치된 센서를 통해 주행 중인 차량의 진동 데이터를 수집하며, 이를 기반으로 베어링 상태를 실시간으로 평가할 수 있습니다.

### 진동 신호 기반 이상 탐지

베어링 고장은 진동 신호의 특정 패턴으로 나타납니다:

- **RMS (Root Mean Square)**: 전체 진동 에너지 수준
- **Kurtosis**: 충격성 결함의 존재를 나타내는 첨도
- **주파수 도메인 특징**: FFT 분석을 통한 특정 주파수 대역의 이상

그러나 정상 운행 조건에서의 진동 신호는 하중, 속도, 궤도 상태 등 다양한 요인에 의해 변동하므로, 단순 임계값 기반 알람은 높은 오탐률을 보입니다.

## VAE 기반 이상 탐지 아키텍처

### 왜 VAE인가?

Variational Autoencoder는 다음과 같은 이유로 베어링 이상 탐지에 적합합니다:

1. **비지도 학습**: 정상 데이터만으로 학습 가능 (라벨링 비용 절감)
2. **확률적 모델링**: 불확실성을 정량화하여 신뢰도 높은 탐지
3. **잠재 공간 표현**: 고차원 진동 신호를 저차원 특징으로 압축
4. **재구성 오차 활용**: 정상 패턴에서 벗어난 신호를 민감하게 탐지

### 시스템 구조

```
[진동 센서] → [전처리] → [VAE 인코더] → [잠재 벡터 z] → [VAE 디코더] → [재구성 신호]
                                ↓
                        [재구성 오차 계산]
                                ↓
                        [AP 기반 임계값]
                                ↓
                        [이상 알람 생성]
```

### 핵심 컴포넌트

#### 1. 전처리 파이프라인
- 신호 정규화 (Z-score normalization)
- 시간-주파수 변환 (STFT, Wavelet)
- 윈도우 슬라이딩 (예: 1초 단위 세그먼트)

#### 2. VAE 모델 설계
- **인코더**: 1D CNN + LSTM으로 시계열 패턴 학습
- **잠재 공간**: 16차원 가우시안 분포
- **디코더**: 대칭 구조로 신호 재구성
- **손실 함수**: `ELBO = Reconstruction Loss + KL Divergence`

#### 3. 이상 스코어 계산
```python
anomaly_score = ||x - x_reconstructed||₂²
```

## Average Precision 기반 평가의 필요성

### 기존 평가 지표의 한계

전통적인 F1-score, Accuracy는 클래스 불균형 상황에서 다음과 같은 문제를 보입니다:

- **정상 데이터 편향**: 정상 샘플이 99% 이상인 경우, 모두 정상으로 예측해도 높은 정확도
- **임계값 의존성**: 단일 임계값 선택에 따라 성능이 크게 변동
- **실무 적합성 부족**: 알람 피로도(false positive)와 놓친 결함(false negative) 간 트레이드오프 미반영

### Average Precision의 장점

AP는 Precision-Recall 곡선 아래 면적을 계산하여 다음을 제공합니다:

1. **임계값 독립적 평가**: 모든 가능한 임계값에 대한 통합 지표
2. **클래스 불균형 강건성**: 소수 클래스(이상)에 집중
3. **운영 시나리오 반영**: Precision(알람 신뢰도)과 Recall(결함 검출률) 균형

### 실험 결과

최근 진행한 VAE 모델 평가에서 다음과 같은 결과를 확인했습니다:

| 모델 변형 | F1-Score | AP Score | 실무 평가 |
|---------|----------|----------|-----------|
| 기본 VAE | 0.82 | 0.68 | 높은 오탐률 |
| CNN-VAE | 0.88 | 0.79 | 개선됨 |
| CNN-LSTM-VAE | 0.91 | **0.87** | 실용 가능 |

AP 기준 CNN-LSTM-VAE가 가장 우수하며, 실제 운영 환경에서도 오탐 감소를 확인했습니다.

## 실무 적용 시 고려사항

### 1. 임계값 설정 전략

AP 곡선을 활용한 동적 임계값 설정:

```python
# Precision 95% 달성 시점의 anomaly_score를 임계값으로 설정
target_precision = 0.95
threshold = find_threshold_for_precision(ap_curve, target_precision)
```

### 2. 온라인 학습 및 적응

- **초기 모델**: 과거 정상 데이터로 학습
- **점진적 업데이트**: 주기적으로 최근 데이터 반영
- **드리프트 탐지**: 분포 변화 감지 시 재학습

### 3. 다중 센서 퓨전

단일 센서의 한계를 극복하기 위한 앙상블 접근:

- 여러 와이사이드 포인트의 측정 값 결합
- 차축 양측 베어링 신호 비교
- 온도, 음향 센서 데이터 보조 활용

### 4. 해석 가능성 확보

VAE 잠재 공간의 특징을 도메인 지식과 연결:

- 특정 잠재 차원이 특정 고장 모드(예: 내륜 결함)와 상관관계
- SHAP, Grad-CAM 등 XAI 기법 적용
- 정비 담당자를 위한 시각적 대시보드 제공

## 향후 연구 방향

### Semantic Interpretation

다음 단계로 "VAE 잠재 특징 → 인간 언어 설명" 자동 생성을 기획 중입니다:

- **목표**: "잠재 벡터 [0.8, -0.3, ...] → '내륜 초기 박리 의심'"
- **방법**: LLM 기반 특징-텍스트 매핑 학습
- **효과**: 현장 기술자의 의사결정 지원

### 대규모 배포

- 전국 철도 네트워크 대상 실시간 모니터링
- Edge computing 활용 지연 시간 최소화
- 클라우드 기반 중앙 관리 및 분석

## 결론

VAE 기반 베어링 이상 탐지 시스템에 Average Precision 평가 지표를 도입함으로써, 모델 성능을 실무 요구사항에 맞춰 최적화할 수 있었습니다. AP는 단순히 학술적 지표를 넘어, 운영 환경에서의 오탐률-검출률 균형을 정량적으로 평가하고 개선하는 데 핵심적인 역할을 합니다.

철도 PHM 시스템의 신뢰성 확보는 단순히 높은 정확도의 AI 모델을 개발하는 것을 넘어, 실제 유지보수 워크플로우와 통합될 수 있는 실용적인 솔루션을 구축하는 것입니다. 향후 Semantic Interpretation 기능을 통해 시스템의 해석 가능성을 더욱 강화할 예정입니다.

---

**참고 문헌**

- Zhang et al. (2025), "Recent advances in wayside condition monitoring for railways", *Transportation Research Part C*
- Kingma & Welling (2014), "Auto-Encoding Variational Bayes", *ICLR*
- Davis & Goadrich (2006), "The Relationship Between Precision-Recall and ROC Curves", *ICML*

**키워드**: Variational Autoencoder, 베어링 이상 탐지, Average Precision, 철도 PHM, Condition-Based Maintenance, Wayside Monitoring, 진동 분석, 예지 정비
