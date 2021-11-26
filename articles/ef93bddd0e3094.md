---
date updated: '2021-11-27T05:16:09+09:00'

---

## モチベーション

1. github actionで、`secrets.GITHUB_TOKEN` の権限を超えてGitHubを操作する場合、Personal Access Tokenを使って操作することが多い。
2. Personal Access Tokenはセキュリティの都合上、有効期限を設定することがベターとされており、無期限で作成したくない。
3. 個人で管理するrepoなどであれば↑でも問題ないが、Organizationで複数人が管理する場合、個人のAccess Tokenを使うことはそもそも避けたい。
4. GitHub Appsを用いてチーム管理Tokenを払い出す（意訳）ことで、Personal Access Tokenに依存せずに、`secrets.GITHUB_TOKEN` の権限を超えてGitHubを操作できるようにする。

## ゴール

Organization配下のリポジトリ上のGitHub Actionsで、`secrets.GITHUB_TOKEN`の権限を超えた操作が行えている。

## 流れ

1. GitHub AppsをOrganization配下に作成する
2. Organizationに、[1]で作成したGitHub Appsをインストールする
3. GitHub AppsのApp IDと、発行したPrivate Keyを取得し、secretsに登録する
4. GitHub Actions上でtokenを取得する

## GitHub AppsをOrganization配下に作成する

<https://github.com/organizations/<org_name>/settings/apps/new> より、GitHub Appsを作成する。

例として、以下情報で設定を行っている。

- org: harvestasya
- GitHub App name: `sample app`
- Homepage URL: `https://example.com`
- Webhook/Active: false

App NameとHomepage URLは任意のものを入れる

![](Pasted%20image%2020211127045245.png)

Permissionについては、

- (GitHub ActionsがPrivate repository上で実行される場合)　`Repository permissions -> Administration` を `Read-only` に設定する
  - ![](Pasted%20image%2020211127045149.png)
- その他については、GitHub Actions上で行いたい操作に基づいて権限を設定する
  - org memberを管理したい場合は、 `Organization permissions -> Members` など。

`Where can this GitHub App be installed?` については、自orgでのみ使えれば良いので `Only on this account` を選択する。

![](Pasted%20image%2020211127045414.png)

## Organizationに、作成したGitHub Appsをインストールする

GitHub Appを作成したら、左のメニューにある `Install App` に行き、Installボタンを押してOrganizationにインストールする.

![](Pasted%20image%2020211127045621.png)

どのリポジトリをreadさせるかは任意で。

![](Pasted%20image%2020211127045650.png)

## GitHub Appsを操作するためのPrivate keyを発行する

GitHub Appsでtoken取得を行うために、Private keyを発行する必要がある。

GitHub AppsのGeneral項目を下にたどると、 `Private keys` 項目があるので、 `Generate a private key`で秘密鍵を作成する。

![](Pasted%20image%2020211127045831.png)

できたやつ

![](Pasted%20image%2020211127045851.png)

## GitHub AppsのApp IDと、発行したPrivate Keyを取得し、secretsに登録する

ここから、GitHub Actionsを動かすリポジトリのSettingsに移動する。

`Organization secrets` か `Repository secrets` かは任せるが、とにかくSecretを登録する。

![](Pasted%20image%2020211127050101.png)

- APP_ID: 154715
- PRIVATE_KEY: `-----BEGIN RSA PRIVATE KEY-----` で始まる複数行

![](Pasted%20image%2020211127050203.png)

## GitHub Actions上でtokenを取得する

github actions上でGitHub Appsによるtokenを取得するのに、 `tibdex/github-app-token` のactionsを利用する。

とりあえず取得するだけのサンプルworkflowを下記に記載する

```yaml
name: CI
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
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

`uses: tibdex/github-app-token@v1` のoutputにtokenが格納されるため、 `${{ steps.generate_token.outputs.token }}` とかでtokenを持ってきて、任意の認証に使えばよい。

![](Pasted%20image%2020211127051620.png)

## さいごに

Personal Access Tokenは初動はとてもコストが低くて便利だが、組織でメンテナンスをしていく場合は後の運用コストのほうが高くなるため、GitHub Appsを用いてメンテナンスするほうが中長期的に見て運用コストは下がる。
GitHub Appsはまだメジャーに使われていない印象ではあるが、一度使い方を覚えてしまえばこちらのほうがいろいろと便利。