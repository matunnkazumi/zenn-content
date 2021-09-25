---
title: "asdf管理下のNode.jsでCorepackを使ったメモ"
emoji: "😊"
type: "tech"
topics: ["nodejs", "corepack", "asdf", "yarn", "pnpm"]
published: false
---

:::message
この記事では、Node.js 16.10.0 時点で実験的扱いの機能について書いています。もし参考にされる場合は、最新の状況を各自で調べ直して下さい。
:::

先日、Node.jsの[16.9.0](https://github.com/nodejs/node/blob/v16.9.0/doc/changelogs/CHANGELOG_V16.md#16.9.0)でCorepackが実験的扱いで取り込まれましたね。

以前から気になっていたのでこの機会に試そうと思ったのですが、掲題にあるように、私はasdfで諸々のバージョンを管理していまして、Corepackでyarn、pnpmを使うまでにひと手間必要だったので、そのメモです。

解決までの過程で調べたことも書き残しておきます。

Node.js、Corepack、asdf-vmについては以下にリンクを貼っておきます。

https://nodejs.org/

https://nodejs.org/dist/latest-v16.x/docs/api/corepack.html

https://asdf-vm.com/

# 環境
* OS: Debian GNU/Linux 10.10 (buster)
* asdf-vm: v0.8.0
* 導入済み asdf プラグイン
  * [nodejs](https://github.com/asdf-vm/asdf-nodejs)
  * [yarn](https://github.com/twuni/asdf-yarn)
  * [pnpm](https://github.com/jonathanmorley/asdf-pnpm)

yarn、pnpmはグローバルインストールしていません

# 手順
まずは手順から

1. Node.jsの16.9.0をインストール

```bash
> asdf install nodejs 16.10.0
```
普通にasdfでインストールします。


2. 適当なディレクトリを作って移動します
```bash
> mkdir tekitou_na_dexirectori ; cd $_
```

3. インストールしたNode.jsをそのディレクトリで有効化します
```bash
> asdf local nodejs 16.10.0
```

4. Corepackを有効化します
```bash
> corepack enable
```
16.10.0の時点ではCorepackは実験的機能なので、使う前に有効化する必要があるとのことです。この手順が将来的にどうなるかは分かりません。

5. asdfのshimを作り直します
```bash
> asdf reshim nodejs
```
通常はあまり使わないかと思います（少なくとも私は）。corepackでインストールした yarn、pnpm を使うためにこれが必要です

6. ではpackage.jsonを作ってパッケージマネージャとバージョンを指定しましょう
```json
{
  "packageManager": "yarn@3.0.0"
}
```
Corepackで一番やりたいやつですね。

7. 動作確認
```bash
> yarn --version
3.0.0
```
特にインストールコマンドなどを実行する必要はありませんね。初回実行時は若干時間がかかるかと思います。 ```.yarnrc.yml```の```yarnPath```による実行ファイルの指定も不要です。


他のパッケージマネージャやバージョンも試しにやってみましょう。
```json
{
  "packageManager": "pnpm@6.15.1"
}
```
```bash
> yarn --version
Usage Error: This project is configured to use pnpm

$ yarn ...
> pnpm --version
6.15.1
```

# reshim が必要な理由
asdfは仕組みとして、インストールしたコマンドを、```~/.asdf/shims```に置かれているshimと呼ばれるファイルを経由して実行します。

rbenvなど、この手のツールは大体そんな仕組みなんじゃないかと思います。

で、asdfの場合このshimはツールをインストールしたタイミングで更新しているようです。上記の手順でいうと```asdf install nodejs 16.10.0```を実行した時ですね。この時にダウンロードされたファイルをもとに自動的に作成・更新していると思われます。

具体的に言えば、今回の場合だと```~/.asdf/installs/nodejs/16.10.0/bin```にあるファイルです。

これが、インストール直後のタイミングだと
```bash
> ls ~/.asdf/installs/nodejs/16.10.0/bin
corepack  node  npm  npx
```
という状態になっているので、これを元にshimを更新しているようです。

そして```corepack enable```を実行すると、
```bash
> corepack enable
> ls ~/.asdf/installs/nodejs/16.10.0/bin
corepack  node  npm  npx  pnpm  pnpx  yarn  yarnpkg
```
と、bin以下に、yarn、pnpmのファイルが増えます。これはこれでCorepackが用意したshimですね。

このように、インストール直後には存在しないファイルを、asdfのshimに反映させる必要があるので```asdf reshim nodejs```を実行する必要があるわけですね。


もし将来的にCorepackの実験的扱いが無くなった時に、このreshimは不要になるんじゃないかと期待していますが、当面は様子見ですねー。


# 16.9.0より前のNode.jsやasdfのyarn, pnpmプラグインとの共存
asdfでNode.jsを管理している人は、yarnなどもasdfで管理してるんじゃあないかと思います。

Corepackを有効化した場合も、yarn、pnpmのプラグインのアンインストールは不要で、共存というか使い分けが可能です。

asdf使いの人はご存知でしょうが、使うコマンドのバージョンの切り替えは```.tool-version```というファイルで制御されています。

## Node.js 16.9.0以降 corepack有効化の場合
上述の手順通りではありますが、以下のような感じになっていればよいでしょう。

```.tool-version```
```
nodejs 16.9.0
```
ここでyarnが入っていない状態にしましょう。Node.js Corepack管理下のyarnが使われます。

```package.json```
```json
{
  "packageManager": "yarn@3.0.0"
}
```

## Node.js 16.9.0より前、yarn v1系を使う場合
```.tool-version```
```
nodejs 14.17.6
yarn 1.22.10
```
こっちは asdf 管理下のyarnを使いたいのでこうします。


## Node.js 16.9.0より前、yarn v2以降を使う場合
```.tool-version```
```
nodejs 14.17.6
yarn 1.22.11
```
という状態にした上で
```bash
> yarn set version 3.0.0
```
などなど。

asdf的には1.22.11をインストールした状態にして、そのyarnのコマンドを使ってv2以降に上げます。

詳しくはコチラ。

https://github.com/twuni/asdf-yarn

yarn のv2以降のインストール・設定などは既に色々解説などがありますのでそちらをご覧ください。

## 要点
つまるところ、nodejsプラグイン管理下のyarnと、yarnプラグイン管理下のyarnの、どちらか一つだけを参照するように```.tool-version```を設定する、ということですね。

両方参照できる状態にするのは多分やめたほうがいいんじゃないかなー？

まあちょっとややこしい気もしますが、Corepackを有効化して試すディレクトリと、ちょっと古いNode.jsを使うディレクトリとで、特に無理なく切り替えられると思います。

# 感想
というわけで自分の環境でCorepackを試したときのあれやこれやでした。

個人的に、yarn v2以降のバージョン指定はちょっと微妙だなー、って思ってたので、package.json に一本化出来るCorepackには期待が大きいですね。

流石に今すぐ完全移行ってのは難しいにしても、asdf管理で、当面の実験用のディレクトリと元の設定で運用するディレクトリとを分けられたので一安心です。


ちなみに私は pnpm 派です。
