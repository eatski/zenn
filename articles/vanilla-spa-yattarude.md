---
title: "Vanilla JSでSPAやったるで" # 記事のタイトル
emoji: "😄" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["frontend","react","spa","javascript"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---
# TL;DR
JS史上最軽量フレームワークと名高いVanilla JSでSPAしてみました。

# 今回目指すSPAの仕様
- URLに対応するページが表示される。アンカーリンクをクリックするとURLが変更され、ページが遷移する。その際、画面のリロードは発生しない。
- ブラウザバックを押下すると、画面のリロードをせずに前表示していたページに戻る。
- URLから変数を受け取り、画面に反映させる。

# 画面を用意

以下のようなindex.htmlを用意しました。
ヘッダー内ににアンカーリンクと、コンテンツにSPAの描画先として`id="app"`のdivタグを配置しています。

```html:index.html
<html>
<head>
	<title>バニラでSPA</title>
	<meta charset="UTF-8" />
</head>
<body>
	<header>
		<a href="/">TOP</a>
		<a href="/home">HOME</a>
		<a href="/profile">PROFILE</a>
	</header>
	<div id="app"></div>
	<script src="src/index.ts">
	</script>
</body>
</html>
```

# リロードなしのページ遷移
まず手始めに、リロード無しでのページ遷移を実装します。
ページ遷移と言っても、**URLの変更**と**DOMの変更**を同時に行うだけなので簡単ですね。

## URLの変更
ブラウザにはURLを変更するAPIが用意されております。

```js
window.history.pushState({name:"taro"}, "", "/hoge");
```

とりあえず第１、２引数は気にしなくていいです。
第３引数に遷移先のURLを指定することで、URLを変更することができます。
変更されますが、画面はリロードされたりURLでサーバーにアクセスしたりしません。

では、この関数を雑に画面上のアンカーリンクに設定しましょう。
以下のスクリプトでクリックイベントを仕込みます。

```js:src/index.ts
document.querySelectorAll("a").forEach(a => {
  a.onclick = event => {
    event.preventDefault();//アンカーリンクのデフォルト挙動をdisable
    window.history.pushState(null, "", a.href);
  };
});
```

アンカーリンクを押してみましょう。URLの変更が確認できますね。
![1.gif](https://storage.googleapis.com/zenn-user-upload/2nl1q0krmmr6idbefk23rg3ir7kg)

## URLの変更と同時にDOMを変更
updateView関数としてURLが変更された時にDOMを操作する挙動を定義しましょう。
```window.location.pathname```で現在のURLを取得し、予め定義した`pages`(パスと描画するhtmlのマッピング)からhtmlを取得しています。

```ts
const updateView = () => {
  const pages = {
    "/": `
      <h1>Vanilla SPA</h1>
    `,
    "/home": `
      <h1>ようこそ！</h1>
    `,
    "/profile": `
      <h1>私は太郎です。</h1>
    `
  };
  document.getElementById("app").innerHTML = pages[window.location.pathname];
};

document.querySelectorAll("a").forEach(a => {
  a.onclick = event => {
    event.preventDefault();//アンカーリンクのデフォルト挙動をdisable
    window.history.pushState(null, "", a.href);
    updateView();//ここで定義した関数を発火
  };
});
```
こんな風にSPAぽい画面ができました。
![2.gif](https://storage.googleapis.com/zenn-user-upload/bfzmptpiekcy0fqqao4ydk5gqxbc)

## 初期表示時のURLによって表示するページを切り替える
この要領で画面初期化時にもURLによって表示するページを切り替えましょう。
先ほどの```updateView```関数を初期化処理として呼び出します。
また、ユーザーは`/hoge`など、予期せぬURLにアクセスしてくる場合があります。
404のページも用意しておきましょう。

```ts 
const updateView = () => {
  /* 省略 */
  const page = pages[window.location.pathname];
  const render = page || `<h1>404 : Not Found<h1>`;
  document.getElementById("app").innerHTML = render;
};
//初期化処理
updateView();
```

# ユーザーのブラウザバックのハンドリング
window.history.pushState関数による画面遷移はただURLが書き変わるだけでなく、ブラウザにその遷移履歴を保存します。
どういうことかというと、`/fuga`から`/hoge`にSPAで遷移した後にブラウザバックボタンを押下すると画面リロード無しでURLが`/fuga`に書き変わります。
このイベントを監視し、そのタイミングでDOMを更新してあげると、さもブラウザバックで元の画面に戻ったような挙動がリロードなしで実現できます。
windowはブラウザバックやブラウザフォワード（？）実行時に`popstate`イベントを発火しますので、これにリスナーを付与しましょう。

```ts
window.addEventListener("popstate", () => {
  updateView();
});
```

これで最低限のSPA機能は出来上がりました。

# URLで変数を受け取れるようにしてSPAを完成させる。
仕上げです。
`react-router`を始めとした、SPAライブラリには遷移したURLから変数を取得できる機能が備わっています。
例えば`/members/:id`というURLでページを待機しておき、`/members/100`というURLにユーザーがアクセスしてきたら`{id:100}`という変数が取得できる。というような塩梅です。

## 変数を抽出する関数を作る

折角なのでここも実装しましょう。
（あなたがもし熱心なバニラJS信者ではないなら大人しく[ライブラリ](https://www.npmjs.com/package/path-to-regexp)を使いましょう）

マッチャー（`/hoge/:id`などの文字列）を受け取り、URLから変数を抽出もしくはマッチしない場合にnullを返却する関数を実装します。

```ts:matcherToParamResolver.ts
export type Params = Record<string, string | undefined>;

const EXTRACT = /\/:[^\/]+/g;
const OPTIONAL_SYMBOL = /\?$/;
const OPTIONAL_MATCHER_STRING = "(?:/([^/]+?))?";
const REQUIRED_MATCHER_STRING = "/([^/]+?)";

export default (matcher: string) => {
  const extracted = matcher.match(EXTRACT);
  const keys = extracted && extracted.map(e => e.replace(OPTIONAL_SYMBOL,"")).map(e => e.replace(/^\/:/,""));
  const exp = matcher.replace(EXTRACT, e => OPTIONAL_SYMBOL.test(e) ? OPTIONAL_MATCHER_STRING : REQUIRED_MATCHER_STRING);
  const reg = new RegExp(`^${exp}(?:/)?$`);
  return (path: string): Params | null => {
    const res = reg.exec(path);
    if (res) {
      if(keys){
        const params: Params = {};
        res.slice(1).forEach((e, i) => {
          params[keys[i]] = e;
        });
        return params;
      }
      return {};
    } else {
      return null;
    }
  };
};
```
```ts:実行結果
import f from "./matcherToParamResolver"

f("/b")("/a"); // null
f("/")("/a"); // null
f("/")("/"); // Object {}
f("/:id")("/aa"); // Object {id: "aa"}
f("/hoge/:id")("/hoge/aaa"); // Object {id: "aaa"}
f("/hoge/:locale/:id")("/hoge/jp/15"); // Object {locale: "jp", id: "15"}

```
## ルーターの処理を実装

作った関数、そして今までの実装を利用しながらルーターの処理を書いていきます。
ユーザーが定義する部分を引数に逃し、SPAを作成する関数として実装します。引数の説明は以下
`pages` : 各ページの定義。keyがURLのマッチャー、valueが描画するHTML（もしくはHTMLを返す関数）です。
`notfound` : マッチャーがマッチするページを見つけられなかった場合に表示するページ
`element` : ページを表示させる要素

```ts:router.ts
import matcherToParamExtractor, { Params } from "./matcherToParamExtractor";
export default (
  pages: { [key: string]: ((params: Params) => string) | string },
  notfound: string,
  element: HTMLElement
) => {
  //引数で受け取ったpagesをビルドする。
  const builtPages = Object.keys(pages).map(matcher => ({
    render(params: Params) {
      const page = pages[matcher];
      return typeof page === "string" ? page : page(params);
    },
    test: matcherToParamExtractor(matcher)
  }));
  //DOM書き換え処理
  const updateView = () => {
    //innerHTMLを使ってDOMを書き換える
    const mount = (html: string) => {
      element.innerHTML = html;
    };
    const path = window.location.pathname;
    //マッチャーにマッチするページまでforで繰り返し
    for (const page of builtPages) {
      const params = page.test(path);
      if (params) {
        mount(page.render(params));
        //見つかればreturn
        return;
      }
    }
    //見つからなければ404のページを表示
    mount(notfound);
  };

  //アンカーリンクにルーターのリンクを仕込む
  document.querySelectorAll("a").forEach(a => {
    a.onclick = event => {
      event.preventDefault();
      window.history.pushState(null, "", a.href);
      updateView();
    };
  });

  //ブラウザバックを監視
  window.addEventListener("popstate", () => {
    updateView();
  });

  //初期化
  updateView();
};
```

## SPAしたいページを定義する。
最後に、ここまで作った機能を利用してページを定義しましょう。
HTML側も少し変えて、taroとhanakoのプロフィールを表示できるようにしました。

```html:index.html
<html>

<head>
	<title>バニラでSPA</title>
	<meta charset="UTF-8" />
</head>

<body>
	<header id="header">
		<a href="/">TOP</a>
		<a href="/home">HOME</a>
		<a href="/profile/taro">Taro's PROFILE</a>
		<a href="/profile/hanako">hanako's PROFILE</a>
	</header>
	<div id="app"></div>
	<script src="src/index.ts">
	</script>
</body>

</html>
```

```ts:index.ts
import router from "./router";

const pages = {
  "/": `
    <h1>Vanilla SPA</h1>
  `,
  "/home": `
    <h1>ようこそ！</h1>
  `,
  "/profile/:name": (params: { name: string }) => `
    <h1>私は${params.name}です。</h1>
  `
};

router(pages, `<h1>404 : Not Found<h1>`, document.getElementById("app"));

```
これで完成です。
![3.gif](https://storage.googleapis.com/zenn-user-upload/pcp5c39mfzxdbn4507qicnwrsro1)

# 注意:ホスティングサーバー側で気をつけること
さて、クライアント側の処理は出来上がりましたが、これをいざ普通にホスティングサーバーにデプロイしてSPAだ！とやると、ユーザーがブラウザからルート以外のURLにアクセスしようとするとこでコケます。
理由は単純で、アクセスしようとしたURLに対応するリソースがないからになります。
今まで実装してきたSPAの仕組み自体は ***index.html上にしかないので、*** ユーザーがindex.html以外のURLでアクセスしてきたらjs+htmlでは手も足も出ないのです。

## 解決策
ユーザーがアクセスしてきたURLにリソースが無い場合に`index.html`を返すようにホスティングサーバーの設定を変更します。（一般的にこの操作を`fallback`と言ったりします。）
`webpack-dev-server`や、`Firebase Hosting`、`Vercel`等、主流なホスティングのアプリケーションは大抵この機能を備えています。設定を見直しましょう。

[参考(vercel)](https://vercel.com/docs/configuration?query=fallback#routes/advanced/spa-fallback)

# じゃあ結局SPAってなんなのよ
さて、ここまでVanilla JSにてSPAの実装サンプルをお見せしてきました。
これを書いた目的は「SPA = **Next.jsやreact-routerなどのヘビーなViewライブラリを使用しないと出来ない高級テクニック** 」というような先入観を解くことです。

もちろん、ここで紹介した方法は現実的なものではありません。普通に開発するならライブラリを使わない手は無いと思います。
しかしながら、この記事が改めて **「SPAってなんなのよ。というかWebアプリケーションってなんなのよ」** ということを考えてみる良いきっかけになってくれれば。と思っております。

# サンプル実装
（2020/10/16 追記）サンプル実装を作りました。 ➡️ https://github.com/itskihaga/vanilla-spa-sample
