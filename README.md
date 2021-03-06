# Deep Learningによる分布推定

## 例えばこのような連続する事象の確率分布がある
<div align="center">
  <img width="100%" src="https://user-images.githubusercontent.com/4949982/36883632-15300886-1e1f-11e8-8f96-0875da438c83.png">
</div>

横軸を時系列、縦軸を例えば企業の株価上がり下がり幅などとした場合、何か大局的なトレンドど業界のトレンドと国などのトレンドが入り混じり、単純な正規分布やベータ分布などを仮定できるものではなくなります。  

このとき、系列から学習して未来や未知の分布を直接求めることができ、かつ、異常値の検知などもしやすくすることなどを示したいと思います　

## 各系列で十分サンプリングでき、かつ、連続する事象の確率分布に対して予想したい場合  

例えば、この分布が日付のような連続なものとして扱われる場合、ある日のデータがサンプルできなかったり、まだサンプルが済んでない未来に対して予想しようとした場合、そういうことは可能なのでしょうか。  

ベイズでも可能ですが、せっかく十分にサンプリングできているので、ディープラーニングを用いて、KL距離、mean squre error距離などの距離関数で損失を決定して、分布を仮定せず、直接、確率分布を予想可能であることを示したいと思います  

## 上記の分布を生成する関数  
正規分布２つから導かれる頻度を計算していって、最大となる点で割ることにより確率の密度を計算していってます  
SIZEで定義される最大の値（時間）に対して、少しずつi(時間)をずらすことで、分布の中心点もまた変わっていくように、設計しました
```python
import numpy as np

SIZE = 25000
for i in range(SIZE):
  sample1 = np.random.normal(loc=-2*np.sin(i/10), scale=1, size=10000)
  sample2 = np.random.normal(loc=5*np.cos(i/10), scale=2, size=30000)
```
微妙に距離がある２つの分布から構成されており、i(時系列)でlocのパラメータが変化し、支配的な正規分布と非支配的な分布が入りまじります    

## 時刻tから分布Dを予想するための距離関数

用いた損失関数は、Kullback-Leibler情報量 + 平均誤差の混合した損失関数です  

確率分布のような図が、問題なく各時間tにおいてDが定義できるので、何か特別な細工をせずとも、学習が可能です。  

コードで表すとこんな感じです  
```python
def custom_objective(y_true, y_pred):
  mse = K.mean(K.square(y_true-y_pred), axis=-1)
  y_true_clip = K.clip(y_true, K.epsilon(), 1)
  y_pred_clip = K.clip(y_pred, K.epsilon(), 1)
  kullback_leibler = K.sum(y_true_clip * K.log(y_true_clip / y_pred_clip), axis=-1)
  return mse + kullback_leibler / 1000.0
```
数式で表現すると以下のようになります  
<div align="center">
  <img width="400px" src="https://user-images.githubusercontent.com/4949982/36884095-ec1bd6a2-1e21-11e8-9e62-7cb9ec35c81a.png">
</div>
k:定数,0.001と今回は設定  
これは、Image to Image[1]の論文と、この発表[2]に参考にしました（有効性の検証は別途必要でしょう）　　  

## 問題設定1. 欠けた部分から、もとの分布を予想する  
ある日のデータが何らの原因で欠けてしまった場合、周りの傾向を学習することで、欠けてしまったログから予想を試みます  

<div align="center">
  <img width="100%" src="https://user-images.githubusercontent.com/4949982/36878980-d1de3932-1e04-11e8-8c16-f52644929c9c.png">
</div>

ところどころ、データが欠損しています。  

これに対してDeepLearningのモデルで欠けた分布の予想をしていきましょう  

<div align="center">
  <img width="100%" src="https://user-images.githubusercontent.com/4949982/36879007-f460eedc-1e04-11e8-839a-887280cba7c0.png">
</div>

このように周辺の値が非常に小さくなる分布となる非常に細かいところは欠けでしまいました（多分活性化関数の工夫の次第です）が、おおよそ再現できることがわかりました。  

ディープラーニングはサンプリング数が十分に多ければ、（正規分布、ベータ分布などの）分布を仮説せずとも、直接、求めることができることがわかりました。

### 操作方法
```console
$ python3 distgen.py > dump.txt # 幾つかの分布からランダムに値をサンプリング
$ python3 make-dataset.py --invert #　dump.txtを転地する
$ python3 make-dataset.py --np # numpyのアレイにする

$ python3 unlimited-dimention-spectre.py --train #学習
$ python3 unlimited-dimention-spectre.py --predict #抜けた穴の予想
```

## 問題設定2.　異常値検知を行う  
検定の話ですが、一般的に（95%などの）信頼区間に入るかどうかがよく使われる手法です。  

信頼区間は確率密度関数が判明している必要がありますが、複雑な分布を本質的にもつようなものに当て推量で分布を仮定することが、事前に知識を外挿していることと等価であり、やらなくて済むケースが今回のケースです。  

<div align="center">
  <img width="400px" src="https://user-images.githubusercontent.com/4949982/36883467-e29e14b8-1e1d-11e8-9f61-4cb9acc3f954.png">
</div>

<div align="center">
  <img width="400px" src="https://user-images.githubusercontent.com/4949982/36883133-6244b530-1e1b-11e8-9b89-5b3997570bfb.png">
</div>
この時、この分布が仮に確率分布と定義できるようなサンプリングをした場合、異常値の検出は、あるサンプルした点から95%の全体の面積を占める範囲外であるとき、と考えられれそうです。（これはディープラーニングの出力値が、離散値なので上から順に足し算、下から順に足し算でかんたんに求まります）

例えば、上記のような端っこの方に①のポイントにサンプルされたような場合、それが異常値かどうかは、②までの積分値（足し算）に、③の全体を割って、0.05以下になるとかと定義することが可能そうです  

（今回は十分に未来の確率密度までディープラーニングは学習できたので、一度作ったモデルでモデルの予想する分布から外れたところにあるものが、異常とみなせそうです）  

### 操作方法
```console
$ python3 distgen.py > dump.txt # 幾つかの分布からランダムに値をサンプリング
$ python3 make-dataset.py --invert #　dump.txtを転地する
$ python3 make-dataset.py --np # numpyのアレイにする

$ python3 unlimited-dimention-spectre.py --train #学習
$ python3 unlimited-dimention-spectre.py --future #未来の100系列を予想
```

## 少し細工した点
入力に対して、時系列のラベルは通常、一次元の数字なのですが、これを利用するとうまく収束しないです。（値が大きすぎるか小さすぎ、ネットワークが安定しない）  

Dirty Hackっぽいですが、数字を２進数にしてN次元にします  
```console
ex)
5 -> [0, 0, 1, 0, 1]
```
このようにすると、今回のケースでは、うまく収束します  

## コード
[https://github.com/GINK03/probability-density-distribution-prediction]

## まとめ
- 何か分布を仮定しない→知識の外挿なしなので、全く未知のこと、ドメイン知識がないことをやろうとするときなど便利です  
- その代わり完全な分布を各時系列などで構築している必要があり、これはどのようにデータを集めるかが鍵なきがします
- データがそんなにないよ、ここまでパワフルな予想が必要がないよってときには、prophetがおすすめです

そもそも、DeepLearningは無限の重ね合わせによりガウス過程を構築していることと等価[3]（これは感覚的に理解できます）ので、こういったタスクに実は向いているのではないでしょうか。テストデータからのハズレ具合による確信度の低さ（これはベイズの確信度の低さに該当するみたいです）みたいなのは今回利用していませんが、利用することはできそうです。つまり、もう一つのモデルの可能性として、ここまで大量の分布がサンプリングできないとき、testデータとの差を最小化するポイントを見つけることで、ガウス過程により導出したことと等価とみなせるモデルが構築可能でしょう。それでも、十分な期間とそれに該当するデータサンプル量が必要となりますが。  

## 参考文献
- [1] [Image-to-Image Translation with Conditional Adversarial Networks](https://arxiv.org/pdf/1611.07004.pdf)
- [2] [確率分布間の距離推定：機械学習分野における最新動向](https://www.jstage.jst.go.jp/article/jsiamt/23/3/23_KJ00008829126/_pdf)
- [3] [深層学習はガウス過程](http://machine-learning.hatenablog.com/entry/2018/01/13/142612)
