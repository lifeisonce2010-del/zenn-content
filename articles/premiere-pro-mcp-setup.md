---
title: "Claude CodeからPremiere Proを操作する（MCP導入編）"
emoji: "🎬"
type: "tech"
topics: ["premierepro", "mcp", "claudecode", "adobe"]
published: false
---

動画編集って、同じ作業の繰り返しが多いんですよね。毎回同じテロップを入れて、同じ書き出し設定を選んで……。

この記事では、AIアシスタントのClaude CodeからPremiere Proを直接操作できるようにするMCPサーバーの導入手順をまとめます。Windows環境での手順です。

## そもそもどうやって動くのか

Premiere Proには外部から叩けるAPIがありません。なので、どのツールも同じ構造を取っています。

Claude → MCPサーバー（Node.js）→ ブリッジ → CEP拡張（Premiere内）→ ExtendScript → Premiere Pro

ポイントは「CEP拡張」です。Premiere Proの中に小さなパネルを常駐させておいて、外から届いた命令をそのパネルが実行する、という仕組みなんです。だからPremiereを起動してパネルを開いておかないと動きません。

## どのMCPサーバーを選ぶか

現時点で主要な選択肢は3つ。いずれも有志によるOSSで、Adobe公式ではありません。

| リポジトリ | 対応OS | ツール数 | 通信方式 |
| --- | --- | --- | --- |
| antipaster/Adobe-Premiere-Pro-MCP | Windows専用 | 170+ | WebSocket |
| leancoderkavy/premiere-pro-mcp | Win / Mac | 269 | ファイルIPC |
| ayushozha/AdobePremiereProMCP | Win / Mac | 1,027 | - |

今回はWindows環境なので、インストーラーがClaude Codeの設定まで自動でやってくれるantipaster版を選びました。

できることは、クリップの配置・カット・トリミング、エフェクトやLUTの適用、キーフレーム操作、マーカー、書き出し、プロジェクト構造の読み取りなど。タイムライン編集のほぼ全域をカバーしています。

## 導入手順

### 事前準備

PowerShellで、Node.jsとGitが入っているか確認します。

```powershell
node -v
git --version
```

Node.jsが18未満、または未インストールならnodejs.orgからLTS版を入れてください。

### インストール

```powershell
cd $HOME
git clone https://github.com/antipaster/Adobe-Premiere-Pro-MCP.git
cd Adobe-Premiere-Pro-MCP
.\install.bat
```

install.batが以下を自動で行います。

1. 未署名CEP拡張の許可（レジストリのPlayerDebugModeを有効化）
2. CEPパネルをPremiereの拡張フォルダにリンク
3. 依存パッケージのインストールとビルド
4. Claude Desktop / Claude CodeのMCP設定を追記

### 接続確認

1. Premiere Proを起動する
2. メニューの「ウィンドウ → 機能拡張 → MCP Bridge」を開く
3. パネルの表示が「Connected」になればOK
4. Claude Codeを再起動すると、Premiereのツールが使えるようになる

## 導入前に知っておきたいこと

### セキュリティ設定を変更します

手順の1つ目、「未署名CEP拡張の許可」は、Adobeが署名していない拡張機能を動かせるようにする設定です。つまりこのPCでは今後、出所不明のCEP拡張も動くようになります。

有志製ツールを動かす以上は避けられない工程なんですが、内容を理解したうえで実行してください。

### 本番プロジェクトでいきなり使わない

AIがタイムラインを直接書き換えます。意図しない編集が入る可能性は普通にあるので、まずはダミープロジェクトで挙動を確かめるのがおすすめです。

### Premiereのバージョンアップで壊れることがある

非公式ツールの宿命ですね。Adobeのアップデート後に動かなくなったら、リポジトリのIssueを覗いてみるといいと思います。

## 実際に使ってみた（追記予定）

<!-- 導入が完了したら、ここに実際に試した内容と所感を追記する -->

- 導入でつまずいたポイント
- 実際に指示してみた作業と結果
- 実用に耐えるか、どこまで任せられるか

## まとめ

- Premiere ProはCEP拡張を経由してAIから操作できる
- Windows環境ならantipaster版がインストーラー付きで手軽
- 未署名CEP拡張の許可というセキュリティ設定の変更を伴う
- まずはダミープロジェクトで試すのが安全

繰り返し作業を任せられるようになれば、編集の体感はかなり変わりそうです。実際に使い込んだら、この記事に追記していきます。
