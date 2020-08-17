# Gaussian_lambda
elmo式学習のlambda設定のGaussian化による学習の最適化

# elmo式学習のlambda設定のGaussian化とは何か？
elmo式学習のlambdaは0で教師の勝敗結果のみから学習し(Q-learning)、1で教師の深い探索の評価値を勝率変換したものから浅い探索の評価値を勝率変換したものを引いたものだけを学習します。(Rootstrap)

詰み寸前の局面では勝敗結果だけあれば十分で、僅か十数手先の探索結果など重視した場合、長手数の頓死の危険性もあるかもしれません。
逆に平手開始局面では、探索不可能な百数十手先の未来の勝敗結果はあまり影響を大きくすべきでは無いと考えおり、指し手が固定化されて戦型が偏ってしまう懸念もあります。

YaneuraOuではlambda・lambda2の2段階設定が可能ですが、序盤と終盤はある程度狙った学習ができるのかもしれませんが、中盤は上手く学習しずらいのではないか、さらに
lambda2のlambda_limit境界でlambdaが大きく変化するのは、評価方法の大きな変化による評価の間違いや読み抜け等が起きる事もあるのではないかと思い、
評価値の変化によってlambdaを滑らかに変動させるためにガウス関数で書き換え、Gaussian_lambdaと名付けて実装しました。
(実装は2019年の６月頃に終えていたのですが、その後は学習の実験をずっと続けていました。)

# Gaussian_lambda = lambda * exp(-x^2/2σ^2)
xに教師の深い探索の評価値を使用し、σには定数値をいれる(σ＝標準偏差、σ^2＝分散)。
評価値0で最大値(lambda設定値)になり、評価値が大きくなるにつれてlambdaが0に近づいていく。

分散を大きくすると、落ち幅が小さいなだらかな曲線になる。σ(標準偏差)の値が最も強さに影響します。

![Gaussian_lambda(lambda0.5_σ600・800・1000)](https://raw.githubusercontent.com/Bonta0729/Gaussian_lambda/master/Gaussian_lambda(lambda0.5_%CF%83600%E3%83%BB800%E3%83%BB1000).png)

σ＝600～1200位まで色々な数値をlambdaとの組み合わせ50パターン程学習を試しました。

一番最初にσにponanza定数に使われている600を使ってみたが、lambdaの下げ幅が大きすぎるのか全然強くなりませんでした。

σ＝900の時はまあまあmove accuracyが良かったが、test_cross_entropyがやや悪くなる。
σ＝1000の方がmove accuracyが悪かったが、test_cross_entropyが良くなる。(move accuracy・test_cross_entropyは、YaneuraOuの学習logで使用されている数値です。)

色々な数値を試した結果、σ＝900～1000辺りが良いという結論に達しました。特にσ＝1000が強そうでしたが、膨大なパターンが試せず断定は出来ませんでした。

評価値無限大、即ち勝率100％の時の局面でlambdaは0が理想であるはずなので、lambdaと勝率の間には明確な関係があると仮定してσの値を決める事にしました。

正規分布近似のガウス関数のグラフでは、横軸に標準偏差σが使用されます。
Gaussian_lambdaをグラフにした場合、グラフの横軸が評価値xになり、縦軸がlambdaになります。

統計学における信頼区間1σの確率は約68.27％ですが、x＝0の時の確率が50％からスタートする場合、信頼区間1σの確率は約84.1％になります。

現在、多くの将棋プログラムが勝率＝1/(1+exp(-評価値/600))にしており、評価値=-600*log((1/勝率)-1) である事から、

勝率84.1％の評価値は約999.4になるので計算しやすいように切り上げてσ＝1000に決定しました。

![Gaussian_lambda(σ1000_lambda0.6・0.5・0.4)](https://raw.githubusercontent.com/Bonta0729/Gaussian_lambda/master/Gaussian_lambda(%CF%831000_lambda0.6%E3%83%BB0.5%E3%83%BB0.4).png)

序盤と最終盤の設定はそれ程難しくは無いが、中盤の指し手がガウス曲線lambdaに影響される為、標準偏差σの決定が重要になります。
σ＝1000が正解の値なのかは自分には証明不可能ですが、勝率連動型可変lambdaとして考えれば悪くない数値のような気がします。
誰か追試して正解の値を見つけてくれる事を期待しています。

この学習法はNNUEに限らず、elmo式であればKPPTの学習等にも使用可能なので汎用性が高いと思います。

YaneuraOuの場合、learner.cppのlambdaが入った更新式のlambdaをGaussian_lambdaに置き換えるだけでOKです。



# 学習持のlambda設定
元々のノーマルなlambdaは、数値を大きくするとmove accuracyが良くなりtest_cross_entropyが悪くなる傾向があります。
逆にlambdaを小さくすると、move accuracyが悪くなりtest_cross_entropyが良くなる傾向があります。

Gaussian_lambdaはこれらの中間のバランスの取れた学習結果になります。

Gaussian_lambdaではlambda設定を0.5よりも少しだけ大きめの値にすることをお勧めします。depth10の教師で0.50～0.52辺りが良さそうな感じでした。
教師データでもっと深い探索をさせている場合は、評価値の信頼性が高まるのでlambdaをもう少しだけ上げても良いかもしれません。

終盤は自動的にGaussian_lambdaが小さくなるのであまり気にする必要はありません。いかに序盤の学習をlambda設定でコントロールするかが焦点になります。
ただし、lambdaを大きくしすぎると終盤力に影響するので程々の値にするか、lambda2・lambda_limitが併用可能なので序盤だけ大きくするという方法もあるかもしれません。

lambda2・lambda_limitを併用する場合、中終盤に大きな変化をさせるのは危険なので、もし使用するならば序盤のせいぜい250点程度がlambda_limitの限界ではないかと考えていますが、
根拠は無いので実際はどうなのかは分かりません。
