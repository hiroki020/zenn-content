---
title: "Mapbox で window is not defined——Next.js の dynamic と use client の使い分け"
emoji: "🗺️"
type: "tech"
topics: ["javascript", "typescript", "react", "nextjs", "mapbox"]
published: false
---

:::message
この記事は Next.js を使い始めたばかりの**初学者向け**です。「SSR って何？」という段階から説明しています。
:::

## はじめに

しまなみ海道の観光ガイドサイトを個人で開発しています。地図機能に Mapbox（`mapbox-gl` / `react-map-gl`）を使っているのですが、最初に組み込んだとき Next.js がこんなエラーを出して動かなくなりました。

```
ReferenceError: window is not defined
```

「`window` は普通に使えるのでは？」と思いましたが、これは Next.js の SSR（サーバーサイドレンダリング）が原因でした。この記事は、なぜこのエラーが起きるのか・どう直すのかの記録です。

---

## なぜ SSR で `window is not defined` になるのか

Next.js はデフォルトで SSR を行います。サーバー（Node.js）でコンポーネントをレンダリングして HTML を生成してからブラウザに送る仕組みです。

```
ブラウザ → Next.js サーバー（Node.js）→ HTML を生成 → ブラウザで React がマウント
              ↑
          ここで一度コンポーネントが実行される
```

Node.js には `window` がありません。`window` はブラウザの API です。

`mapbox-gl` はインポートした瞬間にモジュールの初期化処理が走り、その中で `window` / `navigator` / `document` を参照します。Next.js がサーバーでこのモジュールをインポートしようとすると、`window` が存在しないためエラーになります。

```ts
import mapboxgl from 'mapbox-gl'
// ↑ これだけでサーバーサイドでは ReferenceError: window is not defined
```

これは Mapbox に限りません。`window` や `document` をモジュールレベルで参照するライブラリ全般で起きます。

---

## 解決策1：`dynamic(() => import(...), { ssr: false })`

Next.js には動的インポートをサポートする `next/dynamic` があります。`{ ssr: false }` を渡すと、そのコンポーネントはサーバーサイドでレンダリングされなくなります。

```tsx
import dynamic from 'next/dynamic'

const PowerMap = dynamic(() => import('@/components/PowerMap'), {
  ssr: false,
  loading: () => <div>地図を読み込み中...</div>,
})

export default function MapPage() {
  return <PowerMap />
}
```

`ssr: false` にすることで：

- **サーバーサイドでは** このコンポーネントを完全にスキップする
- **ブラウザでは** JavaScript が実行されてからコンポーネントが動的にロードされる

`loading` に渡したコンポーネントは、ブラウザ側でのロード中に表示されます。

### Pages Router ではこれが唯一の解

Next.js の Pages Router（`pages/` ディレクトリ）には「クライアント専用」をファイル単位で指定する仕組みがありません。ブラウザのみで動くコンポーネントを扱うには `dynamic({ ssr: false })` が必要です。

---

## 解決策2：`"use client"`（App Router の場合）

Next.js 13 以降の App Router では、ファイルの先頭に `"use client"` と書くことでそのコンポーネントをクライアント専用にできます。

```tsx
// src/components/PowerMap.tsx
'use client'

import mapboxgl from 'mapbox-gl'
import 'mapbox-gl/dist/mapbox-gl.css'
import Map from 'react-map-gl'
// ...
```

`"use client"` を書いたファイルとそこからインポートするモジュールは、すべてクライアント側でのみ実行されます。サーバーサイドには含まれないため、`window` を参照しても問題が起きません。

このプロジェクトでは `PowerMap.tsx` に `"use client"` を書くことで解決しています。

```
app/map/page.tsx          ← Server Component（fs で spots データを取得）
  └── HomeClient.tsx      ← "use client"
        └── PowerMap.tsx  ← "use client"（mapbox-gl のインポートはここで止まる）
```

Server Component である `map/page.tsx` は `HomeClient.tsx` を呼び出しますが、`"use client"` の境界以降はサーバーサイドで実行されないため、`mapbox-gl` がサーバーに持ち込まれることがありません。

---

## どちらを使うべきか

| 状況 | 解決策 |
|------|--------|
| **Pages Router**（`pages/` ディレクトリ） | `dynamic({ ssr: false })` 一択 |
| **App Router** で「このコンポーネントは常にクライアント専用」 | `"use client"` がシンプル |
| **App Router** で「Server Component の中でだけ動的にロードしたい」 | `dynamic({ ssr: false })` が有効 |
| ロード中の表示（スケルトン等）を出したい | `dynamic({ loading: ... })` が便利 |
| バンドルサイズを分割したい（コード分割） | `dynamic()` が分割単位になる |

App Router で開発しているなら、**基本は `"use client"` で対応して、コード分割やローディング状態を細かく制御したいときだけ `dynamic()` を追加**するのが実用的だと感じています。

---

## App Router でも `dynamic()` が役立つ場面

`"use client"` で解決しても、`dynamic({ ssr: false })` が役立つ場面があります。

```tsx
// Server Component の中で、特定の箇所だけ遅延ロードしたい
import dynamic from 'next/dynamic'

const HeavyChart = dynamic(() => import('@/components/HeavyChart'), {
  ssr: false,
  loading: () => <p>グラフ読み込み中...</p>,
})

export default function DashboardPage() {
  // この関数自体は Server Component のままにできる
  return (
    <div>
      <h1>ダッシュボード</h1>
      <HeavyChart />
    </div>
  )
}
```

`"use client"` はファイル単位で「このコンポーネント以下は全部クライアント」になります。一方 `dynamic()` は「この import だけ遅延・SSR無効」という細かい制御が可能です。

また、Pages Router のコードを読んだり移行したりするときに `dynamic({ ssr: false })` が頻出するので、意味を知っておくと読みやすくなります。

---

## まとめ

- `mapbox-gl` 等のブラウザ専用ライブラリは、SSR（Node.js）環境で `window is not defined` になる
- **Pages Router** では `dynamic(() => import(...), { ssr: false })` で SSR を無効化する
- **App Router** では `"use client"` でそのファイル以降をクライアント専用にするのがシンプル
- `dynamic()` は「コード分割」「ローディング表示」「Server Component 内での部分的な SSR 無効化」に使う

---

実際に動いているサイトはこちらです。

https://www.shimanami-guide.jp/map
