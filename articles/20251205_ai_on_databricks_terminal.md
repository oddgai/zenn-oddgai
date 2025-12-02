---
title: "Databricks Webターミナルでコーディングエージェントを動かす"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics:
  - "databricks"
  - "claudecode"
  - "codex"
  - "データ分析"
published: false # 公開設定(falseにすると下書き)
published_at: 2025-12-05
publication_name: "genda_jp"
---

この記事はGENDA Advent Calendar 2025#2 および Databricks Advent Calendar 2025#1 5日目の記事です。

https://qiita.com/advent-calendar/2025/genda
https://qiita.com/advent-calendar/2025/databricks

最近知ったのですが、1記事に2つのアドカレを紐づけてもOKらしいです。

> Organizationが紐付いているカレンダー＋Organizationが紐付いていないカレンダーであれば、1つの記事で2つのカレンダーに紐付け可能です。

https://qiita.com/advent-calendar/2025

## TL;DR

- Databricks Webターミナル上でClaude CodeやCodex CLIなどのコーディングエージェントを動かせる
- 1窓しか開けないが、screenなどのTerminal Multiplexerを使えば1窓で複数の作業を並行できる
- init Scriptsを使って自動インストールすれば、Compute起動のたびに手動でインストールする手間が省ける
- やや動作がもっさりするのが難点だが、Databricks Computeのリソースを使いながらAIと協業できるのは便利

![alt text](/images/20251205_ai_on_databricks_terminal/demo.png)
*Databricks Webターミナルで開発しつつ、別窓でワークスペースブラウザを開いて成果物を確認できる*

## はじめに

こんにちは、データサイエンティストのoddgaiです。積みゲーがあるのにSteamのブラックフライデーでまたゲームを買ってしまいました。

さて、普段の業務でデータ分析をするとき、ローカルPCからDatabricks上のデータにアクセスして作業をしています。Claude CodeやCodex CLIなどのコーディングエージェントにコードやクエリを書いてもらうことも多いです。

しかし、重い処理を実行するときはローカルPCの計算リソースでは心もとなく、Databricks Computeの計算リソースを使いたくなります。ローカルからJobとして投げる方法もありますが、開発中はよりスピーディーに試行錯誤したいものです。そこで **「Databricks上でClaudeやCodexが使えればいいのでは？」** と思って試してみたら、意外とできたのでその方法を共有します。

## Databricks Webターミナルとは

Databricksでは、`Compute > [任意のCompute] > Apps > Terminal` からWebターミナルを起動できます。文字通りブラウザ上で動作するターミナルで、指定したComputeにSSHで接続しているようなイメージです。

https://docs.databricks.com/aws/ja/compute/web-terminal

![alt text](/images/20251205_ai_on_databricks_terminal/databricks_terminal.png)
*この画面から起動できる*

作業ディレクトリ（筆者の環境では`/Workspace/Users/[自分のメールアドレス]`）の下にGitフォルダを作成しておけば、外部環境との同期も可能です。

https://docs.databricks.com/aws/ja/repos/

ターミナル経由で編集したファイルはワークスペースブラウザからも確認できます。そのため

- AIにノートブックを書いてもらう
- それをブラウザから確認・実行する

といったこともできます。

## カスタマイズ

Databricks Webターミナルは便利ですがいくつか制約もあるため、使いやすくなるようにカスタマイズをしています。

### Terminal Multiplexerを使う

Databricks Webターミナルは1窓しか開けません。しかし、Claude CodeやCodex CLIを使う場合、AIの実行を見守りながら別の作業をしたり、複数のセッションを並行したりしたいことがあります。

そこでターミナルマルチプレクサを使います。`tmux`や`Zellij`などいくつかの選択肢がありますが、Databricks環境では一部が正常に動作しませんでした（インストールや起動はできるが、Workspaceフォルダにアクセスできなくなるなど）。試した中では`screen`は安定して動作することを確認できました。

### init Scriptを使う

Databricks ComputeはUbuntuなので、ターミナル経由でClaudeを含むさまざまなツールをインストールできます。
Computeを停止するとまっさらな状態に戻りますが、init Scriptを使えば起動時に各種設定を自動化できます。

https://docs.databricks.com/aws/ja/init-scripts#cluster-scoped-init-script

下記は筆者のinit Scriptの一部です。Screen, Claude Code, Codex CLIをインストールしています。なおClaudeとCodexの認証は起動後に手動でやる必要があります。

```bash:init.sh
#!/bin/bash

# screenをインストール
apt-get update
apt-get install screen

# screenの設定
SCREEN_CONF="$HOME/.screenrc"
cat > "$SCREEN_CONF" << 'EOF'
# Encoding
defutf8 on
defencoding utf8
encoding utf8 utf8

# 分割後に自動でフォーカスと新規ウィンドウ作成
bind _ eval "split" "focus down" "screen"
bind | eval "split -v" "focus right" "screen"

# スクロール行数
defscrollback 1000000
startup_message off
autodetach on

# マウススクロール有効
termcapinfo xterm* ti@:te@

# ステータスライン
hardstatus alwayslastline "%{= cd} %-w%{= wk} %n %t* %{-}%+w"
EOF

source "$SCREEN_CONF"

# 各種インストール
WORKING_DIR="/Workspace/Users/[自分のメールアドレス]"

## mise
curl https://mise.run | sh

cat > "$WORKING_DIR/mise.toml" << 'EOF'
[settings]
trusted_config_paths = [
  "/Workspace/Users/[自分のメールアドレス]"
]
EOF

cd "$WORKING_DIR"
/root/.local/bin/mise use node@lts
/root/.local/bin/mise use uv@0.9.2
/root/.local/bin/mise use npm:@openai/codex@latest

echo 'eval "$(/root/.local/bin/mise activate bash)"' >> ~/.bashrc

## Claude
curl -fsSL https://claude.ai/install.sh | bash -s latest
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc

# デフォルトでtmuxを起動させない
echo 'export DISABLE_TMUX=true' >> ~/.bashrc

# プロンプトをカスタマイズ
echo "PS1='\e[34m\w\n\e[0m\$ '" >> ~/.bashrc

# 起動時にこのパスに移動する
echo 'cd $WORKING_DIR' >> ~/.bashrc

# 設定を反映
source ~/.bashrc
```

## まとめ

Databricks Webターミナル上でもClaude CodeやCodex CLIを使って開発できます。やや動作が重いのは気になりますが、大規模データを扱う場合やローカル環境では難しい処理を行う場合には十分に価値があります。ぜひ試してみてください！
