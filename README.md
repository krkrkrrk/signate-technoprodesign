# テクノプロ・デザイン社 食品パッケージ画像解析チャレンジ（一般部門・学生部門）の解法

本ページは2023年8月11日～9月29日に開催されたテクノプロ・デザイン社主催の食品パッケージ画像解析チャレンジにおいて、以下の成績を収めた解法についてまとめたページです。  
コンペページ：https://signate.jp/competitions/1106  
  
・精度のみのランキングで一般・学生を合わせて3位（301人中）  
・審査発表会を含めた総合ランキングで学生部門4位（97人中）  
  
## コンペ概要
本コンペは画像に写る商品が「飲料」か「食料」かを分類するコンペです。

<p align="center">
 <img src="https://github.com/krkrkrrk/signate-technoprodesign/assets/93073869/2890a0eb-f9bc-4b28-9e6b-5675645b0e27" width="800">
</p>

## データセット
訓練用データセット：2175枚  
評価用データセット：2180枚

各画像サイズは512×512で、データ拡張は適用済みです。  
データセットは規約に違反するため公開できません。

## 評価方法
AUC (Area Under the Curve)

## 審査発表会
学生部門においては、AUCランキング上位10名が審査発表会に進むことができます。最終的なランキングは審査発表会の内容を踏まえて決定されます。  
審査発表会で用いたスライドはこちら：https://drive.google.com/file/d/1DX5BDk5TKewiHqKUhjiqWy6Lt71cNQx2/view?usp=sharing

### 審査項目
・スコアの結果  
・基礎集計で得られた知見  
・実施した前処理とそれを行った理由  
・検討したアプローチ  
・一番工夫した点  
・資料のストーリー性  
・誤判定になった理由についての考察と、対処方法の提案  

## 解法
### 実行環境
・Google Colab  
・A100 GPUを使用

### 基本戦略
方針決定のために様々な検証を行ったところ、パラメーターや手法を変更するだけで精度が大幅に向上することに気づきました。
そのため、多くのパターンを試す方針を採りました。
  
### 基礎集計・前処理
#### ・食品の数が1182枚、飲料の数が994枚  
　少し偏りがありましたが、そこまで問題にはならないと考えました

#### ・人間でも判別の難しい画像あり  
　実際にTrainデータを１枚１枚判別可能かどうかを人力で調査したところ、判別可能な割合は約92%でした。そのため、機械による正解率も最大で約92%が限界だろうと推測しました。

#### ・データ拡張が既に適用済  
　左右反転、上下反転、回転といったシンプルなデータ拡張のみ追加

### ベースライン
様々なモデルを調査した結果、eva02が最も精度が良く、eva02_base_patch14_224をベースラインとしました。(CV = 交差検証)

<p align="center">
 <img src="https://github.com/krkrkrrk/signate-technoprodesign/assets/93073869/adc2bab1-e1c4-48b6-9b22-cd2bd3eaa902" width="800">
</p>

### ベースライン改善
#### ・eva02_large_patch14_448に変更  
　コンペ初期に実験したときはパラメーターの設定ミスでbaseよりも精度が低くなったため、不採用にしていました。しかし、再度baseと全く同じ設定で実験を行ったところ、CVが約3.5%、Public LBが約1%向上しました。

#### ・モデルのバージョン変更  
　timmモデルのimagenetでのスコアをまとめた表を発見しました。eva02_large_patch14_448のデフォルトバージョンが以下の表における2番目のものであることに気づき、1番上のものに変更したところ、CVが約2.1%、Public LBが約1%向上しました。

<p align="center">
  <img src="https://github.com/krkrkrrk/signate-technoprodesign/assets/93073869/2df2ca1a-d96a-4e60-b575-0f5a4dc94626" width="700">
</p>

#### ・Warmupの導入
　Warmupの導入により、CV、Public LBが向上したのですが、実際にはPrivate LBが少し低下しており、精度向上には繋がっていないことが分かりました。しかし、コンペ後の調査でepochとWarmupのステップ数を調整すると、CVが0.977から0.988まで向上することが分かりました。Private LBの値を見ないと確実なことは言えませんが、Warmupは精度向上に繋がるアプローチであった可能性が高いと考えています。

<p align="center">
  <img src="https://github.com/krkrkrrk/signate-technoprodesign/assets/93073869/223fd91a-32d8-45bc-ab05-90f467ec5a3e" width="800">
</p>

#### ・交差検証とアンサンブルの組み合わせ  
　乱数シードのみを変えたモデルを５つ用意して、それぞれにおいて交差検証を行い、その結果を平均しました。これにより、乱数による性能のぶれを小さくすることができ、より安定したアルゴリズムにすることができたと考えています。

 <p align="center">
  <img src="https://github.com/krkrkrrk/signate-technoprodesign/assets/93073869/76c0322c-4aa4-4371-90eb-7cb29076659e" width="800">
 </p>

### 精度向上に寄与しなかったアプローチ
#### ・オプティマイザーの変更
基本戦略で述べた「手法を変えるだけで精度が大幅に向上する」という気づきから、Adadelta、dagrad、Adam、Lion、momentum_SGD、Radam、Ranger21、RMSprop、SAM、SGDといったオプティマイザーを試しましたが、結局デフォルトのAdamWが最も安定しており精度が高いことが分かりました。

#### ・ハイパーパラメーターチューニング
Optunaを用いてハイパーパラメーターチューニングを行いました。CV、Public LBは上がりましたが、Private LBを見るとあまり効果はありませんでした。また、手法を変えると再度チューニングする必要があり、コストパフォーマンスが良くありません。これらのことから、最終的にはチューニングは行いませんでした。

### 最終的なパラメーター・手法
'model_name': 'eva02_large_patch14_448.mim_m38m_ft_in22k_in1k',
'n_fold': 5  
'epochs': 4  
'criterion': 'CrossEntropy'  
'image_size': (448, 448)  
'train_batch_size': 16  
'test_batch_size': 32  
'seed': 777  
'optimizer': 'AdamW'  
'learning_rate': 1.5e-05  
'scheduler': 'warmup'  
'warmup_step': '108'  
'min_lr': 1e-06  
'betas': (0.9, 0.999)  

## まとめ
今回が初コンペ&授業以外で機械学習を扱うのも初だったので、分からないことだらけで大変でした。

結果的には単純な解法になったのですが、設定ミスなどがあり、今回の解法に到達するのにかなり時間がかかってしまいました。

機械学習について幅広く知ることができたので、今後の研究や開発に生かしていきたいです。
