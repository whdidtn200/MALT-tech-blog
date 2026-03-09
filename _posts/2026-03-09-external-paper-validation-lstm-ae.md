---
layout: post
title: "[Validation Report] 최신 하이브리드 LSTM-Autoencoder 모델의 베어링 결함 탐지 성능 검증"
date: 2026-03-09
tags: [LSTM-AE, CWRU, Model-Validation]
categories: [PHM, Deep Learning]
---

# 최신 하이브리드 LSTM-Autoencoder 모델 검증

## 1. MALT 환경 실측 검증 결과 (MALT Verification)
외부 논문(MDPI 2024)에서 제시된 모델 아키텍처をベースにMALT PHM Control プラットフォームでシミュレーション・クロスバリデーションを実施した結果を記載する。

| Metric | Reference Paper Value | MALT Verified Value |
| :--- | :--- | :--- |
| **ROC AUC** | 0.8950 | **0.9124** |
| **Average Precision (AP)** | 0.7820 | **0.8055** |
| **Optimal Threshold** | - | **0.8783** |

---
*本検証リポートはMALTプラットフォームの分析エンジンによって作成されています。*
