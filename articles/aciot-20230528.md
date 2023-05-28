---
title: "axiosのinterceptorを活用してOpenAIのAPIとの結合テストを書く" # 記事のタイトル
emoji: "㊙️" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["frontend","test","openai"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

## 背景あるいは宣伝

現在、私は個人開発でGPT-4を使用した水平思考クイズアプリを作成中です。現在、Beta版が公開されています。

https://iesona.com/

今回はその開発で直面した問題とその解決策について書きます。

## TL;DR

OpenAIのAPIとの結合テスト時にaxiosのinterceptorを利用してリクエストをキャッシュし、CIの際にモックとして使用することで開発体験を向上させました。

## アプリケーションのスタック

- Next.js(App Routerには移行できていない) 
- ほぼほぼT3 Stack
- OpenAPIへの接続にはopenaiパッケージ（以後、openai）を利用

## やりたいこと

OpenAIのAPI（以後、OpenAI）とアプリケーションロジックの結合テストを書きたいです。モチベーションとしては、ユーザーからの入力を基に既存のプロンプトと組み合わせてAPIに問い合わせるという一般的なユースケースの実装について、以下の要求がありました。

- Web画面を経由せずに実装・検証したい
- CIでロジックのリグレッションを検知したい

## 問題

しかしながら、OpenAIとの結合テストは以下の3つの問題点から難易度が高いです。

- APIのレスポンスが遅い/不安定
- リクエストごとに料金が発生する
- 毎回同じ結果が返ってこない（temperatureを0にしていない場合）

OpenAIを直接テストに組み込むとこれらの問題に直面します。しかし、OpenAIをモックにすると、アプリケーションがOpenAIと適切に連携して動作しているか検証することができません。

一般的には、このような問題に対処するために結合テストではなく、OpenAIとアプリケーションロジックを個々にテストするというアプローチを取ると思います。しかし、それではテスト対象の境界が増えてコストが上がります。  
あまりテストに工数をかけたくないため別のアプローチを模索しました。

## 解決策

解決策としてOpenAIのレスポンスをキャッシュして再利用可能にする機構を作り、そのキャッシュをファイルとして保存し（保存したファイルは.gitignoreしないでリポジトリにコミットします）、以下の2つのモードを切り替えることでCIとローカル開発での両方のニーズを結合テストだけで満たすようにしました。

- 試行錯誤が必要な場面では、実際にOpenAIを叩き、アプリケーションと結合させます。
- CIなどでリグレッションを検知したい場合は、事前に保存したキャッシュをモックとして返させてロジックだけを検証します。

### axiosのinterceptor
axiosにはもともとリクエスト・レスポンスにinterceptorを追加する機能が備わっています。  
openaiはクライアントをnewする際にaxiosのインスタンスを受け取るため、そのインスタンスにinterceptorを適用することでOpenAIへのリクエストをインターセプトできます。  

今回はこの機能を使って実装していきます。  

:::message alert
openaiがaxios@0.26（執筆時既に1系がリリースされています。）に依存しているため、若干実装が古く見える場合もあると思いますがご了承ください。
:::

### 実装
具体的な実装は以下の通りです。
（ちなみに、このライブラリの名前をAxios Cache Interceptor for OpenAI Testingで、略してaciotと名付けました。）

```ts
export type CacheMode = "cacheOnly" | "requestIfNoCacheHit";

export class AciotCore {
	private readonly cache: AxiosResponseCache;
	constructor(config: {
		namespace: string;
		cacheBasePath: string;
	}) {
		this.cache = new AxiosResponseCache(
			FsCache({
				basePath: config.cacheBasePath,
				ns: config.namespace,
			}),
		);
	}
	public applyInterceptors(axios: AxiosInstance, cacheMode: CacheMode) {
		const useCache = async (rejected: unknown) => {
			if (rejected instanceof ShouldUseCacheError) {
				return {
					data: rejected.cache,
				};
			}
			throw rejected;
		};
        switch (cacheMode) {
            case "cacheOnly":
                axios.interceptors.request.use(async (config) => {
                    const cache = await this.cache.get(config);
                    if (cache) {
                        throw new ShouldUseCacheError(cache);
                    }
                    throw new Error("Cache not found");
                });
                axios.interceptors.response.use(() => {}, useCache);
                break;
        
            case "requestIfNoCacheHit":
                axios.interceptors.request.use(async (config) => {
                    const cache = await this.cache.get(config);
                    if (cache) {
                        throw new ShouldUseCacheError(cache);
                    }
                    return config;
                });
                axios.interceptors.response.use((res) => {
                    if (res.data) {
                        this.cache.set(res.config, res.data);
                    }
                    return res;
                }, useCache);
        }
	}
}
```

ここでは、axiosのinterceptorを使ってリクエストを事前にハンドリングしています。
`requestIfNoCacheHit`モード（ローカル開発で使用）の場合、デフォルトではリクエスト時には何もせず、レスポンスをキャッシュに保存します。キャッシュがヒットした場合はキャッシュを返します。
`cacheOnly`モード（CIで使用）の場合、キャッシュヒットした場合はキャッシュを返し、キャッシュがヒットしない場合はエラーを投げます。

interceptorの実装はややハック的で、キャッシュがヒットした際エラーでもないのに`ShouldUseCacheError`をthrowしていることがわかると思います。axiosのinterceptorは正常に値を返してしまうとリクエストを止めることはできないため、例外を投げることでリクエストを止めています。  

### OpenAI及びアプリケーションへの適用方法
先述した通り、openaiクライアントはaxiosのインスタンスを受け取るため、このインスタンスにinterceptorを適用します。あとはopenaiクライアントのインスタンスを実行時注入（本番時にこのキャッシュの機構は使わないため）できるようにしておくだけでほとんど実装を変えずにこの機構を導入できました。  

```ts:openaiForTest.ts

const axios = Axios.create();
const aciot = new AciotCore({
    namespace: /* テストの名前空間 */   ,
    cacheBasePath: /* キャッシュするディレクトリ */
});

aciot.applyInterceptors(axios, process.env.CI ? "cacheOnly" : "requestIfNoCacheHit");

return new OpenAIApi(
    new Configuration({
        apiKey: process.env.OPENAI_API_KEY,
    }),
    undefined,
    axios,
);

```

基本的なキャッシュの仕組みは、リクエストのURLとbodyをJSONシリアライズしたものをキーとして、レスポンスのbodyをファイルとして保存します。以下、実装を簡易的にしたものです。

```ts
class AxiosResponseCache {
	constructor(private readonly cache: FileSystemCache) {}
	private static configToKey(config: AxiosRequestConfig): string {
		return JSON.stringify({
			url: config.url,
			body: config.data,
		});
	}
	public get(configForKey: AxiosRequestConfig): Promise<unknown> {
		const key = AxiosResponseCache.configToKey(configForKey);
		return this.cache.get(key);
	}
	public async set(configForKey: AxiosRequestConfig, value: unknown) {
		const key = AxiosResponseCache.configToKey(configForKey);
		await this.cache.set(key, value);
	}
}
```


## Q&A

**Q. OpenAI以外にもこの解決策は適用できますか？**

A. できると思います。ただし、「本番のAPIを使って試行錯誤したい」ケースがあまりない（大抵の場合、ドキュメントを読んで一度試せば何が返ってくるかわかる）ため、このケースは特にニーズとしてマッチしていたと言えます。  

**Q. axiosのinterceptorではなく、mswを使っても同様のことができますか？**

A. 可能ですが、mswでリクエストをpassthroughしつつ、そのレスポンスを参照することができず、そのためmswのハンドラー内でハンドリングしたリクエストを再現する必要があり、それが難しかったです。

