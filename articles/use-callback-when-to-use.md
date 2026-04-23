---
title: "useCallback が必要な場面・不要な場面——「とりあえず入れとく」はパフォーマンスを悪化させる"
emoji: "🎯"
type: "tech"
topics: ["react", "nextjs", "typescript", "javascript", "hooks"]
published: true
canonical_url: "https://mojitonews.hateblo.jp/entry/use-callback-when-to-use"
---

しまなみ海道の観光サイト勉強も兼ねて作り始めたとき、カスタムフックを書くたびに `useCallback` を巻いていました。

「関数の再生成を防ぐんでしょ？　入れておいて損はないよね」——そう思っていました。

実際には、不要な `useCallback` はコードを読みにくくするだけでなく、パフォーマンスをわずかに悪化させることもあります。どこで使うべきで、どこでは要らないのかを整理しました。

## useCallback がやっていること

```ts
const handleClick = useCallback(() => {
  doSomething();
}, []);
```

`useCallback` はメモ化された関数を返します。依存配列の中身が変わらない限り、前回レンダリング時と同じ関数参照を返し続けます。

逆に言うと、それだけです。関数の中身を「高速化」するわけではありません。

## 入れなくていい場面

### 1. ネイティブ DOM 要素への onClick

```tsx
// ❌ useCallback は不要
const handleClick = useCallback(() => {
  setIsOpen(true);
}, []);

return <button onClick={handleClick}>開く</button>;
```

`<button>` などのネイティブ要素は `React.memo` で囲まれていません。関数参照が毎回変わっても再レンダリングは起きません。

```tsx
// ✅ これで十分
return <button onClick={() => setIsOpen(true)}>開く</button>;
```

### 2. コンポーネント内だけで使う関数

```tsx
// ❌ 不要
const formatLabel = useCallback((name: string) => {
  return `${name}（スポット）`;
}, []);
```

他のコンポーネントに渡さない、`useEffect` の依存にもならない関数は、メモ化しても意味がありません。

## 入れる意味がある場面

### 1. React.memo で囲んだ子コンポーネントへ渡すとき

```tsx
const Child = React.memo(({ onClose }: { onClose: () => void }) => {
  return <button onClick={onClose}>閉じる</button>;
});

function Parent() {
  // ❌ useCallback なし → 毎レンダリングで新しい関数参照 → memo が意味をなさない
  const handleClose = () => setOpen(false);

  // ✅ useCallback あり → 参照が安定 → Child が不要に再レンダリングしない
  const handleClose = useCallback(() => setOpen(false), []);

  return <Child onClose={handleClose} />;
}
```

`React.memo` は props の参照で前回と比較します。関数を毎回新しく作ると、中身が同じでも「変わった」と判断されて再レンダリングが走ります。

### 2. 別の useCallback の依存配列に入れるとき

`useCallback` の依存配列に入るケースは `useEffect` だけとは限りません。別の `useCallback` から呼ばれる場合も同じ理由で参照の安定が必要になります。

このサイトの `useHotelMapClickHandler.ts` はその実例です。フック自体は `handleHotelClick` を返り値として返し、呼び出し側の `PowerMap.tsx` で別の `useCallback` の依存配列に入っています。

```tsx
// PowerMap.tsx
const handleClick = useCallback(
  (e) => {
    handleHotelClick(e); // ← useHotelMapClickHandler が返した関数
  },
  [activeLayer, closeHotelPopup, handleHotelClick, mapRef], // ← 依存配列に入っている
);
```

`handleHotelClick` の参照が毎レンダリングで変わると、`handleClick` も毎回再生成されます。`handleClick` がさらに別の依存配列に入っていれば、その連鎖が続きます。参照を安定させることで、この再生成の連鎖を断ち切れます。

![useCallbackをReact.memoと組み合わせた例と、PowerMap.tsxの依存配列チェーンのコード例](/images/use-callback-memo-deps.png)
*React.memo との組み合わせ（上）と useCallback の依存配列チェーン（下）——どちらも関数参照を安定させる必要がある場面です。*

### 3. useEffect の依存チェーンが連なるとき

`Sheet.tsx`（ボトムシート）では `finalizeClose` と `requestClose` を `useCallback` で定義しています。

```tsx
const finalizeClose = useCallback(() => {
  if (showRef.current) return;
  if (closeTimerRef.current !== null) {
    window.clearTimeout(closeTimerRef.current);
    closeTimerRef.current = null;
  }
  // ...
}, [onClose]);
```

構造はこうなっています。

- `finalizeClose` → `useEffect` の依存配列に入っています
- `requestClose` → `finalizeClose` を内部で呼び、さらに別の `useEffect` の依存配列に入っています
- `requestClose` → `useImperativeHandle` にも渡しています

`useImperativeHandle` で公開しているのはその延長であって、主な理由は `useEffect` の依存チェーンの安定化です。`finalizeClose` の参照が毎回変わると、それを依存に持つ `requestClose` も再生成され、さらにその `useEffect` が毎回再実行される連鎖が起きます。

## useCallback 自体にコストがある

`useCallback` はタダではありません。

- 依存配列の値を毎レンダリングで比較する処理が走る
- 前回の関数参照をメモリに保持し続ける

関数の生成コスト（ほぼゼロに近い）より、メモ化のオーバーヘッドが上回るケースもあります。「入れておけば安全」ではなく、入れる理由がある場面でだけ使うのが正しい判断です。

## 判断フローチャート

```
この関数、useCallback 巻く？
│
├─ React.memo を使った子コンポーネントに渡す？
│   YES → 巻く
│
├─ useEffect の依存配列に入っている？
│   YES → 巻く
│
├─ 別の useCallback や useEffect の依存配列に入る？
│   YES → 巻く（依存チェーンの連鎖を断ち切る）
│
└─ 上記以外
    NO → 巻かない
```

## まとめ

| 場面 | useCallback |
|------|-------------|
| `<button onClick={fn}>` などネイティブ要素 | 不要 |
| コンポーネント内だけで使う関数 | 不要 |
| `React.memo` の子コンポーネントへ渡す | 必要 |
| `useEffect` の依存配列に入る | 必要 |
| 別の `useCallback` の依存配列に入る | 必要 |
| `useEffect` 依存チェーンの末端で `useImperativeHandle` にも渡す | 必要 |

「とりあえず入れとく」のをやめてから、カスタムフックのコードが読みやすくなりました。依存配列に `useCallback` が並んでいたら、「この関数は子や Effect から参照される重要な関数だ」という意味になります。ノイズがなくなると、コードが語れる情報量が増えます。

:::message
この記事は[はてなブログ](https://mojitonews.hateblo.jp/entry/use-callback-when-to-use)でも公開しています。
:::

@[card](https://mojitonews.hateblo.jp/entry/tanstack-query-migration-custom-cache)

@[card](https://mojitonews.hateblo.jp/entry/custom-hooks-useismobile-usegeolocate-usecloseall)
