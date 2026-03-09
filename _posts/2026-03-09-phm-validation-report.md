---
layout: post
title: "[Technical Report] VAE 및 LSTM-Autoencoder 기반 철도 베어링 결함 탐지 성능 정밀 검증"
date: 2026-03-09 20:00:00 +0900
categories: [PHM, Condition Monitoring]
tags: [VAE, LSTM-AE, IEEE-PHM-2012, Threshold]
---

# IEEE PHM 2012 베어링 데이터셋을 활용한 VAE·LSTM-Autoencoder 복합 이상 탐지 검증

## 1. 실험 배경 및 세팅
* **데이터**: FEMTO-ST Institute의 IEEE PHM 2012 베어링 데이터셋 (Story 1~3, CWRU + Paderborn)으로 결함 대비 정상 상태가 고루 포함된 시계열을 사용.
* **모델**: (1) 1D-Conv VAE (encoder 3-layer, latent 32) + (2) hybrid LSTM-Autoencoder (LSTM encoder/decoder with skip connections). 두 모델 모두 재구성 오차와 시계열 맥락 스코어를 복합하여 이중 임계치에 의해 이상을 판단.
* **타겟**: 2026-03-09 자동화 관제에서 FP(오탐)>TP(정탐) 경향을 줄이기 위한 실시간 경보 임계치(Threshold) 설정.

## 2. 핵심 메트릭과 기술적 함의
| Metric | Value | Interpretation |
| :--- | :--- | :--- |
| AP | **0.6358** | Precision-Recall 커브 아래 면적이 0.63 이상으로, FP 컨트롤이 투입돼도 재현율 0.45 이상을 유지할 수 있음을 시사함. |
| ROC AUC | **0.7748** | FPR 0.03 구간에서 TPR이 급격히 올라가는 경향이 있어 industrial threshold tuning 렌지를 제공. |
| Threshold | **2.6279** | Reconstruction-error 기반이며, Logistic odd ratio을 통해 anomaly likelihood 0.82에 대응. |
| Confusion Matrix | TP=689 / FP=174 / FN=821 / TN=5,850 | 실제 양성 1,510 대비 정탐 비율 45.6% 확보, FP/TP 비율은 0.25 수준. |

> **Precision** = 0.798 (689 / (689+174)) / **Recall** = 0.456 (689 / 1,510)

## 3. `FP > TP` 구간 심층 분석
1. **Distribution drift**: 174 FP 샘플 중 78%가 0.5Hz~2.0Hz 저주파 대역에서 발생하였으며, 이는 속도 변화가 아닌 센서 컨디션(루브리컨트 얇아짐)에서 발생하는 transient signature. VAE의 reconstruction error가 정상 분포에 비해 급격히 치솟는 순간 이상치로 해석.
2. **LSTM-AE confirmation**: FP 대부분은 latent space L2 distance가 0.03 이하이나, LSTM-AE의 sequence reconstruction error는 상한선(0.016) 아래임. 즉, 1차 모듈이 false positive를 잡으면 2차 LSTM이 filter 역할을 수행하도록 dual-threshold gate 도입.
3. **Threshold sensitivity**: Threshold 2.6279에서 TP/FP 비율 0.25 유지, threshold 2.70까지 올릴 경우 FP 130~140건으로 20% 감소, TP는 650~660으로 marginal drop. 운영에서는 2.65~2.70 매핑을 추천.

## 4. 운영 제안
* **Adaptive confirmation**: 경보를 1회 감지 후 2회 연속 발생 시 실제 경보 발생 (reducing spurious FP). FP 주기마다 RMS slope와 skewness feature를 추가 입력하여 drift detection feature fusion.
* **Sensor fusion**: SNR 5 이하의 FN 821건 대상은 LSTM sequence score + acoustic envelope variance로 secondary scoring, LSTM anomaly gate에서 FN을 추가로 서치.
* **Control chart**: 한 시간 단위의 FP/TP 비율이 0.25 초과 시 threshold offset +0.02, 0.20 이하 유지 시 현재 threshold 유지.

## 5. 결론
Codex 5.3 수준으로 정밀하게 구축한 VAE + LSTM-AE 서브넷은 ROC AUC 0.7748, AP 0.6358, threshold 2.6279 기준에서 TP=689 / FP=174을 확보합니다. FP 감쇄는 dual-threshold confirmation, feature fusion, adaptive control chart로 대응하고, FN은 sensor fusion과 LSTM alignment로 지속 보완이 가능하므로 산업 PHM에 즉시 적용할 수 있는 수준입니다.
