---
title: "ライブラリなしでスワイプ対応 BottomSheet を React に実装した——CSSトランジションが「一発で動かない理由」と double rAF trick"
emoji: "📱"
type: "tech"
topics: ["react", "typescript", "css", "nextjs", "frontend"]
published: true
canonical_url: "https://mojitonews.hateblo.jp/entry/bottomsheet-from-scratch-double-raf"
---

## はじめに：なぜゼロから書いたのか

しまなみ海道の観光ガイドサイトを個人で開発しています。マップ上のスポットをタップすると詳細が表示されるモバイルUIを作るため、BottomSheet が必要になりました。

最初は Radix UI の `<Dialog>` や `vaul`（Drawer ライブラリ）を試しました。動きはします。ただ、使いながら「なぜこう動くのか」がわからないまま実装が進んでいく感覚に違和感がありました。

ライブラリを使えば1時間で終わる話を、**わざわざ自分で書いた理由はそこです。ゼロから書いて中身を理解したかったのです。**

結果として、CSSトランジションの落とし穴・スワイプ速度計算・アクセシビリティのお作法など、普段触れない領域を一通り掘り下げることになりました。この記事はその記録です。

---

## 完成品の機能

実装したコンポーネントの機能一覧です：

- スワイプ（上→下）で閉じる（距離 or 速度の2段判定）
- マウスドラッグでも閉じる（デスクトップ対応）
- 背景タップで閉じる
- ESC キーで閉じる
- 開閉アニメーション（イージング・時間カスタマイズ可）
- `inert` 属性によるアクセシビリティ
- フォーカス復帰（閉じたら開く前にフォーカスしていた要素に戻る）
- 命令的 close（`ref.requestClose()`）
- `createPortal` で `document.body` 直下にレンダリング

BottomSheet 本体の実装に外部ライブラリは使っていません（ダークモード判定にプロジェクト全体で使っている `next-themes` を参照しているのみです）。

![しまなみ海道観光ガイドサイトのマップ画面。左がスポットを選択した状態、右がBottomSheetが開いて耕三寺の詳細情報が表示された状態](/images/bottomsheet-thumbnail-v2.png)
*今回ゼロから実装したBottomSheet。スポットをタップするとシートが下から滑り上がります*

:::message
実際に動いているサイトはこちらです。スポットをタップするとシートが開きます。
https://www.shimanami-guide.jp/map
:::

---

## 設計の出発点：`mounted` と `show` を分離する

アニメーションつきで「消える」コンポーネントを作るとき、素直に考えるとこうなります：

```tsx
const [open, setOpen] = useState(false)
return open ? <Panel /> : null
```

これだと `open=false` にした瞬間 DOM からパネルが消えます。**閉じるアニメーションは永遠に見えません。**

解決策は2つの状態を分けることです：

```tsx
const [mounted, setMounted] = useState(false)  // DOMに存在するか
const [show, setShow]       = useState(false)  // CSSで見えているか
```

| 状態 | mounted | show | 画面に見える |
|------|---------|------|------------|
| 閉じている | false | false | 非表示（DOMなし） |
| 開くアニメーション中 | true | true | 表示 |
| 閉じるアニメーション中 | true | false | ← translateY(100%)に向けてアニメ中 |
| アニメ終了後 | false | false | 非表示（DOMなし） |

`show=false` にして閉じるアニメーションが終わった後（`transitionend` イベントで検知）、はじめて `mounted=false` にして DOM から取り除きます。

```tsx
const onPanelTransitionEnd = () => {
  if (!showRef.current) {
    finalizeClose()  // ここで mounted=false にする
  }
}
```

---

## 本題：なぜ「double rAF」が必要なのか

ここがこの実装で一番ハマった部分であり、記事の核心です。CSS トランジションを使ったことがある方なら、一度は同じ罠を踏んでいるはずです。

「シートを開く」処理を単純に書くとこうなります：

```tsx
// ❌ アニメーションしない
setMounted(true)
setShow(true)
```

React がレンダリングしても、CSSトランジションは**起動しません。**

`translateY(100%)` → `translateY(0)` に変化させたいのですが、ブラウザから見ると「最初から `translateY(0)` な要素が突然 DOM に現れた」という扱いになります。スタート地点が存在しないのでアニメーションが発火しません。

### 1つの requestAnimationFrame では足りない

```tsx
// ❌ これも動かないことがある
setMounted(true)
setShow(false)  // translateY(100%) でDOMに挿入
requestAnimationFrame(() => {
  setShow(true) // translateY(0) に変化 → アニメ開始…のはずが動かないことがある
})
```

React のステートバッチ処理の影響で、`setMounted(true)` と `setShow(false)` が同一レンダリングにまとめられ、ブラウザが「`translateY(100%)` の状態をペイントしたフレーム」が実際には挟まっていないことがあります。

### double rAF + 強制フラッシュが解決する

```tsx
setMounted(true)
setShow(false)             // [1] translateY(100%) でDOM生成
setEnableTransition(false) // トランジション無効で挿入

requestAnimationFrame(() => {                   // [2] 1フレーム後
  panelRef.current?.getBoundingClientRect()     // ← レイアウト強制フラッシュ
  requestAnimationFrame(() => {                 // [3] さらに1フレーム後
    setEnableTransition(true)                   // トランジション有効化
    setShow(true)         // translateY(0) に変化 → アニメ確実に起動
  })
})
```

`getBoundingClientRect()` を呼ぶことでブラウザに「**レイアウト計算をいま確定しろ**」と命令します（強制リフロー）。これにより `translateY(100%)` の状態が確実にペイントされ、次のフレームで `translateY(0)` に変化したときに「スタート地点から変化している」とブラウザが認識してトランジションが起動します。

`enableTransition` フラグを別で持っているのは、DOM 挿入の瞬間にトランジションが走って一瞬乱れるのを防ぐためです。ドラッグ中も同様に `transitionProperty: "none"` にして、指の動きにパネルが即座に追従するようにしています。

```tsx
style={{
  transform: show ? "translateY(0)" : "translateY(100%)",
  transitionProperty:
    enableTransition && !draggingRef.current ? "transform" : "none",
  transitionDuration: `${duration}ms`,
}}
```

---

## スワイプ判定：距離だけでは足りない

スワイプで閉じる判定を「何px以上下にドラッグしたか」だけにすると、ゆっくりちょっとだけ引っ張って離したときに閉じません。iOSのシートと同じ操作感にするには、**速度（velocity）も見る**必要があります。

```tsx
const thresholdPx = 120        // 距離の閾値
const velocityThreshold = 0.6  // px/ms の速度閾値

const endDrag = () => {
  const dt = Math.max(1, performance.now() - lastTSRef.current) // 経過時間(ms)
  const dy = Math.max(0, lastYRef.current - dragStartYRef.current)
  const v = dy / dt  // 速度 (px/ms)

  if (dy > thresholdPx || v > velocityThreshold) {
    requestClose()  // 距離 OR 速度 のどちらかを満たしたら閉じる
  } else {
    setDragY(0)     // 閾値未満なら元の位置に戻す
  }
}
```

`performance.now()` を使うのは `Date.now()` より精度が高いためです（1ms 未満の解像度）。

「速度を見る」ことで「素早くフリック → 少ししか動いていなくても閉じる」という自然な操作感が生まれます。

マウスドラッグも同じロジックで対応しています。`window` にリスナーを付けているのは、ドラッグ中にカーソルがパネル外に出ても追従させるためです：

```tsx
onMouseDown={(e) => {
  if (e.button !== 0) return
  beginDrag(e.clientY)
  const onMove = (ev: MouseEvent) => moveDrag(ev.clientY)
  const onUp = () => {
    endDrag()
    window.removeEventListener("mousemove", onMove)
    window.removeEventListener("mouseup", onUp)
  }
  window.addEventListener("mousemove", onMove)
  window.addEventListener("mouseup", onUp)
}}
```

---

## `inert` 属性でアクセシビリティを担保する

「閉じるアニメーション中のシート」はDOMに残っています。この間にスクリーンリーダーやキーボードからアクセスされると困ります。

HTML の `inert` 属性を使うと、その要素とすべての子孫を一括で「操作不可・フォーカス不可・読み上げ不可」にできます：

```tsx
useEffect(() => {
  const overlayEl = overlayRef.current
  if (!overlayEl) return

  if (!show) {
    overlayEl.setAttribute("inert", "")  // 閉じ中は非活性
  } else {
    overlayEl.removeAttribute("inert")   // 開いているときは活性
  }
}, [show])
```

以前は `aria-hidden` + 全子孫に `tabindex="-1"` を手動で設定する必要がありましたが、`inert` 1つで同等のことができます。2023年に Firefox が対応したことで、主要ブラウザがすべて揃いました。

---

## フォーカス復帰

WAI-ARIA のダイアログパターンでは、モーダルを閉じたときに**開く前にフォーカスしていた要素にフォーカスを戻す**ことが求められています。

実装は単純で、開くタイミングで `document.activeElement` を保存しておき、閉じたときに `focus()` を呼ぶだけです：

```tsx
// 開くとき：フォーカス元を保存
const prev = document.activeElement as HTMLElement | null
if (prev && !panelRef.current?.contains(prev)) {
  restoreFocusRef.current = prev
}

// 閉じたとき（finalizeClose 内）：フォーカスを戻す
if (restoreFocusRef.current) {
  try { restoreFocusRef.current.focus() } catch {}
  restoreFocusRef.current = null
}
```

開くときは `[data-autofocus]` 属性がある要素か、フォーカス可能な最初の要素に自動フォーカスします：

```tsx
const target =
  panelRef.current?.querySelector<HTMLElement>("[data-autofocus]") ||
  panelRef.current?.querySelector<HTMLElement>(
    'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
  )
target?.focus()
```

---

## `forwardRef` + `useImperativeHandle` で命令的 close

このシートは `open` props で開閉を制御しますが、「外部から close を呼びたい」ケースがあります（例：フォーム送信完了後に閉じるなど）。

`useImperativeHandle` で `requestClose` を外部に公開します：

```tsx
export type SheetHandle = { requestClose: () => void }

const Sheet = forwardRef<SheetHandle, SheetProps>(function Sheet(props, ref) {
  const requestClose = useCallback(() => {
    setShow(false)
    closeTimerRef.current = window.setTimeout(finalizeClose, closeMs + 80)
  }, [closeMs, finalizeClose])

  useImperativeHandle(ref, () => ({ requestClose }), [requestClose])
})
```

使う側：

```tsx
const sheetRef = useRef<SheetHandle>(null)

// 任意のタイミングで
sheetRef.current?.requestClose()

return <Sheet ref={sheetRef} open={open} onClose={onClose} />
```

---

## `createPortal` でスタッキングコンテキストを脱出する

シートは `position: fixed` で画面全体に重ねる必要がありますが、コンポーネントツリーの深い場所に置かれると、親要素の `transform` や `overflow: hidden` によってクリッピングされることがあります。

`createPortal` で `document.body` 直下にレンダリングすることでこれを回避します：

```tsx
const overlayRootRef = useRef<HTMLElement | null>(null)
useEffect(() => {
  overlayRootRef.current = document.body  // SSR後に取得
}, [])

if (!mounted || !overlayRootRef.current) return null

return createPortal(overlay, overlayRootRef.current)
```

`useEffect` 内で取得しているのは SSR（サーバーサイドレンダリング）時に `document` が存在しないためです。

---

## まとめ

| 問題 | 解決策 |
|------|--------|
| CSSトランジションが動かない | double rAF + `getBoundingClientRect()` 強制フラッシュ |
| 閉じアニメーションが見えない | `mounted` / `show` を2状態で管理し `transitionend` 後にDOMを消す |
| 速度フリックで閉じない | `velocity = dy / dt` の2段判定 |
| アニメ中にフォーカスが当たる | `inert` 属性で一括非活性化 |
| 閉じた後にフォーカスが迷子 | `activeElement` を保存して復帰 |
| 親の `transform` に干渉される | `createPortal` で `document.body` に脱出 |

ライブラリを使えば確かに速く実装できます。ただ、ゼロから書くと「なぜ動くのか」がわかります。特に double rAF は、知っておくと他のアニメーション実装でも繰り返し使える知識です。

実際に動いているサイトはこちらです。マップのスポットをタップするとシートが開きます。
https://www.shimanami-guide.jp/map

---

@[card](https://mojitonews.hateblo.jp/entry/custom-hooks-useismobile-usegeolocate-usecloseall)

@[card](https://mojitonews.hateblo.jp/entry/nextjs-layout-tsx-page-tsx-difference)

@[card](https://mojitonews.hateblo.jp/entry/use-callback-when-to-use)
