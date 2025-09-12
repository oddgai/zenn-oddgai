---
title: "Claude Codeと数理最適化をやってみる"
emoji: "🚚"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics:
  - "数理最適化"
  - "claudecode"
  - "データ分析"
  - "zennfes2025ai"
published: true # 公開設定（falseにすると下書き）
publication_name: "genda_jp"
---

こんにちは、ペルソナ4 リバイバルを楽しみにしているデータサイエンティストのoddgaiです。
先日、Claude CodeでKaggleをやってみた記事を書いたのですが、数理最適化もできるの？と思ったのでやってみました。

https://zenn.dev/genda_jp/articles/20250909_kaggle_with_claude_code

## 結論

- 単純な問題ならざっくり指示しても割とちゃんと解いてくれる
  - [OR-tools](https://developers.google.com/optimization?hl=ja), [PuLP](https://coin-or.github.io/pulp/)などのライブラリも使える
  - 他分野よりネットに情報が落ちてない印象があったので心配してたけど意外と大丈夫だった
  - 数秒で数百行のコードを書いてくるので人間による確認＆精度担保が大変
    - 今回はテストデータなので甘めにやってしまった・・・
  - C++やらRustやらでヒューリスティックをゴリゴリ書いてもらうとかは未調査だが、こちらも強そう
    - 参考：[AI vs 人間まとめ【AtCoder World Tour Finals 2025 Heuristic エキシビジョン】 - chokudaiのブログ](https://chokudai.hatenablog.com/entry/2025/07/21/190935)
- AIらしいミスはするのでちゃんと確認は必要
  - モデラーはPython-MIPを使ってと指示したのにPuLPを使う
  - これが最適解です！と言いつつ微妙に違う解を出してくる
  - 配送計画問題の解を可視化させて、何かおかしいと思ったら頂点の添字が1つズレていた

## やったこと

以下の2つのことをやってもらいました。AI（Claude Code）に各種ソルバーが使えるかを検証したかったので、どちらも問題およびデータはそこまで難しくないものを選びました。

- [CBC](https://github.com/coin-or/Cbc)で施設配置問題を解く
  - MIPソルバーおよびモデラーを使えるかの検証
  - 本当は[HiGHS](https://github.com/ERGO-Code/HiGHS)を使ってほしかったけど、AIに「このサイズの問題ならCBCで十分」と断られた
- OR-toolsで配送計画問題（Vehicle Routing Problem, VRP）を解く
  - ヒューリスティックソルバーを使えるかの検証

使ったデータやコードは以下のリポジトリにあります。なお、今回の検証で人間は1行もコードを書いておらず、すべてAIに指示して書いてもらいました。

https://github.com/oddgai/datascience-playground/tree/main/vrp-with-claude-code

### CBCで施設配置問題を解く

#### 使ったデータ

こちらのサイトからRochat and Taillard (1995)による`tai100a`を拝借しました。本来はVRP用のベンチマークデータですが、地理データであれば何でもよかったのでこれにしました。後述するVRPでもこのデータを使っています。

https://vrp.atd-lab.inf.puc-rio.br/index.php/en/

![](https://storage.googleapis.com/zenn-user-upload/c3aef1f2f4b4-20250912.png)
*tai100a（今回は赤と青の点を一緒くたにして施設配置問題を解く）*

サイトでダウンロードできるのは以下のようなやや癖のあるテキストデータなのですが、何の説明もせずファイルを与えただけでこれをパースするコードを書いてくれました。たすかる！

```txt:tai100a.vrp
NAME : Tai100a
TYPE : CVRP  # Capacitated Vehicle Routing Problem 用のデータという意味
COMMENT : (No of trucks: 11, Optimal value: 2041.34)  # 車両数と最適値
DIMENSION : 101  # 頂点数
CAPACITY : 1409  # 車両ごとの容量
EDGE_WEIGHT_TYPE : EUC_2D  # 頂点間の距離はユークリッド距離
NODE_COORD_SECTION  # 頂点ごとの座標
1 0 0
2 -5 70
3 -17 69
...
100 12 41
101 24 53
DEMAND_SECTION  # 頂点ごとの需要量
1 0
2 13
3 122
...
100 5
101 10
DEPOT_SECTION  # 車両が出発する拠点の頂点番号
1
-1
EOF
```

#### 実験管理

実験管理は[冒頭に載せた記事](https://zenn.dev/genda_jp/articles/20250909_kaggle_with_claude_code#%E5%AE%9F%E9%A8%93%E7%AE%A1%E7%90%86)と同じようにやったので、詳細はこちらを参照してください。

AIにコードを書いてもらうためには、人間がAIの出力を確認しやすい仕組みを作るのが大切だと思っています。特にデータサイエンス系のプロジェクトでは、AIが書いたコードだけでなく関連する入出力（パラメータ、評価指標、グラフなど）もちゃんと確認・比較したいです。そのために、今回はMLflowという実験管理ツールを使っています。

MLflowとは機械学習のライフサイクル管理（MLOps）を目的としたオープンソースライブラリです。機械学習プロジェクトの実験管理で使うのが一般的ですが、**パラメータなどの入力、評価指標などの出力、それらに付随する画像やHTMLなどのファイルをまとめて管理できるため、数理最適化プロジェクトでもかなり重宝すると思っています**（特にPoCなどの初期段階）。

https://mlflow.org/

#### AI（Claude Code）による実装

以下のような流れで実装を進めました。

1. 人間がAIに指示する
2. AIがコードを書いて実行する
3. 人間がコードなどの出力を確認して、気になる部分があればAIに修正を指示する
4. AIがコードを書いて実行する
5. 以下同様

AIへの指示はざっくりめにしました。たとえば最初の指示は以下のとおりで、解いてほしい問題ややってほしいことを箇条書きでまとめています。問題はいわゆる[p-メディアン問題](https://scmopt.github.io/opt100/40kmedian.html)ですが、あえてその単語は出していません。うまく察してくれることを期待します。

> 以下のような施設配置問題をMIPとして定式化してPython-MIPで解いて、MLflowに記録して。
> - tai100aを使った施設配置問題
> - tai100aの各点を施設候補地とし、各点に需要があるとする
> - 施設の設置コストは0とする
> - 施設の設置数は10個とする
> - 各需要点は最も近い施設からの距離と、需要点のDemandを掛けたものがコストになる
>   - ある需要点のDemandが5で、最も近い施設までの距離が3なら、その需要点のコストは15になる
> - 全需要点のコストの総和を最小化するような施設の設置場所を決定する
> - instances/facility-location というフォルダを作って、その下の experiments/exp001, results/exp001 に実験過程のノートブックや結果を保存する
> - MLflowには以下の情報を記録する
>   - 施設の設置場所（リスト、あるいはカンマ区切りの文字列）
>   - 全需要点のコストの総和（目的関数値）
>   - 最適化にかかった時間
>   - 最適化のギャップ (最適化が完了しなかった場合)
>   - 需要点と施設の位置を可視化したもの（Artifactsとして保存）

これだけの指示で、以下のような合計1000行くらいのコードを数分で書いて実行してくれます。すげ〜。

- データをパースするコード

https://github.com/oddgai/datascience-playground/blob/main/vrp-with-claude-code/src/utils.py

- データをロードするコード

https://github.com/oddgai/datascience-playground/blob/main/vrp-with-claude-code/instances/facility-location/src/facility_utils.py

- 数理最適化モデル実装
  - ちゃんとp-メディアン問題の定式化に準拠して実装してくれました。えらい！

https://github.com/oddgai/datascience-playground/blob/main/vrp-with-claude-code/instances/facility-location/src/facility_mip.py

- 可視化

https://github.com/oddgai/datascience-playground/blob/main/vrp-with-claude-code/instances/facility-location/src/facility_visualization.py

- 実行ファイル

https://github.com/oddgai/datascience-playground/blob/main/vrp-with-claude-code/instances/facility-location/experiments/exp001/facility_location_experiment.py


MLflowを確認すると、ちゃんとMIPソルバーで最適解を出してくれたようです。

![](https://storage.googleapis.com/zenn-user-upload/5c99de80a835-20250912.png)
*MLflowの画面*

ただ、可視化したものが見づらかったので修正をお願いしてみます。日本語が微妙だったりtypoもありますがうまく読み取ってくれます。

> facility_location_solution.png は以下のことが気になるから直してほしい
> - 色が見づらい
> - 需要点と施設を結ぶ線が細すぎる
> - 需要点を、需要の大きさに応じて点の大きさを変える（需要が大きいほど点が大きくなる）
> - 需要店のラベルは要らない
> - 画像サイズが大きすぎるからdpi=80くらいにする

いくつかの修正を経て出してくれた図がこちらです。まだまだ気になる部分はありますが、内々で使う分にはまあいいでしょう。

![](https://storage.googleapis.com/zenn-user-upload/6e157c64dd05-20250912.png)
*これもMLflowの画面。画像を表示してくれるのが嬉しい*

という感じで、このくらいの問題・データであればすぐに解いてくれました。

### OR-toolsで配送計画問題を解く

配送計画問題（VRP）とは、1つの拠点から複数の車両が複数の配送先へ配送するとき、全ての車両の走行距離の総和が最小になるような配送ルートを求める問題です。全ての車両の走行距離の総和が評価指標（目的関数値）となり、この値が小さいほど良い解になります。

VRPは古くから研究されている問題で多くのバリエーションがありますが、今回は容量制約付きの配送計画問題（Capacitated Vehicle Routing Problem, CVRP）を解いてもらいます。配送先ごとに需要（例：N個の商品がほしい）をもっているが、1つの車両に載せられる量には上限があり、どの車両がどの配送先に向かえば需要を満たしたうえで走行距離を短くできるか、という問題です。

#### 使ったデータ

先ほどと同じシリーズの`tai75a`, `tai100a`, `tai150a`, `tai385`を使いました。入力データだけでなく最適解も提供されているので、解いた結果と最適解を比較できます。

#### AIによる実装

実験管理、および実装の流れも先ほどと同様です。施設配置問題のときよりもざっくり指示してみます。

> instances/tai75a/data にVRPのデータがある。instances/tai75a/experiments/exp001 にこれを解くアルゴリズムを書きたい。OR-toolsで実装できる？結果はinstances/tai75a/results/exp001 において

VRPやOR-toolsの説明はあえて省いたのですが、これだけでもコードを書いてくれました。すごい〜。

- 汎用的なモジュール（データを読み込んだり、距離行列を作ったり）

https://github.com/oddgai/datascience-playground/blob/main/vrp-with-claude-code/instances/tai75a/experiments/exp001/utils.py

- ソルバーによる実装（ノートブック）
  - [OR-toolsの公式ドキュメント](https://developers.google.com/optimization/routing/vrp?hl=ja)をほぼコピペしてそうだが、すなわち適切ということ

https://github.com/oddgai/datascience-playground/blob/main/vrp-with-claude-code/instances/tai75a/experiments/exp001/exp001.ipynb

MLflowを見ると、以下のように最適解に近い解を出してくれているようです。本来はOR-toolsによる解の目的関数値（1615）が最適解（1618.36）を下回ることはないのですが、コードを見るかぎり距離を整数に丸めたときの誤差で下回ってしまったんだろうと推測します。

![](https://storage.googleapis.com/zenn-user-upload/ac8dcf3454ba-20250912.png)
*上がOR-toolsによる解、下が既知の最適解*

上は75頂点でしたが、もう少し大きいデータでも解かせてみます。

> それぞれに対してVRPの実験を行い、結果をまとめてください。
> - tai100a
> - tai150a
> - tai385
>
> おそらく計算に時間がかかるインスタンスも含まれるため、最適解とのGAPが3％以下の解が求まらなかった場合は以下の秒数を順に試していってください。
> - 60 秒
> - 120 秒
> - 300 秒
> - 600 秒
>
> すべての実験はそれぞれ今までと同じようにインスタンス名に対応したディレクトリを作成し、その中に experiments/expXXX, results/expXXX というディレクトリを作成して、実験結果してください。1つの実験が終わるたびにMLflowに結果を記録してください。

しばらくすると実験が終わって、以下のようにMLflowに結果がまとまってました。`tai385`を除いて、最適解とのGAPが3%以下の解が求まったようです。本当は`solve_time`には計算にかかった時間を書いてほしかったのですが、あらかじめ設定した制限時間を書いてしまっています。ちゃんと指示しなかった人間が悪いです。

![](https://storage.googleapis.com/zenn-user-upload/d32b74803e87-20250912.png)
*MLflowの実験リスト*

## 感想

今回は扱った問題が簡単だったり有名なものだったこともあると思いますが、AIは数理最適化も割とできることがわかりました。テストデータではなく実務でもどんどん使っていきたいです。もちろん、実務の問題の多くは今回と比べ物にならないくらい複雑なのでこんなに一筋縄ではいかないですが。。。

AIは指示したことを守らなかったりミスもするので、私用ならともかく実務での完全な自動開発はまだ難しいですが、ソフトウェア開発と同じようにデータサイエンスの領域でもAIはたくさん活躍してくれそうです。というか、AIにたくさん活躍してもらうために人間ががんばらないといけないのだと思います。やっていきましょう。
