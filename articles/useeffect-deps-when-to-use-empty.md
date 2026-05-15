---
title: "`useEffect` の第2引数（依存配列）を空にしていい場面・してはいけない場面——しまなみガイドの実装から整理した"
emoji: "🎣"
type: "tech"
topics: ["javascript", "typescript", "react", "nextjs", "hooks"]
published: true
---

## はじめに

しまなみ海道の観光ガイドサイトを個人で開発しています。カスタムフックを書くたびに `useEffect` の第2引数（依存配列）をどう書くか迷いました。

```ts
useEffect(() => {
  // 何か処理
}, []); // ← これは正しいのか？
```

「とりあえず `[]` にしとけばいい」と思っていた時期があります。実際には `[]` が正しい場面と、`[]` にしてはいけない場面があります。

このサイトで書いたカスタムフックを素材に、4パターンに整理しました。

---

## まず：依存配列とは何か

`useEffect` の第2引数は「この値が変わったときに再実行する」というリストです。

```ts
useEffect(() => {
  // 処理
}, [a, b]); // a または b が変わったときに再実行
```

- **`[]`（空）** → マウント時に1回だけ実行。以降は再実行しない
- **`[a, b]`** → `a` か `b` が変わるたびに再実行
- **省略（第2引数なし）** → 毎レンダリングのたびに実行

---

## パターン1：`[]` でいい場面——「初回だけ取得すればいい」静的データ

```ts
// src/hooks/map/useHotelsData.ts
useEffect(() => {
  let cancelled = false;

  (async () => {
    const res = await fetch("/data/hotels.json", { cache: "no-store" });
    if (!cancelled) setHotels(await res.json());
  })().catch(() => {});

  return () => {
    cancelled = true;
  };
}, []); // ← [] で正しい
```

`useHotelsData` はホテルの GeoJSON ファイルをフェッチするフックです。フェッチ対象の URL は固定で、props や state に依存しません。「マウント時に1回取得すれば十分」なケースは `[]` が正解です。

同じパターンは `useBusTimetable`（バス時刻表）でも使っています。どちらも「URL が変わることはない・ページを開いたときに1回取得すれば十分」というデータです。

**`[]` を使っていい条件：**

- effect 内で参照している値がコンポーネントの外（定数・モジュールスコープ）にある
- effect 内で参照している値が props や state ではない
- 「何かが変わったら再実行したい」という要件がない

---

## パターン2：`[依存値]` が必要な場面——「URLが変わったら取り直す」

```ts
// src/hooks/map/useMapStyle.ts
export function useMapStyle(styleUrl: string) {
  const [styleSpec, setStyleSpec] = useState<MapboxStyle | null>(null);

  useEffect(() => {
    const ac = new AbortController();
    setStyleSpec(null);
    (async () => {
      try {
        const spec = await fetchAndLocalizeStyle(styleUrl, ac.signal);
        if (!ac.signal.aborted) setStyleSpec(spec);
      } catch {
        /* noop */
      }
    })();
    return () => ac.abort();
  }, [styleUrl]); // ← styleUrl が変わるたびに再実行

  return { styleSpec };
}
```

`useMapStyle` は地図のスタイルURL（衛星モード・通常モードなど）を受け取ってスタイル定義を取得するフックです。ユーザーがレイヤーを切り替えると `styleUrl` が変わるため、`[styleUrl]` にしています。

`[]` にしていたら、スタイルを切り替えても最初のURLのスタイルを使い続けます。

また `return () => ac.abort()` でクリーンアップしています。`styleUrl` が変わって effect が再実行されるとき、前回の fetch を中断するためです。**依存配列に値を入れたら、クリーンアップも一緒に書く**のがセットです。

---

## パターン3：`[query]` が必要な場面——「設定が変わったらリスナーを張り直す」

```ts
// src/hooks/useIsMobile.ts
export function useMediaQuery(query: string) {
  const [matches, setMatches] = useState(false);

  useEffect(() => {
    if (typeof window === "undefined") return;
    const mql = window.matchMedia(query);
    const onChange = () => setMatches(mql.matches);
    onChange();
    mql.addEventListener("change", onChange);
    return () => mql.removeEventListener("change", onChange);
  }, [query]); // ← query が変わるたびにリスナーを張り直す

  return matches;
}
```

`useMediaQuery` はメディアクエリ文字列（`"(max-width: 768px)"` など）を受け取ってマッチ状態を返すフックです。

`query` が変わったとき、`[]` にしていると前の `query` に対するリスナーが残り続けます。`[query]` にすることで、`query` が変わると前のリスナーを外して（`removeEventListener`）新しい `query` のリスナーを張り直します。

---

## パターン4：複数の依存値——「渡された値が変わるたびに再セットアップ」

```ts
// src/hooks/map/useHotelMarkers.ts（抜粋）
useEffect(() => {
  if (!map) return;

  // syncOnce・各ハンドラはすべて effect 内部で定義（依存配列への追加不要）
  const syncOnce = (): void => {
    /* マーカーの差分更新処理 */
  };
  const onMoveEnd = (): void => {
    syncOnce();
  };
  const onIdle = (): void => {
    syncOnce();
  };

  map.on("moveend", onMoveEnd);
  map.on("idle", onIdle);

  return () => {
    map.off("moveend", onMoveEnd);
    map.off("idle", onIdle);
    // マーカーを全削除
  };
}, [map, sourceId, idProp, thumbProp, nameProp, onPinClick]);
```

`useHotelMarkers` はホテルのマーカーを地図に表示するフックです。`map`（Mapboxのインスタンス）・`sourceId`（データソースID）・各種設定を受け取ります。これらのどれかが変わったら、リスナーを外してマーカーを削除し、新しい設定で再セットアップが必要です。

`onMoveEnd` / `onIdle` / `syncOnce` は effect の**内部で定義**しているため依存配列には含めません。依存配列に書くのは「**effect の外から来た値**」だけです。effect 内部で作った変数・関数は毎回新しく作られるので、依存配列に入れる意味がありません。

---

## なぜ `[]` にしてはいけない場面があるのか：stale closure 問題

`[]` を誤って使うと「stale closure（古い値を参照し続ける）」というバグが起きます。

```ts
// ❌ よくある間違い
function SearchBox({ query }: { query: string }) {
  const [result, setResult] = useState(null);

  useEffect(() => {
    fetch(`/api/search?q=${query}`) // query を参照しているのに
      .then((r) => r.json())
      .then(setResult);
  }, []); // ← [] にしている → 最初の query 値を永遠に使い続ける
}
```

この例では `query` が変わっても effect が再実行されないため、初回マウント時の `query` 値でフェッチし続けます。

正しくは：

```ts
// ✅ 依存配列に query を入れる
useEffect(() => {
  fetch(`/api/search?q=${query}`)
    .then((r) => r.json())
    .then(setResult);
}, [query]); // query が変わるたびにフェッチする
```

---

## ESLint の `react-hooks/exhaustive-deps` を使う

どの値を依存配列に入れるべきか、人間が毎回判断するのは難しいです。ESLint の `react-hooks/exhaustive-deps` ルールを有効にすると、「依存配列に入れるべき値が漏れている」ときに警告してくれます。

Next.js 15 はデフォルトで flat config（`eslint.config.mjs`）を採用しており、`next/core-web-vitals` を継承するだけで `react-hooks/exhaustive-deps` が**すでに有効**になっています。

```js
// eslint.config.mjs（Next.js 15 のデフォルト）
import { FlatCompat } from "@eslint/eslintrc";

const compat = new FlatCompat({ baseDirectory: __dirname });

export default [...compat.extends("next/core-web-vitals", "next/typescript")];
// ↑ next/core-web-vitals に react-hooks/exhaustive-deps が含まれる
```

追加設定なしで警告が出るので、このルールが出ているときは `// eslint-disable-line` で黙らせる前に「本当に `[]` でいいのか」を一度考える習慣にしています。

「`[]` でいい理由」が説明できるなら `[]` で問題ありません。説明できないなら、依存配列に値を追加するのが正解です。

---

## 4パターンの整理

| パターン                     | 依存配列               | 使う場面                                              |
| ---------------------------- | ---------------------- | ----------------------------------------------------- |
| 静的データのフェッチ         | `[]`                   | URL固定・マウント時1回で十分（`useHotelsData`）       |
| props が変わったら再フェッチ | `[url]`                | URLが外から渡される（`useMapStyle`）                  |
| 外部リスナーの張り直し       | `[query]`              | 設定が変わったらリスナーを付け替え（`useMediaQuery`） |
| 複数設定の再セットアップ     | `[map, sourceId, ...]` | 渡された全設定に追従（`useHotelMarkers`）             |

依存配列の選び方の判断基準は1つです。**「effect 内で外部から来た値を参照しているか」**。参照していたら依存配列に入れる。参照していなければ `[]` でいい。

---

実際に動いているサイトはこちらです。今回紹介したフックはすべてこのマップ機能の中で動いています。

https://www.shimanami-guide.jp/map

:::message
この記事は[はてなブログ](https://mojitonews.hateblo.jp/entry/useeffect-deps-when-to-use-empty)でも公開しています。
:::
