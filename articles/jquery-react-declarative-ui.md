---
title: "jQueryとReactの対比で学ぶ宣言的UI——本サイトの絞り込み機能で気づいた「状態を渡すだけ」の正体"
emoji: "⚛️"
type: "tech"
topics: ["react", "jquery", "nextjs", "javascript", "typescript"]
published: true
canonical_url: "https://mojitonews.hateblo.jp/entry/jquery-react-declarative-ui"
---

## はじめに

「宣言的UI」という言葉、React の入門記事でよく目にするのですが、最初はいまいちピンときませんでした。

「命令的との違い」を読んで、なるほど。と思いつつ、実際に手を動かすまでは腹落ちしていませんでした。その感覚が変わったのは、本サイトを Next.js で作り始めて、スポットの絞り込み機能を実装したときのことです。

この記事では、jQuery を知っている人に向けて「宣言的UIとは何か」を before/after の対比で整理します。

---

## jQuery で書くとしたら——「ワイが全部やる」スタイル

観光スポットをキーワードで絞り込む機能を jQuery で書くとしたら、こうなります。

```javascript
$('#spot-search').on('input', function () {
  const q = $(this).val().toLowerCase();

  // 全カードをいったん非表示
  $('.spot-card').hide();

  // キーワードに一致するカードだけ表示
  $('.spot-card').filter(function () {
    return $(this).data('name').toLowerCase().includes(q); // data-name属性から取得
  }).show();

  // カウンターを更新
  const count = $('.spot-card:visible').length;
  $('#result-count').text(count + '件');
});
```

やっていることは明確です。入力されたら、全部隠して、一致するものだけ出して、カウンターを書き換える。**手順を一行ずつ書いています。**

なお、このコードは各カード要素に `data-name="スポット名"` 属性がついている前提です。

DOM を自分の手で動かしている感覚があります。これはこれで直感的ですし、小規模なら十分に機能します。

---

## React で書いたら——「状態を渡すだけ」

同じ機能を Next.js（React）で実装したとき、コードはこうなりました。

```tsx
const [query, setQuery] = useState("");

const filtered = useMemo(() => {
  const q = query.trim().toLowerCase();
  if (!q) return spots;
  return spots.filter(
    (s) =>
      s.name.toLowerCase().includes(q) ||
      s.shortDescription?.toLowerCase().includes(q) ||
      s.tags?.some((t) => t.toLowerCase().includes(q))
  );
}, [spots, query]);

// 入力で状態を更新するだけ
<input
  value={query}
  onChange={(e) => setQuery(e.target.value)}
  placeholder="スポット名・キーワードで検索…"
/>

// あとは状態に基づいてレンダリング
{filtered.map((spot) => (
  <SpotCard key={spot.id} spot={spot} />
))}
```

最初に書いたとき、正直戸惑いました。

**「え、これだけ？ カードを hide/show しなくていいの？」**

`setQuery` で文字列を更新するだけで、`filtered` が自動的に再計算され、`SpotCard` が再レンダリングされます。自分で DOM を触るコードはどこにもありません。

`useMemo` は「`query` が変わったときだけ再計算する」という最適化ですが、本質は同じです。状態（`query`）が変われば、画面が勝手についてきます。

---

## 実際に動いている——本サイトのスポット検索機能

上のコードは、本サイトのスポット一覧ページ（[shimanami-guide.jp/spots](https://www.shimanami-guide.jp/spots)）でそのまま動いています。

ページを開くと、最初はすべてのスポットが表示されています。

![しまなみ海道の観光スポット一覧ページ。検索バーは空欄で全29件が表示されている](/images/search-before.png)
*しまなみ海道の観光スポット一覧。全29件が表示された初期状態です。*

検索バーに「尾道」と入力すると、一致するスポットだけが残ります。

![「尾道」と入力した後の絞り込み結果。12件が表示されている](/images/search-after.png)
*「尾道」と入力した瞬間に12件へ絞り込まれました。検索ボタンは配置していません。*

DOM を直接触るコードは1行もありません。`setQuery("尾道")` が走った瞬間に `filtered` が再計算され、画面が切り替わります。状態を変えただけで、これが起きています。

---

## 「あ、これが React の正体か」という瞬間

本サイトの地図コンポーネントを作っていて、ある瞬間に気づきました。

jQuery のときは**「どう動かすか」を書いていました。**  
React のときは**「どういう状態のとき、何を表示するか」を書いています。**

jQuery は手順書です。「Aをして、Bをして、Cを変える」と書きます。  
React は完成図です。「この状態のとき、画面はこう見えるべき」と宣言します。

これが**宣言的UI（Declarative UI）**の正体でした。

> **命令的（jQuery）**：「ステップ1: 非表示にする。ステップ2: 対象を表示する。ステップ3: カウンターを更新する」  
> **宣言的（React）**：「`query` がこの値のとき、画面はこう見えるべき」

---

## 宣言的UIの何がうれしいのか

**状態の管理が一元化されます。**

jQuery だと、複数の場所で DOM を操作するうちに「どれが正しい状態か」が分からなくなることがあります。カードは非表示なのにカウンターが更新されていない、といったバグが起きやすいです。

React は状態（`query`）が唯一の真実です。状態が変われば、それに依存するすべての UI が自動的に整合性を保って更新されます。

```
状態（state）
  └── filtered（派生）
        └── SpotCard の表示（UI）
        └── resultCount の表示（UI）
```

**状態を変えれば、UI は勝手についてきます。**

---

## jQuery が悪いわけじゃない

誤解してほしくないのですが、jQuery は今も現役で正解なケースがあります。

- 既存の HTML テンプレート（WordPress など）に少し動きを加えたい
- ページ遷移のある MPA（マルチページアプリ）で軽い操作だけしたい
- バンドルサイズを最小限にしたい小規模サイト

こういうときに React を持ち込むのは過剰です。jQuery は今も優秀なツールです。

ただ、**状態が複数あって、それに応じた UI が連動する**ような場面では、宣言的UIの考え方が圧倒的にコードを整理してくれます。

---

## まとめ

| | jQuery（命令的） | React（宣言的） |
|---|---|---|
| 書くもの | 「何をするか」の手順 | 「どう見えるべきか」の宣言 |
| DOM 操作 | 自分で書く | React が担う |
| 状態管理 | 散在しやすい | 一元化される |
| 向いている場面 | 既存 HTML への追加・小規模 | 状態が多い・連動する UI |

**jQueryは命令書、Reactは完成図を渡す。**

この一文が、宣言的UIを一番シンプルに表していると思います。本サイトを作りながら、私が手を動かして初めてその意味が腹落ちしました。

「Reactよくわからん」と思っていた頃の私に、この記事を渡したかったです。

---

:::message
この記事は[はてなブログ](https://mojitonews.hateblo.jp/entry/jquery-react-declarative-ui)でも公開しています。
:::

*本サイト（Next.js + TypeScript + MDX）を作りながら学んでいます。*