---
title: "useSWRをざっくり理解したので、概要と不満を書きます"
emoji: "🗂"
type: "tech"
topics:
  - "react"
  - "swr"
published: true
published_at: "2021-07-22 13:51"
---
<!-- textlint-disable -->

### 前置きと学んだ経緯

今開発しているプロダクトでは、APIからのデータ取得・キャッシュ戦略が場所によりバラバラなので、便利なライブラリを用いて綺麗に書きたい揃えたい…という思いからSWRの導入をしました。

キャッシュ周りやSWRについて自分が全く理解していなかったので、公式から掻い摘んで必要なところを自分で解釈しつつ説明していきます。


## useSWRとは何者なのか。概要

[SWR](https://github.com/vercel/swr)とは、みんな大好きNext.jsを作成しているVercel製のライブラリです。

SWRとは、[stale-while-revalidate](https://web.dev/stale-while-revalidate/)というRFC 5861で策定されたキャッシュ戦略の略称らしいです。

ライブラリの使い方としては`useSWR`というReact Hooksを用いることで、APIを通じたデータの取得・キャッシュを簡単に記述する手助けをしてくれます。

stale while revalidatesのキャッシュ戦略は、まずキャッシュからデータを返し（stale）、次にフェッチリクエストを送り（revalidate）、最後に最新のデータを持ってくるという戦略です。端的にいうと、非同期でキャッシュの新しさを確認する時間を設けています。

↓こちらで定義されています。

[https://datatracker.ietf.org/doc/html/rfc5861](https://datatracker.ietf.org/doc/html/rfc5861#page-2)

SWRが自称している特徴としては(公式ページより引用)

```text
たった 1 行のコードで、プロジェクト内のデータ取得のロジックを単純化し、さらにこれらの素晴らしい機能をすぐに利用できるようになります：

- Jamstack 指向
- 速い、 軽量 そして 再利用可能 なデータの取得
- 組み込みの キャッシュ とリクエストの重複排除
- リアルタイム な体験
- トランスポートとプロトコルにとらわれない
- TypeScript 対応
- React Native

SWR は、スピード、正確性、安定性のすべての面をカバーし、より良い体験を構築するのに役立ちます：

高速なページナビゲーション
定期的にポーリングする
データの依存関係
フォーカス時の再検証
ネットワーク回復時の再検証
ローカルキャッシュの更新（Optimistic UI）
スマートなエラーの再試行
ページネーションとスクロールポジションの回復
React Suspense
...
```

らしいです。

公式なので当然いいことしか書いていませんが、悪いところも結構あります。

実際使ってみて（自分の理解が甘いだけけもしれませんが）よくないな…と思う箇所も結構あったので、最後にまとめて書きます。

## useSWRのざっくり[Overview](https://swr.vercel.app/#overview)

最初にどんな感じで書くのか。標準的な書き方を例示します。

公式に書いてあるやつそのまま持ってきました。

```jsx
import useSWR from 'swr'

function Profile() {

	const { data, error } = useSWR('/api/user', fetcher)

	if (error)return <div>failed to load</div>
	if (!data)return <div>loading...</div>

	return <div>hello {data.name}!</div>
}

```

詳しい書き方は後ほど解説します。

ざっくり説明すると`useSWR`は`キー`と`fetcher`を受け取るだけで、キャッシュで高速化された`data`と`error`を返してきます。

`fetcher` はデータを返す任意の非同期関数で、 `Axios`のようなツールを使うことができます。

通常だとpromis.then.catch...とか色々書かないといけませんが、

このようにデータ取得・キャッシュ周りの機構を、シンプルに１行だけで書くことができます。



## stale while revalidateについて

ここは概要を理解するための余談です。

useSWRの仕組みがこの設定を使っているわけではなさそうなので、適当に読み飛ばしてください

前提として、ブラウザが持つキャッシュの目的は

- リソースの取得を高速化し、ユーザの待機時間を軽減
- サーバーへのリクエスト回数を減らし、負担を軽減すること

があると思っています。

既存のキャッシュコントロールヘッダーの場合

`Cache-Control`で`max-age`を指定した場合、例えばこのように記述すると、

```text
Cache-Control: max-age=600
```

最大600秒までは、何度アクセスしてもこのキャッシュした値を使ってくれます。

600秒経過後は、再度fetchしてくるという仕様です


![](https://storage.googleapis.com/zenn-user-upload/4e3ff0c42369754436f99df7.png)


ここにstale while revalidateを追加すると

```text
Cache-Control: max-age=600, stale-while-revalidate=30
```

600秒までは、このキャッシュされた値を使用しますが、追加で30秒検証期間が設けられます。

600秒経過後に、改めて今の値が最新のものであるのかの問い合わせが行われます。(revalidate)

（もし30秒以内に完了しない場合はキャッシュは削除されます）

![](https://storage.googleapis.com/zenn-user-upload/246e5b24a87c9a066923c427.png)

ちなみに…このような設定だと、

サーバーの負荷はあまり変わりませんが、リソースの読み込みは高速化できます。

```text
Cache-Control: max-age=1, stale-while-revalidate=600
```

RFC 5861で策定されたstale while revalidateのお話でした。



## useSWRの基本的な使い方

ここから本題です。

今まではuseSWRの元となったキャッシュ戦略のお話でしたが、ここからuseSWR のお話です。

考え方をは使っていますが、このヘッダー使っているわけではなさそうなので、もう忘れてしまって大丈夫です。

改めて、useSWRは

- API からのデータ取得、キャッシュ
- ローディング
- エラーハンドリング

がシンプルに記述できることが強みでした。

### 基本的な使い方の例

最初にも示しましたが、公式のコードそのままです。

```jsx
import useSWR from 'swr'

function Profile() {
	const { data, error } = useSWR('/api/user', fetcher)

	if (error)return <div>failed to load</div>
	if (!data)return <div>loading...</div>

	return <div>hello {data.name}!</div>
}

```

この例では、`useSWR`は `key` 文字列と `fetcher` 関数を受け取ります。

 `key` はデータの一意な識別子（通常は API の URL）で、`fetcher` に渡されます。

fetcherに渡さないように実装もできます。それは後ほど示します。

 `fetcher` はデータを返す任意の非同期関数で、

ネイティブの fetch や Axios のようなツールを使うことができます。

デフォルトのfetcherを使う場合は第二引数が不要でした。

keyに渡したパスに対してGETが走ります。

このフックは、リクエストの状態にもとづいて `data` と `error` の二つの値を返します。

データはこの２つしか返してくれないので、基本的にはこれに基づいて、データ、エラーのハンドリングを行います。



### ちょっと詳しく

厳密には渡せる引数や返り値ももう少しあるので紹介していきます。
公式に載っているものにちょっと

```jsx
const { data, error, isValidating, mutate } = useSWR(key, fetcher, options)
```

#### **[パラメーター](https://swr.vercel.app/ja/docs/options#%E3%83%91%E3%83%A9%E3%83%A1%E3%83%BC%E3%82%BF%E3%83%BC)**

- `key`: このリクエストのためのユニークなキー文字列（または関数、配列、null） [（高度な使用法）](https://swr.vercel.app/docs/conditional-fetching)
- `fetcher`: （*任意*） データをフェッチするための Promise を返す関数 [（詳細）](https://swr.vercel.app/docs/data-fetching)
- `options`: （*任意*） この SWR フックのオプションオブジェクト
    - ここで色々設定できますが細かく説明していると時間がなくなる＆全ては理解してないので割愛します。

#### **[返り値](https://swr.vercel.app/ja/docs/options#%E8%BF%94%E3%82%8A%E5%80%A4)**

- `data`: `fetcher` によって解決された、指定されたキーのデータ（もしくは、ロードされていない場合は undefined）
- `error`: `fetcher` によって投げられたエラー （もしくは undefined）
- `isValidating`: リクエストまたは再検証の読み込みがあるかどうか
    - [詳しくはこちら](https://swr.vercel.app/ja/advanced/performance#%E4%BE%9D%E5%AD%98%E9%96%A2%E4%BF%82%E3%81%AE%E3%82%B3%E3%83%AC%E3%82%AF%E3%82%B7%E3%83%A7%E3%83%B3) がわかりやすいと思います。正直使い道はわかってないです。
    - errorとdata のみでloadingを表現したくない場合、エラーが帰ってきているが、再試行しその検証中はloadingにしたい。などの要望があればこれ使えそうです。
- `mutate(data?, shouldRevalidate?)`: キャッシュされたデータを更新する関数
    - 後述しますがこれがPUT実装の鍵です。

#### **[再利用可能にする](https://swr.vercel.app/ja/getting-started#%E5%86%8D%E5%88%A9%E7%94%A8%E5%8F%AF%E8%83%BD%E3%81%AB%E3%81%99%E3%82%8B)**

同じデータを複数の場所で扱いたい場合など、単純にuseSWRを直接記述するだけでは不便なので、再利用可能な形に切り出して使うことが推奨されています。

以下のように実装します。公式に書いてあるコードそのままです。

```jsx
function useUser (id) {
const { data, error } = useSWR(`/api/user/${id}`, fetcher)
return {
    user: data,
    isLoading: !error && !data,
    isError: error
  }
}
```

これを使用する時はこのような形になります。

```jsx
function Avatar ({ id }) {
	const { user, isLoading, isError } = useUser(id)

	if (isLoading)return <Spinner />
	if (isError)return <Error />

	return <img src={user.avatar} />
}

```

このパターンを採用することで、リクエストを開始し、ロード状態を更新し、最終的な結果を返す、

という命令的な方法でデータを**取得**することを忘れることができます。 

代わりに、コードはより宣言的になります：コンポーネントが使用するデータを指定するだけです。

キャッシュしたからには再検証が必要です。

どんなタイミングで再検証が走るのか、自動的な再検証は３つあります

#### **[Revalidate on Focus](https://swr.vercel.app/docs/revalidation#revalidate-on-focus)**

フォーカス時の再検証。

ページを再フォーカスしたり、タブを切り替えたりすると、SWRは自動的にデータを再検証してくれます。

#### **[Revalidate on Interval](https://swr.vercel.app/docs/revalidation#revalidate-on-interval)**

intervalでの検証の仕組みも用意してくれています。

常に裏で再検証されていたら、何回もAPIリクエストが発生するのでは？と思いましたが

「フックに関連付けられたコンポーネントが画面に表示されている場合にのみ再取得が行われるというスマートなものです。」

とのことでした。賢いです。（どうやって実現しているのかまだみてないです）

こっちは、デフォルトでは設定されていないため、自分で設定する必要があります。 `[refreshInterval](https://swr.vercel.app/docs/options)` 

```jsx
useSWR('/api/todos', fetcher, { refreshInterval: 1000 })
```

※同様なことを設定するために `refreshWhenHidden` 、 `refreshWhenOffline`  がありますがデフォルトでオフらしいです。

#### Revalidate on Reconnect

名前の通りです。ネットワークが切断されて再度接続された場合に再検証されます。

### マニュアルでの再検証 **[Revalidate](https://swr.vercel.app/docs/mutation#revalidate)**

もちろんマニュアルでの再検証をすることもできます。

 `mutate(key)`を呼ぶことで、同じキーを持つ全てのキャッシュに対してrevalidateをかけることができます。

```jsx
// １回目
const { data, error } = useSWR(`/api/user/`, fetcher)

// 手動でrevalidate
mutate('/api/user')

```

### **[Mutation and POST Request](https://swr.vercel.app/docs/mutation#mutation-and-post-request)**

ここまで聞くとpostできないじゃん！

と思う人も出てくると思いますが安心してください。POSTできます。

できるというか、POSTとmutateを組み合わせて使用します。

これが賢いのが、postした段階でローカルのキャッシュを書き換えつつ、postリクエストを送信。

その後データの再検証を行います。

そのため、エンドユーザーはpostの結果を待つことなく更新後のデータが表示されます。

```jsx
import useSWR, { mutate } from 'swr'

function Profile () {
  const { data } = useSWR('/api/user', fetcher)

  return (
    <div>
      <h1>My name is {data.name}.</h1>
      <button onClick={async () => {
        const newName = data.name.toUpperCase()
        
        // update the local data immediately, but disable the revalidation
        mutate('/api/user', { ...data, name: newName }, false)
        
        // send a request to the API to update the source
        await requestUpdateUsername(newName)
        
        // trigger a revalidation (refetch) to make sure our local data is correct
        mutate('/api/user')
      }}>Uppercase my name!</button>
    </div>
  )
}
```

## 仕組み

先ほどキャッシュ戦略のお話をしましたが、useSWRがどのようにして実現しているかをみてみます。

[https://github.com/vercel/swr](https://github.com/vercel/swr)

use-swr.ts

```
function useSWR<Data = any, Error = any>(
  ...args:
    | readonly [Key]
    | readonly [Key, Fetcher<Data> | null]
    | readonly [Key, SWRConfiguration<Data, Error> | undefined]
    | readonly [
        Key,
        Fetcher<Data> | null,
        SWRConfiguration<Data, Error> | undefined
      ]
): SWRResponse<Data, Error> {
  const [_key, fn, config] = useArgs<Key, SWRConfiguration<Data, Error>, Data>(
    args
  )
  const cache = config.cache

// (めちゃくちゃ長いのでバッサリ中略)

  return state
}
```

config.ts

```tsx
// config
const defaultConfig = {

// (長いのでバッサリ前略)
 
  // providers
  fetcher,
  compare: dequal,
  isPaused: () => false,
  cache: wrapCache(new Map()),

  // presets
  ...webPreset
} as const
```

cache.ts

```tsx
export function wrapCache<Data = any>(provider: Cache<Data>): Cache {
  // We might want to inject an extra layer on top of `provider` in the future,
  // such as key serialization, auto GC, etc.
  // For now, it's just a `Map` interface without any modifications.
  return provider
}
```

なるほど、シンプルなMapらしいです。（将来的に変わりそうな雰囲気を感じます）


## 置き換えの例

React, TypeScriptです


### 標準的なGET

元の実装

```jsx
const [sample, setSample] = React.useState<SampleDTO | null>(null);

// (中略)

const { call: fetchSample } = useAPI(() => SampleAxiosGet(SampleId), {
    onError: () => showError(),
    onSuccess: (data) => setSample(data)
  });

// (中略)

useMount(() => {
    fetchSample();
  });

// 他にも reloadの処理とか…
```

元はこんな感じでした。なんかスッキリしているようにも見えますが`SampleAxiosGet`の中身が結構大変だったり、マウント時にfetchしてくるように設定が必要だったり、reloadを明示的に定義して色々なところで渡したり。キャッシュも各PageでuseStateを用いてどうにかしていたり。という状態でした。



置き換え実装

```jsx
const { sampleData, isError, reload } = useSampleApi(sampleId);

// (中略)

// もとのerrorハンドリングに則るため追加
React.useEffect(() => {
    if (isError) {
      showError();
    }
  }, [isError, showError]);

// 取得したデータをメモ化
  const sample = React.useMemo(() => {
    if (sampleData && !isError) {
      return sampleData;
    }
    return undefined;
  }, [sampleData, isError]);
```

実際使う場面はこんな感じ


```tsx

export default function useSampleApi(id: number | undefined) {

	// もともと使用しているaxiosの関数を使うため独自定義
  const fetcher = React.useCallback((_, id) => {
    return SampleAxiosIdGet(id).then((res) => res.data);
  }, []);

	// 一意なkeyを設定する
  const key = 'sampleAxiosIdGet';
  const { data, error, mutate } = useSWR(id ? [key, id] : null, fetcher);
  
	// mutateを呼ぶことで、再検証できる
  const reload = React.useCallback(() => mutate(), [mutate]);

  return {
    jobData: data,
	// loadingはエラーもdataもまだ帰ってきていないことで判断
    isLoading: !error && !data,
    isError: !!error,
    error,
    reload
  };
}
```


### POSTを含んだ実装


```tsx

// （前略）

export default function useSamplePostApi(sampleId: number | undefined) {
  const key = 'samplePostApi';

  const fetcher = (_: string, id: number) => {
    return SampleAxoisIdGet(id).then((res) => res.data);
  };

  const { data, error, mutate } = useSWR(sampleId ? [key, sampleId] : null, fetcher, {
	// 初期値を設定するためinitalDataを使用。
    initialData: getDefaultValue(sampleId),
	// それだけでは検証してくれないのでその他設定を追加
    revalidateOnMount: true,
    revalidateOnFocus: false,
    shouldRetryOnError: false
  });

  const reload = React.useCallback(() => mutate(), [mutate]);

// POSTリクエストを送りつつ、mutateを実行する。
  const editForm = React.useCallback(
    async (id: number, param: SampleForm) => {
      const result = await SampleAxiosApiIdPut(id,param);
      const editedItem = result.data;
// 事前にバインドされたmutateを使用する場合はkeyは不要なので、データを渡すだけでOKです。
      await mutate({ ...editedItem });
    },
    [mutate]
  );

  return {
    sampleData: data,
    editForm,
    isLoading: !error && !data,
    isError: !!error,
    error,
    reload
  };
}
```

## 気をつけるべきこと、実装時の失敗談

他にもハマるポイントはあると思いますが、ぱっと思い出すものを書いておきます。


### idがない場合

このようにidでデータの取得をする場合、
idが　nullだと空文字でキャッシュされてしまうので、バグの原因になります。
タイミングによってはidが渡される前にuseSWRが実行されるので考慮する必要があります。
空文字がkeyとみなされ、キャッシュのタイミングがおかしかったり、他と共用してしまったり。

（他にも同様のkeyが貼られている場合バグります）

```jsx
const fetcher = React.useCallback((_, id) => {
    return SampleIdGet(id).then((res) => res.data);
  }, []);
  const key = 'keyName';
  const { data, error, mutate } = useSWR(id , fetcher);
```

ちなみにswrのコードをみると、ここにkeyの記載があります。

空文字入れてますよね…？？？（なんで…）

src/libs/serialize.ts

```tsx
export function serialize(key: Key): [string, any, string, string] {
  let args = null
  if (typeof key === 'function') {
    try {
      key = key()
    } catch (err) {
      // dependencies not ready
      key = ''
    }
  }

  if (Array.isArray(key)) {
    // args array
    args = key
    key = hash(key)
  } else {
    // convert falsy values to ''
    key = String(key || '')
  }

  const errorKey = key ? 'err@' + key : ''
  const isValidatingKey = key ? 'req@' + key : ''

  return [key, args, errorKey, isValidatingKey]
}
```

### 初期値を入れてから更新されない件

initialDataを入れるだけだと、再検証してくれないので、mount時に検証するように設定する必要があります

```jsx
 const { data, error, mutate } = useSWR(Id ? [key, jobId] : null, fetcher, {
    initialData: getDefaultValue(Id),
    revalidateOnMount: true,
    revalidateOnFocus: false,
    shouldRetryOnError: false
  });
```



## 所感
確かに単純なfetchやキャッシュはだいぶ楽になりますが、
まだ内部を理解しないとバグる箇所が多く、もう少し丁寧にハンドリングできるようになってくれたらこっちで対応することも少なくて済むのに…と思う場面も結構あります。外部のツールを使うなら当然なのですが…
やってくれそうな雰囲気だけ出して実は何もやってないところが何箇所かありました。（完全自分の感覚ですが）

他にもmutateが結構汎用性高く、意図しない使い方ができてしまったり、独自ルールもそこそこあるように思います。

この辺を解消するために、useSWRをそのまま使うのではなく、自分でwrapして制約を加えたりハンドリングを加えたりしながら使っていくのが、現時点での最適解かと思いました。






## 参考にしたページ等

[SWR: React Hooks for Data Fetching](https://swr.vercel.app/)

[https://blog.jxck.io/entries/2016-04-16/stale-while-revalidate.html](https://blog.jxck.io/entries/2016-04-16/stale-while-revalidate.html)

[https://datatracker.ietf.org/doc/html/rfc5861](https://datatracker.ietf.org/doc/html/rfc5861#page-2)

[https://web.dev/stale-while-revalidate/](https://web.dev/stale-while-revalidate/)