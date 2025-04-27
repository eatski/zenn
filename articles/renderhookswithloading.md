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
- データフェッチはそのデータを使うUIのマークアップのコードの近くで書きたい
- 関心ごとの単位で切ったコンポーネント内でデータフェッチをすることが理想

## 立ちはだかる壁　ローディング
- かといってコンポーネント内でデータフェッチする場合避けて通れない問題、それはローディング
- ローディングの表現を考えることが面倒
- 各コンポーネントが個別でデータフェッチをしてると激しいレイアウトシフトが発生する
- 関心ごとの単位とローディングしたい単位はイコールじゃない。大抵ローディングが大きい。 
- レイアウトシフトについて考慮すべきなのは大抵、ページなどの大きな粒度のコンポーネントであり、子が考えることではない。

## Suspseneによる対処
- それに対処できる代表的な手段がSuspense
- Suspsenseによってコンポーネントのローディングの問題を親に丸投げすることができるようになった

### 問題
- Suspenseを使ったローディング時promiseをthrowするタイプのデータフェッチは、ウォーターフォール問題を起こす
- Suspenseが300ms待たないとfallbackが終わらない https://github.com/facebook/react/issues/31819
- 正直まだまだプロダクションで使うの厳しいかもね

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

- 使う側にローディングの責務を移譲できる
- シンプルなJS関数として実装できる
- https://qiita.com/uhyo/items/cb6983f52ac37e59f37e
