---
title: "`Promise.all` で複数APIを並列フェッチしつつ、1件失敗しても残りを使う設計——しまなみガイドのバス時刻表実装から"
emoji: "🔀"
type: "tech"
topics: ["javascript", "typescript", "react", "nextjs", "promise"]
published: false
canonical_url: "https://mojitonews.hateblo.jp/entry/promise-all-partial-success-bus-timetable"
---

## はじめに

しまなみ海道の観光ガイドサイトを個人で開発しています。マップ上にバス停を表示し、タップすると時刻表を見られる機能を実装したとき、`Promise.all` の使い方でひとつ判断が必要になりました。

このサイトはしまなみ海道エリアをカバーするため、複数のバス事業者の時刻表データを扱います。

- 中国バス
- 尾道バス
- 井笠バス
- 鞆鉄バス

4事業者それぞれの JSON ファイルを読み込んで、ひとつにマージして使います。このとき「4件を並列で取得したいが、1件が404でも残り3件は表示したい」という要件がありました。

素直に `Promise.all` を書くと、1件でも失敗した瞬間に全体がエラーになります。この記事はその問題をどう解決したかの記録です。

![しまなみ海道観光ガイドサイトのマップ画面。左側はバス停を非表示にした状態、右側はバス停ピンが表示された状態。](/images/promise-all-partial-success-bus-timetable.png)
_左：バス停レイヤーOFF、右：バス停レイヤーON。4事業者のデータを並列フェッチし、取得できた分だけマージして表示している。_

---

## `Promise.all` の「1件失敗で全滅」問題

まず基本動作の確認です。`Promise.all` は渡したすべての Promise が成功したときだけ解決します。

```ts
const results = await Promise.all([
  fetch("/data/chugokubus/stop-times.json").then((r) => r.json()),
  fetch("/data/onomichibus/stop-times.json").then((r) => r.json()),
  fetch("/data/ikasabus/stop-times.json").then((r) => r.json()),
  fetch("/data/tomotetsubus/stop-times.json").then((r) => r.json()),
]);
```

4事業者のうち1事業者のファイルがまだ存在しなかった（404）場合、`Promise.all` 全体が reject されます。残り3事業者のデータも捨てられます。

今回はデータ追加を順次進めている途中で、まだファイルが存在しない事業者があります。「404のものは警告だけ出してスキップ、存在するものだけマージ」という動作が必要でした。

---

## 解決策：内側で catch して `null` を返す

解決策はシンプルです。**各 fetch を個別の try/catch で囲み、失敗したら `null` を返す**。`Promise.all` には「絶対に reject しない Promise」だけを渡します。

```ts
const results = await Promise.all(
  STOP_TIMES_SOURCES.map(async (src) => {
    try {
      const res = await fetch(src.url, { cache: "force-cache" });

      if (res.status === 404) {
        console.warn(`[useBusTimetable] not found: ${src.label}`);
        return null; // ← 失敗ではなく null を返す
      }

      if (!res.ok) {
        console.warn(`[useBusTimetable] HTTP ${res.status}: ${src.label}`);
        return null;
      }

      const json = (await res.json()) as BusStopTimetableByStop | null;
      if (!json || typeof json !== "object") {
        console.warn(`[useBusTimetable] invalid JSON: ${src.label}`);
        return null;
      }

      return json; // ← 成功したものだけ返る
    } catch (e) {
      console.warn(`[useBusTimetable] error: ${src.label}`, e);
      return null; // ← ネットワークエラーなどもここで吸収
    }
  }),
);
// results の型: (BusStopTimetableByStop | null)[]
// Promise.all 自体は絶対に reject しない
```

このパターンの肝は「`Promise.all` の外側に reject を漏らさない」ことです。各 async 関数が成功なら `json`、失敗なら `null` を返すことで、`results` は必ず `(T | null)[]` になります。

---

## null を除いてマージする

`results` には成功した事業者のデータと `null` が混在しています。`null` を飛ばしながらマージします。

```ts
const merged: BusStopTimetableByStop = {};

for (const json of results) {
  if (!json) continue; // null はスキップ
  for (const [stopId, record] of Object.entries(json)) {
    merged[stopId] = record;
  }
}

const hasAnyData = Object.keys(merged).length > 0;

if (hasAnyData) {
  setBusTimetables(merged);
  setError(null);
} else {
  // 全事業者が失敗したときだけエラー表示
  setBusTimetables(null);
  setError("バス時刻表データの読み込みに失敗しました");
}
```

「全事業者が失敗した」ときだけ `error` をセットし、1件でも成功していればユーザーにはエラーを見せません。

:::message
上記のマージ処理は「事業者間で `stopId` が重複しない」ことを前提にしています。今回扱うバス停IDは事業者ごとに名前空間が分かれているため後勝ちでも問題が起きませんが、IDが衝突しうるデータに同じパターンを適用する場合は、事業者プレフィックスを付けるか、配列で持つかといった工夫が必要になります。
:::

---

## `Promise.allSettled` との違い

同じことを `Promise.allSettled` でも書けます。

```ts
const results = await Promise.allSettled(
  STOP_TIMES_SOURCES.map((src) => fetch(src.url).then((r) => r.json())),
);

for (const result of results) {
  if (result.status === "fulfilled") {
    // result.value を使う
  } else {
    // result.reason がエラー
  }
}
```

`Promise.allSettled` は ES2020 で追加されたメソッドで、全件が settle（成功または失敗）するまで待ち、`{ status: 'fulfilled' | 'rejected', value/reason }` の配列を返します。

今回 `Promise.allSettled` を使わなかった理由は2つです。

**1つ目：ステータスコード単位の処理を細かく書きたかった。**
`Promise.allSettled` の `rejected` はネットワークエラーやパースエラーをひとまとめにします。404 だけ `warn` で通過させて他のエラーコードは別処理、という分岐を書くには内側 catch のほうが素直でした。

**2つ目：型が少し手間になる。**
`PromiseSettledResult<T>` を受け取る場合、`fulfilled` かどうかをナローイングしてから `value` にアクセスする必要があります。内側 catch で `T | null` に統一しておくほうが後続処理がシンプルになりました。

どちらが正解というわけではありません。「fetch の成否だけを見る場合」なら `Promise.allSettled` のほうが意図が明確です。「ステータスコードごとに細かく分岐したい場合」は内側 catch パターンのほうが書きやすいと感じました。

---

## このパターンを使うべきでない場面

「部分的成功を許す」設計は、すべての場面で正しいわけではありません。次のようなケースでは、むしろ「1件でも失敗したら全体を止める」素の `Promise.all` のほうが適切です。

- **決済前のバリデーション**：在庫チェック・与信チェック・配送可否チェックを並列で投げるような処理。1件でも失敗していれば、その先に進んではいけません。
- **依存関係のあるAPI呼び出し**：取得した4件のデータを掛け合わせて1つの結果を計算する処理。欠損があれば計算自体が成立しないので、部分的成功には意味がありません。
- **トランザクション的な書き込み**：複数のAPIに「保存」「更新」を並列で投げる処理。一部だけ成功すると整合性が壊れるため、失敗時にはむしろロールバックを検討すべきです。

今回のバス時刻表は「事業者ごとに独立した表示データで、欠けても残りで価値が出る」ものだったので、部分的成功を許す設計が噛み合いました。設計を選ぶときは「欠損があってもユーザーに価値を返せるか」を基準に判断しています。

---

## クリーンアップ：`cancelled` フラグ

フックのクリーンアップも必要です。コンポーネントがアンマウントされた後に `setState` を呼ぶと React が警告を出します（React 18 以降は警告なしですが、後続処理が走り続けるのは無駄です）。

```ts
useEffect(() => {
  let cancelled = false;

  (async () => {
    try {
      const results = await Promise.all(/* ... */);

      if (cancelled) return; // アンマウント済みなら setState しない

      // マージ・setState の処理
    } catch (e) {
      // fetch エラーは内側 try/catch で null を返すので、
      // ここに来るのは設計バグや setState 周りの想定外例外
      console.error("[useBusTimetable] unexpected error:", e);
      if (!cancelled) {
        setBusTimetables(null);
        setError("バス時刻表データの読み込みに失敗しました");
      }
    }
  })().catch(() => {});
  // outer try/catch が先にエラーを拾うため、
  // 末尾の .catch(() => {}) は「万が一 outer try/catch すら抜けた場合」の最後の安全網

  return () => {
    cancelled = true; // クリーンアップ時にフラグを立てる
  };
}, []);
```

`AbortController` を使ってフェッチ自体をキャンセルする方法もあります。今回は静的 JSON ファイルのフェッチで、キャンセルの恩恵が小さいため `cancelled` フラグだけにしています。重い API コールや長時間のストリームには `AbortController` が適しています。

---

## `cache: 'force-cache'` で2回目以降を高速化

```ts
const res = await fetch(src.url, { cache: "force-cache" });
```

`force-cache` はブラウザのHTTPキャッシュを最大限に利用するオプションです。一度ダウンロードした JSON はキャッシュから返るため、マップを開き直しても再ネットワーク通信が走りません。

バス時刻表は日次更新がなく、デプロイ時にのみ変わるファイルなので `force-cache` が適切です。リアルタイム性が必要なデータには `no-store` や `no-cache` を使います。

:::message
ここで触れている `cache` オプションは `useEffect` 内で実行される**クライアント側 `fetch`** の話で、ブラウザの HTTP キャッシュに乗ることを期待しています。Next.js 15 ではサーバー側 `fetch` のデフォルトキャッシュ挙動が変わっており、サーバーコンポーネント内の `fetch` とは挙動が異なる点には注意してください。
:::

---

## 全体像

最終的なフックの構造をまとめるとこうなります。

```
useEffect
  └── async IIFE
        └── Promise.all([
              fetch A → try/catch → T | null,
              fetch B → try/catch → T | null,
              fetch C → try/catch → T | null,
              fetch D → try/catch → T | null,
            ])
            ↓ (T | null)[] が必ず返る
            null を除いてマージ
            ↓
            1件以上成功 → setBusTimetables(merged)
            全件失敗   → setError("失敗しました")
  return () => { cancelled = true }
```

呼び出し側はシンプルです：

```ts
const { busTimetables, error } = useBusTimetable();
```

`busTimetables` が `null` でなければマップのバス停タップ時に時刻表を表示し、`null` のときはバス停を非表示にします。

---

## まとめ：今回の判断基準

今回の実装で得た判断軸を3点に整理しておきます。

- 並列フェッチで**部分的成功を許したい**なら、`Promise.all` の内側で catch して `null` を返すのが素直。外側に reject を漏らさないのがポイント。
- ステータスコードごとに挙動を分けたいなら、`Promise.allSettled` よりも内側 catch のほうが書きやすい。「fetch の成否だけ見れば十分」なら `Promise.allSettled` のほうが意図が明確。
- クリーンアップは `cancelled` フラグで十分なケースが多い。重い処理や長時間のストリームのときだけ `AbortController` を検討する。

どのパターンを選ぶかは「欠損があってもユーザーに価値を返せるか」「失敗をどの粒度で扱いたいか」で決まります。まずはこの2点を自問してから書き始めると、後から設計を直す手戻りが減らせると感じました。

---

実際に動いているものはこちらで確認できます。マップ右上のレイヤー切替でバス停を有効にするとバス停ピンが表示されます。

https://www.shimanami-guide.jp/map

:::message
この記事は[はてなブログ](https://mojitonews.hateblo.jp/entry/promise-all-partial-success-bus-timetable)でも公開しています。
:::
