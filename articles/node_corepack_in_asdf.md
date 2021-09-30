---
title: "asdf管理下のNode.jsでCorepackを使ったメモ"
emoji: "🙂"
type: "tech"
topics: ["nodejs", "corepack", "asdf", "yarn", "pnpm"]
published: true
---

:::message
この記事では、Node.js 16.10.0時点で実験的扱いの機能について書いています。もし参考にされる場合は、最新の状況を各自で調べ直して下さい。
:::

先日、Node.jsの[16.9.0](https://github.com/nodejs/node/blob/v16.9.0/doc/changelogs/CHANGELOG_V16.md#16.9.0)でCorepackが実験的扱いで取り込まれましたね。

以前から気になっていたのでこの機会に試そうと思いまして。私はasdfで諸々のバージョンを管理しているのですが、Corepackでyarn、pnpmを使うまでにひと手間必要だったので、そのメモです。

解決までの過程で調べたことも書き残しておきます。

Node.js、Corepack、asdf-vmについては以下にリンクを貼っておきます。

https://nodejs.org/

https://nodejs.org/dist/latest-v16.x/docs/api/corepack.html

https://asdf-vm.com/

# 環境
* OS: Debian GNU/Linux 10.10 (buster)
* asdf-vm: v0.8.0
* 導入済みasdfプラグイン
  * [nodejs](https://github.com/asdf-vm/asdf-nodejs)
  * [yarn](https://github.com/twuni/asdf-yarn)
  * [pnpm](https://github.com/jonathanmorley/asdf-pnpm)

yarn、pnpmはグローバルインストールしていません。

# 手順
まずは手順から。

## Node.jsの16.9.0以降をインストール

```shell-session
> asdf install nodejs 16.10.0
```
普通にasdfでインストールします。記事作成時点で16.10.0が出ていました。


## 適当なディレクトリを作って移動します
```shell-session
> mkdir tekitou_na_dexirectori ; cd $_
```

## インストールしたNode.jsをそのディレクトリで有効化します
```shell-session
> asdf local nodejs 16.10.0
```

## Corepackを有効化します
```shell-session
> corepack enable
```
16.10.0の時点ではCorepackは実験的機能なので、使う前に有効化する必要があるとのことです。この手順が将来的にどうなるかは分かりません。

## asdfのshimを作り直します
```shell-session
> asdf reshim nodejs
```
通常はあまり使わないでしょう（少なくとも私は）。corepackでインストールしたyarn、pnpmを使うためにこれが必要です。

## ではpackage.jsonを作ってパッケージマネージャとバージョンを指定しましょう
```json:package.json
{
  "packageManager": "yarn@3.0.0"
}
```
Corepackで一番やりたいやつですね。

## 動作確認
```shell-session
> yarn --version
3.0.0
```
特にインストールコマンドなどを実行する必要はありませんね。初回実行時は若干時間がかかります。 ```.yarnrc.yml```の```yarnPath```による実行ファイルの指定も不要です。


他のパッケージマネージャやバージョンも試しにやってみましょう。
```json:package.json
{
  "packageManager": "pnpm@6.15.1"
}
```
```shell-session
> yarn --version
Usage Error: This project is configured to use pnpm

$ yarn ...
> pnpm --version
6.15.1
```

私は```asdf reshim```コマンドを使ったことが無く、と言いますか今回の件で初めて知ったぐらいなので、ここにたどり着くまでにしばらくハマッてました。

# reshim が必要な理由
asdfは仕組みとして、インストールしたコマンドを、```~/.asdf/shims```に置かれているshimと呼ばれるファイルを経由して実行します。

このshimは各ツールをインストールしたタイミングで更新しているようです。上記の手順でいうと```asdf install nodejs 16.10.0```を実行した時ですね。この時にダウンロードされたファイルをもとに自動的に作成・更新していると思われます。

具体的に言えば、今回の場合だと```~/.asdf/installs/nodejs/16.10.0/bin```にあるファイルです。

これが、インストール直後のタイミングだと
```shell-session
> ls ~/.asdf/installs/nodejs/16.10.0/bin
corepack  node  npm  npx
```
という状態になっているので、これを元にshimを更新しているようです。

ここで```corepack enable```を実行してみましょう。
```shell-session
> corepack enable
> ls ~/.asdf/installs/nodejs/16.10.0/bin
corepack  node  npm  npx  pnpm  pnpx  yarn  yarnpkg
```
と、bin以下に、yarn、pnpmのファイルが増えます。これはこれでCorepackが用意したshimですね。

このように、インストール直後には存在しないファイルを、asdfのshimに反映させる必要があるので```asdf reshim nodejs```を実行する必要があるわけですね。


将来的にCorepackの実験的扱いが無くなった時に、このreshimの手順は不要になるんじゃないかと期待していますが、当面は様子見ですねー。


# 16.9.0より前のNode.jsやasdfのyarn, pnpmプラグインとの共存
Corepackを有効化した場合も、yarn、pnpmのプラグインのアンインストールは不要で、共存というか使い分けが可能です。

asdf使いの人はご存知でしょうが、使うコマンドのバージョンの切り替えは```.tool-versions```というファイルで制御されています。

## Node.js 16.9.0以降 corepack有効化の場合
上述の手順通りではありますが、以下のような感じになっていればよいでしょう。

```plain:.tool-versions
nodejs 16.9.0
```
ここにyarnプラグイン指定が無い状態にしましょう。Node.js管理下つまりCorepackでインストールしたyarnが使われます。

```json:package.json
{
  "packageManager": "yarn@3.0.0"
}
```

## Node.js 16.9.0より前(Corepack無し)、yarn v1系を使う場合
```plain:.tool-versions
nodejs 14.17.6
yarn 1.22.15
```
こっちはasdf管理下のyarnを使いたいのでこうします。


## Node.js 16.9.0より前(Corepack無し)、yarn v2以降を使う場合
```plain:.tool-versions
nodejs 14.17.6
yarn 1.22.15
```
同様にasdf管理下のyarnを使います。
```shell-session
> yarn set version 3.0.0
```
asdf的にはv1系最新をインストールした状態にして、そのyarnのコマンド側の設定でv2以降を使うようにします。

asdfで直接指定できないのは仕様ですね。

https://github.com/twuni/asdf-yarn

yarnのv2以降のインストール・設定などは既に色々解説などがありますのでそちらをご覧ください。

## 要点
nodejsプラグイン管理下のyarnと、yarnプラグイン管理下のyarnの、どちらか1つだけを参照するように```.tool-versions```を設定する、ということですね。

両方参照できる状態にするのは多分やめたほうがいいでしょう。仕様的にどちらが優先されるのかがよく分からなかったので……。

まあちょっとややこしい気もしますが、Corepackを有効化して試すディレクトリと、ちょっと古いNode.jsを使うディレクトリとで、特に無理なく切り替えられます。

# 感想
というわけで自分の環境でCorepackを試したときのあれやこれやでした。

個人的に、yarn v2以降のバージョン指定はちょっと微妙だなー、って思ってたので、```package.json```に一本化出来るCorepackには期待が大きいですね。

流石に今すぐ完全移行ってのは難しいにしても、asdf管理で、当面の実験用のディレクトリと元の設定で運用するディレクトリとを分けられたので一安心です。


ちなみに私はpnpm派です。
