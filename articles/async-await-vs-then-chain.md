---
title: "async/awaitと.then()チェーンの使い分け——同じプロジェクトで両方使う理由"
emoji: "🔗"
type: "tech"
topics: ["javascript", "react", "nextjs", "typescript"]
published: false
canonical_url: "https://mojitonews.hateblo.jp/entry/async-await-then-chain-usage-map-site"
---

JavaScriptの非同期処理を書くとき、async/awaitと.then()チェーンのどちらを使うか迷ったことはありませんか？

「async/awaitの方がモダンだから常にそっちを使う」と思っていた時期が私にもありました。しかし実際にしまなみ海道の観光サイトを開発していると、同じプロジェクトの中で両方を意図的に使い分けている箇所が出てきました。この記事ではその理由を整理します。

---

![しまなみ海道観光マップ——ホテルのピンをタップすると吹き出し型のポップアップが表示され（左）、詳細を見るボタンからボトムシートが開いてホテルの詳細情報が表示されているスマートフォン画面（右）](/images/hotel-map-popup.png)
*ピンをタップすると吹き出し型のポップアップが表示され（左）、詳細を見るボタンからボトムシートが開いてホテルの空室確認や詳細情報を確認できます（右）*

## まず基本の比較

同じ処理を両方の書き方で並べます。

**.then()チェーン**

```js
fetch("/data/hotels.json")
  .then((res) => res.json())
  .then((data) => {
    console.log(data);
  })
  .catch((e) => {
    console.error(e);
  });
```

**async/await**

```js
async function loadHotels() {
  try {
    const res = await fetch("/data/hotels.json");
    const data = await res.json();
    console.log(data);
  } catch (e) {
    console.error(e);
  }
}
```

どちらも同じ処理です。読みやすさの点ではasync/awaitの方が「上から下へ順番に読める」ため、多くの場面で優れています。

---

## async/awaitを使った例：useHotelsData

地図上にホテルのピンを表示するために、`/data/hotels.json` をフェッチするカスタムフック `useHotelsData` を書きました。

```ts
// src/hooks/map/useHotelsData.ts
useEffect(() => {
  let cancelled = false;

  (async () => {
    try {
      setLoading(true);

      const [hotelRes, validRes] = await Promise.all([
        fetch("/data/hotels.json", { cache: "no-store" }),
        fetch("/data/valid-image-ids.json", { cache: "no-store" }),
      ]);

      if (!hotelRes.ok) throw new Error(`HTTP ${hotelRes.status}`);

      const json = await hotelRes.json();
      let parsed = parseHotels(json);

      if (validRes.ok) {
        const validData = await validRes.json();
        const validSet = new Set(validData.hotelNos);
        parsed = {
          ...parsed,
          features: parsed.features.filter((f) =>
            validSet.has(f.properties.hotelNo)
          ),
        };
      }

      if (!cancelled) setHotels(parsed);
    } catch (e) {
      if (!cancelled) setError("読み込みに失敗しました");
    } finally {
      if (!cancelled) setLoading(false);
    }
  })().catch(() => {});

  return () => { cancelled = true; };
}, []);
```

`await Promise.all([...])` で2つのフェッチを並列実行し、その後に絞り込み処理をしています。処理が直線的に流れるので、async/awaitが読みやすい場面です。

---

## .then()チェーンを使った例：useHotelDetail

一方、マップ上でホテルをタップしたとき詳細データをオンデマンド取得する `useHotelDetail` では、useEffectの中で.then()チェーンを使っています。

```ts
// src/hooks/hotels/useHotelDetail.ts
useEffect(() => {
  // ...前略（キャッシュ確認・AbortController設定）

  const p = existing ?? fetchDetail(normalizedNo, signal);
  if (!existing) inFlight.set(normalizedNo, p);

  p.then((data) => {
      if (latestNoRef.current !== normalizedNo) return;
      memoryCache.set(normalizedNo, data);
      setDetail(data);
      setLoading(false);
    })
    .catch((e: unknown) => {
      if (signal.aborted) return;
      if (latestNoRef.current !== normalizedNo) return;
      setError(e instanceof Error ? e : new Error(String(e)));
      setLoading(false);
    })
    .finally(() => {
      if (!existing) inFlight.delete(normalizedNo);
    });

  return () => controller.abort();
}, [normalizedNo]);
```

useEffectのコールバックは「クリーンアップ関数をreturnする」という役割があります。async関数は必ずPromiseを返すため、useEffectのコールバックに直接asyncを付けるとクリーンアップ関数を返せなくなるという問題があります。

```ts
// ❌ これはできない
useEffect(async () => {
  await fetchDetail(normalizedNo, signal);
  return () => controller.abort(); // Promiseに包まれてしまいクリーンアップとして機能しない
}, [normalizedNo]);
```

`useHotelsData` ではIIFE（即時実行関数）でasyncを閉じ込めることで回避しています。しかし `useHotelDetail` では事情が異なります。`p` は `inFlight.get()` から取り出した「すでに走っているかもしれないPromise」です。IIFEで `await p` と書くこともできますが、その場合 `.finally()` で `inFlight.delete()` するタイミング管理が複雑になります。`.then().catch().finally()` のメソッドチェーンにすることで、キャッシュへの書き込み・エラー処理・inFlightの削除がどこで行われているかが一目瞭然になりました。

---

## 判断基準をまとめると

| 状況 | 推奨 |
|---|---|
| 処理が直線的に流れる（順番にawaitするだけ） | async/await |
| try/catch/finallyで複数のエラーを整理したい | async/await |
| useEffectの中で書きたいが処理が単純 | IIFE + async/await |
| すでにPromise変数が手元にある | .then()チェーン |
| useEffectの外でPromiseを共有・合流させたい | .then()チェーン |
| .finally()だけ使いたい（クリーンアップ処理） | .then()チェーンの末尾に.finally() |

---

## まとめ

async/awaitは「読みやすい」という点でほとんどの場面で優れています。ただし、useEffectの制約（クリーンアップをreturnする必要がある）や、すでにPromiseとして存在している変数に処理を繋ぎたい場面では、.then()チェーンの方が構造に合うことがあります。

「常にasync/await」ではなく、処理の形に合わせて選ぶ——それが判断基準です。

:::message
この記事は[はてなブログ](https://mojitonews.hateblo.jp/entry/async-await-then-chain-usage-map-site)でも公開しています。
:::