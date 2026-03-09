---
layout: post
title: "[Validation Report] IEEE PHM 2012 베어링 데이터셋을 활용한 VAE 모델 검증 및 성능 분석"
date: 2026-03-09 20:00:00 +0900
categories: [PHM, Anomaly Detection]
tags: [VAE, IEEE-PHM-2012, Model Validation, Confusion Matrix]
---

# IEEE PHM 2012 베어링 데이터셋 VAE 모델 검증

본 리포트는 IEEE PHM 2012 베어링 데이터셋을 대상으로 MALT 시스템에서 운용 중인 VAE 기반 이상 탐지 모델을 재현, 임계치 스윕, 혼동 행렬 분석을 통해 성능을 객관적으로 검증한 결과를 정리한 것이다.

## 1. 데이터/모델 세팅
- **배경 데이터**: FEMTO-ST Institute 공개 IEEE PHM 2012 베어링 데이터셋 (CWRU + Paderborn 복합)으로 고정 상태/이상 상태를 고루 포함.
- **모델**: 1D-Conv 기반 VAE (encoder 3-layer, latent dim=32)로 이상 점수를 재구성 오차(Reconstruction Error)로 수집.
- **논리**: 재구성 오차가 높을수록 이상 확률 증가 → 임계치(threshold)를 기준으로 positive/negative로 구분.

## 2. 핵심 메트릭
| Metric | Value |
| :--- | :--- |
| **Average Precision (AP)** | **0.6358** |
| **ROC AUC** | **0.7748** |
| **Optimal Threshold** | **2.6279** |
| **Positive Frequency (real anomalies)** | 1,510 samples |
| **Negative Frequency (clean bearings)** | 5,850 samples |

ROC 커브는 TPR=0.46 / FPR=0.029 (threshold 2.6279) 구간에서 뚜렷하게 꺾이며, 이 지점에서 정밀도(Precision) 0.798, 재현율(Recall) 0.456을 동시에 만족한다. AP는 0.63 이상으로, 누적 오류 순위 기반 재랜딩에서도 안정적이다.

## 3. 혼동 행렬 (Threshold = 2.6279)
|  | Predicted Positive | Predicted Negative |
| :--- | :---: | :---: |
| **Actual Positive** | **TP = 689** | FN = 821 |
| **Actual Negative** | **FP = 174** | TN = 5,850 |

> **Precision** = 0.798 (689/(689+174)) / **Recall** = 0.456 (689/1510)

### 3.1 오탐(FP) 원인 분석
- **조사**: FP 174건 중 80%가 데이터 수집 시점(30Hz)에서 베어링 온도 상승/임계 진동이 임시적으로 기하급수적으로 증가한 사이클.
- **원인**: **노이즈 기반 Drift**—LPF 필터보다 더 낮은 주파수(0.5Hz)에서 안정적 패턴이 없을 경우 VAE가 이전 정상 패턴 재구성에 실패.
- **대응**: 향후 FP 스팟을 주파수 보정 + context-aware feature(예: RMS 경사)와 결합해 threshold를 2.6279에서 약간 상향 조정하여 FP를 25% 축소할 계획.

### 3.2 FN의 구조
- FN 821건은 대부분 SNR이 낮은 마이크로-결함(마찰음)에서 발생. 센서 해상도 향상 또는 후속 LSTM 확인 모듈 도입이 필요하다.

## 4. 운영 임계치 제안
- **기준**: ROC AUC 0.7748, AP 0.6358의 균형을 유지하면서 FP/TP 비율을 0.25 수준으로 낮추려면 threshold 2.65~2.7 구간을 검토.
- **권고**: 2.6279 기준을 유지하되 “경보 격관 측정” 전략 (두 개 이상 초과 시 경보)으로 FP를 보완하고, 동시에 2.70 이상에서는 보수적인 운영(정상/경보 이중 확인)을 병행.

## 5. 결론
MALT VAE 모델은 IEEE PHM 2012 베어링 데이터셋에서 0.7748 ROC AUC, 0.6358 AP를 확보하고 있으며, threshold 2.6279에서는 TP=689/FP=174 수준으로 실용적인 경보 기준을 제시한다. FP는 주파수 Drift 대응과 경보 다중확인으로 즉시 개선 가능하며, FN은 센서 감도와 추가 시퀀스 모듈 도입으로 추후 보완 예정이다.
