---
title: "Slack にファイルを送る GitHub Action を作成した"
emoji: "📬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubactions", "slack"]
published: true
---

## 作成背景

[files.upload API](https://api.slack.com/methods/files.upload) が deprecated になってしまいました。

その代わりに、
- [files.getUploadURLExternal](https://api.slack.com/methods/files.getUploadURLExternal)
- [files.completeUploadExternal](https://api.slack.com/methods/files.completeUploadExternal)

を使うこととなりました。

非推奨になった背景については以下の記事が参考になりました。

https://zenn.dev/slack/articles/7ce5065cc4daa7

この影響で、愛用していた https://github.com/adrey/slack-file-upload-action が、  
中で呼んでいる API が `files.upload` であったために、新規アプリでの利用ができなくなってしまいました。

#### ファイルアップの流れ

1. `files.getUploadURLExternal` でファイルアップロードに利用する Upload 用の URL とファイルIDを取得し
2. Upload 用 URL にファイルを渡してアップロード
3. `files.completeUploadExternal` で、1 で取得したファイルIDとチャンネルIDを渡してアップロードを完了

という流れです。

## Create Action

慣れているため、TS で action を作成することにしました。  
また、公式が出している [Creating a JavaScript action](https://docs.github.com/en/actions/sharing-automations/creating-actions/creating-a-javascript-action) が参考になりました。

Project の雛形作成は、以下の記事が参考になりました。

https://zenn.dev/ring_belle/articles/github-actions-ts

#### 引数を受け取れるようにする

一旦はシンプルな実装を目的にしているため、最小限の引数のみで構成します。

- `token` Slack App の Token
- `path` 送りたいファイルのパス
- `channel_id` 送付されるチャンネルのID

```yaml:action.yaml
name: 'yanskun/slack-file-upload-action'
description: "GitHub Action for uploading a file to Slack"
author: "yanskun"
branding:
  icon: file
  color: orange

inputs:
  token:
    description: Slack Bot Token
    required: true
  path:
    description: file path
    required: true
  channel_id:
    description: Slack Channel ID
    required: true

runs:
  using: "node20"
  main: "dist/index.js"
```

取得方法は、上記の記事でも紹介されていますが、

```ts:src/index.ts
import { getInput, setFailed } from "@actions/core";

const path = getInput("path");

if (!path) {
  setFailed("No filename provided!");
}

const channelId = getInput("channel_id");

if (!channelId) {
  setFailed("No channel ID provided!");
}
```

取得できなかったら、エラーを返すようにしています。

#### Slack Client を用意する

Slack API を呼び出すために、Slack SDK を install します

```bash
npm i @slack/web-api
```

そして Slack Client を呼び出します。

```ts:src/slackClient.ts
import { getInput, setFailed } from "@actions/core";
import { WebClient } from "@slack/web-api";

const token = getInput("token");

if (!token) {
  setFailed("No token provided!");
}

export const slackClient = new WebClient(token);
```

#### Slack API を実行する

```ts:src/index.ts
import { slackClient } from "./slackClient";


// 略

return slackClient.files.getUploadURLExternal({
  length: fileSize,
  filename,
});
```

こんな感じで呼び出すことができます。

#### file を用意する

ここは ts-node の File system を使って実装をします。

```bash
npm install @types/node
```

```ts:src/index.ts
import * as fs from "fs";
import FormData from "form-data";

const filename = path.split("/").at(-1) ?? path;

const stats = fs.statSync(path);
const fileSize = stats.size;

const formData = new FormData();
formData.append("file", fs.createReadStream(path), filename);
```

`fileSize` は `files.getUploadURLExternal` で、  
`formData` は、 `files.getUploadURLExternal` の Response である `upload_url` を使った、POST Reqest に渡します。

また、`upload_url` を使う箇所は Slack API とは違うため、
axios を使って POST API を実行します。

```ts:src/index.ts
import axios from "axios";

await axios.post(postFileResult.upload_url, formData, {
  headers: {
    ...formData.getHeaders(),
  },
});
```

## 完成

あとは、これらを組み合わせてあげることで完成します。

```ts:src/index.ts
import { getInput, setFailed } from "@actions/core";
import { slackClient } from "./slackClient";
import * as fs from "fs";
import axios from "axios";
import FormData from "form-data";

const path = getInput("path");

if (!path) {
  setFailed("No filename provided!");
}

const channelId = getInput("channel_id");

if (!channelId) {
  setFailed("No channel ID provided!");
}

const filename = path.split("/").at(-1) ?? path;

function postFile() {
  let fileSize = 0;

  try {
    const stats = fs.statSync(path);
    fileSize = stats.size;
    console.log("File size: ", fileSize);
    console.log("Filename: ", filename);
    return slackClient.files.getUploadURLExternal({
      length: fileSize,
      filename,
    });
  } catch {
    setFailed("Error getting file stats");
  }
}

async function uploadFile(fileId: string) {
  try {
    slackClient.files.completeUploadExternal({
      files: [
        {
          id: fileId ?? "",
          title: filename,
        },
      ],
      channel_id: channelId,
    });
  } catch {
    setFailed("Error uploading file");
  }
}

async function main() {
  const postFileResult = await postFile();
  if (!postFileResult?.ok) {
    setFailed("Error posting file");
  }

  const formData = new FormData();
  formData.append("file", fs.createReadStream(path), filename);

  if (!postFileResult?.upload_url) {
    setFailed("Error getting upload URL");
  } else {
    try {
      await axios.post(postFileResult.upload_url, formData, {
        headers: {
          ...formData.getHeaders(),
        },
      });
      console.log("File uploaded successfully");
    } catch (error) {
      console.error("Error uploading file:", error);
      setFailed("Error uploading file");
    }
  }

  if (!postFileResult?.file_id) {
    setFailed("Error getting file ID");
  } else {
    uploadFile(postFileResult.file_id);
  }
}

main().catch((error) => {
  console.error("Error in main function:", error);
  setFailed("Error in main function");
});
```

かなり汚いですが、こんな感じで仕上がりました。

最後に build を実行して Release して終わりです。

出来上がったものが以下です。

https://github.com/marketplace/actions/yanskun-slack-file-upload-action

## 使い方

例えば、Nuxt などの Project でコーポレートサイトを構築している場合、  
ドキュメントなどの更新に関しては、非技術者にも関心のあることだと思います。

その場合に、  
CI で Release をトリガーに、Latest との差分を取得して、Slack に通知することで、  
いつ変更があったか？などを通知することができます。

主に Repository 内にあるドキュメントを共有したい際に活用できるのではないかなと思っています。

もっと面白い使い方があれば、ぜひ教えてください！
