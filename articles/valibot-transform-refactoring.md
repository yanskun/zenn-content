---
title: "ややこしい API Request を Valibot の transoform() で解決する"
emoji: "🦋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["valibot", "reacthookform", "schema"]
published: true
---

前回、Valibot を v0.31 への Migration ができた。
https://zenn.dev/yanskun/articles/valibot-migrate-to-031

Migration にて、
`transform()` API は手動で置き換える必要があり、  
その際に、かなりの数の transform() の置換を行い、理解があったまってきていた。  

そんな中で、Valibot の transform() をいい感じに使えた気がするので、紹介する。

## 他の key に影響される transform 

### 前提

以下のような関数があるとする

```ts:post-user.ts
type RequestBody = {
  name: string;
};

export async function postUser(data: RequestBody) {
  await fetch("example.com", {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(data),
  });
}
```

さらに UI が、以下のような形になっているとする

![](/images/valibot-transform-refactoring/sample-form.png)

さらに、日本人だった場合、  
firstName と lastName をあべこべにして、API に投げたい。

となると以下のような構成が思いつく

### API Request 前に加工してあげる

```ts:schema.ts
import * as v from "valibot";

export const schema = v.object({
  name: v.object({
    firstName: v.string(),
    lastName: v.string(),
  }),
  isJapanese: v.boolean(),
});

export type IFormInput = v.InferInput<typeof schema>;
```

```tsx:form.tsx
"use client";

import { useForm, SubmitHandler, FormProvider } from "react-hook-form";
import { valibotResolver } from "@hookform/resolvers/valibot";
import { IFormInput, schema } from "./schema";
import { postUser } from './post-user'

export function Form() {
  const formMethods = useForm<IFormInput>({
    resolver: valibotResolver(schema),
  });
  const { register, handleSubmit } = formMethods;

  const onSubmit: SubmitHandler<IFormInput> = (data) => {
    postUser({
      name: data.isJapanese
        ? `${data.name.lastName} ${data.name.firstName}`
        : `${data.name.firstName} ${data.name.lastName}`,
    });
  };
  return (
    <div>
      <FormProvider {...formMethods}>
        <form onSubmit={handleSubmit(onSubmit)}>
          <div className="name">
            <div>
              <label>First Name</label>
              <input {...register("name.firstName")} />
            </div>
            <div>
              <label>Last Name</label>
              <input {...register("name.lastName")} />
            </div>
          </div>

          <div className="is-japanese">
            <input
              id="default-checkbox"
              type="checkbox"
              {...register("isJapanese")}
            /> <label htmlFor="default-checkbox">日本人ですか？</label>
          </div>

          <button type="submit">Submit</button>
        </form>
      </FormProvider>
    </div>
  );
}
```

`handleSubmit` をする際に、加工をしてあげるパターンである。  
僕は最初はこれを書いていた。  
しかしこれだと、 Form の型と Request の型が分離してしまい、じドメインなのに別で定義しなくてはならず  
せっかく使っている Schema Validate を満足に使いきれていない気がする。  

そこで、

### Schema で Form と Request Body を定義してあげる

```ts:schema.ts
import * as v from "valibot";

export const schema = v.pipe(
  v.object({
    name: v.object({
      firstName: v.string(),
      lastName: v.string(),
    }),
    isJapanese: v.boolean(),
  }),
  v.transform((input) => {
    const { firstName, lastName } = input.name;
    return {
      name: input.isJapanese
        ? `${lastName} ${firstName}`
        : `${firstName} ${lastName}`,
    };
  })
);
export type IFormInput = v.InferInput<typeof schema>;
```

```ts:form.tsx
"use client";

import { useForm, SubmitHandler, FormProvider } from "react-hook-form";
import { valibotResolver } from "@hookform/resolvers/valibot";
import { IFormInput, schema } from "./schema";
import { postUser } from './post-user'

export function Form() {
  const formMethods = useForm<IFormInput>({
    resolver: valibotResolver(schema),
  });
  const { register, handleSubmit } = formMethods;

  const onSubmit: SubmitHandler<IFormInput> = (data) => postUser(data);
  return (
    <div>
      <FormProvider {...formMethods}>
        <form onSubmit={handleSubmit(onSubmit)}>
          <div className="name">
            <div>
              <label>First Name</label>
              <input {...register("name.firstName")} />
            </div>
            <div>
              <label>Last Name</label>
              <input {...register("name.lastName")} />
            </div>
          </div>

          <div className="is-japanese">
            <input
              id="default-checkbox"
              type="checkbox"
              {...register("isJapanese")}
            />
            <label htmlFor="default-checkbox">日本人ですか？</label>
          </div>

          <button type="submit">Submit</button>
        </form>
      </FormProvider>
    </div>
  );
}
```

かなりシンプルになった。  
schema の定義はやや複雑になったが、  
schema に Form が何か？と API Request が何か？がセットで見えるので、  
ごちゃごちゃ感はかなり抑えることができている気がする。

## 終わりに

今回は Valibot を使った実装だが、Zod にも `transform()` や `coerce()` があるので似たようなことはできる。

僕は今回これを通じて、Valibot Schema の定義を少し綺麗に書くことができ、  
JSX 部分の実装と分離させることができて大変満足である。  

ただ、パフォーマンスの差など十分に検討できていないので、その辺りもしっかり調べていきたい。  

今回の成果物は以下

https://github.com/yanskun/valibot-hook-form

https://codesandbox.io/p/github/yanskun/valibot-hook-form/main
