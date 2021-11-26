---
title: GitHub Apps経由で発行したtokenを用いて、GitHub Actionsで使う
emoji: 🔑
type: tech
topics:
  - githubactions
published: true
date updated: 2021-11-27 05:40
---

## モチベーション

1. github action で、`secrets.GITHUB_TOKEN` の権限を超えて GitHub を操作する場合、Personal Access Token を使って操作することが多い。
2. Personal Access Token はセキュリティの都合上、有効期限を設定することがベターとされており、無期限で作成したくない。
3. 個人で管理する repo などであれば ↑ でも問題ないが、Organization で複数人が管理する場合、個人の Access Token を使うことはそもそも避けたい。
4. GitHub Apps を用いてチーム管理 Token を払い出す（意訳）ことで、Personal Access Token に依存せずに、`secrets.GITHUB_TOKEN` の権限を超えて GitHub を操作できるようにする。

## ゴール

Organization 配下のリポジトリ上の GitHub Actions で、`secrets.GITHUB_TOKEN`の権限を超えた操作が行えている。

## 流れ

1. GitHub Apps を Organization 配下に作成する
2. Organization に、[1]で作成した GitHub Apps をインストールする
3. GitHub Apps の App ID と、発行した Private Key を取得し、secrets に登録する
4. GitHub Actions 上で token を取得する

## GitHub Apps を Organization 配下に作成する

https://github.com/organizations/org_name/settings/apps/new より、GitHub Apps を作成する。

例として、以下情報で設定を行っている。

- org: harvestasya
- GitHub App name: `sample app`
- Homepage URL: `https://example.com`
- Webhook/Active: false

App Name と Homepage URL は任意のものを入れる

![](/images/how-to-use-github-apps-token-in-github-actions/Pasted%20image%2020211127045245.png)

Permission については、

- (GitHub Actions が Private repository 上で実行される場合)　`Repository permissions -> Administration` を `Read-only` に設定する
  - ![](/images/how-to-use-github-apps-token-in-github-actions/Pasted%20image%2020211127045149.png)
- その他については、GitHub Actions 上で行いたい操作に基づいて権限を設定する
  - org member を管理したい場合は、 `Organization permissions -> Members` など。

`Where can this GitHub App be installed?` については、自 org でのみ使えれば良いので `Only on this account` を選択する。

![](/images/how-to-use-github-apps-token-in-github-actions/Pasted%20image%2020211127045414.png)

## Organization に、作成した GitHub Apps をインストールする

GitHub App を作成したら、左のメニューにある `Install App` に行き、Install ボタンを押して Organization にインストールする.

![](/images/how-to-use-github-apps-token-in-github-actions/Pasted%20image%2020211127045621.png)

どのリポジトリを read させるかは任意で。

![](/images/how-to-use-github-apps-token-in-github-actions/Pasted%20image%2020211127045650.png)

## GitHub Apps を操作するための Private key を発行する

GitHub Apps で token 取得を行うために、Private key を発行する必要がある。

GitHub Apps の General 項目を下にたどると、 `Private keys` 項目があるので、 `Generate a private key`で秘密鍵を作成する。

![](/images/how-to-use-github-apps-token-in-github-actions/Pasted%20image%2020211127045831.png)

できたやつ

![](/images/how-to-use-github-apps-token-in-github-actions/Pasted%20image%2020211127045851.png)

## GitHub Apps の App ID と、発行した Private Key を取得し、secrets に登録する

ここから、GitHub Actions を動かすリポジトリの Settings に移動する。

`Organization secrets` か `Repository secrets` かは任せるが、とにかく Secret を登録する。

![](/images/how-to-use-github-apps-token-in-github-actions/Pasted%20image%2020211127050101.png)

- APP_ID: 154715
- PRIVATE_KEY: `-----BEGIN RSA PRIVATE KEY-----` で始まる複数行

![](/images/how-to-use-github-apps-token-in-github-actions/Pasted%20image%2020211127050203.png)

## GitHub Actions 上で token を取得する

github actions 上で GitHub Apps による token を取得するのに、 `tibdex/github-app-token` の actions を利用する。

とりあえず取得するだけのサンプル workflow を下記に記載する

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
jobs:
  terraform:
    name: test
    runs-on: ubuntu-latest

    defaults:
      run:
        shell: bash
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Generate github token
        id: generate_token
        uses: tibdex/github-app-token@v1
        with:
          app_id: ${{ secrets.APP_ID }}
          private_key: ${{ secrets.PRIVATE_KEY }}
      - name: use token
        run: echo "$GITHUB_TOKEN"
        env:
          GITHUB_TOKEN: ${{ steps.generate_token.outputs.token }}
```

`uses: tibdex/github-app-token@v1` の output に token が格納されるため、 `${{ steps.generate_token.outputs.token }}` とかで token を持ってきて、任意の認証に使えばよい。

![](/images/how-to-use-github-apps-token-in-github-actions/Pasted%20image%2020211127051620.png)

## さいごに

Personal Access Token は初動はとてもコストが低くて便利だが、組織でメンテナンスをしていく場合は後の運用コストのほうが高くなるため、GitHub Apps を用いてメンテナンスするほうが中長期的に見て運用コストは下がる。
GitHub Apps はまだメジャーに使われていない印象ではあるが、一度使い方を覚えてしまえばこちらのほうがいろいろと便利。
