---
title: "TypeScriptの型述語をちょっと理解した"
emoji: "🧩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

# TypeScript の型述語をちょっと理解した

## はじめに

TypeScript の型述語をちょっと理解したので、意味や使い方についてまとめます。

## 型述語とは

サバイバル TypeScript の[型述語の説明](https://typescriptbook.jp/reference/functions/type-guard-functions)について見てみると、以下のように書いてあります。

> 型述語という言葉を分解してみると「型+述語」となります。つまり型についての述語です。この述語という用語は元々は論理学に由来するものであり、その意味を知ることで型ガード関数についての理解を深めることができます。
>
> 元々、述語(predicate)とは、論理学において対象が持つ属性や関係などを表現するものです。たとえば、「X は素数である(X is a prime number)」という命題 P があったとき、X を変数として P(x)のように述語を表現できます。この述語 P(X)は変数 X が素数の 3 などであれば真を返し、非素数の 4 などであれば偽を返します。これはまさに真か偽(真理値)を返す関数です。
>
> このように述語とは変数を含んだ命題(=真理値を持つ判断)のことです。型述語(型についての述語)とはそのまま「型を変数に取る命題」ということができます。x is number のような型述語は「x は number 型である」という変数 x が持つ型についての判断を表現しています。

つまり、型述語とはユニオン型を取る変数が今何の型なのかを TypeScript に明示するための機能であると言えます。

## 使用法

型述語の使用法は以下の通りです。

```ts:isNumber.ts

function isNumber(x: unknown): x is number {
  return typeof x === "number";
}
```

isNumber 関数の戻り値の場所に見慣れない`x is number`という記述があると思います。
これが型述語の使用法となっており、`boolean` 型を返す関数の戻り値の型を記述する部分に`hoge is [示したい型]`とすることで `hoge` が示したい型になっていることを TypeScript に明示します。

しかし、この時 TypeScript が判断することは型が正しいかだけであり、実行時の値の整合性については保証してくれません。

## 使用例

React と TypeScript を使用してマインスイーパーを作っていたときにつまずいた事を例に挙げて説明をします。

まず、型述語を使用した完成形について示します。

```ts:gameSettings.ts
export type GameSettings = {
  rows?: number;
  cols?: number;
};
```

```ts:validation.ts
import { GameSettings } from "./gameSettings.ts";

export function isGameSettingsDefined(input: GameSettings): input is {
  rows: number;
  cols: number;
} {
  return input.rows !== undefined && input.cols !== undefined;
}
```

```ts:main.ts
import type { GameSettings } from "./gameSettings.ts";
import { isGameSettingsDefined } from "./validation.ts";

const hoge: GameSettings = {
  rows: 3,
  cols: 3,
};

console.log(hoge.rows);

if (isGameSettingsDefined(hoge)) {
  console.log(hoge.rows);
}
```

ここで、gameSettings.ts は `GameSettings` 型として `rows` と `cols` というメンバを持っています。
`rows` と `cols` はオプションなのでこれらの型は `number | undefined` 型を取ります。

この時、main.ts で `rows` を `number` 型として別の関数に渡したい時はどうすればよいでしょうか？

main.ts を

```ts:main.ts
if (hoge.rows !== undefined && hoge.cols !== undefined) {
  console.log(hoge.rows);
}
```

とすれば、TypeScript は if 文の中で `rows`,`cols` が `undefined` ではなく `number` 型であることを理解してくれるため、console.log 内の `hoge.rows` をホバーした時型を `number` として認識してくれていることが分かります。

しかし、`rows !== undefined && cols !== undefined`を関数 isGameSettingsDefined としてそのまま validation.ts に切り出してしまうと、TypeScript は`if(isGameSettingsDefined(hoge))`としても `rows` の型が `number` 型に絞られていることを理解してくれず、`hoge.rows` をホバーしても `number | undefined` 型と表示されてしまいます。

そこで、型述語の出番です。`isGameSettingsDefined` 関数を

```ts:validation.ts
function isGameSettingsDefined(input: GameSettings): input is {
  rows: number;
  cols: number;
}
```

として型述語を利用しています。

今回 `input` は `GameSettings` 型で、中のメンバの `rows`,`cols` の型を示したいので`input is {~}`となっています。
このようにすることで、main.ts において TypeScript から `undefined` である可能性を示すエラーを受け取らないようにすることができます。

## おわりに

今回は型述語についてまとめました。undefined や null の場合分けが書きやすくなるため、今後のコードベースに取り入れていきたいです。

私の解釈なので、もしかしたら間違っている部分があるかもしれません。間違いがありましたら、是非コメント等で指摘して頂けると助かります。
