---
title: "自前キャッシュをTanStack Queryに移行したらコードが半分になった"
emoji: "🗂️"
type: "tech"
topics: ["typescript", "react", "nextjs", "reactquery"]
published: false
canonical_url: "https://mojitonews.hateblo.jp/entry/tanstack-query-migration-custom-cache"
---

## はじめに

![しまなみ海道観光マップのホテル詳細表示。左がポップアップ、右がボトムシートで詳細情報を表示している状態](/images/tanstack-query-hotel-detail.png)

本サイトの地図画面には、ホテルのマーカーをタップすると詳細情報をAPIから取得して表示する機能があります。

この機能、最初は自分でキャッシュを実装していました。同じホテルを2回タップしたときにAPIを叩かないように、`Map` オブジェクトを使ってメモリに保存していたのです。

動いてはいました。でも、コードを見るたびに「これ、本当に正しく動いているのか？」という不安がありました。今回 TanStack Query（React Query）に移行したら、そのコードが丸ごと消えました。

---

## 移行前のコード——自前で全部やっていた

`src/hooks/hotels/useHotelDetail.ts` の中身です。

```typescript
// メモリキャッシュ（同じホテル再閲覧の高速化）
const memoryCache = new Map<number, HotelDetail>();
// 同時リクエストの集約
const inFlight = new Map<number, Promise<HotelDetail>>();
```

ファイルのトップレベルに2つの `Map` が定義されていました。

`memoryCache` は「一度取得したホテルの詳細データを保存しておく箱」です。同じホテルをタップしたとき、まずここを見てデータがあればAPIを叩かずに返します。

`inFlight` は「今まさに進行中のリクエスト」を保存する箱です。同じホテルに対して2つのリクエストが同時に走るのを防ぐための仕組みで、「すでにリクエスト中なら、その Promise を共有する」という処理をしていました。

```typescript
export function useHotelDetail(hotelNo: number | null) {
  const [detail, setDetail] = useState<HotelDetail | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [loading, setLoading] = useState<boolean>(false);

  // 競合回避用に最新 hotelNo を保持
  const latestNoRef = useRef<number | null>(null);
  useEffect(() => {
    latestNoRef.current = hotelNo;
  }, [hotelNo]);

  useEffect(() => {
    // キャッシュ命中なら即返す
    const cached = memoryCache.get(normalizedNo);
    if (cached) {
      setDetail(cached);
      setLoading(false);
      return;
    }

    const controller = new AbortController();

    // 進行中の同一リクエストがあればそれを共有
    const existing = inFlight.get(normalizedNo);
    const p = existing ?? fetchDetail(normalizedNo, signal);
    if (!existing) inFlight.set(normalizedNo, p);

    p.then((data) => {
      if (latestNoRef.current !== normalizedNo) return; // 競合チェック
      memoryCache.set(normalizedNo, data);
      setDetail(data);
      setLoading(false);
    })
    .catch((e) => {
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
}
```

やっていることを整理すると：

1. キャッシュ（`memoryCache`）にデータがあれば即返す
2. なければ `inFlight` を確認して、進行中なら Promise を共有、なければ新しくfetch
3. 完了したら `memoryCache` に保存
4. `latestNoRef` でホテルを切り替えたときの競合（古いデータが後から届く問題）を防ぐ
5. `AbortController` でアンマウント時のリクエストをキャンセル

動いてはいましたが、コードが複雑で「この `latestNoRef` のチェック、本当に全パスで正しいか？」という不安が常にありました。

---

## インストール

```bash
pnpm add @tanstack/react-query
```

これだけです。

:::message
**2026年5月のサプライチェーン攻撃について**

2026年5月11日、`@tanstack/*` の42パッケージに悪意あるバージョンが公開されるサプライチェーン攻撃が発生しました。ただし、**`@tanstack/react-query`（および `query*` ファミリー）は対象外**であり、安全が確認されています。影響を受けたのは主に `@tanstack/router` 周辺のパッケージです。詳細は[公式ポストモーテム](https://tanstack.com/blog/npm-supply-chain-compromise-postmortem)を参照してください。
:::

---

## Step 1：QueryClientProvider をルートに追加

TanStack Query はアプリ全体で `QueryClient` を共有します。このプロジェクトでは `src/components/ClientProviders.tsx` にすべての Provider が集まっているので、ここに追加しました。

**変更前**

```typescript
import { ThemeProvider } from "next-themes";

export default function ClientProviders({ children }) {
  return (
    <ThemeProvider ...>
      {children}
    </ThemeProvider>
  );
}
```

**変更後**

```typescript
import { useState } from "react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ThemeProvider } from "next-themes";

export default function ClientProviders({ children }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 1000 * 60 * 5, // 5分
            retry: 1,
          },
        },
      }),
  );

  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider ...>
        {children}
      </ThemeProvider>
    </QueryClientProvider>
  );
}
```

**なぜ `useState` で生成するのか**

`QueryClient` をコンポーネントの外で `const queryClient = new QueryClient()` と書くと、モジュールが読み込まれたタイミングで1度だけ生成されます。`useState(() => new QueryClient())` にしておくと「コンポーネントが初回レンダリングされたときに1度だけ生成する」ことが保証されます。SSRで複数インスタンスが生まれる問題を防ぐための書き方です。

**`staleTime: 1000 * 60 * 5` とは**

「5分間はキャッシュを新鮮とみなし、再fetchしない」という設定です。ホテルの詳細情報は数分で変わるものではないので、5分はちょうどよい値でした。

---

## Step 2：useHotelDetail.ts を useQuery に書き換え

ここがメインの変更です。

**変更前**

```typescript
// ファイルトップレベルの自前キャッシュ
const memoryCache = new Map<number, HotelDetail>();
const inFlight = new Map<number, Promise<HotelDetail>>();

export function useHotelDetail(hotelNo: number | null) {
  const [detail, setDetail] = useState<HotelDetail | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [loading, setLoading] = useState<boolean>(false);
  const latestNoRef = useRef<number | null>(null);

  // ...キャッシュ確認・inFlight管理・競合チェック・AbortControllerの手動管理
}
```

**変更後**

```typescript
import { useMemo } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";

export function useHotelDetail(hotelNo: number | null) {
  const queryClient = useQueryClient();

  const normalizedNo = useMemo(() => {
    if (hotelNo == null) return null;
    const n = Number(hotelNo);
    return Number.isFinite(n) ? n : null;
  }, [hotelNo]);

  const { data: detail, isLoading: loading, error, refetch } = useQuery({
    queryKey: ["hotelDetail", normalizedNo],
    queryFn: ({ signal }) => fetchDetail(normalizedNo!, signal),
    enabled: normalizedNo != null,
  });

  const refresh = async () => {
    if (normalizedNo == null) return;
    await queryClient.invalidateQueries({ queryKey: ["hotelDetail", normalizedNo] });
    await refetch();
  };

  return {
    detail: detail ?? null,
    loading,
    error: error instanceof Error ? error : null,
    refresh,
  };
}
```

削除されたコードと、その代わりに何が担っているかの対応です。

| 削除したコード | TanStack Query が代わりに担うこと |
|---|---|
| `memoryCache = new Map()` | `queryKey` ごとに自動でキャッシュ。`staleTime` 内なら再fetchしない |
| `inFlight = new Map()` | 同じ `queryKey` への同時リクエストを自動で1本に集約 |
| `latestNoRef` | `queryKey` が変わると自動で前のクエリを無効化 |
| `AbortController` の手動管理 | `queryFn` の引数 `signal` に自動で渡される |

`fetchDetail` 関数自体も少し変わっています。移行前は第3引数 `opts` で `cache: "no-store"` を切り替える仕組みがありました。

```typescript
// 移行前
async function fetchDetail(
  hotelNo: number,
  signal?: AbortSignal,
  opts?: FetchDetailOptions, // ← refresh() 時に no-store を渡すための引数
): Promise<HotelDetail> {
  const res = await fetch(`...`, {
    cache: opts?.noStore ? "no-store" : "default",
    signal,
  });
}
```

移行後は `refresh()` が `invalidateQueries` でキャッシュを無効化するようになったため、`no-store` を渡す必要がなくなり、`opts` ごと削除できました。`signal` の受け取り方も `queryFn` の引数から受け取る形に変わっています。

```typescript
// 変更後（queryFn の引数から signal を受け取る）
queryFn: ({ signal }) => fetchDetail(normalizedNo!, signal),
```

また、`useQuery` が返す値の名前は `isLoading` ですが、既存のコンポーネントは `loading` という名前で受け取っています。分割代入時に読み替えることで、呼び出し側のコンポーネントは一切変更不要でした。

```typescript
const { data: detail, isLoading: loading, error, refetch } = useQuery({...});
```

---

## 動作確認

![DevToolsのNetworkタブ。hotel-detailへのリクエストが走り、X-Vercel-CacheがSTALEになっている](/images/tanstack-query-network-tab.png)
*X-Vercel-Cache: STALE はキャッシュはあるが期限切れと判断され、再fetchが走った状態です。次のタップでは HIT になりキャッシュから即返ります*

ブラウザの DevTools（Network タブ）を開いてホテルマーカーをタップします。`/api/rakuten/hotel-detail?hotelNo=xxxxx` へのリクエストが1回走ります。

同じマーカーをもう一度タップすると、リクエストが走りません。キャッシュから即座に表示されます。

5分後に再タップすると、もう1回リクエストが走ります。`staleTime` の5分が経過して「古いデータ」と判断されたためです。

---

## 削除できたコードの量

| | 移行前 | 移行後 |
|---|---|---|
| ファイルの行数 | 176行 | 82行 |
| トップレベルの Map | 2個 | 0個 |
| useRef | 1個 | 0個 |
| useState | 3個 | 0個 |
| useEffect | 2個 | 0個 |

半分以下になりました。そして残ったコードは「何をするか」だけが書いてあり、「どうやるか」は TanStack Query に任せています。

「動いてはいるけど不安」というコードを抱えているなら、TanStack Query への移行を検討する価値はあると思います。

---

@[card](https://mojitonews.hateblo.jp/entry/async-await-then-chain-usage-map-site)

@[card](https://mojitonews.hateblo.jp/entry/use-client-where-to-put-server-client-component)
