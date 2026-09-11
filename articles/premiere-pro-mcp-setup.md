---
title: "Claude CodeからPremiere Proを操作する（MCP導入編）"
emoji: "🎬"
type: "tech"
topics: ["premierepro", "mcp", "claudecode", "adobe"]
published: false
---

動画編集って、同じ作業の繰り返しが多いんですよね。毎回同じテロップを入れて、同じ書き出し設定を選んで……。

この記事では、AIアシスタントのClaude CodeからPremiere Proを直接操作できるようにするMCPサーバーの導入手順と、実際にやってみてハマったポイントをまとめます。Windows環境での話です。

## そもそもどうやって動くのか

Premiere Proには外部から叩けるAPIがありません。なので、どのツールも同じ構造を取っています。

Claude → MCPサーバー（Node.js）→ WebSocket → CEP拡張（Premiere内）→ ExtendScript → Premiere Pro

ポイントは「CEP拡張」です。Premiere Proの中に小さなパネルを常駐させておいて、外から届いた命令をそのパネルが実行する、という仕組みなんです。だからPremiereを起動してパネルを開いておかないと動きません。

## どのMCPサーバーを選ぶか

現時点で主要な選択肢は3つ。いずれも有志によるOSSで、Adobe公式ではありません。

| リポジトリ | 対応OS | ツール数 | 通信方式 |
| --- | --- | --- | --- |
| antipaster/Adobe-Premiere-Pro-MCP | Windows専用 | 170+ | WebSocket |
| leancoderkavy/premiere-pro-mcp | Win / Mac | 269 | ファイルIPC |
| ayushozha/AdobePremiereProMCP | Win / Mac | 1,027 | - |

今回はWindows環境なので、インストーラーがClaude Codeの設定まで自動でやってくれるantipaster版を選びました。実際に接続したら180個のツールが見えました。

## 導入手順

### 事前準備

PowerShellで、Node.jsとGitが入っているか確認します。

```powershell
node -v
git --version
```

Node.jsが18未満、または未インストールならnodejs.orgからLTS版を入れてください。私の環境はNode v24.14.1とGit 2.54.0でした。

### インストール

Premiere Proは閉じた状態で実行します。

```powershell
cd $HOME
git clone https://github.com/antipaster/Adobe-Premiere-Pro-MCP.git
cd Adobe-Premiere-Pro-MCP
.\install.bat
```

install.batが以下を自動で行います。

1. 未署名CEP拡張の許可（レジストリのPlayerDebugModeを有効化）
2. CEPパネルをPremiereの拡張フォルダにジャンクションで接続
3. 依存パッケージのインストールとビルド
4. Claude Desktop / Claude CodeのMCP設定を追記

私の環境では管理者権限なしで通りました。npm audit の警告が12件出ますが、`npm audit fix` は実行しない方が無難です。依存が壊れてツールが動かなくなることがあります。

## ここでハマった：起動の順番

READMEには「Premiere Proを起動 → パネルを開く → Connectedを確認 → Claude Codeを再起動」と書かれています。**この順番だとつながりません。**

パネルを開いても、こうなります。

```
Connecting to MCP server on port 8097...
WebSocket error
Disconnected from MCP server
```

理由はシンプルで、**ポート8097で待ち受けるサーバーを立ち上げるのはClaude Code側**だからです。CEPパネルは「つなぎに行く側」なので、Claude Codeが起動していないと接続先が存在しません。

### 正しい順番

1. **先にClaude Codeを起動する**（`claude` コマンド）
2. `/mcp` で `premiere-pro · connected` を確認
3. **そのあと**Premiereのパネルで「Reconnect」をクリック

これで一発で `Connected` になりました。

### Claude Codeを閉じるとサーバーも止まる

MCPサーバーはClaude Codeの子プロセスとして動いています。ターミナルを閉じるとサーバーも落ちて、パネルは `Disconnected` に戻ります。作業中はClaude Codeを開きっぱなしにしておく必要があります。

### Claude DesktopとClaude Codeは同時に使えない

インストーラーは両方に設定を書き込みますが、**ポート8097を使えるのは1つのプロセスだけ**です。片方を使っている間はもう片方から操作できません。どちらで使うか決めておくのがよさそうです。

## もうひとつの罠：ログイントークンの期限切れ

接続できたので意気揚々と指示を出したら、これが返ってきました。

```
API Error: 401 OAuth access token has expired. Re-authenticate to continue.
```

Premiereとは無関係で、単にClaude Code自体のログインが切れていただけです。`/login` を実行して再認証すれば解決します。久しぶりにClaude Codeを起動した人は、先に `/login` を通しておくと切り分けがラクだと思います。

## 実際に動かしてみた

再認証後、こう頼んでみました。

```
Premiere で今開いているプロジェクトの情報を教えて
```

31秒ほどで、こんな情報が返ってきました。

- プロジェクト名とファイルパス、ドキュメントID
- WebSocketポート8097で接続中であること
- アクティブシーケンス名、解像度（1080×1920）、再生ヘッド位置
- ビデオトラック3本・オーディオトラック3本の構成と、各トラックのクリップ数
- プロジェクトパネルのbin構造と、中のクリップ名・元ファイルパス

しかも「縦型素材で9:16のショート動画を編集中ですね。未保存なので早めに保存を」とまで言われました。**プロジェクトの状況を読んで文脈を理解している**のが、地味にすごいところです。

### ツールの癖もある

`get_project_info` が `numSequences: 0` を返す一方で、実際にはアクティブシーケンスが存在する、という食い違いがありました。取得経路によって見える範囲が違うようです。有志製ツールなので、こういう不整合は前提として付き合う必要があります。

## 導入前に知っておきたいこと

### セキュリティ設定を変更します

「未署名CEP拡張の許可」は、Adobeが署名していない拡張機能を動かせるようにする設定です。つまりこのPCでは今後、出所不明のCEP拡張も動くようになります。有志製ツールを動かす以上は避けられない工程ですが、内容を理解したうえで実行してください。

### 本番プロジェクトでいきなり使わない

AIがタイムラインを直接書き換えます。まずは読み取り系（プロジェクト情報、クリップ一覧）から始めて、次にマーカー追加のような戻せる操作、最後にカットや配置、という順で慣らすのがおすすめです。

## まとめ

- Premiere ProはCEP拡張を経由してAIから操作できる
- Windows環境ならantipaster版がインストーラー付きで手軽
- **起動順はClaude Codeが先、Premiereのパネルは後**。逆だとつながらない
- Claude Codeを閉じるとサーバーも止まる
- Claude DesktopとClaude Codeはポートの取り合いになるので同時使用は不可
- 未署名CEP拡張の許可というセキュリティ設定の変更を伴う

つまずいたポイントはほぼ全部「順番」と「認証」でした。そこさえ越えれば、プロジェクトの中身を読んで文脈を理解してくれるので、繰り返し作業を任せられる未来がかなり現実的に見えてきます。
