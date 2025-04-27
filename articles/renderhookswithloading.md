---
title: "Render hooks with loading のすすめ" # 記事のタイトル
emoji: "🐺" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["frontend"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

## この記事
- Render hooks with loading という設計パターンを推奨したい

## 背景にある設計思想
- データフェッチ及びその状態管理のコードはそのデータを使うUIのマークアップのコード、適切な粒度の1つのContainerコンポーネントとして1つにまとめたい
  - 可読性: データフェッチとマークアップの関係性がわかりやすくなる
  - テスタビリティ: データフェッチとマークアップが結合した状態をテストできる

## 立ちはだかる壁　ローディング
- かといってコンポーネント内でデータフェッチする場合避けて通れない問題、それはローディング
  - ローディング状態の表現を考えることが面倒
  - 各コンポーネントが個別でデータフェッチをしてると激しいレイアウトシフトが発生する
- 関心事の分割単位とローディングしたい領域の単位はイコールじゃない。大抵ローディングの方が大きい。 
  - レイアウトシフトについて考慮すべきなのはページなどの大きな粒度のコンポーネントであり、子が考えることではない。


## そこで Render hooks with loading

```tsx
const usePostsViewWithLoading = () => {
    const {isLoading,data} = useSWR("/api/posts",fetcher);
    return {
        isLoading,
        view: data ? <PresentationalPosts posts={data} /> : null
    }
}
```

- どういうもの？
  - https://qiita.com/uhyo/items/cb6983f52ac37e59f37e の1つの形
  - ローディングの状態はisLoadingとしてbooleanで返す。このhooksにおけるviewはローディングの状態について考慮しない
- メリット
  - 使う側にローディングの責務を移譲できる
  - シンプルなJS関数として実装できる


## Suspseneによる対処
- 他の代表的な手段がSuspense（というかこっちが一般的）
- Suspsenseによってコンポーネントのローディングの問題を親に丸投げすることができるようになった

### 問題
- ただ以下の問題がありそう
  - Suspenseを使ったローディング時promiseをthrowするタイプのデータフェッチは、ウォーターフォール問題を起こす
  - https://github.com/facebook/react/issues/31819
- 正直まだまだプロダクションで使うの厳しいかもね
