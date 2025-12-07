---
title: "Reactコンポーネントの適切な分割方法：render hooksのすすめ" # 記事のタイトル
emoji: "🪝" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["frontend","react","typescript","hooks"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

## 概要

本記事では、肥大化しがちなReactコンポーネントをより読みやすく分割する方法として「render hooks」というパターンを紹介します。

## ソースコード分割の課題

ソースコードが大きくなると読みづらくなります。そのため、ファイルや関数を分割することが一般的です。しかし、分割は読み手にコンテキストの切り替えを要求するため、闇雲に分割すればよいというわけではありません。**どのような切り口で分割するか**が重要です。

## 切り口の例：ロジックとマークアップで分ける

Reactのhooksが登場して以来、「ロジックとマークアップを分離する」という考え方が広まりました。stateやイベントハンドラなどのロジックをカスタムhooksに切り出し、JSXは別の場所に書くという方法です。

しかし、この分割方法は読みづらくなることがあります。なぜなら、**マークアップとロジックは密に関係しており、結合度が高くなる**からです。

具体例を見てみましょう。ダイアログ内にフォームがあり、入力値のバリデーションを行う場合を考えます。

:::message
以下の例では、UIライブラリとして[smarthr-ui](https://github.com/kufu/smarthr-ui)を使用しています。本質的でないマークアップの詳細を抽象化し、コード分割のパターンに集中するためです。
:::

```tsx
import { Button, FormControl, Input, Textarea, FormDialog } from 'smarthr-ui'

// ロジック部分（カスタムhooks）
const useFormDialog = () => {
  const [isOpen, setIsOpen] = useState(false);
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [answer, setAnswer] = useState("");
  const [nameError, setNameError] = useState("");
  const [emailError, setEmailError] = useState("");

  const validateName = (value: string) => {
    if (value.length === 0) {
      setNameError("名前を入力してください");
      return false;
    }
    setNameError("");
    return true;
  };

  const validateEmail = (value: string) => {
    if (!value.includes("@")) {
      setEmailError("正しいメールアドレスを入力してください");
      return false;
    }
    setEmailError("");
    return true;
  };

  const handleNameChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setName(value);
    validateName(value);
  };

  const handleEmailChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setEmail(value);
    validateEmail(value);
  };

  const handleAnswerChange = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
    setAnswer(e.target.value);
  };

  const handleSubmit = async (close: () => void) => {
    if (validateName(name) && validateEmail(email)) {
      // 送信処理
      await submitAnswers({ name, email, answer });
      close();
    }
  };

  return {
    isOpen,
    setIsOpen,
    name,
    email,
    answer,
    nameError,
    emailError,
    handleNameChange,
    handleEmailChange,
    handleAnswerChange,
    handleSubmit,
  };
};

// マークアップ部分（コンポーネント）
const SurveyFormDialog = () => {
  const {
    isOpen,
    setIsOpen,
    name,
    email,
    answer,
    nameError,
    emailError,
    handleNameChange,
    handleEmailChange,
    handleAnswerChange,
    handleSubmit,
  } = useFormDialog();

  return (
    <>
      <Button onClick={() => setIsOpen(true)}>フォームを開く</Button>
      <FormDialog
        isOpen={isOpen}
        title="アンケート"
        actionText="送信"
        onSubmit={handleSubmit}
        onClickClose={() => setIsOpen(false)}
      >
        <FormControl label="名前" errorMessages={nameError || undefined}>
          <Input type="text" value={name} onChange={handleNameChange} />
        </FormControl>
        <FormControl label="メールアドレス" errorMessages={emailError || undefined}>
          <Input type="email" value={email} onChange={handleEmailChange} />
        </FormControl>
        <FormControl label="ご意見をお聞かせください">
          <Textarea value={answer} onChange={handleAnswerChange} />
        </FormControl>
      </FormDialog>
    </>
  );
};
```

この分割の問題点は以下の通りです:

- hooksの戻り値として多数の値や関数を返す必要がある
- どの値がどのマークアップで使われるかを理解するには、hooksとコンポーネントの両方を行き来する必要がある
- マークアップを見ても、それに対応するロジックがどこにあるのか分かりにくい

## 推奨する切り口：画面の構成要素で分ける

では、どのように分割すればよいのでしょうか。私が推奨したいのは「**画面の構成要素（UI の関心）で分ける**」という切り口です。

先ほどの例を、以下のような構成要素に分けることを考えます:

- ダイアログの外枠と送信処理
- ユーザー情報の入力（名前、メールアドレス）
- アンケート内容の入力（質問への回答など）

しかし、ここで問題が生じます。各構成要素をコンポーネントとして分割すると、**親コンポーネントから子コンポーネント内のstateを参照できない**のです。

```tsx
import { Button, FormControl, Input, Textarea, FormDialog } from 'smarthr-ui'

// ユーザー情報の入力部分をコンポーネントに切り出す
const UserInfoForm = () => {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");

  return (
    <>
      <FormControl label="名前">
        <Input type="text" value={name} onChange={(e) => setName(e.target.value)} />
      </FormControl>
      <FormControl label="メールアドレス">
        <Input type="email" value={email} onChange={(e) => setEmail(e.target.value)} />
      </FormControl>
    </>
  );
};

// アンケート内容の入力部分をコンポーネントに切り出す
const SurveyForm = () => {
  const [answer, setAnswer] = useState("");

  return (
    <FormControl label="ご意見をお聞かせください">
      <Textarea value={answer} onChange={(e) => setAnswer(e.target.value)} />
    </FormControl>
  );
};

// ダイアログ本体
const SurveyFormDialog = () => {
  const [isOpen, setIsOpen] = useState(false);

  const handleSubmit = async (close: () => void) => {
    // 問題: UserInfoForm と SurveyForm 内の state にアクセスできない
    // name, email, answer の値をどうやって取得する？
    await fetch("/api/answers", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ /* ??? */ }),
    });
    close();
  };

  return (
    <>
      <Button onClick={() => setIsOpen(true)}>フォームを開く</Button>
      <FormDialog
        isOpen={isOpen}
        title="アンケート"
        actionText="送信"
        onSubmit={handleSubmit}
        onClickClose={() => setIsOpen(false)}
      >
        <UserInfoForm />
        <SurveyForm />
      </FormDialog>
    </>
  );
};
```

## render hooksによる解決

この問題を解決するのが **render hooks** です。render hooksは、**JSXを返すカスタムhooks**です。

render hooksでは、以下の2つを返します:

- `view`: レンダリングするJSX
- その他の値: 親コンポーネントが必要とする値（フォームの入力値など）

先ほどの例をrender hooksで書き直してみます。

```tsx
import { Button, FormControl, Input, Textarea, FormDialog } from 'smarthr-ui'

// ユーザー情報の入力部分（render hooks）
const useRenderUserInfoForm = () => {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [nameError, setNameError] = useState("");
  const [emailError, setEmailError] = useState("");

  const validateName = (value: string) => {
    if (value.length === 0) {
      setNameError("名前を入力してください");
      return false;
    }
    setNameError("");
    return true;
  };

  const validateEmail = (value: string) => {
    if (!value.includes("@")) {
      setEmailError("正しいメールアドレスを入力してください");
      return false;
    }
    setEmailError("");
    return true;
  };

  const handleNameChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setName(value);
    validateName(value);
  };

  const handleEmailChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setEmail(value);
    validateEmail(value);
  };

  const view = (
    <>
      <FormControl label="名前" errorMessages={nameError || undefined}>
        <Input type="text" value={name} onChange={handleNameChange} />
      </FormControl>
      <FormControl label="メールアドレス" errorMessages={emailError || undefined}>
        <Input type="email" value={email} onChange={handleEmailChange} />
      </FormControl>
    </>
  );

  return {
    view,
    values: { name, email },
    validate: () => validateName(name) && validateEmail(email),
  };
};

// アンケート内容の入力部分（render hooks）
const useRenderSurveyForm = () => {
  const [answer, setAnswer] = useState("");

  const view = (
    <FormControl label="ご意見をお聞かせください">
      <Textarea value={answer} onChange={(e) => setAnswer(e.target.value)} />
    </FormControl>
  );

  return {
    view,
    values: { answer },
  };
};

// ダイアログ本体
const SurveyFormDialog = () => {
  const [isOpen, setIsOpen] = useState(false);
  const userInfo = useRenderUserInfoForm();
  const survey = useRenderSurveyForm();

  const handleSubmit = async (close: () => void) => {
    if (!userInfo.validate()) {
      return;
    }

    // 各 render hooks から値を取得して送信
    await submitSurvey({
      ...userInfo.values,
      ...survey.values,
    }); // { name: "...", email: "...", answer: "..." }
    close();
  };

  return (
    <>
      <Button onClick={() => setIsOpen(true)}>フォームを開く</Button>
      <FormDialog
        isOpen={isOpen}
        title="アンケート"
        actionText="送信"
        onSubmit={handleSubmit}
        onClickClose={() => setIsOpen(false)}
      >
        {userInfo.view}
        {survey.view}
      </FormDialog>
    </>
  );
};
```

このアプローチの利点は以下の通りです:

- **関心の分離**: ユーザー情報の入力とアンケートの入力が、それぞれ独立した単位で管理される
- **ロジックとマークアップの共存**: 各render hooks内で、関連するstateとJSXが近くに配置される
- **親からのアクセス**: 親コンポーネントは、必要な値（`values`、`validate`など）にアクセスできる

各render hooksが1つの関心（UI の構成要素）を持ち、そのロジックとマークアップをまとめて管理します。

## 最後に

render hooksは、JSXを返すカスタムhooksという単純なパターンです。
特別なライブラリも複雑な設定も不要で、各UIの関心ごとにロジックとマークアップをまとめて管理できます。

コンポーネントの分割方法に悩んだときは、ぜひ一度試してみてください。



