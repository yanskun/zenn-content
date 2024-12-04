---
title: "Storybook の Test Runner を並列実行して、GitHub Actions の実行時間を短縮した"
emoji: "⛄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Test', 'Storybook', 'GitHubAction', 'CICD']
published: false
---

**師走** は言葉の通りで、坊主も走り回るほど12月は忙しいです。  
CI は年がら年中走っています。

そんな CI の話です。

# はじめに

弊社 [Finswer](https://github.com/finswer) では、  
Storybook を使った Integration Test を行っています。

Integration Test はとても便利ですが、困ったことに時間がとってもかかります。

流石にストレスだったので、  
10m くらいかかっていたのを、並列実行することで 2m くらいにできたのでそれを紹介します。

## 元の構成

```
.
├── .github
│   ├── actions
│   │   └── test.yaml
│   └── workflows
│       └── ci.yaml
├── package.json
├── storybook.sh
```

```json:package.json
{
  "scripts": {
    "storybook": "bash ./storybook.sh",
    "storybook:test": "pnpm storybook test",
  }
}
```

Storybook のための実行環境の setup などを Shell Script でまとめています。
`storybook xxx` で、ある程度初期設定が決まった storybook command を実行できるようになっています。 

```sh:storybook.sh
# 略
case "$1" in
    test)
        storybook build --test
        
        concurrently \
            --kill-others \
            --success first \
            --names "STORYBOOK,TEST" \
            --prefix-colors "auto" \
            "http-server storybook-static --port 6006 --silent" \
            "wait-on http://127.0.0.1:6006 && test-storybook"
        ;;
```

```yaml:.github/workflows/ci.yaml
name: CI

# 略

jobs:
  test:
    runs-on: ubunts-latest

    steps:
      - name: integration test
        uses: ./.github/actions/test
```

GitHub Actions に関しては、Composite Action[^1] を使って、  
頻出するような Actions を使いまわせるようにしています。  
Integration Test に関しても同様です。

```yaml:.github/actions/test.yaml
name: Test

runs:
  using: 'composite'

  steps:
    # 略
    # Node と Playwright の setup など

    - name: Component test
      shell: bash
      run: pnpm storybook:test:ci
```

ざっくりこんな感じの構成です。

これの `integration test` と言う step がかなり時間がかかってしまっている。という状態です。  

## 並行実行に向けての準備

### test-storybook の shard option

Storybook の Test runner の Document をみると  
`--shard`[^2]  optinal があります。

`test-storybook --shard=1/8` とすると、全体の 8分の1のテストを実行してくれるもの。

かなり便利そう。

### GitHub Actions の matrix strategies

matrix strategies[^3] とは、  
変数を設定して、Job を変数の組み合わせによる Job の並列実行をしてくれるもの。

公式の使い方を見ると

```yaml
jobs:
  example_matrix:
    strategy:
      matrix:
        version: [10, 12, 14]
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.version }}
```

複数のバージョンに対して、CI を実行することもできるよう。
これもかなり便利ですね。

## 並行実行の設定

GitHub Actions に matrix strategis を設定していきます。
今回は試しに 5分割にしてみます。

```diff yaml:.github/workflows/ci.yaml
name: CI

# 略

jobs:
  test:
    runs-on: ubunts-latest

+   strategy:
+     matrix:
+       shardIndex: [1, 2, 3, 4, 5]

    steps:
      - name: integration test
        uses: ./.github/actions/test
+       with:
+         shardIndex: ${{ matrix.shardIndex }}
+         shardTotal: 5
```

Composite Action が引数を受け取れるようにして、  
受け取った引数を command に渡します。

npm-run-script には `--`[^4] をつけると、 option が渡せるので  

`-- --shard=y/x`

のようにします。

```diff yaml:.github/actions/test.yaml
 name: Test
+inputs:
+  shardIndex:
+    description: "The index of the shard to run (0-based)"
+    default: "1"
+  shardTotal:
+    description: "The total number of shards"
+    default: "1"

 runs:
   using: 'composite'

   steps:
     # 略
     - name: Component test
       shell: bash
-      run: pnpm storybook:test:ci
+      run: pnpm storybook:test:ci -- --shard=${{ inputs.shardIndex }}/${{ inputs.shardTotal }}
```

Shell Script 側でも option を受け取れるようにします。

storybook.sh は第一引数を case 文に使っているので、  
第二引数移行を test-storybook に渡してあげる必要があります。

`sh ./storybook.sh test --options`

case の中に入ったら、第一引数は必要ないので、
`shift`[^5] で破棄します。  

JS の `.shift()`[^6] と一緒で、一番最初を破棄する関数です。

第一引数を破棄したら、引数をそのまま雑に渡してあげます。  
今回は `--shard` しか渡しませんが、他にも渡すことも可能です。  

```diff sh:storybook.sh
## 略
case "$1" in
    test)
+       shift

        storybook build --test
        
        concurrently \
            --kill-others \
            --success first \
            --names "STORYBOOK,TEST" \
            --prefix-colors "auto" \
            "http-server storybook-static --port 6006 --silent" \
-           "wait-on http://127.0.0.1:6006 && test-storybook"
+           "wait-on http://127.0.0.1:6006 && test-storybook $*"
        ;;
```

終わり

実行してみると

![summary](/images/storybook-integration-test/summary.png)
![jobs](/images/storybook-integration-test/jobs.png)

一部ですが、こんな感じになりました

フォルダのような UI もかなり可愛くて嬉しいです。  

もともと、10m かかっていた integration test が、5分割されたので2ｍくらいにすることができました。  
かなり満足な実行時間です。  

[^1]: [Creating a composite action](https://docs.github.com/en/actions/sharing-automations/creating-actions/creating-a-composite-action)
[^2]: [Test runner](https://storybook.js.org/docs/writing-tests/test-runner)
[^3]: [Running variations of jobs in a workflow](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/running-variations-of-jobs-in-a-workflow)
[^4]: [npm-run-script - description](https://docs.npmjs.com/cli/v8/commands/npm-run-script?v=true&utm_source=chatgpt.com#description)
[^5]: [Bash - shift](https://ss64.com/bash/shift.html)
[^6]: [JavaScript Array shift()](https://www.w3schools.com/jsref/jsref_shift.asp)
