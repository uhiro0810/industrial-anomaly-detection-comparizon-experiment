---
title: "GLAD・EfficientAD・PUADをColabで再現して5カテゴリ比較した話"
emoji: "🧪"
type: "tech"
topics: ["異常検知", "機械学習", "深層学習", "画像処理", "colab"]
published: true
---

# はじめに

産業異常検知の論文実装を比較しようとすると、実際には「モデルの中身」より先に、

- 公開実装がそのまま動かない
- Colab だと依存関係が壊れる
- 手法ごとに出力形式や評価指標が違う
- image-level と pixel-level を分けて見ないと比較にならない

といった問題にぶつかります。

今回は、代表的な3手法である

- **GLAD**
- **EfficientAD**
- **PUAD**

を対象に、**MVTec-AD の 5カテゴリ（bottle / cable / capsule / pill / grid）**で比較実験を行いました。

この検証では、単に公開コードを実行するだけでなく、

- Colab 上での環境構築
- 依存関係の修正
- checkpoint の配置
- 複数手法の評価指標統一
- 比較表と平均表の作成
- anomaly-wise AUROC の整理
- 可視化画像の保存

まで一通り行っています。

この記事では、**結果そのもの**だけでなく、  
**どこが大変だったか / 何が分かったか / どのような成果物になったか** までまとめます。

---

# 対象手法

## GLAD
GLAD は diffusion ベースの再構成と DINO 特徴を使って異常を捉える手法です。  
ざっくりいうと、**正常らしい再構成画像とのズレ**や**特徴差**から異常箇所を推定します。

今回比較してみて感じたのは、GLAD は **「画像内のどこが異常か」** という局所化がかなり強いということです。

## EfficientAD
EfficientAD は teacher / student / autoencoder を組み合わせた軽量な異常検知手法です。  
シンプルで高速な構造ながら高精度で、実運用目線でもかなり魅力があります。

今回の比較では、**画像全体として異常かどうかを判定する image-level 指標が特に強い**という結果でした。

## PUAD
PUAD は EfficientAD をベースに、picturable anomaly と unpicturable anomaly の両方を扱う発展的な手法です。  
今回使った公開実装では主に **image-level の性能評価** が中心になります。

ただし anomaly-wise AUROC を出せるので、**異常タイプごとの得意・不得意が見える**のが面白いです。

---

# なぜこの比較をやったか

異常検知は、単に AUROC が高ければよいわけではありません。  
実際には次の2種類の問いがあります。

- **この画像は異常か？**
- **異常だとして、どこが異常か？**

この2つを同時に見ないと、実運用で役立つかどうかを判断しづらいです。

そこで今回は、

- **Image_AUROC**
- **Pixel_AUROC**
- **Pixel_AP**
- **PRO / AUPRO**
- **PUAD の anomaly-wise AUROC**

を使って、手法ごとの特徴が見える形にしました。

---

# 実験設定

## データセット
- [MVTec AD Dataset](https://www.mvtec.com/company/research/datasets/mvtec-ad)

## 使用カテゴリ
- bottle
- cable
- capsule
- pill
- grid

物体系とテクスチャ系を混ぜて、手法の得意・不得意が見えやすい構成にしています。

---

# 実装面で大変だったこと

今回、一番時間がかかったのは「モデルそのもの」よりも、**再現実験を回せる状態にすること**でした。

具体的には次のような問題に対応しました。

## GLAD 側
- `numpy`, `scikit-image`, `imgaug` の不整合
- `bitsandbytes`, `kornia` など依存不足
- diffusers / transformers / accelerate のバージョン差
- checkpoint パスとコード側の期待パスのズレ
- Colab 再起動時の再開手順整理
- 公開実装内の typo / NameError 修正

## EfficientAD / PUAD 側
- CPU版 torch が入ってしまい CUDA が使えない問題
- pretrained model の配置パス整理
- dataset path の期待構造の確認
- image-level と pixel-level 出力形式の整理

このあたりは、いわゆる「研究コードを実務寄りに扱う」時にかなりよくある問題だと思っています。  
今回の作業は、むしろここに一番価値があった気がしています。

---

# 実験環境

## GLAD 側
- numpy: `1.26.4`
- torch: `2.10.0+cu128`
- diffusers: `0.29.2`
- transformers: `4.41.2`
- accelerate: `0.27.2`
- GPU: `Tesla T4`
- CUDA available: `True`

## EfficientAD / PUAD 側
- torch: `2.5.1+cu121`
- CUDA version: `12.1`
- GPU: `Tesla T4`
- CUDA available: `True`

## 実行環境
- Google Colab
- GPU runtime (T4)

---

# 結果

## 5カテゴリ平均

| Method | Image_AUROC | Pixel_AUROC | Pixel_AP | PRO / AUPRO |
|---|---:|---:|---:|---:|
| EfficientAD | 98.772 | 97.989 | 45.683 | 91.009 |
| GLAD | 98.654 | 98.516 | 66.926 | 96.070 |
| PUAD | 98.749 | - | - | - |

この表だけでもかなり面白いです。

- **Image_AUROC は3手法ともかなり高い**
- ただし平均では **EfficientAD と PUAD がわずかに上**
- 一方、**pixel-level 指標では GLAD が明確に上**

つまり、  
**「画像全体の異常判定に強い手法」と「異常位置の局所化に強い手法」が分かれる**  
という構図になっています。

---

## カテゴリ別の比較結果

| Method | Category | Image_AUROC | Pixel_AUROC | Pixel_AP | PRO_or_AUPRO |
|---|---|---:|---:|---:|---:|
| GLAD | bottle | 100.000 | 98.650 | 85.250 | 96.100 |
| EfficientAD | bottle | 100.000 | 97.162 | 55.720 | 88.990 |
| PUAD | bottle | 99.921 | - | - | - |
| GLAD | cable | 99.100 | 98.000 | 66.720 | 93.540 |
| EfficientAD | cable | 97.245 | 98.231 | 66.039 | 90.743 |
| PUAD | cable | 97.770 | - | - | - |
| GLAD | capsule | 99.600 | 98.640 | 53.700 | 94.980 |
| EfficientAD | capsule | 98.444 | 98.605 | 33.174 | 91.914 |
| PUAD | capsule | 98.045 | - | - | - |
| GLAD | pill | 94.570 | 97.770 | 75.200 | 97.560 |
| EfficientAD | pill | 98.172 | 97.752 | 48.868 | 89.810 |
| PUAD | pill | 98.009 | - | - | - |
| GLAD | grid | 100.000 | 99.520 | 53.760 | 98.170 |
| EfficientAD | grid | 100.000 | 98.195 | 24.614 | 93.589 |
| PUAD | grid | 100.000 | - | - | - |

---

# 考察

## 1. image-level は EfficientAD / PUAD が強い
平均の `Image_AUROC` では、EfficientAD と PUAD が GLAD をわずかに上回りました。  
これは、**画像全体として異常かどうかを判定する力**が軽量系手法で非常に高いことを示しています。

実運用でまず「NG画像を弾く」用途なら、EfficientAD 系はかなり魅力的です。

## 2. pixel-level は GLAD が強い
一方で、`Pixel_AP` と `PRO / AUPRO` は GLAD が大きく上回りました。  
これは、GLAD が **異常領域をより自然に、より安定して拾えている**ことを意味します。

「異常の位置も見たい」「根拠のある可視化が欲しい」用途では、GLAD の強みが出やすいです。

## 3. pill がかなり面白い
`pill` では、GLAD の `Image_AUROC` は 94.57 と他カテゴリよりかなり低いです。  
一方で pixel-level 指標は強いままです。

この結果から、GLAD は

- 局所異常自体は拾えている
- ただし画像全体スコアへの集約で不利になる場合がある

と考えられます。  
つまり、GLAD 側は score aggregation を工夫するとさらに伸びる余地がありそうです。

## 4. PUAD は image-level 補強として見るのが自然
今回の公開実装では、PUAD は pixel-level map を直接比較していません。  
そのため、今回の比較では **image-level 強化手法** として扱いました。

一方で anomaly-wise AUROC はかなり有用で、  
「全体では強いが、特定の異常タイプだけ難しい」という分析ができます。

---

# 可視化例

## image-level の比較グラフ
![](/images/figures/image_auroc_by_category.png)

## 可視化比較について

今回の可視化は、各手法の公開実装が標準で出力する形式に合わせています。  
そのため、**GLAD と EfficientAD で完全に同一の表示形式にはしていません**。

- **GLAD**: 入力画像・再構成画像・異常ヒートマップ・予測異常領域
- **EfficientAD**: 入力画像・重畳ヒートマップ・anomaly map

つまり今回の可視化比較で見たいのは、  
**「同じ異常画像に対して、それぞれの手法がどこを異常と判断しているか」**  
という点です。

出力フォーマット自体は異なりますが、同一カテゴリ・同一異常タイプ・同一入力画像で揃えているため、  
異常領域の反応傾向を比較する材料としては十分に意味があると考えています。

## GLAD の可視化例

### bottle
**GLAD visualization example**  
左から、入力画像、再構成画像、異常ヒートマップ、予測異常領域。

![](/images/GLAD/bottle/glad_bottle_1.png)
![](/images/GLAD/bottle/glad_bottle_2.png)

### cable
**GLAD visualization example**  
左から、入力画像、再構成画像、異常ヒートマップ、予測異常領域。

![](/images/GLAD/cable/glad_cable_1.png)
![](/images/GLAD/cable/glad_cable_2.png)

### capsule
**GLAD visualization example**  
左から、入力画像、再構成画像、異常ヒートマップ、予測異常領域。

![](/images/GLAD/capsule/glad_capsule_1.png)
![](/images/GLAD/capsule/glad_capsule_2.png)

### grid
**GLAD visualization example**  
左から、入力画像、再構成画像、異常ヒートマップ、予測異常領域。

![](/images/GLAD/grid/glad_grid_1.png)
![](/images/GLAD/grid/glad_grid_2.png)

### pill
**GLAD visualization example**  
左から、入力画像、再構成画像、異常ヒートマップ、予測異常領域。

![](/images/GLAD/pill/glad_pill_1.png)
![](/images/GLAD/pill/glad_pill_2.png)

## EfficientAD の可視化例

### bottle
**EfficientAD visualization example**  
左から、入力画像、重畳ヒートマップ、anomaly map。

![](/images/EfficientAD/bottle/000_broken_large_score_1.4279.png)
![](/images/EfficientAD/bottle/001_broken_large_score_2.6971.png)

### cable
**EfficientAD visualization example**  
左から、入力画像、重畳ヒートマップ、anomaly map。

![](/images/EfficientAD/cable/000_bent_wire_score_0.6386.png)
![](/images/EfficientAD/cable/001_bent_wire_score_1.3788.png)

### capsule
**EfficientAD visualization example**  
左から、入力画像、重畳ヒートマップ、anomaly map。

![](/images/EfficientAD/capsule/000_crack_score_0.4557.png)
![](/images/EfficientAD/capsule/001_crack_score_3.5638.png)

### grid
**EfficientAD visualization example**  
左から、入力画像、重畳ヒートマップ、anomaly map。

![](/images/EfficientAD/grid/000_bent_score_2.1464.png)
![](/images/EfficientAD/grid/001_bent_score_2.3808.png)

### pill
**EfficientAD visualization example**  
左から、入力画像、重畳ヒートマップ、anomaly map。

![](/images/EfficientAD/pill/000_color_score_0.2514.png)
![](/images/EfficientAD/pill/001_color_score_2.5432.png)

---

# リポジトリ構成

```text
comparison/
├── EfficientAD/
│   ├── bottle/
│   │   └── pixel_metrics.json
│   ├── cable/
│   │   └── pixel_metrics.json
│   ├── capsule/
│   │   └── pixel_metrics.json
│   ├── grid/
│   │   └── pixel_metrics.json
│   └── pill/
│   │   └── pixel_metrics.json
├── GLAD/
│   ├── bottle/
│   │   └── metrics.json
│   ├── cable/
│   │   └── metrics.json
│   ├── capsule/
│   │   └── metrics.json
│   ├── grid/
│   │   └── metrics.json
│   └── pill/
│   │   └── metrics.json
├── PUAD/
│   ├── bottle/
│   │   ├── anomaly_breakdown.json
│   │   └── metrics.json
│   ├── cable/
│   │   ├── anomaly_breakdown.json
│   │   └── metrics.json
│   ├── capsule/
│   ├── grid/
│   │   ├── anomaly_breakdown.json
│   │   └── metrics.json
│   └── pill/
│   │   ├── anomaly_breakdown.json
│   │   └── metrics.json
├── tables/
│   ├── glad_multi_metrics.csv
│   ├── puad_anomaly_breakdown_multi.csv
│   ├── puad_efficientad_multi_metrics.csv
│   ├── multi_category_comparison_raw.csv
│   ├── multi_category_comparison_final.csv
│   ├── multi_category_summary.csv
│   └── multi_category_pixel_compare.csv
├── figures/
│   └── image_auroc_by_category.png
scripts/
├── GLAD_eval_multi.ipynb
├── PUAD_eval_multi.ipynb
└── comparison_multi.ipynb
```

---

# 再現に必要なデータ・重み

容量の都合で、データセットと pretrained weights は GitHub には含めていません。  
以下から取得して配置する想定です。

## Dataset
- [MVTec AD Dataset](https://www.mvtec.com/company/research/datasets/mvtec-ad)
- [DTD Dataset](https://www.robots.ox.ac.uk/~vgg/data/dtd/)

## Pretrained Weights
- [GLAD pretrained weights](https://stuhiteducn-my.sharepoint.com/personal/23b903042_stu_hit_edu_cn/_layouts/15/onedrive.aspx?id=%2Fpersonal%2F23b903042%5Fstu%5Fhit%5Fedu%5Fcn%2FDocuments%2FGLAD&ga=1)
- [EfficientAD pretrained weights](https://drive.google.com/drive/folders/1-NDUVHFbLTI3CmL8FYSYFXpbLExRu_N5)

---

# この検証を通じて得たもの

今回の比較で一番大きかったのは、単に数値が分かったことではなく、

- image-level と pixel-level は分けて見るべき
- 軽量手法は image-level に強い
- diffusion 系は pixel-level に強い
- 公開実装比較では、コード読解と再現環境整備そのものが重要

という整理ができたことです。

実際、モデル本体よりも

- 実装差分の吸収
- Colab での安定実行
- 結果保存形式の統一
- 比較 notebook の整備

の方が時間がかかりました。  
ただ、この部分こそ実務の ML engineering に近いと思っています。

---

# おわりに

今回の比較を通して、異常検知は「どの手法が一番強いか」を1行で言い切るのが難しい分野だと改めて感じました。

- **image-level 判定なら EfficientAD / PUAD**
- **pixel-level 局所化なら GLAD**

という形で、用途に応じて評価軸を分ける必要があります。

また、公開実装の再現と比較をきちんとやること自体が、かなり実装力を問われる作業でした。  

もし同じように anomaly detection の比較をしたい人がいれば、  
このリポジトリや記事が出発点になればうれしいです。
