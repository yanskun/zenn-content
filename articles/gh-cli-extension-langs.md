---
title: "GitHub CLI Extension でユーザーが使用してる言語のリストを取得"
emoji: "🥷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "GitHubCLI", "Go"]
published: true
---

複数のプロジェクトが organization に紐付いていて、  
EM とかをやっていると人事から「あれ？うちってなんの言語使ってるんだっけ？」って頻繁に聞かれます。  
採用資料とかを作ってもらったり、開発に関する説明をしてもらったり

それ以外にも、自分自身がふと気になった時なども、横断で開発していない限り、かなり記憶ベースになってしまいます。  

Repository のページに行くと、その Repository で使われてる言語をパッとみれるので、  
「あー、そうだったそうだった」となれますがポリレポでやってたりすると、もはやどう調べていいのかもわかりません。  
知らないことは調べられないのだなと、無力になります。

## アイデア

- Repository から言語を取得できる
- Organization/User から Repositopry を取得できる

これを組み合わせれば、できそう。

## 実装

### Project 作成

[公式](https://docs.github.com/ja/github-cli/github-cli/creating-github-cli-extensions) の手順に乗っ取って

```bash
gh extension create --precompiled=go gh-langs
```

[前回](https://qiita.com/yanskun/items/a983bf24da9bb95930db) は Rust で書いてみたが、  
Go だとなんか色々楽そうっぽいので、今回は Go

### API Call

以下の API を叩いていく
```bash
gh api users/yanskun/repos
gh api repos/yanskun/dotfiles
```

#### GET Repositories

```go
func getRepositories(client *api.RESTClient, account string) ([]github.Repository, error) {
	var repos []github.Repository
	page := 1

	t, err := findAccountType(client, account)
	if err != nil {
		return nil, err
	}
	endpoint := fmt.Sprintf("%s/%s/repos", t, account)

	for {
		url := fmt.Sprintf("%s?per_page=100&page=%d", endpoint, page)
		response, err := client.Request(http.MethodGet, url, nil)
		if err != nil {
			fmt.Printf("%s is not a valid GitHub username\n", account)
			return nil, err
		}

		decoder := json.NewDecoder(response.Body)
		data := []github.Repository{}
		if err := decoder.Decode(&data); err != nil {
			return nil, err
		}
		if err := response.Body.Close(); err != nil {
			return nil, err
		}

		if len(data) == 0 {
			break
		}

		repos = append(repos, data...)
		page++
	}
	return repos, nil
}
```

#### GET Languages

```go
func getLanguages(client *api.RESTClient, data []github.Repository) (LanguagesList, error) {
	results := make(LanguagesList, 0, len(data))

	var wg sync.WaitGroup

	for _, repo := range data {
		wg.Add(1)
		go func(repo github.Repository) {
			defer wg.Done()

			fullName := repo.GetFullName()
			response, err := client.Request(http.MethodGet, fmt.Sprintf("repos/%s/languages", fullName), nil)
			if err != nil {
				log.Fatal(err)
				return
			}

			decoder := json.NewDecoder(response.Body)
			data := Languages{}
			if err := decoder.Decode(&data); err != nil {
				log.Fatal(err)
				return
			}

			if err := response.Body.Close(); err != nil {
				log.Fatal(err)
				return
			}

			results = append(results, data)
		}(repo)
	}
	wg.Wait()

	return results, nil
}

取得した Repository の言語情報をマージしていく

func sumLanguages(list LanguagesList) Languages {
	results := Languages{}

	for _, languages := range list {
		for lang, lines := range languages {
			results[lang] += lines
		}
	}

	return results
}

type Pair struct {
	Key   string
	Value int
}
```

### Release

```yaml:.github/workflows/release.yml
name: release
on:
  push:
    tags:
      - "v*"
permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: cli/gh-extension-precompile@v1
        with:
          go_version_file: 'go.mod'
```

前回 Rust で作った時は、Build したファイルを Release して、  
`gh extension install gh-xxx` した時に、それを Download する。

というなんかすごいことになっちゃっていたのですが、  

`--precompiled=go` をすると、いい感じの GHA を組んでくれます。  
とても便利、これだけで Go でいいかってなる。

## 実行

```bash
$ gh extension install yanskun/gh-langs

$ gh langs octocat

+------------+---------+
| LANGUAGE   |   LINES |
+------------+---------+
| Ruby       | 204,865 |
| CSS        |  14,950 |
| HTML       |   4,338 |
| Shell      |     910 |
| JavaScript |      48 |
+------------+---------+
| TOTAL      | 225,111 |
+------------+---------+
https:github.com/octocat has 8 repositories
Last updated after 2023-07-26

```

いい感じですね。  

これで今後採用チームから「うちで使ってる言語ってなんでしたっけ？」って言われたら、この結果を元に回答する事ができます。  

## 成果物

https://github.com/yanskun/gh-langs

@[speakerdeck](93453c3b1e124eddbb31127280644c60)
