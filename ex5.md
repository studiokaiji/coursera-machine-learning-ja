# プログラミング演習5: 正則化された線形回帰とバイアス対分散

機械学習

## はじめに

この演習では、正則化された線形回帰を実装し、これを使用してさまざまなバイアス・分散特性を持つモデルを学習します。
プログラミング演習を始める前に、ビデオ講義を見て、関連トピックのレビュー質問を完了することを強くお勧めします。

演習を開始するには、スターター・コードをダウンロードし、演習を行うディレクトリーにその内容を解凍する必要があります。
必要に応じて、この演習を開始する前にOctave/MATLABの`cd`コマンドを使用して、このディレクトリーに移動してください。

また、コースウェブサイトの「環境設定手順」にOctave/MATLABをインストールするための手順も記載されています。

## この演習に含まれるファイル

 - `ex5.m` - 演習の手順を示すOctave/MATLABスクリプト
 - `ex5data1.mat` - データセット
 - `submit.m` - 解答をサーバーに送信するスクリプト
 - `featureNormalize.m` - フィーチャーの正規化関数
 - `fmincg.m` - 最小化ルーチンの関数（`fminunc`と同様）
 - `plotFit.m` - 多項式近似をプロットする
 - `trainLinearReg.m` - コスト関数を使用して線形回帰をトレーニングする
 - [\*] `linearRegCostFunction.m` - 正則化された線形回帰コスト関数
 - [\*] `learningCurve.m` - 学習曲線を生成する
 - [\*] `polyFeatures.m` - データを多項式フィーチャー空間にマップする
 - [\*] `validationCurve.m` - クロス・バリデーション曲線を生成する
 
 \* はあなたが完了する必要があるものを示しています

演習では、スクリプト`ex5.m`を使用します。
これらのスクリプトは、問題に対するデータセットをセットアップし、あなたが実装する関数を呼び出します。
こららのスクリプトを変更する必要はありません。
この課題の指示に従って、他のファイルの関数を変更することだけが求められています。

### 助けを得る場所

このコースの演習では、数値計算に適した高度なプログラミング言語であるOctave（※1）またはMATLABを使用します。
OctaveまたはMATLABがインストールされていない場合は、コースWebサイトのEnvironment Setup Instructionsのインストール手順を参照してください。

Octave/MATLABコマンドラインでは、`help`の後に関数名を入力すると、組み込み関数のドキュメントが表示されます。
たとえば、`help plot`はプロットのヘルプ情報を表示します。
Octave関数の詳細なドキュメントは、[Octaveのドキュメントページ](http://www.gnu.org/software/octave/doc/interpreter/)にあります。
MATLABのドキュメントは、[MATLABのドキュメントページ](http://jp.mathworks.com/help/matlab/?refresh=true)にあります。

また、オンライン・ディスカッションを使用して、他の学生との演習について話し合うことを強く推奨します。
しかし、他人が書いたソースコードを見たり、他の人とソースコードを共有したりしないでください。

※1：Octaveは、MATLABの無料の代替ソフトウェアです。
プログラミング演習は、OctaveとMATLABのどちらでも使用できます。

## 1. 正則化された線形回帰

演習の前半では、貯水池の水位の変化を使ってダムから流出する水の量を予測するために、正則化された線形回帰を実装します。
後半では、デバッグ学習アルゴリズムのいくつかの診断を行い、バイアス対分散の影響を調べます。

提供されたスクリプト`ex5.m`は、この演習を段階的に手助けします。

### 1.1. データセットの可視化

水位の変化(<img src="https://latex.codecogs.com/gif.latex?x" title="x" />)、ダムからの水の流出量(<img src="https://latex.codecogs.com/gif.latex?y" title="y" />)についての歴史的記録を含むデータセットを可視化することから始めます。

このデータセットは3つのパートに分かれています。

 - モデルが学習するトレーニング・セット： `X`、`y`
 - 正則化パラメーターを決定するためのクロス・バリデーション・セット： `Xval`、`yval`
 - パフォーマンスを評価するためのテストセット。モデルがトレーニング中に参照しなかった「未参照」のサンプル： `Xtest`、`ytest`

`ex5.m`の次のステップでトレーニング・データをプロットします（図1）。
以降のパートで、線形回帰を実装し、それを使用してデータを直線にフィットさせ、学習曲線をプロットします。
それに続いて、多項式回帰を実装して、データにさらによくフィットするようにします。

![図1:データ](images/ex5/ex5-F1.png)

&nbsp;&ensp;&nbsp;&ensp; 図1: サンプルデータセット1

### 1.2. 正則化された線形回帰のコスト関数

正則化された線形回帰は、以下のコスト関数となることを思い出してください。

![式1](images/ex5/ex5-NF1.png)

ここで、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />は正則化の程度を制御する正則化パラメーターです（したがって、オーバーフィッティングを防ぐのに役立ちます）。
正則化項はコスト<img src="https://latex.codecogs.com/gif.latex?J" title="J" />全体にペナルティーを課します。
モデル・パラメーター<img src="https://latex.codecogs.com/gif.latex?\inline&space;\theta&space;_{j}" title="\theta _{j}" />の大きさが増加するにつれて、ペナルティーも増加します。
<img src="https://latex.codecogs.com/gif.latex?\inline&space;\theta&space;_{0}" title="\theta _{0}" />項は正則化しないでください（Octave/MATLABのインデックス付けが1から始まるので、Octave/MATLABでは<img src="https://latex.codecogs.com/gif.latex?\inline&space;\theta&space;_{0}" title="\theta _{0}" />項は`theta(1)`として表されます）。

これで、ファイル`linearRegCostFunction.m`のコードを完成させられるはずです。
あなたがすべきことは、正則化された線形回帰コスト関数を計算する関数を書くことです。
可能であれば、コードをベクトル化してループを作成しないようにしてください。
終了したら、`ex5.m`の次のパートは`[1; 1]`で初期化された`theta`を使用してコスト関数を実行します。
出力は`303.993`となるはずです。

*ここで解答を提出する必要があります。*

### 1.3. 正則化された線形回帰の勾配

これに対応して、正則化された線形回帰の<img src="https://latex.codecogs.com/gif.latex?\inline&space;\theta&space;_{j}" title="\theta _{j}" />に対するコストの偏微分は、以下のように定義されます。

![式2](images/ex5/ex5-NF2.png)

`linearRegCostFunction.m`に、勾配を計算するためのコードを追加し、変数`grad`でそれを返します。
終了したら、`ex5.m`の次のパートは`[1; 1]`で初期化された`theta`を使って勾配関数を実行します。
`[-15.30; 598.250]`の勾配が確認できるはずです。

*ここで解答を提出する必要があります。*

### 1.4. 線形回帰のフィッティング

コスト関数と勾配が正しく機能したら、`ex5.m`の次のパートは`trainLinearReg.m`のコードを実行して<img src="https://latex.codecogs.com/gif.latex?\inline&space;\theta" title="\theta" />の最適値を計算します。
このトレーニング関数は、コスト関数を最適化するために`fmincg`を使用します。

このパートでは、正則化パラメーター<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />をゼロに設定します。
現行の線形回帰の実装は2次元の<img src="https://latex.codecogs.com/gif.latex?\inline&space;\theta" title="\theta" />に適合しようとしているため、このような低次元の<img src="https://latex.codecogs.com/gif.latex?\inline&space;\theta" title="\theta" />では正則化はあまり有用ではありません。
演習の後半では、多項式回帰と正則化を使用します。

最後に、`ex5.m`スクリプトは、図2に類似した画像となる最良適合線をプロットするはずです。
最良適合線は、データが非線形パターンであるため、モデルがデータに適していないことを示しています。
学習アルゴリズムをデバッグするには、ここで示されているような最良適合線を可視化する方法がありますが、データとモデルを可視化することは必ずしも容易ではありません。
次のセクションでは、データを可視化することが容易でない場合でも、学習アルゴリズムをデバッグするのに役立つ学習曲線を生成する関数を実装します。

![図2:線形近似](images/ex5/ex5-F2.png)

&nbsp;&ensp;&nbsp;&ensp; 図2: 線形近似

## 2. バイアス-分散

機械学習における重要な概念はバイアス-分散のトレードオフです。
バイアスが大きいモデルはデータに対して十分に複雑ではなくアンダーフィットする傾向がありますが、分散が大きいモデルはトレーニング・データにオーバーフィットします。

この演習では、トレーニング誤差とテスト誤差を学習曲線にプロットし、バイアス・分散問題を診断します。

### 2.1. 学習曲線

では、学習アルゴリズムのデバッグに役立つ学習曲線を生成するコードを実装しましょう。
学習曲線は、トレーニング・セットのサイズの関数としてトレーニング・セットの誤差とクロス・バリデーション・セットの誤差をプロットすることを思い出してください。
あなたがすべきことは、`learningCurve.m`を実装し、トレーニング・セットの誤差とクロス・バリデーション・セットの誤差のベクトルを返すことです。

学習曲線をプロットするには、異なるトレーニング・セットのサイズに対して、トレーニング・セットの誤差とクロス・バリデーション・セットの誤差が必要です。
具体的には、トレーニング・セットのサイズが`i`の場合は、最初の<img src="https://latex.codecogs.com/gif.latex?i" title="i" />個のサンプル（つまり、`X(1:i,:)`と`y(1:i)`）を使います。

`trainLinearReg`関数を使用してパラメーター<img src="https://latex.codecogs.com/gif.latex?\inline&space;\theta" title="\theta" />を見つけることができます。
`lambda`は、パラメーターとして`learningCurve`関数に渡されることに注意してください。
パラメーター<img src="https://latex.codecogs.com/gif.latex?\inline&space;\theta" title="\theta" />を学習した後、トレーニング・セットの誤差とクロス・バリデーション・セットの誤差を計算する必要があります。
データセットのトレーニング誤差は次のように定義されることを思い出してください。

![式3](images/ex5/ex5-NF3.png)

特に、トレーニング誤差には正則化項は含まれないことに注意してください。
トレーニング誤差を計算する方法の1つは、既存のコスト関数を使用して、トレーニング誤差とクロス・バリデーション誤差を計算するときにのみ<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />を0に設定することです。
トレーニング・セット誤差を計算するときは、（トレーニング・セット全体ではなく）トレーニング・サブセット（つまり、`X(1:n,:)`と`y(1:n)`）で必ず計算してください。
ただし、クロス・バリデーション誤差の場合は、クロス・バリデーション・セット全体で計算する必要があります。
計算された誤差をベクトル`error_train`と`error_val`に格納する必要があります。
終了したら、`ex5.m`は学習曲線をプリントし、図3と同様のプロットを作成します。

*ここで解答を提出する必要があります。*

図3では、トレーニング・サンプルが増えると、トレーニング誤差とクロス・バリデーション誤差の両方が高いことが分かります。
これは、モデルにおける高バイアス問題を反映しています。

![図3:線形回帰学習曲線](images/ex5/ex5-F3.png)

&nbsp;&ensp;&nbsp;&ensp; 図3: 線形回帰学習曲線1

線形回帰モデルは、あまりにも単純で、我々のデータセットをうまく適合させることができません。
次のセクションでは、このデータセットに対してより良いモデルに合うように多項式回帰を実装します。

## 3　多項式回帰

我々の線形モデルの問題は、それがデータにとってはあまりにも単純であり、アンダーフィッティング（高バイアス）であるということでした。
演習のこのパートでは、より多くのフィーチャーを追加することでこの問題に対処します。
多項式回帰を使用するための仮説は以下のようになります。

![式4](images/ex5/ex5-NF4.png)

<img src="https://latex.codecogs.com/gif.latex?x_{1}&space;=&space;(waterLevel),&space;x_{2}&space;=&space;(waterLevel)^{2}&space;,...,&space;x_{p}&space;=&space;(waterLevel)^{p}" title="x_{1} = (waterLevel), x_{2} = (waterLevel)^{2} ,..., x_{p} = (waterLevel)^{p}" />を定義することに注意してください。
フィーチャーが元の値<img src="https://latex.codecogs.com/gif.latex?(waterLevel)" title="(waterLevel)" />のさまざまな累乗である線形回帰モデルが得られます。

ここで、データセット内の既存のフィーチャー<img src="https://latex.codecogs.com/gif.latex?x" title="x" />の高次の累乗を使って、より多くのフィーチャーを追加します。
このパートであなたがすべきことは、関数が<img src="https://latex.codecogs.com/gif.latex?m&space;\times&space;1" title="m \times 1" />の元のトレーニング・セット`X`に、より高い乗数をマップするように、`polyFeatures.m`のコードを完成させることです。
具体的には、サイズ<img src="https://latex.codecogs.com/gif.latex?m&space;\times&space;1" title="m \times 1" />のトレーニング・セット`X`が関数に渡されたら、関数は<img src="https://latex.codecogs.com/gif.latex?m&space;\times&space;p" title="m \times p" />行列の`X_poly`を返すように実装する必要があります。
列1には`X`の元の値が格納され、列2には`X.^2`の値が格納され、列3には`X.^3`の値が格納されます。
このフィーチャーで0乗を考慮する必要はありません。

これで、フィーチャーを高次元にマップする関数ができました。`ex5.m`のパート6では、トレーニング・セット、テストセット、クロス・バリデーション・セット（まだ使用していない）にそれを適用します。

*ここで解答を提出する必要があります。*

### 3.1. 多項式回帰の学習

`polyFeatures.m`を完了すると、`ex5.m`スクリプトは線形回帰のコスト関数を使用して、多項式回帰をトレーニングします。

フィーチャー・ベクトルに多項式の項があるとしても、線形回帰の最適化の問題を解決しているということを覚えていてください。
多項式の項が、線形回帰に使用できるフィーチャーに変わっただけです。
この演習の前半で書いたのと同じコスト関数と勾配を使用しています。

演習のこのパートでは、8次の多項式を使用します。
予定されたデータに対して直接トレーニングを実行すると、フィーチャーがひどく縮小されてしまい、うまく機能しません
（例：<img src="https://latex.codecogs.com/gif.latex?x&space;=&space;40" title="x = 40" />では、フィーチャー<img src="https://latex.codecogs.com/gif.latex?x_{8}&space;=&space;40^{8}&space;=&space;6.5&space;\times&space;10^{12}" title="x_{8} = 40^{8} = 6.5 \times 10^{12}" />になります）。
したがって、フィーチャーの正規化を使用する必要があります。

多項式回帰のパラメーター<img src="https://latex.codecogs.com/gif.latex?\inline&space;\theta" title="\theta" />を学習する前に、`ex5.m`は`featureNormalize`を呼び出し、トレーニング・セットのフィーチャーを正規化し、パラメーター`mu`、`sigma`を別々に記憶します。
この関数はすでに実装されており、最初の演習から呼び出された関数と同じです。
パラメーター<img src="https://latex.codecogs.com/gif.latex?\inline&space;\theta" title="\theta" />を学習すると、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;0" title="\lambda = 0" />の多項式回帰のために2つのプロット（図4,5）が生成されます。

図4から、多項式近似がデータ点に非常によく追従できることが分かります。
したがって、トレーニング・誤差が低くなります。
しかし、多項式近似は非常に複雑であり、極端に低下もします。
これは、多項式回帰モデルがトレーニング・データを上回っており、一般化できないという指標です。
非正則化（<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;0" title="\lambda = 0" />）モデルの問題をよりよく理解するために、学習曲線（図5）を見てください。トレーニング誤差が低く、クロス・バリデーション誤差が高い場合と同じ効果を示すことが分かります。
トレーニング誤差とクロス・バリデーション誤差の間にはギャップがあり、高分散の問題であることを示しています。

![図4:多項式近似、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;0" title="\lambda = 0" />](images/ex5/ex5-F4.png)

&nbsp;&ensp;&nbsp;&ensp; 図4: 多項式近似、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;0" title="\lambda = 0" />

![図5:多項式学習曲線、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;0" title="\lambda = 0" />](images/ex5/ex5-F5.png)

&nbsp;&ensp;&nbsp;&ensp; 図5: 多項式学習曲線、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;0" title="\lambda = 0" />

オーバーフィット（高分散）の問題に対処する1つの方法は、モデルに正則化を追加することです。
次のセクションでは、正則化がどのようにしてより良いモデルを導くか確認するために、さまざまな<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />パラメーターを試してみます。

### 3.2. オプション（非評価）演習：正則化パラメーターの調整

このセクションでは、正則化パラメーターが正則化された多項式回帰のバイアス・分散にどのように影響するかを観察します。
`ex5.m`の`lambda`パラメーターを変更して<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;1,&space;100" title="\lambda = 1, 100" />を試してみるべきです。
これらの値のそれぞれについて、スクリプトはデータと学習曲線の多項式近似を生成するはずです。

<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;1" title="\lambda = 1" />の場合、クロス・バリデーション誤差とトレーニング誤差の両方が比較的低い値に収束していることを示す、データ傾向（図6）と学習曲線（図7）に従った多項式近似を確認するはずです。
これは、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;1" title="\lambda = 1" />の正則化された多項式回帰モデルが高バイアスまたは高分散の問題を持たないことを示しています。
事実上、バイアスと分散との間の良好なトレードオフを達成します。

<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;100" title="\lambda = 100" />の場合、データによく従わない多項式近似（図8）が表示されます。
この場合、正則化があまりにも大きく、モデルはトレーニング・データに適合できません。

*このオプションの（非評価）演習は、解答を提出する必要はありません。*

![図6:多項式近似、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;1" title="\lambda = 1" />](images/ex5/ex5-F6.png)

&nbsp;&ensp;&nbsp;&ensp; 図6: 多項式近似、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;1" title="\lambda = 1" />

![図7:多項式学習曲線、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;1" title="\lambda = 1" />](images/ex5/ex5-F7.png)

&nbsp;&ensp;&nbsp;&ensp; 図7: 多項式学習曲線、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;1" title="\lambda = 1" />

![図8:多項式近似、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;100" title="\lambda = 100" />](images/ex5/ex5-F8.png)

&nbsp;&ensp;&nbsp;&ensp; 図8: 多項式近似、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;100" title="\lambda = 100" />

### 3.3. クロス・バリデーション・セットを使用して<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />を選択する

演習の前のパートからは、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />の値がトレーニング・セットとクロス・バリデーション・セットで正則化された多項式回帰の結果に、大きく影響する可能性があることが分かりました。
特に、正則化のないモデル（<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;0" title="\lambda = 0" />）は、トレーニング・セットによく適合しますが、一般化しません。
逆に、正則化があまりにも大きいモデル（<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;100" title="\lambda = 100" />）は、トレーニング・セットとテストセットにうまく適合しません。 
<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />の良好な選択（たとえば、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;1" title="\lambda = 1" />）は、データに良好な適合を提供することができます。

このセクションでは、パラメーター<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />を選択するための自動化された方法を実装します。
具体的には、クロス・バリデーション・セットを使用して、各<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />値がどれだけ良いかを評価します。
クロス・バリデーション・セットを使用して最良の<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />値を選択した後、テストセット上のモデルを評価して、モデルが実際の見えないデータに対してどれだけうまく機能するかを推定することができます。

あなたのすべきことは、`validationCurve.m`のコードを完成させることです。
具体的には、異なる値の<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />を使用して`trainLinearReg`関数を使用してモデルをトレーニングし、トレーニング誤差とクロス・バリデーション誤差を計算する必要があります。
次の範囲で<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />を試してください：<img src="https://latex.codecogs.com/gif.latex?\left&space;\{&space;0,&space;0.001,&space;0.003,&space;0.01,&space;0.03,&space;0.1,&space;0.3,&space;1,&space;3,&space;10\right&space;\}" title="\left \{ 0, 0.001, 0.003, 0.01, 0.03, 0.1, 0.3, 1, 3, 10\right \}" />

![図9:クロス・バリデーション・セットを使用してλを選択する](images/ex5/ex5-F9.png)

&nbsp;&ensp;&nbsp;&ensp; 図9: クロス・バリデーション・セットを使用して<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />を選択する

コードを完成すると、`ex5.m`の次のパートがあなたの関数を実行し、誤差対<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />のクロス・バリデーション曲線をプロットすることができます。
これにより、使用するパラメーター<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />を選択できます。
図9と同様のプロットが表示されます。
この図からは、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />の最良値は約3であることが分かります。
データセットをトレーニングセットとバリデーションセットに分割する際のランダム性のために、クロス・バリデーション誤差はトレーニング誤差より低い場合があります。

*ここで解答を提出する必要があります。*

### 3.4. オプション（非評価）演習：テストセット誤差の計算

演習の前半では、正則化パラメーター<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />のさまざまな値に対するクロス・バリデーション誤差を計算するコードを実装しました。
しかし、現実世界でのモデルのパフォーマンスをより正確に把握するためには、トレーニングのどの部分でも使用されなかったテストセットの「最終」モデルを評価することが重要です（つまり、 パラメーター<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />を学習したり、モデル・パラメーター<img src="https://latex.codecogs.com/gif.latex?\inline&space;\theta" title="\theta" />を学習することはできません）。

このオプションの（非評価）演習では、見つかった<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda" title="\lambda" />の最良の値を使ってテスト誤差を計算する必要があります。
クロス・バリデーションでは、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;3" title="\lambda = 3" />で`3.8599`のテスト誤差が得られました。

*このオプションの（非評価）演習は、解答を提出する必要はありません。*

### 3.5. オプション（非評価）演習：ランダムに選択されたサンプルによる学習曲線のプロット

実際には（特に小さなトレーニング・セットでは）、学習曲線をプロットしてアルゴリズムをデバッグする際に、ランダムに選択された複数のサンプルセットの平均を取って、トレーニング誤差とクロス・バリデーション誤差を判断すると便利です。

具体的には、サンプル<img src="https://latex.codecogs.com/gif.latex?i" title="i" />のトレーニング誤差とクロス・バリデーション誤差を特定するには、まずトレーニング・セットのサンプル<img src="https://latex.codecogs.com/gif.latex?i" title="i" />とクロス・バリデーション・セットのサンプル<img src="https://latex.codecogs.com/gif.latex?i" title="i" />をランダムに選択する必要があります。
ランダムに選択されたトレーニング・セットを使用してパラメーター<img src="https://latex.codecogs.com/gif.latex?\inline&space;\theta" title="\theta" />を学習し、ランダムに選択されたトレーニング・セットとクロス・バリデーション・セットでパラメーター<img src="https://latex.codecogs.com/gif.latex?\inline&space;\theta" title="\theta" />を評価します。
上記のステップを複数回（たとえば50回）繰り返す必要があり、平均化された誤差を使用して<img src="https://latex.codecogs.com/gif.latex?i" title="i" />のサンプルのトレーニング誤差とクロス・バリデーション誤差を決定すべきです。

このオプションの（非評価）演習では、学習曲線を計算するための上記の戦略を実装する必要があります。
参考までに、図10は、<img src="https://latex.codecogs.com/gif.latex?\inline&space;\lambda&space;=&space;0.01" title="\lambda = 0.01" />の多項式回帰のために得られた学習曲線を示す。
サンプルはランダムに選択されているため、数字が多少異なる場合があります。

*このオプションの（非評価）演習は、解答を提出する必要はありません。*

![図10:オプション（非評価）演習：無作為に選択されたサンプルによる学習曲線](images/ex5/ex5-F10.png)

&nbsp;&ensp;&nbsp;&ensp; 図10: オプション（非評価）演習：ランダムに選択されたサンプルによる学習曲線

## 提出と採点

この課題が完了したら、送信機能を使用して解答を我々のサーバーに送信してください。
以下は、この演習の各パートの得点の内訳です。

| パート | 提出するファイル | 点数　|
----|----|---- 
| 正則化された線形回帰のコスト関数 | `linearRegCostFunction.m` | 25 点 |
| 正則化された線形回帰の勾配 | `linearRegCostFunction.m` | 25 点 |
| 学習曲線 | `learningCurve.m` | 20 点 |
| 多項式フィーチャー・マッピング | `polyFeatures.m` | 10 点 |
| クロス・バリデーション曲線 | `validationCurve.m` | 20 点 |
| 合計点 |  | 100 点 |

解答を複数回提出することは許可されており、最高のスコアのみを考慮に入れます。
