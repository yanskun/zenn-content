---
title: "GitHub API x Ollama で、自分が先週何やったかを思い出してくれる GitHub CLI Extension を作った"
emoji: "🗯️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ollama", "githubcli", "go"]
published: true
---

週の初めに、先週自分が何やったか？を思い出すのに時間がかかったりします。

github.com/pulls とかをみて思い出したりしています。

正直面倒なので、プログラムにやってもらった方が良いですね

## 作ったもの

https://github.com/yanskun/gh-recall

GitHub API や git command を使って、Repository の

- Pull Request
- Issue
- Commit Message

を取得して、

Ollama を使って要約してもらいます。

https://ollama.com/

### どんなものか

僕の dotfiles 配下で実行してみると、

```shell
-❯ gh recall -l ja
# 2025-01-23 ~ 2025-01-24

## 🔄 調整とメンテナンス

この期間中、ユーザーはプロジェクトにおける「lspsaga」の使用を調整しました。具体的には、「chore: too light lspsaga」と「chore: use lspsaga」というタスクが含まれています。これらは、プロジェクト内で「lspsaga」の設定や機能を改善し、最適化するための作業です。

## 📋 問題

この期間中には特定の問題報告がありません。したがって、ユーザーは既存の機能や設定に対して直接的な修正を行わず、調整と改善に焦点を当てました。

## ✨ 機能強化

「lspsaga」の導入および最適化は、ユーザーインターフェースや機能性の向上を目的としています。これにより、プロジェクト全体のパフォーマンスが向上し、使いやすさも改善されることが期待されます。
---
これらの作業は、ユーザーの技術的な専門性とプロジェクトへの貢献を示しています。

```

こんな感じで、要約してくれます。

## 作り方

```bash
$ gh extension create --precompiled=go gh-recall
```

Go の Project が出来上がるので、
書いていきます。

構成としてはこんな感じ。

```
.
├── config
│   └── config.go
├── git
│   └── git.go
├── go.mod
├── go.sum
├── main.go
└── ollama
    └── ollama.go

```

| package | やること                                     |
| ------- | -------------------------------------------- |
| main    | option を処理                                |
| git     | GitHub API へのリクエスト                    |
| ollama  | Prompt や Local の Ollama API へのリクエスト |
| config  | `config.toml` の読み込み                     |

### git

git 配下はこんな感じ

```go
type PullRequest struct {
	Title     string `json:"title"`
	Body      string `json:"body"`
	State     string `json:"state"`
	CreatedAt string `json:"createdAt"`
	IsDraft   bool   `json:"isDraft"`
}

type Issue struct {
	Title     string `json:"title"`
	Body      string `json:"body"`
	State     string `json:"state"`
	CreatedAt string `json:"createdAt"`
}

type GitService struct {
	startDate string
	endDate   string
}

type GitServiceInterface interface {
	getPullRequests() []PullRequest
	getIssues() []Issue
	getCommits() string
	GenerateSummary() string
}
```

PR, Issue, Commit を取得する関数と、
それをまとめてくれる関数を実装していきます。

Service にしたのは、日付を何度も使い回すから

https://github.com/yanskun/gh-recall/blob/main/git/git.go

### ollama

それを ollama 配下に渡して、prompt にしていく。

```go
type Request struct {
	Model    string    `json:"model"`
	Messages []Message `json:"messages"`
	Stream   bool      `json:"stream"`
	Raw      bool      `json:"raw"`
	Template string    `json:"template"`
}

type Message struct {
	Role    string `json:"role"` Content string `json:"content"`
}

type Response struct {
	Model              string    `json:"model"`
	CreatedAt          time.Time `json:"created_at"`
	Message            Message   `json:"message"`
	Done               bool      `json:"done"`
	TotalDuration      int64     `json:"total_duration"`
	LoadDuration       int       `json:"load_duration"`
	PromptEvalCount    int       `json:"prompt_eval_count"`
	PromptEvalDuration int       `json:"prompt_eval_duration"`
	EvalCount          int       `json:"eval_count"`
	EvalDuration       int64     `json:"eval_duration"`
}

type OllamaService struct {
	content  string
	url      string
	model    string
	locale   string
	sections int
}

type OllamaServiceInterface interface {
	GenerateSummaries() string
}
```

https://github.com/yanskun/gh-recall/blob/deb175702a21d86906d5697230300121a0f8a0ca/ollama/ollama.go#L63-L118

ChatGPT とかは日々使ってるが、
プログラミングの中でプロンプトと初めてまともに向き合ったので、めちゃくちゃ苦労しました。

### config

ここは、[viper](https://github.com/spf13/viper) を使って、`config.toml` を読み込んでるだけなので、割愛

https://github.com/spf13/viper

あとは main で、[pflag](https://github.com/spf13/pflag) を使って、 option を受け取れるようにしたり

https://github.com/spf13/pflag

[spinner](https://github.com/briandowns/spinner) を使ってローディングを表示したり、
https://github.com/briandowns/spinner

普段使ってる CLI Tool でよくみる、便利だなって感じたものを真似ているだけです。

## 終わり

も LLM を使った「何か」を作ってみたかったのですが、アイデアが浮かばずにいました。
今回ちょうどいいアイデアが閃いたので、作れてよかったです。

こういった、煩雑なものは LLM も活用して解決していきたいし、
今 prompt としっかり向き合わないと、かなり機を逃している感じもするので、
まだまだ味がするので楽しんでいこうと思います。
