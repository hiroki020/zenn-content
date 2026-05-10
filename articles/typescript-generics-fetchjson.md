---
title: "TypeScript の Generics を実際に使いたくなった場面——汎用フェッチ関数 fetchJson<T> の設計"
emoji: "🔧"
type: "tech"
topics: ["typescript", "react", "nextjs"]
published: true
---

## はじめに

しまなみ海道の観光ガイドサイトを個人で開発しています。コードを書いていて、同じパターンが繰り返されていることに気づきました。

```ts
// useRestaurantsData.ts
let json = (await res.json()) as RestaurantsGeoJSON;

// useBusTimetable.ts
const json = (await res.json()) as BusStopTimetableByStop | null;

// HotelCard.tsx
const json = (await res.json()) as ApiDetail;

```

`fetch` → `res.ok` チェック → `.json()` → `as SomeType`。同じ 3 ステップをファイルごとに書いていました。

「これをまとめられないか」と考えたとき、TypeScript の **Generics** を初めて実用的に使うことになりました。

---

## 繰り返しているパターンを確認する

典型的なコードはこうです。

```ts
const res = await fetch("/data/hotels.json");
if (!res.ok) throw new Error(`HTTP ${res.status}`);
const json = (await res.json()) as HotelsData;
```

どのファイルでも：

1. `fetch` でリクエストを送る
2. `res.ok` が false なら例外を投げる
3. `.json()` の結果を `as SomeType` でキャストする

ステップ 1・2 は毎回まったく同じです。変わるのはステップ 3 の型だけです。

---

## Generics とは

Generics は「**型を引数として受け取る仕組み**」です。

値を渡すように、型を渡すことができます。

```ts
// 普通の関数：値を受け取る
function identity(value: string): string {
  return value;
}

// Generics を使った関数：型を受け取る
function identity<T>(value: T): T {
  return value;
}
```

`<T>` の `T` は型変数です。呼び出し時に具体的な型が決まります。

```ts
identity<string>("hello")  // T = string
identity<number>(42)       // T = number
```

TypeScript は多くの場合、値から型を推論するので `<string>` を省略できます。

```ts
identity("hello")  // T は string と推論される
```

---

## fetchJson\<T\> を実装する

この仕組みを使うと、「返す型は呼び出し側が決める」汎用 fetch 関数を作れます。

```ts
async function fetchJson<T>(url: string, options?: RequestInit): Promise<T> {
  const res = await fetch(url, options);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json() as Promise<T>;
}
```

ポイントは 2 つです。

- `<T>` で型変数を受け取る
- `Promise<T>` を返すので、`await` すると型 `T` の値が得られる

---

## 使い方（before / after）

**Before**（各フックにボイラープレートが散在している状態）:

```ts
// useRestaurantsData.ts
const res = await fetch("/data/restaurants.json", {
  cache: "no-cache",
  signal: ac.signal,
});
if (!res.ok) throw new Error(`HTTP ${res.status}`);
let json = (await res.json()) as RestaurantsGeoJSON;

// HotelCard.tsx
const res2 = await fetch(`/api/rakuten/hotel-detail?hotelNo=${id}`);
if (!res2.ok) throw new Error(`HTTP ${res2.status}`);
const json2 = (await res2.json()) as ApiDetail;
```

**After**（fetchJson\<T\> を使う）:

```ts
// useRestaurantsData.ts
const json = await fetchJson<RestaurantsGeoJSON>("/data/restaurants.json", {
  cache: "no-cache",
  signal: ac.signal,
});

// HotelCard.tsx
const json2 = await fetchJson<ApiDetail>(
  `/api/rakuten/hotel-detail?hotelNo=${id}`
);
```

`if (!res.ok) throw` の繰り返しが消えます。エラーハンドリングの抜け漏れも防げます。

---

## `as T` は実行時に型を保証しない

`fetchJson<T>` を使っても、`T` の安全性は完全ではありません。

`as SomeType` も `as Promise<T>` も、TypeScript がコンパイル時にチェックするだけで、**実行時に JSON の中身を検証していません**。

```ts
// 型エラーにはならないが、実行時に壊れる可能性がある
const data = await fetchJson<{ name: string }>("...");
console.log(data.name.toUpperCase()); // JSON に name がなければ実行時エラー
```

外部 API のように「構造が変わりうる」データには、型ガードを組み合わせるのが適切です。

---

## 型ガードとの組み合わせ

このサイトの `useHotelDetail.ts` では、エラーレスポンスの解析に型ガードを使っています。

```ts
type ApiErrorJson = { error?: string; status?: number };

function isApiErrorJson(v: unknown): v is ApiErrorJson {
  if (typeof v !== "object" || v === null) return false;
  const r = v as Record<string, unknown>;
  return (
    r.error === undefined ||
    typeof r.error === "string" ||
    r.status === undefined ||
    typeof r.status === "number"
  );
}

async function fetchDetail(
  hotelNo: number,
  signal?: AbortSignal,
): Promise<HotelDetail> {
  const res = await fetch(`/api/rakuten/hotel-detail?hotelNo=${hotelNo}`, { signal });

  if (!res.ok) {
    let msg = `HTTP ${res.status}`;
    try {
      const j: unknown = await res.json();
      if (isApiErrorJson(j) && typeof j.error === "string") msg = j.error;
    } catch {
      /* noop */
    }
    throw new Error(msg);
  }

  const data: unknown = await res.json();
  return data as HotelDetail;
}
```

`const j: unknown` で受け取り、型ガードで絞り込んでから使う。これが型安全な fetch の正しい形です。

`v is ApiErrorJson` という返り値の型が型ガードの構文です。`true` を返すと、その後のスコープで `v` が `ApiErrorJson` として扱われます。

---

## どちらを使うか

| データの種類 | 推奨パターン |
|------------|------------|
| 自分で管理している内部 JSON（`/data/hotels.json` など） | `fetchJson<T>` + as キャスト |
| 外部 API（構造が変わりうる） | `unknown` + 型ガード関数 |
| エラーレスポンスの詳細を取得したい | `unknown` + 型ガード関数 |

自分で管理している JSON は、型が壊れることはほぼありません。`fetchJson<T>` での as キャストで十分です。外部 API は型ガードで実行時の検証を加えるのが安全です。

---

## まとめ

- `fetch` → `res.ok` チェック → `.json()` → `as SomeType` という繰り返しを `fetchJson<T>` にまとめた
- `<T>` という型変数を使うことで「返す型は呼び出し側が決める」汎用関数を作れる
- `as T` は実行時の検証をしないので、外部 API には型ガード（`value is T`）を組み合わせるのが安全

Generics は「同じ構造で型だけ違う」コードが複数あるときに使うと、重複を減らしながら型を保てます。`fetchJson<T>` はその最もシンプルな例の一つです。
