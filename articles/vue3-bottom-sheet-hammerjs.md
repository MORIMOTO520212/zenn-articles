---
title: "Vue3 + HammerJS でスワイプ式ボトムシートを自作する"
emoji: "👆"
type: "tech"
topics: ["Vue", "Nuxt", "TypeScript", "HammerJS", "UI"]
published: false
publication_name: "sonicmoov"
---

## この記事で分かること

- HammerJS を使ってスワイプジェスチャーを検出する方法
- `open` / `half` / `close` の3状態を持つボトムシートの設計思想
- スワイプ中のリアルタイム追従と、離した後のスナップをスムーズに切り替えるテクニック
- Vue3 の Composition API と組み合わせる実装パターン

対象読者：Vue3 でモバイルUIを作っているエンジニア

---

## なぜ自作するのか

ボトムシートは既製ライブラリも存在しますが、次の理由から自作を選びました。

- **3状態（open/half/close）への対応**：既製品の多くは open/close の2状態のみ
- **位置の柔軟なカスタマイズ**：各状態の停止位置を画面高さの割合で自由に指定したい
- **親からの状態制御**：地図画面など外部コンポーネントがシートの状態を操作する必要がある

---

## 完成形のイメージ

```
┌──────────────────┐
│                  │  ← open（画面の1%の位置）
│   コンテンツ     │
│                  │
├──────────────────┤  ← half（画面の45%の位置）
│   ▬▬▬           │  ← ドラッグバー
├──────────────────┤  ← close（画面の70%の位置）
```

スワイプの方向と移動量によって3つの状態をスナップしながら遷移します。

---

## HammerJS とは

HammerJS はタッチ・マウスの複合ジェスチャーを検出するライブラリです。

```bash
yarn add hammerjs
yarn add -D @types/hammerjs
```

`pan`（ドラッグ）イベントを使うと、開始座標・移動中座標・終了座標を簡単に取得できます。

:::message
Touch API の `touchmove` でも同じことはできますが、HammerJS を使うと iOS/Android の差異やマウス操作への対応を意識せず書けます。
:::

---

## 設計の核心：状態・位置・動きを分離する

このコンポーネントで重要なのは **3つの概念を分けること** です。

| 概念 | 変数 | 説明 |
|---|---|---|
| 状態 | `state` | `open` / `half` / `close` |
| スナップ位置 | `calcY` (computed) | 状態に対応した `top` の値 |
| スワイプ中の位置 | `y` (ref) | 指に追従するリアルタイムの `top` 値 |

スワイプ中は `y` で動かし、指を離したら `state` が確定して `calcY` に引き渡す——この切り替えが滑らかな動きの鍵です。

```ts
// isMove が true の間は y で追従、false になったら calcY に戻る
:style="{ top: `${isMove ? y : calcY}px` }"
```

---

## 各状態の停止位置を計算する

各状態の停止位置は、**画面高さに対する割合**として Props で受け取ります。

```ts
const props = withDefaults(defineProps<Props>(), {
  openY: 0.01,   // 画面の1%  → ほぼ全画面
  halfY: 0.45,   // 画面の45% → 半分
  closeY: 0.7,   // 画面の70% → 少しだけ見える
})
```

```ts
const calcY = computed(() => {
  if (!rect.value?.height) return 0
  switch (state.value) {
    case 'open':  return rect.value.height * props.openY
    case 'half':  return rect.value.height * props.halfY
    case 'close': return rect.value.height * props.closeY
  }
})
```

画面サイズが変わっても追従できるよう、`window.onresize` で `rect` を更新します。

---

## HammerJS でスワイプを検出する

`onMounted` でドラッグバー要素に HammerJS を設定します。

```ts
mc.value = new Hammer(pan.value)
mc.value.get('pan').set({ direction: Hammer.DIRECTION_ALL })

// スワイプ中：指に追従
mc.value.on('panup pandown', (evt) => {
  y.value = evt.center.y - 16
})

// スワイプ開始：追従モードへ切り替え
mc.value.on('panstart', (evt) => {
  startY.value = evt.center.y
  isMove.value = true
})

// スワイプ終了：移動量で次の状態を決定
mc.value.on('panend', (evt) => {
  isMove.value = false
  const diff = startY.value - evt.center.y  // 上方向が正
  // ここで状態遷移を判定（次のセクション参照）
})
```

---

## 移動量で次の状態を決める

スワイプ終了時、`startY - endY`（上方向が正）の値で遷移先を決めます。

```ts
mc.value.on('panend', (evt) => {
  isMove.value = false
  const diff = startY.value - evt.center.y

  switch (state.value) {
    case 'close':
      if (diff > 320) setState('open')
      else if (diff > 120) setState('half')
      break
    case 'half':
      if (diff > 120) setState('open')
      if (diff < -50) setState('close')
      break
    case 'open':
      if (diff < -120) setState('half')
      break
  }
})
```

各閾値（120px・320px・50px）はデバイス実機で調整した値です。

---

## CSS transition のタイミング制御

スワイプ中に transition が効いていると、指の動きに対してカクつきます。
`data-state` 属性を使って **スワイプ中だけ transition を無効化** します。

```vue
<div
  :data-state="isMove ? 'move' : state"
  :style="{ top: `${isMove ? y : calcY}px` }"
>
```

```css
/* isMove=false のとき（state = open/half/close）だけ transition を適用 */
.card-animation[data-state='open'],
.card-animation[data-state='half'],
.card-animation[data-state='close'] {
  @apply transition-[top] duration-300 ease-out;
}
/* data-state='move' のときは transition なし → 指にリアルタイム追従 */
```

---

## 初期表示でアニメーションをスキップする

マウント直後に transition クラスを付けると、初期位置からスライドインアニメーションが走ってしまいます。

```ts
onMounted(async () => {
  // ... Hammer の設定 ...

  // 1ms 待ってから transition を有効化
  await sleep(1)
  wrapper.value?.classList.add('wrapper-animation')
  card.value?.classList.add('card-animation')
})
```

`sleep(1)` で1フレーム待つことで、初期位置の描画が完了してからアニメーションを有効化できます。

---

## 外部から状態を操作できるようにする

地図画面では「駐車場をタップしたらシートを開く」などの操作が必要です。
`defineExpose` で親から `setState` を呼べるようにします。

```ts
defineExpose({ setState })
```

```vue
<!-- 親コンポーネント -->
<VBottomSheet ref="bottomSheetRef" />

<script setup>
const bottomSheetRef = ref()
bottomSheetRef.value?.setState('open')
</script>
```

---

## クリーンアップを忘れずに

`onBeforeUnmount` で必ずクリーンアップします。

```ts
onBeforeUnmount(() => {
  mc.value?.destroy()       // HammerManager を破棄
  window.onresize = null    // リサイズイベントを解除
})
```

---

## まとめ

| ポイント | 内容 |
|---|---|
| ジェスチャー検出 | HammerJS の `pan` イベントで開始・移動・終了を取得 |
| 状態と位置の分離 | `state`（snap先）と `y`（追従中）を別 ref で管理 |
| スムーズな追従 | `isMove` フラグで CSS transition を動的にON/OFF |
| 閾値ベースのスナップ | `startY - endY` の移動量で遷移先を決定 |
| 初期アニメーション回避 | `await sleep(1)` でtransitionクラスを1フレーム遅延付与 |

既製ライブラリに頼らず自作することで、アプリ固有の要件（3状態・外部制御・位置カスタマイズ）に完全対応できました。スワイプ系 UI の実装で詰まったときの参考になれば幸いです。

---

## 参考

- [HammerJS](https://hammerjs.github.io/)
- [vue-swipeable-bottom-sheet](https://github.com/atsutopia/vue-swipeable-bottom-sheet)（今回の実装のベース）
