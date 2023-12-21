---
title: "React関数の中でReactコンポーネントを定義しない" # 記事のタイトル
emoji: "⚠️" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["frontend","react"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

## 1行で
Reactの関数の中で別のReactコンポーネントを定義しないように気をつけましょう。

```tsx:やってはいけない実装
const Hoge: React.FC<{ username: string }> = ({ username }) => {
  // FugaはHoge FCの中で定義されている。
  const Fuga: React.FC<React.PropsWithChildren<{}>> = ({ children }) => (
    <div>
      <div>こんにちは、{username}さん！</div>
      <div>{children}</div>
    </div>
  );
  return (
    <Fuga>
      <SomeLargeContent />
    </Fuga>
  )
};
```

## FCの中で別のFCを定義することによる問題とその調査

### 何が起こったのか。
上の例の`<SomeLargeContent />`の中にて、DOMが*ユーザー入力によるstateの更新の度に*削除と再構築をしてしまい、パフォーマンスやユーザビリティの劣化等、予期せぬ挙動を引き起こしました。

クリティカルな現象としては以下が挙げられると思います。

- `transition`を使用したアニメーションを定義したが、stateの変更によるスタイルの切り替えをしようとする度にDOMが再構築されるのでアニメーションが発生しない。
- inputにユーザーが入力を行うとinputのDOMが再構築されてフォーカスがinputから外れる。
- 子コンポーネントのuseEffectがReact関数が実行される度に実行されてしまう。

### そもそも通常の場合は何故DOMの再構築が行われないのか
Reactは関数によって生成された仮想DOMを前回生成された仮想DOMと比較し、その差分のみDOMを更新・削除・追加しています。なので、仮想DOMがReactに前回の仮想DOMと同一と判断されたDOMについては再構築が行われません。
[参考記事](https://qiita.com/muraikenta/items/21eb5b2e0d1c7c95e3b8)

### では、この場合はどうだったのか。
Reactの仮想DOMオブジェクトをのぞいてみましょう。

```tsx
  const Fuga: React.FC<React.PropsWithChildren<{}>> = ({ children }) => (
    <div>
      <div>こんにちは、{username}さん！</div>
      <div>{children}</div>
    </div>
  );
  const vdom = (
    <Fuga>
      <SomeLargeContent />
      {/* stateの更新用にチェックボックスを配置 */}
      <input
        type="checkbox"
        checked={state}
        onChange={() => setState(s => !s)}
      />
    </Fuga>
  );
  console.log(vdom)//仮想DOMを出力;
```
以下
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/294714/01d2d05f-a109-37cd-62de-e69aaa622024.png)

`key,props,type`というプロパティを持っていることがわかります。
typeに注目してみましょう。**Fuga関数が格納されています。**
仮想DOMは*「どのようなDOMが構築されるか」*以外にも*「どのコンポーネントからDOMが生成されたのか」*をtypeとして保持するようになっており、Reactはその差分も検出しています。
つまり、最終的に構築される実DOMに差分が無かったとしても、**DOMを構築するコンポーネントが異なっていれば、それは異なるDOMツリーだと解釈されます**
FC(1)の中でFC(2)を定義した場合、**FC(2)はrenderが走る度に定義される**ので、前回のrenderと異なる仮想DOMを生成してしまいます。
これにより、実DOMの差分以上にDOMの再構築が発生してしまうのです。

## まとめ
コンポーネントを定義するときは必ず、トップレベルに定義することをお勧めします。
ルートに近いコンポーネントがこれをやらかすとアプリ全ての末端のコンポーネントまで被害を被りますので（実体験）気をつけましょう。

## おまけ：仮想DOMをハックしてDOMの再構築を食い止める
Reactは仮想DOMオブジェクトのtypeから仮想DOMが異なるコンポーネントが作られているか判断します。
これを利用しtypeをハックすることでReactに差分を誤認識させ、再構築を食い止めてみましょう。

```tsx
  const vdom = (
    <Fuga>
      <SomeLargeContent />
      {/* stateの更新用にチェックボックスを配置 */}
      <input
        type="checkbox"
        checked={state}
        onChange={() => setState(s => !s)}
      />
    </Fuga>
  );
  console.log(vdom);
  //初回に定義されたFugaコンポーネントをrefに格納しておく
  const ref = React.useRef<React.FC | null>(null);
  const firstFuga = ref.current;
  if (!firstFuga) {
    ref.current = Fuga;
  }
  //2回目以降のrenderではtypeを初回に格納したFuga関数に差し替える
  return firstFuga
    ? {
        ...vdom,
        type: firstFuga
      }
    : vdom;

```

絶対に真似しないように。





