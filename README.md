# Multimodal Model Comparison
## Qwen3-VL と OpenAI による画像理解性能の比較

ローカル環境で動作する **Qwen3-VL** と、クラウドAPIで利用する **OpenAIのマルチモーダルモデル** に対して、同一画像・同一プロンプトを入力し、画像理解の違いを比較した検証プロジェクトです。

単純に回答を見比べるだけではなく、以下の評価プロセスを構築しました。

- Ground Truthの作成
- LLM-as-a-Judgeによる一次評価
- 評価結果のレビューと正規化
- Atomic Fact単位での最終評価
- Precision / Recall / F1-scoreによる定量比較
- Ground Truthを設定しない定性比較

---

## 1. 目的

本検証の目的は、各モデルを個別に最適化することではなく、

> **同一の画像・同一のプロンプトを使用し、可能な限り条件を揃えた状態で画像理解の違いを比較すること**

です。

そのため、モデルごとの追加チューニングやプロンプト最適化は原則として行わず、誤認・見落とし・部分的な認識も比較結果として扱いました。

---

## 2. 使用モデル

### 画像理解モデル

#### Qwen3-VL

- Model: `Qwen/Qwen3-VL-4B-Instruct`
- ローカル環境で実行
- Hugging Face Transformersを使用

#### OpenAI

- Model: `gpt-5.6-terra`
- OpenAI API経由で実行

### Judge LLM

- Model: `gpt-5.6-sol`

Judge LLMは画像そのものを見るのではなく、

- Ground Truth
- Qwen3-VLの回答
- OpenAIの回答

を比較し、どこが一致し、どこが部分一致・欠落・追加情報となっているかを一次評価するために使用しました。

---

## 3. 評価画像

定量評価には `001.png` ～ `005.png` の5枚を使用しました。

評価画像には、以下のような要素を含めています。

- 複数物体の認識
- 物体同士の位置関係
- 背景の理解
- OCR
- 図・イラスト全体の意味理解

また、`000.png` はStable Diffusionで生成した通常のイラストを使用し、Ground Truthを設定しない定性比較用画像としました。

---

## 4. 共通プロンプト

Qwen3-VLとOpenAIには同一のプロンプトを使用しています。

評価項目は以下の5項目です。

1. 主な対象
2. 対象同士の関係・配置
3. 背景・周囲
4. 画像内の文字
5. 画像全体の意味・種類

Ground Truthは画像理解モデルには与えていません。

---

## 5. 評価フロー

本検証では、以下の流れで評価を行いました。

```text
評価画像
   │
   ├── Qwen3-VL
   │
   └── OpenAI
          │
          ▼
    画像理解結果を固定
          │
          ▼
      Judge LLM
          │
          │ Ground Truthとの
          │ 一致・差異を一次評価
          ▼
   評価結果をレビュー
          │
          │ 粒度・重複・
          │ 判定基準を確認
          ▼
 Ground TruthをAtomic Fact化
          │
          ▼
 MATCH / PARTIAL / MISS
          │
          ▼
 TP / FP / FNへ変換
          │
          ▼
 Precision / Recall / F1-score
          │
          ▼
        最終比較
```

Judge LLMの段階ではF1-scoreは算出していません。

Judge LLMは、Ground Truthとモデル回答の一致・差異を分類するための一次評価を行います。

その結果を参考に評価単位をAtomic Factへ整理し、最終ラベルを固定した後に、Precision / Recall / F1-scoreを計算しています。

---

## 6. LLM-as-a-Judge

Judge LLMでは、モデル回答とGround Truthを比較し、以下のカテゴリへ分類しました。

### Ground Truth側

- `matched`
  - Ground Truthと十分一致している情報

- `partial`
  - 意味は近いが、数量・属性・種類・配置などに差がある情報

- `missed`
  - Ground Truthに存在するが、モデルが取得できていない情報

### モデルが追加した情報

- `unsupported`
  - Ground Truthと明確に矛盾する、または明確な誤認

- `unverified`
  - Ground Truthには存在しないが、Ground Truthだけでは正誤を判断できない追加情報

Ground Truthは画像内の情報を完全に列挙したものではないため、単にGround Truthに書かれていないという理由だけで誤答とはしません。

このため、`unsupported` と `unverified` を分離しました。

---

## 7. 評価結果のレビュー

LLM-as-a-Judgeは意味比較には有効でしたが、評価する事実の粒度に揺れが生じるケースがありました。

例えば、

- 同じ事実が `partial` と `missed` の両方に含まれる
- 1つの事実が複数の評価単位に分割される
- Ground Truthにない追加情報の扱いが一定しない

といったケースです。

そのため、Judge LLMの結果をそのままF1-scoreへ変換せず、Ground Truthと実際のモデル出力を確認し、評価基準が一定になるようレビューしました。

Notebookでは代表例のみを表示しています。

### 代表例

| Model / Image | Ground Truth | モデル出力 | 一致した点 | 差異 | 最終判定 |
| --- | --- | --- | --- | --- | --- |
| Qwen3-VL / 001 | UFO | 宇宙船 | 黄色い飛行物体を認識 | UFOを「宇宙船」と表現 | PARTIAL |
| Qwen3-VL / 002 | 犬が小さな船の上に立っている | 犬とイルカは認識したが、船を認識していない | 犬自体は認識 | 船、および「犬が船の上に立つ」という関係を取得できていない | MISS |
| Qwen3-VL / 005 | BUTTERFLY | BUTTER FLY | 単語自体は認識 | 単語内に空白がある | PARTIAL |
| Qwen3-VL / 005 | 物体検知結果を模した図 | 分類する図 | 対象識別という大意は一致 | 物体検知という具体的な種類まで特定できていない | PARTIAL |
| OpenAI / 004 | 白黒の線画 | 手描き風イラスト | イラストであることは認識 | 白黒・線画という特徴が不足 | PARTIAL |

全件のレビュー済みJudge結果は以下に保存しています。

```text
results/judge_results_reviewed.json
```

---

## 8. Atomic Fact

Judge LLMの文章単位の評価をそのまま件数として扱うと、1項目が含む情報量の違いによって評価値が変化します。

そこでGround Truthを、より小さな意味単位である **Atomic Fact** に分解しました。

例：

```text
黒猫が描かれている
黄色いUFOが描かれている
黒猫が黄色いUFOに乗っている
右上に別のUFOがある
海が描かれている
海に複数のヨットが浮かんでいる
```

5枚の評価画像から、合計 **47個のAtomic Fact** を定義しました。

各Atomic Factについて、

- `MATCH`
- `PARTIAL`
- `MISS`

のいずれか1つへ最終分類しています。

Atomic Factと最終評価ラベルは以下に保存しています。

```text
results/final_evaluation.json
```

---

## 9. Precision / Recall / F1-score

### Precision

モデルが正しく認識したと評価された情報のうち、実際に正しい情報の割合です。

```text
Precision = TP / (TP + FP)
```

### Recall

Ground Truthに存在する情報のうち、モデルが取得できた情報の割合です。

```text
Recall = TP / (TP + FN)
```

### F1-score

PrecisionとRecallの調和平均です。

```text
F1 = 2 × Precision × Recall / (Precision + Recall)
```

または、

```text
F1 = 2TP / (2TP + FP + FN)
```

と表せます。

PrecisionとRecallの片方だけが高い場合ではなく、両方のバランスを見るための指標です。

---

## 10. 本検証でのStrict評価

本検証では `PARTIAL` を完全一致とは扱わず、比較的厳しいStrict評価を採用しました。

```text
MATCH       → TP
PARTIAL     → FP + FN
MISS        → FN
unsupported → FP
unverified  → 評価対象外
```

例えばQwen3-VLでは、

```text
MATCH       = 27
PARTIAL     = 12
MISS        = 8
unsupported = 2
```

であるため、

```text
TP = 27
FP = 12 + 2 = 14
FN = 12 + 8 = 20
```

となります。

したがって、

```text
Precision = 27 / (27 + 14)
          ≒ 0.659

Recall    = 27 / (27 + 20)
          ≒ 0.574

F1-score  ≒ 0.614
```

となります。

---

## 11. 定量評価結果

| Model | MATCH | PARTIAL | MISS | Precision | Recall | F1-score |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Qwen3-VL | 57.4% | 25.5% | 17.0% | 0.659 | 0.574 | 0.614 |
| OpenAI | 74.5% | 12.8% | 12.8% | 0.833 | 0.745 | 0.787 |

本評価セットでは、OpenAIがPrecision、Recall、F1-scoreのすべてで高い結果となりました。

一方、Qwen3-VLは、

```text
PARTIAL : 25.5%
MISS    : 17.0%
```

となっており、完全に認識できないケースより、

> 対象や場面の大意は認識しているが、数量・種類・属性・配置などの詳細が不足する

ケースが比較的多く見られました。

---

## 12. Ground Truthなしの定性比較

`000.png` ではGround Truthを設定せず、Stable Diffusionで生成した通常のイラストを両モデルへ入力しました。

両モデルとも、

- 黒髪の少女
- 制服
- 雨
- 電柱
- 文字なし
- イラスト

といった主要要素を認識しました。

一方で、Qwen3-VLは、

> 「雨の中の学校制服を着た少女の静かな瞬間」

と、画像の雰囲気や情緒を含めた解釈を行いました。

OpenAIは、

> 「雨の日に屋外で座る学生を描いたアニメ風イラスト」

と、場面と画像種類を比較的明確に分類しました。

この画像はF1-scoreには含めず、定量評価では確認しにくいモデルごとの説明傾向を見るために使用しています。

結果は以下に保存しています。

```text
results/qualitative_000_results.json
```

---

## 13. JSONによる評価結果の固定

本プロジェクトでは、公開用Notebookの `Run All` ごとにAPIやローカルモデルを再実行しません。

一度取得した結果をJSONとして固定し、保存済み結果を読み込んで評価・可視化します。

```text
results/
├── qwen_results.json
├── openai_results.json
├── judge_results_v2.json
├── judge_results_reviewed.json
├── final_evaluation.json
└── qualitative_000_results.json
```

### 各ファイルの役割

#### `qwen_results.json`

Qwen3-VLによる画像認識結果。

#### `openai_results.json`

OpenAIによる画像認識結果。

#### `judge_results_v2.json`

Judge LLMによる一次評価結果。

#### `judge_results_reviewed.json`

Judge結果について、評価粒度や重複などを確認・補正した結果。

#### `final_evaluation.json`

- 47 Atomic Facts
- MATCH / PARTIAL / MISS
- additional informationの件数
- Precision / Recall / F1-score
- 最終評価結果

#### `qualitative_000_results.json`

000.pngを用いたGround Truthなしの定性比較結果。

元のJudge結果を上書きせず別ファイルとして保存することで、評価プロセスの追跡可能性を維持しています。

---

## 14. ディレクトリ構成

```text
multimodal_model_compare/
│
├── multimodal_model_comparison.ipynb
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

## 15. 実行環境

### ハードウェア・OS

- Windows
- NVIDIA GeForce RTX 4070 12GB
- CUDA 12.6系

### Python環境

- Anaconda
- conda environment: `model_compare`
- Jupyter Notebook

### 主なライブラリ

- PyTorch `2.14.0+cu126`
- Transformers `4.57.6`
- Pillow
- OpenAI Python SDK
- Pydantic
- Matplotlib

---

## 16. 公開用Notebookの実行

公開用Notebookでは以下を再実行しません。

- Qwen3-VLによる画像推論
- OpenAI APIによる画像推論
- Judge LLM API

保存済みJSONを読み込み、

- 画像
- モデル回答
- 人間レビューの代表例
- Atomic Fact評価
- Precision / Recall / F1-score
- グラフ
- 定性比較

を再構成します。

そのため、画像と保存済みJSONが存在すれば、APIコストを発生させず `Run All` が可能です。

---

## 17. 本検証の制約

本検証は大規模なベンチマークではありません。

定量評価画像は5枚、Atomic Factは47件であるため、今回算出したF1-scoreを各モデルの一般的な性能値として解釈することはできません。

あくまで、本評価セットにおける比較結果です。

また、Judge LLMにはOpenAI系モデルを使用しているため、同一ベンダーのモデルを評価する際にバイアスが生じる可能性があります。

そのため、

- Judge結果をそのまま最終評価にしない
- Ground Truthとモデル出力を再確認する
- Atomic Fact単位へ正規化する
- 生のJudge結果を保存する
- レビュー済み結果を別ファイルへ保存する

という手順を採用しました。

また、Qwen3-VLはローカルGPU、OpenAIはクラウドAPIで実行しているため、処理時間については直接的な性能比較には使用していません。

---

## 18. まとめ

本プロジェクトでは、Qwen3-VLとOpenAIの画像理解能力を同一条件で比較しました。

単純なモデル回答の比較ではなく、

```text
画像認識
    ↓
LLM-as-a-Judge
    ↓
評価結果のレビュー
    ↓
Atomic Factへの正規化
    ↓
Precision / Recall / F1-score
    ↓
Ground Truthなしの定性比較
```

という評価パイプラインを構築しています。

本評価セットではOpenAIのStrict F1-scoreが高い結果となりました。

一方、Qwen3-VLについても多くのケースで対象や場面の大意を捉えており、完全なMISSよりもPARTIALとなるケースが比較的多く確認されました。

また、LLM-as-a-Judge自体にも評価粒度の揺れが存在したことから、自動評価だけに依存せず、評価単位を明確にした上でレビューすることの重要性も確認できました。

マルチモーダルモデルの評価では、単一のスコアだけではなく、定量評価と実際の回答内容を組み合わせて確認することが重要だと考えています。
