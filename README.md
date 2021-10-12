# Gaussian_lambda
elmo式学習のlambda設定のGaussian化による学習の最適化

# elmo式学習のlambda設定のGaussian化とは何か？
コンピューター将棋の強化学習ではelmo式学習の登場で、当時最強を誇っていたponanzaに引導を渡しました。  
その後NNUEの学習でも採用され、コンピューターチェスのstockfish NNUEでもelmo式学習が使用されています。 

elmo式学習の設定に存在するlambdaは、0から1の間の数値を設定可能で、0で教師の勝敗結果のみから学習し(Q-learning)、1で浅い探索の評価値を勝率変換したものから教師の深い探索の評価値を勝率変換したものを引いたものだけを学習します。(Rootstrap)

詰み寸前の局面では勝敗結果だけあれば十分で、僅か十数手先の探索結果など重視した場合、長手数の頓死の危険性もあるかもしれません。  
逆に平手開始局面では、探索不可能な百数十手先の未来の勝敗結果はあまり影響を大きくすべきでは無いのではと疑問に思います。  
例えば、教師データ生成時に序盤に正解の手を指していたとしても、中終盤などで読み抜けなどがあって負けた場合、序盤の正解の指し手が全て負けとして記録されてしまいます。  
位置評価よりも十数手程度の探索性能が原因で指し手が固定化されて戦型が偏ってしまう懸念があります。(実際に巷の評価関数は既に居飛車の特定の戦形に偏ってしまっているようですが、勝敗結果が少なからず影響している気がします。)  

コンピューター将棋ソフトのYaneuraOuではlambda・lambda2の2段階設定が可能ですが、序盤と終盤はある程度狙った学習ができるのかもしれませんが、中盤は上手く学習しずらいのではないか、  
さらにlambda2使用時のlambda_limit境界でlambdaが大きく変化するのは、学習方法の大きな変化による評価の間違いや読み抜け等が起きる事もあるのではないかと思い、
評価値の変化によってlambdaを滑らかに変動させるためにガウス関数で書き換え、  Gaussian_lambdaと名付けて実装しました。  
(実装は2019年の６月頃に終えていたのですが、その後は学習の実験をずっと続けていました。)  
ガウス関数と言えばベイズ最適化でも使用されていますが、Gaussian_lambdaはlambdaにガウス関数を掛け合わせただけのシンプルなものです。

# Gaussian_lambda = lambda * exp(-x^2/2σ^2)
xに教師の深い探索の評価値を使用し、σには定数値をいれる(σ＝標準偏差、σ^2＝分散)。  
評価値0で最大値(lambda設定値)になり、評価値が大きくなるにつれてlambdaが0に近づいていく。  
分散を大きくすると、落ち幅が小さいなだらかな曲線になる。σ(標準偏差)の値が最も強さに影響します。  
序盤と最終盤の設定はそれ程難しくは無いが、中盤の指し手がガウス曲線lambdaに影響される為、標準偏差σの決定が重要になります。  

![Gaussian_lambda(lambda0.5_σ600・800・1000)](https://raw.githubusercontent.com/Bonta0729/Gaussian_lambda/master/Gaussian_lambda(lambda0.5_%CF%83600%E3%83%BB800%E3%83%BB1000).png)

σ＝600～1200位まで色々な数値をlambdaとの組み合わせ50パターン程学習を試しました。  
一番最初にσにponanza定数に使われている600を使ってみたが、lambdaの下げ幅が大きすぎるのか全然強くなりませんでした。  
色々な数値を試した結果、σ＝900～1000辺りが良いという結論に達しました。  
σ＝900の時はまあまあmove accuracyが良かったが、test_cross_entropyがやや悪くなる。  
σ＝1000の方がmove accuracyが悪かったが、test_cross_entropyが良くなる。(move accuracy・test_cross_entropyは、YaneuraOuの学習logで使用されている数値です。)  
特にσ＝1000付近が最も強そうでしたが、膨大なパターンが試せず断定は出来ませんでした。

評価値無限大、即ち勝率100％の時の局面でlambdaは0が理想であるはずなので、lambdaと勝率の間には明確な関係があると仮定してσの値を決める事にしました。  
統計学でのガウス関数のグラフでは、横軸に標準偏差σが使用されます。  
Gaussian_lambdaをグラフにした場合、グラフの横軸が評価値xになり、縦軸がlambdaになります。  
正規分布近似の信頼区間1σの確率は約0.6827ですが、x＝0の時の確率が0.5からスタートする場合、信頼区間1σの確率は、0.6827/2 +0.5 ≒0.841 になります。  

現在多くの将棋プログラムが、勝率＝1/(1+exp(-評価値/600))にしており、評価値=-600*log((1/勝率)-1) である事から、  
勝率84.1％の評価値は約999.4になるので分かりやすいように切り上げてσ＝1000に決定しました。  
(今後、この評価値算出のシグモイド関数の600という定数が仮に300に変更になった場合は,
σ＝500に変更する必要があります。)

![Gaussian_lambda(σ1000_lambda0.6・0.5・0.4)](https://raw.githubusercontent.com/Bonta0729/Gaussian_lambda/master/Gaussian_lambda(%CF%831000_lambda0.6%E3%83%BB0.5%E3%83%BB0.4).png)

σ＝1000が正解の値なのかはわかりませんが、勝率連動型可変lambdaとして考えれば悪くない数値のような気がします。もしかしたらもっと大きな値が正解の可能性も無いとは言えないですが、もう実験するのがきつくてモチベーションが続かないので、誰か追試して正解の値を見つけてくれる事を期待しています。  

こんな事するより進行度をlambdaに利用すれば？とか言われてしまいそうですが、コードの改造がごく僅かで簡単だったのでこの手法を実装しました。進行度を利用したものを誰か作ってくれることも期待しています。(tttakさんが技巧2をYaneuraOuの教師データで学習する際に進行度をlambdaに利用していましたが。)  
今回の場合は、YaneuraOuのlearner.cppのlambdaが入った更新式のlambdaをGaussian_lambdaに置き換えるだけでOKです。(改変後のsourceはlearner.cppの1121行目～1129行目と1149行目～1153行目)  
この学習法はNNUEに限らず、elmo式であればKPPTの学習等にも使用可能なので汎用性が高いと思います。  

# 学習持のlambda設定
元々のノーマルなlambdaは、数値を大きくするとmove accuracyが良くなりtest_cross_entropyが悪くなる傾向があります。  
逆にlambdaを小さくすると、move accuracyが悪くなりtest_cross_entropyが良くなる傾向があります。  
Gaussian_lambdaはこれらの中間のバランスの取れた学習結果になります。  
Gaussian_lambdaではlambda設定を0.5よりも少しだけ大きめの値にすることをお勧めします。depth10の教師で0.50～0.52辺りが良さそうな感じでした。  
教師データでもっと深い探索をさせている場合は、評価値の信頼性が高まるのでlambdaをもう少しだけ上げても良いかもしれません。  
教師データが大量にに用意できない場合はlambdaを大きめにした方が良いと思います。教師データが少ないと勝敗結果の影響力が大きくなりすぎ、学習を難しくします。  
教師データを十分大量に用意可能ならば、勝敗結果を使用するデメリットを薄めることが可能なので
、lambdaを小さくするのも有りなのかなと思います。どれだけ大量に必要なのかは分かりませんが。

終盤は自動的にGaussian_lambdaが小さくなるのであまり気にする必要はありません。いかに序盤の学習をlambda設定でコントロールするかが焦点になります。  
ただし、lambdaをあまり大きくしすぎると終盤力に影響し、頓死の確率が上がるので程々の値にするか、lambda2・lambda_limitが併用可能なので序盤だけ大きくするという方法もあるかもしれません。  

lambda2・lambda_limitを併用する場合、中終盤に大きな変化をさせるのは危険なので、もし使用するならば序盤のせいぜい250点程度がlambda_limitの限界ではないかと考えていますが、
実際はどうなのかは分かりません。
