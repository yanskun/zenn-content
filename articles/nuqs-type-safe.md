---
title: "型安全な SearchParams を扱える nuqs"
emoji: "🛟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["next", "nuqs"]
published: true
---


型安全に SearchParams を扱える [nuqs](https://nuqs.47ng.com/) の紹介をします。

## 背景

SearchParams を扱う上で、僕は一生 typo に怯えます。

例えば、searchParams でステータスを管理するような設計の画面の際に、
用意していない param を渡すこともできてしまいます。

Stringリテラルを扱うので避けられません。

怯えすぎて、

```ts
import { useSearchParams } from 'next/navigation';
import { Output, picklist, safeParse } from 'valibot';

const ValueSchema = picklist(['normal', 'warn', 'error']);
type Value = Output<typeof ValueSchema>;

type SearchParams = {
  value: Value;
};

export function useValueSearchParams(): {
  setTypedSearchParams: (params: Partial<SearchParams>) => string;
  value: Value;
} {
  const searchParams = useSearchParams();

  const setTypedSearchParams = (params: Partial<SearchParams>) => {
    const newParams = new URLSearchParams(searchParams.toString());

    if (params.value) newParams.set('value', params.value);

    return newParams.toString();
  };

  const value = searchParams.get('value');
  const valueResult = safeParse(ValueSchema, value);

  return {
    setTypedSearchParams,
    value: valueResult.success ? value.output : 'value'
  };
}
```

こんなものをたまに作ってました。

## nuqs

冒頭に記載した通り、nuqs とは
SearchParams を型安全に扱えるものです。

公式の Top ページが Demo になっているので、ぜひ触ってみてください。
https://nuqs.47ng.com/?count=1&hello=nuqs

### useSearchParams との違い

#### next/navigation での実装

```tsx
import { useRouter, usePathname, useSearchParams } from "next/navigation";

type PathType = "normal" | "warn" | "error";

export default function Home() {
  const searchParams = useSearchParams();
  const pathname = usePathname();
  const router = useRouter();

  const handleClick = (value: PathType) => {
    const newSearchParams = new URLSearchParams(searchParams.toString());
    newSearchParams.set("value", value);

    router.push(`${pathname}?${newSearchParams}`);
  };

  const pathValue = searchParams.get("value");

  return (
    <main>
      <div>
        <button onClick={() => handleClick("normal")}>Normal</button>
        <button onClick={() => handleClick("warn")}>Warning</button>
        <button onClick={() => handleClick("error")}>Error</button>
      </div>

      <p>{pathValue}</p>
    </main>
  );
}
```

pathname や、type 定義によって、Stringリテラルの埋め込みを最小限にしています。  
せいぜいこれくらいなのではないかと思う。画面跨ぎでやる場合は、上述したような関数を定義するなどやり方はあると思います。  

#### nuqs での実装

```tsx
import { useQueryState, parseAsStringLiteral } from "nuqs";

export default function Home() {
  const [value, setValue] = useQueryState(
    "value",
    parseAsStringLiteral(["normal", "warn", "error"]).withOptions({
      history: "push",
    })
  );

  return (
    <main className="flex flex-col min-h-screen items-center w-full">
      <div className="flex items-center justify-between p-24 w-full">
        <button onClick={() => setValue("normal")}>Normal</button>
        <button onClick={() => setValue("warn")}>Warning</button>
        <button onClick={() => setValue("error")}>Error</button>
        {/* <button onClick={() => setValue("none")}>None</button> 怒られる */}
      </div>

      <p>{value}</p>
    </main>
  );
}
```

nuqs を使うとかなりシンプルになりました。  
型の定義はなくなりましたし、 `withOptions` で、history.push も行ってくれます。  

### 最後に

Stringリテラルで history.push をすると、  
ずっと、typo に怯えることになります。  
じゃあ拡張を入れればいいじゃないか？テストをたくさん書こう。  
は賛成ですが、あまり前向きな取り組みとは言い難いかなと思います。  

正直、そこにストレスを感じていたし、関数も自前で用意していたくらいなので、  
nuqs のような素晴らしい汎用的なものを先につくりたかったな。と悔しい気持ちになりました。  

感謝しながら使います。

### 余談

nuqs の発音や、意味について記載がありました。
https://nuqs.47ng.com/docs/about

> I realised after the fact that the word nuqs is Urdu for “flaw” or “defect”. It’s a good reminder that:

nuqs はヒンドゥー語で、「欠陥」などの意味を持つそうで、  
命名する前に最初に調べておけばよかった。とのことです。  
ウケる
