---
layout: post
title: "[Validation Report] IEEE PHM 2012 베어링 데이터셋을 활용한 VAE 모델 검증 및 성능 분석"
date: 2026-03-09
categories: [PHM, Anomaly Detection]
tags: [VAE, IEEE-PHM-2012, Model-Validation, Confusion-Matrix]
---

# IEEE PHM 2012 베어링 데이터셋을 활용한 VAE 모델 검증 및 성능 분석

## 1. 개요 (Abstract)
本レポートでは、FEMTO-ST Institute が公開する **IEEE PHM 2012 Prognostics Data Challenge** 複数ベアリングデータセットを用いて、MALT システムの VAE(Variational Autoencoder) による異常検知性能を精査しました。

## 2. 核心検証結果 (Key Metrics)
| Metric | Value |
| :--- | :--- |
| **ROC AUC** | 0.7748 |
| **Average Precision (AP)** | 0.6358 |
| **Optimal Threshold** | 2.6279 |

### 2.1. オセハンリョウ (Confusion Matrix @ Threshold 2.628)
- **True Positive (正検出)**: 689
- **False Positive (誤検出)**: 174
- **False Negative (未検出)**: 821
- **True Negative (正正常)**: 5,850

---
*本レポートは MALT システムによって自動生成・検証されました。*
