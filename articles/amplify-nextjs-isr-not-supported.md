---
title: "AmplifyはNext.jsのISRをサポートしていない"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "amplify", "aws", "isr"]
published: true
---

Next.jsアプリをAWS Amplifyにデプロイする際、`revalidateTag()` や `revalidatePath()` のオンデマンドISRが一切機能しなかった。

ローカル環境では正常に動作するのに、Amplifyにデプロイすると再検証が走らず、古いキャッシュが返され続ける。

## 原因

AWS公式ドキュメントに明記されていた。

> バージョン 12.2.0 以降、Next.js は特定のページの Next.js キャッシュを手動で削除するインクリメンタル・スタティック・リジェネレーション (ISR) をサポートしています。ただし、**Amplify は現在オンデマンド ISR をサポートしていません**。アプリが Next.js のオンデマンド再検証を使用している場合、アプリを Amplify にデプロイしてもこの特徴量は機能しません。
>
> https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/troubleshooting-SSR.html#on-demand-isr-not-supported

つまり、以下のAPIはAmplifyでは動作しない。

- `revalidateTag(tag)` — タグベースの再検証
- `revalidatePath(path)` — パスベースの再検証

## 対処法

`force-dynamic` や `cache: 'no-store'` でキャッシュを無効化する。

```ts
export const dynamic = "force-dynamic";
```

## まとめ

AmplifyでNext.jsを運用する場合、`revalidateTag()` / `revalidatePath()` は機能しない。オンデマンドISRを使いたい場合はVercelなど対応プラットフォームへの移行を検討する。
