---
title: "「Render hooks with loading」パターンでReactのローディング表示を柔軟に制御する" # 記事のタイトル
emoji: "🐺" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["frontend"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

Reactでクライアントサイドのデータフェッチを行う際、ローディング表示は悩みの種です。本記事では、カスタムフックがViewとローディングの状態を返し、UI表示の制御を呼び出し元に委ねる「Render hooks with loading」パターンを提案します。

## 前提条件

このパターンは、以下の技術スタックや設計方針を採用している場合に特に有効です。

*   **React:** React 以外でも同様のパターンが適用可能なライブラリは存在しますが、React のような柔軟な実装が可能であることが望ましいです。
*   **クライアントサイドでのデータフェッチ:** SWR などのライブラリを用いて、クライアントサイドでデータフェッチを行っていること。
*   **機能的関心事を意識したディレクトリ構成:** `package by feature` や、それに類する機能的関心事を優先的に意識したディレクトリ構成を採用していること。

## 実現したいこと：データフェッチとマークアップの集約

機能的関心事で分割されたコンポーネントにおいて、データフェッチとそのデータを利用するマークアップを一つの関数に集約することを目指します。これには以下のメリットがあります。

*   **可読性の向上:** データフェッチとそれを利用するマークアップの関連性が明確になります。
*   **コンポーネントの利用しやすさ:** 内部実装がカプセル化され、呼び出し元は詳細を知らなくても UI を呼び出せます。
*   **テスタビリティの向上:** データフェッチとマークアップが結合した、よりプロダクション環境に近い状態でテストを実行できます。

### 素直な実装例

例えば、前述した方針に従いコンポーネントを素直に実装すると、以下のようになるでしょう。

```tsx
import React from 'react';
import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(res => res.json());

interface Post {
  id: number;
  title: string;
}

const PostsList: React.FC = () => {
  const { data: posts, error, isLoading } = useSWR<Post[]>('/api/posts', fetcher);

  if (error) return <div>Failed to load posts.</div>;
  if (isLoading) return <div>Loading...</div>; // ローディングの表示

  return (
    <ul>
      {posts?.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
};

export default PostsList;
```

## ローディングの課題

クライアントサイドでデータフェッチを行う場合、ローディング時の UI 表示を考慮する必要があります。先ほどの `PostsList` のような実装では、コンポーネント自身がローディング表示の責務を持ちますが、これには以下の課題があります。

*   **UI デザインの負担:** コンポーネントごとにローディング時の UI 表現をデザイン・実装する必要があります。
*   **レイアウトシフト:** 各コンポーネントが個別にデータフェッチとローディング表示を行うと、データフェッチのタイミングによって画面全体のレイアウトが大きく変動 (レイアウトシフト) し、ユーザー体験を損なう可能性があります。

このように、機能的関心事で分割したコンポーネント単位でローディング表示を行うことには一定のコストが伴います。

## 解決策：Render hooks with loading パターン

前述の課題を解決するために、`Render hooks with loading` というパターンを提案します。これは、データフェッチとそれに対応する JSX を返すカスタムフックを作成し、ローディングの状態も併せて返すアプローチです。

```tsx
// Render hooks with loading パターンを用いたカスタムフック
const usePostsViewWithLoading = () => {
    const { data, error, isLoading } = useSWR<Post[]>("/api/posts", fetcher);

    if (error) {
        return { isLoading: false, view: <div>Error loading posts.</div> };
    }
    return {
        isLoading,
        view: !isLoading && data ? (
          <ul>
            {data.map(post => (
              <li key={post.id}>{post.title}</li>
            ))}
          </ul>
        ) : null
    }
}
```

この名称は筆者が考案したものです。 `Render hooks` ([参考記事](https://qiita.com/uhyo/items/cb6983f52ac37e59f37e)) に着想を得ています。`Render hooks` の特徴は、カスタムフックが JSX (View) を返すという点にあり、`Render hooks with loading` パターンもこれに当てはまっています。

### 特徴
このパターンでは、カスタムフック内でデータフェッチを行います (`useSWR`)。フックは以下のものを返します。
*   `isLoading`: データフェッチ中かどうかを示す boolean 値。
*   `view`: データフェッチ完了後に表示する JSX。ローディング中やエラー時は `null` または代替 UI を返すことがあります。

重要な点として、このフック自体は **ローディング中の UI (スピナーなど) を直接は返しません**。ローディングの状態 (`isLoading`) を返すことで、その状態をどう扱うかの判断を呼び出し元に委ねます。フックが `isLoading` のみを返し、UI 表示を行わないことで、「機能的関心事で分割したコンポーネント単位でローディング表示を行う」ことから脱却できます。

このパターンの真価は、特にページレベルのコンポーネントなど、**複数の独立したコンポーネント (それぞれが独自の `Render hooks with loading` パターンで実装される) を統合する必要がある場合**に発揮されます。呼び出し元は、これらの複数のフックから返される `isLoading` 状態を集約し、ローディング UI (例：ページ全体に単一のスピナーやスケルトンを表示) を、より大きな、適切な粒度で実装できます。結果として、コンポーネントごとにローディング UI をデザイン・実装する負担が軽減され、各コンポーネントが個別にローディング表示を行うことによるレイアウトシフトも効果的に抑制できます。

### 例

以下に、呼び出し元のコンポーネント (例: `Page`) が複数のフックから `isLoading` を受け取り、それらを集約しつつ、コンテンツの重要度に応じてローディングの扱いを分ける例を示します。

```tsx
const Page: React.FC = () => {
  // 複数のフックを呼び出して isLoading と view を受け取る
  const { isLoading: isLoadingPosts, view: postsView } = usePostsViewWithLoading();
  const { isLoading: isLoadingProfile, view: profileView } = useProfileViewWithLoading();
  // Settings は重要度が低いと仮定
  const { isLoading: isLoadingSettings, view: settingsView } = useSettingsViewWithLoading();

  // 主要コンテンツ (Posts, Profile) のローディングの状態を集約
  const isMainContentLoading = isLoadingPosts || isLoadingProfile;

  return (
    <div>
      <h1>My Page</h1>
      {/* 主要コンテンツがローディング中ならページ全体のインジケーターを表示 */}
      {isMainContentLoading ? (
        <PageLoadingIndicator />
      ) : (
        <>
          {/* 主要コンテンツの JSX を表示 */}
          <section>
            <h2>Posts</h2>
            {postsView}
          </section>
          <section>
            <h2>Profile</h2>
            {profileView}
          </section>

          {/* Settings セクションは独立してローディングの状態を管理 */}
          <section>
            <h2>Settings</h2>
            {isLoadingSettings ? <SectionLoadingIndicator /> : settingsView}
          </section>
        </>
      )}
    </div>
  );
};

export default Page;
```

この例では、`Page` コンポーネントはまず主要なコンテンツ (Posts と Profile) の `isLoading` (`isMainContentLoading`) を確認します。これらの読み込みが完了するまではページ全体のローディングインジケーターを表示します。

主要コンテンツが表示された後、`Settings` セクションは自身の `isLoadingSettings` に基づいて個別にローディング表示 (ここでは `SectionLoadingIndicator`) またはコンテンツ (`settingsView`) を表示します。これにより、重要度の低いコンテンツの読み込みが、主要なコンテンツの表示をブロックしないように制御できます。
`Render hooks with loading` パターンは、このように柔軟なローディング戦略の実装を可能にします。

## Suspense との比較

ローディングの管理には、React の `Suspense` を利用する方法も一般的です。`Suspense` を使うと、コンポーネントレベルでのローディング処理を上位のコンポーネント (`<Suspense>` でラップした部分) に委任できます。

### Suspense の課題

しかし、現状の `Suspense` をデータフェッチと組み合わせて利用する場合、以下の点が課題となる可能性があります。

*   **直列的なデータフェッチによる遅延:** `Suspense` と `use(Promise)` を組み合わせた場合、`use` は Promise が解決されるまでレンダリングを中断 (サスペンド) します。コンポーネントの構造によっては、このサスペンドの挙動が原因で複数のデータフェッチが直列的に (一つが終わってから次が始まるように) 実行されてしまい、結果として全体の表示完了までの時間が長くなる可能性があります。
*   **Suspense のスロットリング動作 (React 19 以降):** React 19 では、Suspense がトリガーされた際に、たとえデータがすぐに利用可能になったとしても、最低限の時間 (300ms) はフォールバック UI を表示し続けるスロットリング動作が導入されました。これはキャッシュされたデータや高速なレスポンスの場合に、不必要な遅延として体感される可能性があります。
    *   関連する議論: [https://github.com/facebook/react/issues/31819](https://github.com/facebook/react/issues/31819)
*   **チームへの導入:** `Suspense` と `use` を組み合わせたデータフェッチは比較的新しい概念であり、その挙動 (特にサスペンドの仕組み) をチームメンバーが正確に理解し、適切に利用するには学習コストがかかる可能性があります。
    *   一方で、 `Render hooks with loading` パターンは、基本的な概念の応用で構成されています。そのため学習コストが低く、チームへの導入が容易であると言えます。
