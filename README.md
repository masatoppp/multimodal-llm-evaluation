# Multimodal LLM Evaluation
## Qwen3-VL と OpenAI による画像理解性能の比較

ローカル環境で動作する **Qwen3-VL** と、クラウドAPIで利用する **OpenAIのマルチモーダルモデル** に対して、同一画像・同一プロンプトを入力し、画像理解性能の違いを比較した検証プロジェクトです。

単純な回答比較ではなく、Ground Truth、LLM-as-a-Judge、人間レビュー、Atomic Fact、Precision / Recall / F1-scoreを組み合わせて評価しました。

---

## 1. 検証目的

本検証では、モデルごとの個別チューニングではなく、

> **同一画像・同一プロンプト・同一評価条件で、画像理解結果の違いを比較すること**

を目的としています。

モデルごとの追加プロンプト調整は原則として行わず、誤認・見落とし・部分一致もそのまま評価対象としました。

---

## 2. Architecture

Qwen3-VLとOpenAIによる画像推論は事前に実行し、結果をJSONへ保存しています。

公開用NotebookではLLMを再実行せず、保存済み結果を読み込んで評価・集計・可視化を行います。

```text
                ┌──────────────────┐
                │ Evaluation Image │
                └────────┬─────────┘
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
       ┌─────────────┐       ┌─────────────┐
       │  Qwen3-VL   │       │   OpenAI    │
       │ Local Model │       │  Cloud API  │
       └──────┬──────┘       └──────┬──────┘
              │                     │
              └──────────┬──────────┘
                         ▼
                    Save JSON
                         │
                         ▼
             ┌─────────────────────┐
             │ Evaluation Pipeline │
             │                     │
             │ Ground Truth        │
             │ Atomic Fact         │
             │ LLM-as-a-Judge      │
             │ Human Review        │
             │ Precision / Recall  │
             │ F1-score            │
             └──────────┬──────────┘
                        ▼
                  Public Notebook
```

この構成により、APIコストや推論結果の変動に依存せず、同じモデル出力に対して評価処理を再実行できます。

---

## 3. 検証データセット

定量評価には `001.png` ～ `005.png` の5枚を使用しました。

評価対象には、

- 複数物体の認識
- 数量
- 位置関係
- 背景
- OCR
- 画像全体の意味

などを含めています。

### データセット選定の方針

公開画像や既存の版権画像は、モデルの学習データに含まれている可能性があります。

その影響をできるだけ避けるため、

- 定量評価：自身で作成した手描き画像
- 定性評価：自身のStable Diffusion環境で新規生成した画像

を使用しました。

また、自作画像を使用することで、評価したい対象・数量・位置関係などを事前にコントロールし、Ground Truthを明確に設定できるようにしています。

### 定量評価画像の例

<img src="images/001.png" width="450" alt="001.png">

`001.png` では、猫・UFO・ヨット・海など複数の対象と、その位置関係を評価しています。

### 定性評価画像

<img src="images/000.png" width="450" alt="000.png">

`000.png` にはGround Truthを設定せず、自然な画像に対する両モデルの説明傾向を比較しています。

---

## 4. 使用モデル

### 画像理解モデル

**Qwen3-VL**

- `Qwen/Qwen3-VL-4B-Instruct`
- ローカル環境で実行

**OpenAI**

- `gpt-5.6-terra`
- OpenAI API経由

### Judge LLM

- `gpt-5.6-sol`

Judge LLMは画像そのものではなく、

- Ground Truth
- Qwen3-VLの回答
- OpenAIの回答

を比較し、一次評価を行います。

---

## 5. 評価方法

Ground Truthは画像理解モデルには与えていません。

評価時にGround Truthを、1つずつ独立して判定できる **Atomic Fact** へ整理しました。

例：

```text
黒猫が描かれている
黄色いUFOが描かれている
黒猫が黄色いUFOに乗っている
海に複数のヨットが浮かんでいる
```

5枚の画像から合計 **47個のAtomic Fact** を定義しています。

各Atomic Factは、モデル回答との一致度に応じて以下へ分類します。

- `MATCH`：完全一致
- `PARTIAL`：部分一致
- `MISS`：不一致・見落とし

これらは点数ではなく、評価ラベルです。

---

## 6. LLM-as-a-Judge と人間レビュー

Judge LLMでは、

- `matched`
- `partial`
- `missed`
- `unsupported`
- `unverified`

へ一次分類します。

ただし、LLMによる評価にも、

- `partial` と `missed` の境界
- 評価単位の粒度
- Ground Truthにない追加情報の扱い

などで揺らぎが確認されました。

そのため、Judge結果をそのまま最終評価にはせず、Ground Truthと固定済みモデル回答を確認し、人間レビューによって最終ラベルを固定しています。

---

## 7. Strict Precision / Recall / F1-score

本検証では、Atomic Factの最終ラベルを以下のように変換しています。

| 判定 | 評価 |
| --- | --- |
| `MATCH` | TP |
| `PARTIAL` | FP + FN |
| `MISS` | FN |
| `unsupported` | FP |
| `unverified` | 評価対象外 |

`PARTIAL` は完全な誤答ではありませんが、Strict評価では完全一致したAtomic FactのみをTPとしています。

例えば、

```text
Ground Truth:
黄色いUFOが描かれている

Model:
黄色い宇宙船が描かれている
```

の場合、

- 対象の大意は認識している
- ただし「UFO」としては完全一致していない

ため `PARTIAL` とします。

定性的には「惜しい回答」として残しつつ、Strict F1では完全正解として加点しません。

---

## 8. 評価結果

| Model | MATCH | PARTIAL | MISS | Precision | Recall | F1-score |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Qwen3-VL | 57.4% | 25.5% | 17.0% | 0.659 | 0.574 | 0.614 |
| OpenAI | 74.5% | 12.8% | 12.8% | 0.833 | 0.745 | 0.787 |

本評価セットでは、OpenAIがPrecision、Recall、F1-scoreのすべてで高い結果となりました。

Qwen3-VLでは `MISS` より `PARTIAL` が多く、

> **対象や場面の大意は捉えているものの、数量・種類・属性・位置関係などの詳細が不足する**

ケースが比較的多く確認されました。

なお、この結果は5枚・47 Atomic Factsによる独自評価セットの結果であり、各モデルの一般的なベンチマーク値ではありません。

---

## 9. 保存データ

```text
results/
├── qwen_results.json
├── openai_results.json
├── judge_results_v2.json
├── judge_results_reviewed.json
├── final_evaluation.json
└── qualitative_000_results.json
```

| File | 内容 |
| --- | --- |
| `qwen_results.json` | Qwen3-VLの回答 |
| `openai_results.json` | OpenAIの回答 |
| `judge_results_v2.json` | Judge LLMによる一次評価 |
| `judge_results_reviewed.json` | 人間レビュー後の評価 |
| `final_evaluation.json` | Atomic Factと最終評価 |
| `qualitative_000_results.json` | 定性比較結果 |

元のJudge結果とレビュー後の結果を分離して保存し、評価過程を追跡できるようにしています。

---

## 10. ディレクトリ構成

```text
multimodal-llm-evaluation/
│
├── README.md
├── 05_Qwen3-VLとOpenAIによる画像理解比較.ipynb
├── ground_truth.json
│
├── images/
│   ├── 000.png
│   ├── 001.png
│   ├── 002.png
│   ├── 003.png
│   ├── 004.png
│   └── 005.png
│
└── results/
    ├── qwen_results.json
    ├── openai_results.json
    ├── judge_results_v2.json
    ├── judge_results_reviewed.json
    ├── final_evaluation.json
    └── qualitative_000_results.json
```

---

## 11. 実行環境

- Windows
- NVIDIA GeForce RTX 4070 12GB
- CUDA 12.6系
- Anaconda
- Jupyter Notebook
- PyTorch
- Transformers
- OpenAI Python SDK
- Pillow
- Matplotlib

公開用Notebookではモデル推論を再実行しないため、保存済みJSONと画像があればAPIコストを発生させず `Run All` が可能です。

---

## 12. 制約

本検証は大規模なベンチマークではありません。

また、Judge LLMにはOpenAI系モデルを使用しているため、同一ベンダーのモデル評価へ影響する可能性があります。

そのため、

- Judge結果をそのまま最終評価にしない
- モデル回答を固定する
- Atomic Fact単位で評価する
- 人間レビューを行う
- Judge結果とレビュー結果を別々に保存する

という手順を採用しています。

---

## 13. まとめ

本プロジェクトでは、

```text
画像推論
   ↓
JSON保存
   ↓
LLM-as-a-Judge
   ↓
人間レビュー
   ↓
Atomic Fact評価
   ↓
Precision / Recall / F1
```

という流れでQwen3-VLとOpenAIを比較しました。

本評価セットではOpenAIのStrict F1-scoreが高い結果となりました。

一方でQwen3-VLも多くのケースで画像の大意を捉えており、完全な見落としよりも部分一致が多く確認されました。

また、モデル性能だけでなく、**LLM-as-a-Judge自体にも評価の揺らぎが存在すること、そのため評価単位と人間レビューが重要であること**も確認できました。
