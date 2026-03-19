---
title: "GitHub Copilot CLI - hooksとalerterでタスク完了時に通知してくれる仕組み作った"
emoji: "🍇"
type: "tech"
topics: ["githubcopilot", "mac", "shell", "cli"]
published: true
publication_name: "sonicmoov"
---

## はじめに

GitHub Copilot CLIでタスクを依頼すると、処理が終わるまでターミナルを眺め続けることになりがちです。
長めのタスクを投げて別の作業をしていると「いつ終わったかな？」とターミナルを何度もチェックしてしまう…という体験をしていました。

そこで Copilot CLI の **hooks** 機能と macOS 向け通知ツール **alerter** を組み合わせて、タスク完了時に Mac へ通知を飛ばす仕組みを作ってみました。

![通知](https://storage.googleapis.com/zenn-user-upload/cc083242f2ac-20260319.png)

---

## 1. alerter をインストールする

alerter は macOS の通知センターへ通知を送れる CLI ツールです。Homebrew でインストールできます。

```bash
brew install alerter
```

インストール後、以下のコマンドで動作確認できます。

```bash
alerter --title 'テスト' --message '通知のテストです' --sound default
```

Mac の通知センターに通知が届けば OK です。

:::message
**通知が届かない場合は通知の許可設定を確認してください。**
`システム設定 > 通知 > ターミナル` を開き、通知を「許可」にする必要があります。この設定が OFF になっていると通知が一切届きません。
![通知設定](https://storage.googleapis.com/zenn-user-upload/1a34d0d9fe93-20260319.png)
:::

---

## 2. hooks.json を作成する

Copilot CLI の hooks はプロジェクトルートの `.github/hooks/hooks.json` に記述します。

```bash
mkdir -p .github/hooks
touch .github/hooks/hooks.json
```

以下の内容を記述します。

```json:.github/hooks/hooks.json
{
  "version": 1,
  "hooks": {
    "sessionEnd": [
      {
        "type": "command",
        "bash": "alerter --title 'プロジェクト名' --message 'タスクが完了しました。' --sound default",
        "timeoutSec": 10
      }
    ]
  }
}
```

### 設定項目の説明

| キー         | 説明                                                           |
| ------------ | -------------------------------------------------------------- |
| `version`    | hooks スキーマのバージョン。現在は `1`                         |
| `sessionEnd` | セッション終了時に実行されるフック                             |
| `type`       | フックの種類。`command` を指定するとシェルコマンドが実行される |
| `bash`       | 実行するシェルコマンド                                         |
| `timeoutSec` | コマンドのタイムアウト秒数                                     |

---

## 3. 動作確認

Copilot CLI でタスクを実行してセッションが終了すると、Mac の通知センターに以下のような通知が表示されます。

![通知](https://storage.googleapis.com/zenn-user-upload/cc083242f2ac-20260319.png)

タスクを依頼して別のウィンドウで作業しながら待てるようになりました！

---

## カスタマイズ例

`alerter` はオプションが豊富なので、さらに細かくカスタマイズできます。

```bash
# アクションボタンを付ける
alerter --title 'Copilot' --message 'タスクが完了しました。' --sound default --actions '確認'

# 一定時間後に自動で消える（秒数指定）
alerter --title 'Copilot' --message 'タスクが完了しました。' --sound default --timeout 5
```

---

## まとめ

- `.github/hooks/hooks.json` に `sessionEnd` フックを追加するだけで、タスク完了後にコマンドを自動実行できる
- `alerter` と組み合わせることで Mac への通知が簡単に実現できる
- タスク依頼後は別の作業に集中でき、開発効率が上がった

hooks には `sessionEnd` 以外にもフックのタイミングが用意されているので、他の通知方法（Slack通知、ログ記録など）にも応用できそうです。気になる方はぜひ試してみてください！
