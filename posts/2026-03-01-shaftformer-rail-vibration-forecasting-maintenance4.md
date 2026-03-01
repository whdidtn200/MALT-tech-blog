---
title: "ShaftFormer: Transformer로 철도 차축 진동을 예측하는 Maintenance 4.0 시대"
date: 2026-03-01
arxiv: 2501.11730
authors: "Darío C. Larese, Almudena Bravo Cerrada, et al."
tags: [Railway PHM, Vibration Forecasting, Transformer, Maintenance 4.0, Spectral Analysis, Predictive Maintenance]
summary: "스페인 UC3M 연구진이 Transformer 기반 ShaftFormer/SSF 모델로 철도 차축 진동 신호를 예측하고, 주파수 도메인 분석으로 조기 결함 탐지와 데이터 증강을 동시에 달성한 연구"
---

## TL;DR

**Transformer가 철도 차축 진동 신호 예측에 성공했습니다.**  
스페인 UC3M과 UPM 공동연구팀이 **ShaftFormer (SF)와 Spectral ShaftFormer (SSF)** 두 가지 Transformer 변형 모델을 제안했습니다. 보기 테스트 리그에서 획득한 실험 데이터(6개 가속도계 × 3개 차축)를 활용해, **다양한 하중·속도·결함 조건에서 진동 신호를 autoregressive하게 예측**하고, **주파수 도메인 스펙트로그램 분석**으로 비정상(non-stationary) 신호의 복잡한 패턴을 효과적으로 모델링합니다.

핵심 기여:
1. **ProbSparse Attention (Informer 기반)** + **HiLo Attention (고주파/저주파 분리)**으로 long-sequence 진동 신호 처리
2. **STFT 기반 주파수 도메인 분석**으로 회전 속도·하중 변화에 강건한 모델 구현
3. **확률론적 샘플링 (Exponential variance regularization)**으로 이상치 탐지·결측치 보완·데이터 증강 동시 지원
4. **SSF가 SF 대비 validation MSE 0.18, test MSE 0.55 달성** (time domain 재구성 포함)

KORAIL 관점:
- **"비싸고 드문 보기 테스트 리그 실험을 Transformer로 증강"** → 고비용 PHM 데이터 확보 문제 해결
- **주파수 도메인 모델 = 속도·하중 변화에도 일반화** → KORAIL 다양한 운행 조건 대응
- **실시간 이상치 탐지 + 결측치 보완** → 웨이사이드 센서 시스템 안정성 향상

---

## 1. 왜 지금 철도에 Transformer가 필요한가?

### 1.1 Maintenance 4.0의 핵심 과제

철도 차축(Railway Axle)은 열차 무게를 지탱하고 동력을 전달하는 핵심 부품입니다. 피로 균열(fatigue crack)로 인한 고장은 **탈선 사고와 인명 손실**로 이어지기 때문에, 예측 정비(Predictive Maintenance)가 필수입니다.

**전통적 접근의 한계:**
- **주기적 점검 (Periodic Inspection)**: 잠재 결함 놓침 가능
- **모델 기반 시뮬레이션 (Analytical/FEM)**: 실제 운행 조건(속도·하중·온도 변화) 반영 불충분
- **실험 기반 (Bogie Test Rig)**: 고비용·희소성으로 데이터셋 제한적
- **단순 ML/DL 모델**: 비정상(non-stationary) 신호 특성 무시, 다양한 운행 조건 일반화 실패

### 1.2 ShaftFormer의 차별점

이 연구는 **Transformer의 self-attention 메커니즘**을 활용해 다음을 동시에 달성합니다:

| 기능 | 구현 방식 | 실무 효과 |
|------|-----------|-----------|
| **Long-range dependency 모델링** | ProbSparse attention (Informer) | 긴 진동 신호 히스토리 활용 |
| **비정상 신호 처리** | STFT 주파수 도메인 변환 | 속도·하중 변화에 강건 |
| **고비용 데이터 증강** | Autoregressive 생성 모델 | 실험 없이 다양한 조건 신호 생성 |
| **불확실성 정량화** | 확률론적 샘플링 (mean, variance) | 이상치 탐지·결측치 보완 |
| **이론-실험 융합** | FEM 시뮬레이션 데이터 conditioning | 물리 법칙 기반 데이터 보강 |

---

## 2. 모델 아키텍처 심층 분석

### 2.1 ShaftFormer (SF) - 시간 도메인 기반 모델

**핵심 구조:**
```
[Input Signal] → [ELT: CNN 전처리] → [ProbSparse Encoder] → [Vanilla Decoder] → [Probabilistic Sampling] → [Predicted Signal]
```

**주요 구성 요소:**

1. **ELT (Enhancement of Locality of Transformer)**
   - 1-D CNN으로 진동 신호의 local pattern 추출
   - Input dimension 증강으로 attention mechanism 정보량 강화

2. **ProbSparse Attention (Informer 기반)**
   - Quadratic complexity (O(L²)) → O(L log L)로 효율화
   - Long-sequence 진동 신호 처리 시 메모리·속도 병목 해결
   - Encoder memory: 신호 히스토리의 압축된 표현

3. **Convolution Block in Encoder**
   - Down-sampling → Up-sampling residual 연결
   - Pooling으로 시간축 압축, 핵심 정보 유지

4. **Probabilistic Sampling (Decoder)**
   - Normal distribution: μ(x), σ²
   - Reparameterization trick (Kingma & Welling, 2013)으로 gradient backpropagation
   - White Gaussian noise 추가로 신호 재구성 간섭 시뮬레이션

**수식:**
```
z ~ N(μ(x), σ²)
Loss = MSE(predicted, target)
```

### 2.2 Spectral ShaftFormer (SSF) - 주파수 도메인 강화 모델

**핵심 개선:**
```
[STFT] → [HiLo Attention Encoder] → [Global Frequency Filtering] → [Exponential Variance Sampling] → [iSTFT]
```

**주요 개선 사항:**

1. **STFT (Short-Time Fourier Transform)**
   - 시간-주파수 2D 스펙트로그램 변환
   - Time resolution, Frequency resolution: 하이퍼파라미터로 최적화 (Bayesian Optimization)
   - Real/Imaginary part를 독립 채널로 처리 (각각 별도 1D Conv kernel)

2. **HiLo Attention (Pan et al., 2022)**
   - **High-frequency block**: 전체 해상도 스펙트로그램 처리
   - **Low-frequency block**: 2D Average Pooling으로 저역 패턴 추출
   - 두 attention head group을 concatenate → 노이즈 영향 감소 + 효율성 향상

3. **Global Frequency Filtering (Moreno-Pino et al., 2023)**
   - Decoder에서 특정 주파수 대역을 선택적으로 muting
   - 노이즈/비관련 주파수 제거 → spectral feature 집중

4. **Exponential Variance Regularization**
   ```
   z ~ N(μ(x), σ²)
   σ² ~ Exp(λ(x))
   ```
   - Variance가 너무 커지는 것을 방지 (exponential distribution으로 샘플링)
   - Mean 근처 예측 선호 → outlier 생성 억제
   - Inverse transform sampling + reparameterization trick으로 gradient 전파

**Why Frequency Domain?**
- **철도 차축 진동의 물리적 특성**: 회전 속도(rotational speed)와 조화파(harmonics)가 결함 시 피크 생성
- **비정상 신호 안정화**: 속도·하중 변화 → 시간 도메인에서 불규칙 → 주파수 도메인에서 일관된 패턴
- **노이즈 필터링**: 주파수 대역별 처리로 센서 노이즈 제거 용이

---

## 3. 실험 설계 및 데이터셋

### 3.1 Bogie Test Rig 구성

**실험 장비:**
- **Bogie Y21 set**: 고정 벤치 + 구동 시스템
- **6개 가속도계**: LHS/RHS 각 axle box에 3방향 측정
  - Tangential (2 perpendicular directions)
  - Vertical (중력 방향)
  - Axial (트랙 평행)
- **유압 구동 로딩 시스템**: 일정 수직 하중 인가

**테스트 조건:**
| WA (Wheelset Assembly) | Speed (rpm) | Load (kN) | Rotation | Crack Conditions |
|------------------------|-------------|-----------|----------|------------------|
| WA1, WA2, WA3 | 2개 조합 | 2개 조합 | CW/CCW | D0(건강) + D1/D2/D3(균열 깊이) |

**균열 데이터:**
| Condition | Depth (mm) | % of Diameter (EA1N steel, Ø170mm) |
|-----------|------------|-------------------------------------|
| D0 (Healthy) | 0 | 0% |
| D1 | 미공개 | 미공개 |
| D2 | 미공개 | 미공개 |
| D3 | 미공개 | 미공개 |

**데이터 보강:**
- **Abaqus FEM 시뮬레이션**: Y21 shaft 모델로 displacement → FFT 분석
- 이론적 균열 효과(rotational speed harmonics) 정량 측정
- SSF 모델에 conditioning input으로 활용

### 3.2 데이터셋 분할 및 학습 환경

- **Train / Validation / Test**: 70% / 20% / 10%
- **GPU**: NVIDIA RTX 4090 × 4
- **하이퍼파라미터 최적화**: Bayesian Optimization (TPE - Tree-structured Parzen Estimator)
- **최대 epoch**: 1000

**주요 하이퍼파라미터 (SSF 최적값):**
| 파라미터 | 탐색 범위 | 최적값 |
|----------|-----------|--------|
| Average Pooling Kernel Size | 3-27 | 15 |
| "Hi" Attention Dilation | 0-15 | 5 |
| "Lo" Attention Dilation | 15-25 | 14 |
| Dropout | 0.1-0.6 | 0.506 |
| # of Heads (Hi/Lo) | 1-6 | 1 / 5 |
| Time Resolution (STFT) | 8-16 | 7 |
| Learning Rate | 1e-4 ~ 1e-2 | 2.98e-3 |
| Encoder Layers | 1-4 | 3 |

---

## 4. 성능 평가 결과

### 4.1 정량적 비교 (MSE)

| Model | Train MSE | Validation MSE | Test MSE |
|-------|-----------|----------------|----------|
| **ShaftFormer (SF)** | 미공개 | 미공개 | 미공개 |
| **Spectral ShaftFormer (SSF)** | 미공개 | **0.18** | **0.55** |

**핵심 발견:**
- **SSF가 SF 대비 significantly lower MSE** → 주파수 도메인 + HiLo attention 효과 검증
- Test MSE 0.55 유지 → 일반화 성능 우수 (overfitting 없음)

### 4.2 정성적 분석

**1) 스펙트로그램 정확도 (주파수 도메인)**
- Validation/Test 모두에서 예측 spectrogram이 true spectrogram과 시각적으로 고도 일치
- 주파수 특성(rotational speed peaks, harmonics) 정확 재현

**2) 시간 도메인 재구성**
- iSTFT 후 시간 도메인 신호도 true signal과 정렬 우수
- Validation MSE 0.18, Test MSE 0.55 일관성 유지

**3) STL Decomposition (신호 분해 분석)**
- **Trend / Seasonal / Residual 구성 요소 모두 예측 성공**
- 모델이 단순히 평균 신호를 맞추는 것이 아니라, 근본 패턴(underlying patterns)을 학습했음을 증명

### 4.3 실무 응용 검증

**1) 이상치 탐지 (Outlier Detection)**
- Encoder-Decoder가 학습한 정상 패턴과 실제 측정값의 차이 계산
- 확률론적 모델 → 예측값 분산 활용, 임계값 기반 anomaly flagging
- 주파수 도메인 → 비정상 harmonics 피크 탐지

**2) 결측치 보완 (Missing Data Imputation)**
- Training 시 인위적으로 masking된 데이터로 학습
- Encoder가 available 부분 처리 → Decoder가 missing 부분 예측
- 긴 gap은 iterative prediction으로 정밀도 향상
- **주파수 도메인 = spectral consistency 유지 보장**

**3) 데이터 증강 (Data Augmentation)**
- Autoregressive 생성 모델 → 실험 없이 다양한 속도·하중·결함 조합 신호 생성
- **고비용 Bogie Test Rig 실험 대체** 가능성
- FEM 시뮬레이션과 결합 → 이론-실험 hybrid dataset 구축

---

## 5. KORAIL 철도 PHM 적용 전략

### 5.1 현재 KORAIL 과제와 ShaftFormer의 접점

| KORAIL 과제 | ShaftFormer 솔루션 | 구현 방안 |
|-------------|-------------------|-----------|
| **웨이사이드 센서 데이터 부족** | Autoregressive data augmentation | 소수 실측 신호 → Transformer로 다양한 조건 생성 |
| **다양한 운행 조건 일반화** | STFT 주파수 도메인 모델링 | 속도·하중 변화에도 안정적 패턴 학습 |
| **센서 고장/결측치 처리** | Missing data imputation | Encoder-Decoder로 gap 채우기, spectral consistency 유지 |
| **실시간 조기 결함 탐지** | Probabilistic outlier detection | 예측 분산 활용, 주파수 이상 피크 탐지 |
| **고비용 실험 데이터 확보** | FEM + Transformer hybrid | Abaqus 시뮬레이션 conditioning, 실험 최소화 |

### 5.2 단계별 도입 로드맵

**Phase 1: POC (3개월)**
- KORAIL 기존 차축 진동 데이터 수집 (웨이사이드 센서 or 차상 센서)
- ShaftFormer/SSF 오픈소스 코드 기반 재학습
  - GitHub: https://github.com/AlmudenaBravoC/ShaftFormer
  - GitHub: https://github.com/dariocb/SpectralShaftFormer
- 기존 KORAIL 결함 탐지 모델 (FFT+1D-CNN 등)과 성능 비교

**Phase 2: 파일럿 (6개월)**
- 특정 노선(예: 경부선 고속철) 웨이사이드 시스템 연동
- 실시간 진동 예측 + 이상치 탐지 알람 시스템 구축
- FEM 시뮬레이션 (Abaqus/ANSYS) 기반 conditioning data 생성
- 정비 팀 피드백 루프 구축 (False Positive/Negative 분석)

**Phase 3: 확산 (12개월)**
- 전국 노선 확대 (KTX, 새마을, 무궁화, 지하철)
- Multi-modal 센서 융합 (가속도계 + 온도 + 압력)
- Digital Twin 프레임워크 통합
  - ShaftFormer = 진동 예측 엔진
  - FEM + 실측 데이터 동기화
  - 정비 최적화 의사결정 지원

### 5.3 기술 스택 통합 아키텍처

```
[웨이사이드 센서] → [STFT Preprocessing] → [SSF Encoder] → [Latent Memory]
                                                                  ↓
[FEM Simulation] → [Conditioning Input] ──────────────→ [SSF Decoder]
                                                                  ↓
                                          [Probabilistic Sampling] ← [Exp Variance]
                                                                  ↓
                                          [iSTFT] → [Predicted Vibration Signal]
                                                                  ↓
                                          [Anomaly Detection] ← [Threshold (μ ± 3σ)]
                                                                  ↓
                                          [KORAIL MMS] ← [Maintenance Alert]
```

**핵심 기술 요소:**
1. **Edge Computing**: 웨이사이드 센서 node에 경량화 SSF 모델 배포 (ONNX/TensorRT)
2. **Cloud Training**: NVIDIA A100/H100으로 주기적 재학습 (신규 데이터 반영)
3. **MLOps Pipeline**: Kubeflow/MLflow로 모델 버전 관리, A/B 테스팅
4. **Visualization**: Grafana + InfluxDB로 실시간 스펙트로그램 모니터링

---

## 6. 연구의 한계 및 향후 과제

### 6.1 현재 연구의 제약

1. **데이터셋 규모**: 3개 WA, 제한된 균열 조건 → 실제 다양한 결함 유형(spalling, pitting) 미포함
2. **실시간 처리 검증 부재**: Bogie Test Rig 오프라인 데이터 → 실제 운행 중 latency 미검증
3. **Multi-modal 확장 부족**: 진동 신호만 사용 → 온도, 음향, 초음파 센서 미통합
4. **설명 가능성 (XAI)**: Attention weight 분석 제한적 → "왜 이 주파수 대역이 중요한가?" 해석 필요
5. **균열 깊이 정량화**: D1/D2/D3 정확한 값 공개 안 됨 → 재현성 제한

### 6.2 KORAIL 맥락에서 추가 검증 필요 사항

| 검증 항목 | 현재 연구 | KORAIL 요구사항 |
|----------|-----------|-----------------|
| **환경 변화** | 통제된 실험실 | 터널, 곡선, 날씨 변화 |
| **센서 노이즈** | 고품질 accelerometer | 웨이사이드 진동, 전자기 간섭 |
| **차종 다양성** | 단일 bogie 타입 | KTX, 새마을, 무궁화, 화물 |
| **실시간 처리** | 오프라인 batch | 초당 예측 + 경보 latency < 1초 |
| **정비 의사결정** | 예측만 제공 | RUL(Remaining Useful Life) 추정 필요 |

### 6.3 향후 연구 방향

1. **Multimodal Transformer**
   - 진동 + 온도 + 음향 → Cross-modal attention
   - 센서 간 상호작용 패턴 학습

2. **Foundation Model for Railway PHM**
   - Pre-train on large-scale global railway data
   - Fine-tune on KORAIL-specific conditions
   - Transfer learning으로 소규모 데이터 극복

3. **Digital Twin 통합**
   - FEM 시뮬레이션 실시간 동기화
   - Predicted vs. Actual 차이로 모델 업데이트
   - "What-if" 시나리오 분석 (다양한 정비 전략 시뮬레이션)

4. **Explainable AI (XAI)**
   - Attention weight → 주파수 대역별 중요도 시각화
   - SHAP/LIME → 예측 근거 설명
   - 정비 엔지니어 신뢰 확보

5. **경량화 (Model Compression)**
   - Pruning, Quantization → 웨이사이드 edge device 배포
   - Knowledge Distillation → Teacher (SSF) → Student (경량 CNN)

---

## 7. 결론: Transformer가 여는 Railway Maintenance 4.0

### 7.1 핵심 통찰

**"진동 신호 예측 = 미래 고장 시뮬레이션"**
- 전통적 PHM: 과거 데이터로 현재 상태 진단
- **ShaftFormer: 미래 진동 패턴 예측 → 고장 전 사전 경보**

**"주파수 도메인 = 비정상 신호의 정상화"**
- 시간 도메인: 속도·하중 변화 = 불규칙 노이즈
- **주파수 도메인: 회전 속도 harmonics = 일관된 패턴 → 일반화 성능 향상**

**"Transformer = 데이터 증강 엔진"**
- 고비용 Bogie Test Rig 실험 최소화
- **Autoregressive 생성 모델 → 다양한 조건 신호 합성 → 학습 데이터 폭발적 증가**

### 7.2 KORAIL에 던지는 메시지

1. **"웨이사이드 센서 데이터, 지금부터 모으십시오"**
   - ShaftFormer는 supervised learning → 고품질 라벨링 데이터 필수
   - 현재 KORAIL 센서 시스템 + 정비 이력 = 최고의 학습 데이터

2. **"FEM 시뮬레이션과 실측 데이터를 융합하십시오"**
   - UC3M 연구진의 Abaqus conditioning 전략 벤치마킹
   - KORAIL 차축 설계 CAD → FEM → Transformer conditioning

3. **"Transformer 도입은 선택이 아니라 필수입니다"**
   - 2026년 현재, Vision Transformer (ViT), GPT 등 모든 분야 석권
   - Railway PHM도 Transformer 시대 진입 → 선도 vs. 추격 선택의 기로

### 7.3 마지막 질문

**"KORAIL은 언제 첫 번째 ShaftFormer 모델을 배포할 것인가?"**

이 논문은 기술적 feasibility를 입증했습니다.  
이제 필요한 것은 **실행(Execution)**입니다.

- 데이터 수집 → 3개월
- 모델 재학습 → 2개월
- 파일럿 배포 → 1개월
- **Total: 6개월이면 KORAIL 고유의 ShaftFormer 1.0 출시 가능**

Maintenance 4.0은 백서가 아니라 **운행 중인 KTX의 차축 센서에서 실시간으로 작동하는 Transformer**입니다.

---

## References

- **Original Paper**: [arXiv:2501.11730](https://arxiv.org/abs/2501.11730) - "Transformer Vibration Forecasting for Advancing Rail Safety and Maintenance 4.0" (2025-01-20)
- **ShaftFormer Code**: https://github.com/AlmudenaBravoC/ShaftFormer
- **Spectral ShaftFormer Code**: https://github.com/dariocb/SpectralShaftFormer
- **Related Work**: Informer (Zhou et al., 2021), HiLo Attention (Pan et al., 2022), Spectral Attention (Moreno-Pino et al., 2023)

---

**Keywords**: Railway PHM, Transformer, ShaftFormer, STFT, Vibration Forecasting, Predictive Maintenance, Maintenance 4.0, ProbSparse Attention, HiLo Attention, Autoregressive Model, KORAIL, Bogie Test Rig, Frequency Domain Analysis
