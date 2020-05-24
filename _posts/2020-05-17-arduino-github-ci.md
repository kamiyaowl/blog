---
layout: post
title: "Github ActionsでArduinoプロジェクトのビルドを行う"
date: 2020-05-17
description: 'ビルド周りの不安解消の話'
main-class: 'jekyll'
image: 
color: '#B31917'
tags:
- Arduino
- Docker
- C++
categories:
---

## 前置き

最近[Wio Terminal](https://www.switch-science.com/catalog/6360/)を購入して遊んでいる。Twitterを見ていても多くのユーザが色々遊んでいてなんとなく流行に乗った気分になっている。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">やったー！！！！Wio TerminalにRustで書いたファミコンエミュ移植できた！！！！！！！！！！！！！！！（まだUART経由でしか表示できないけど...）<a href="https://t.co/VvBct6k8EX">https://t.co/VvBct6k8EX</a> <a href="https://t.co/V2nC4LusSX">pic.twitter.com/V2nC4LusSX</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1259895486816780289?ref_src=twsrc%5Etfw">May 11, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>


個人的購入したきっかけはLiPo内臓ではないこと、ATMEL社のARMマイコンを以前使っていたことあたり。([これとか](http://logiclover.hatenablog.jp/entry/2017/04/29/163108))

ところで一昔前ならICEつないでArduino環境を取っ払って遊んでしまうのだが、最近は趣向が変わっていて他人が再現しやすい環境・想定された開発環境をできるだけ守るような試みに重きをおいている。
というわけでArduino IDEを使っていたのだが、色々と辛い部分があったので今回の話に移る。

## 気になる点

1. `hogehoge.ino`だけを公開してもBSP,/Libraryはユーザ環境のVersion次第
2. 1.に関連して、仮に別Projectの`fugafuga.ino`がLibraryのVersion依存を持っていると収集がつかない
3. Arduino IDEがないとビルドができない(arm-none-eabi-gccを叩くにしてもオプションを毎回控えるのは手間)

ざっくりまとめると1.2.に関してはenv切り替えのないpythonと同じ問題、3.については...という具合

## 改善・やったこと

今回は作り途中の [kamiyaowl/wfi_monitor - GitHub](https://github.com/kamiyaowl/wfh_monitor) で試した内容である。

### 1.2. ライブラリの参照方法

**2020/05/24修正**

1.2.についてはライブラリをArduino IDEのパス解決に任せないことにした。
今日の多くのライブラリはgithubに公開されているものをzip DL, IDEに追加の流れを踏んでいたのでsubmoduleとしてリポジトリに追加している。

その際に`lib`以下にまとめてsubmoduleとして追加されるようにし、後ほどDockerコンテナ内にマウントしている。

{% highlight text %}
$ cd lib
$ git submodule add <追加したいライブラリのrepo url>
{% endhighlight %}

今回の場合はTFT液晶制御のライブラリを追加している。

{% highlight text %}
$ cd lib
$ git submodule add https://github.com/Bodmer/TFT_eSPI.git
{% endhighlight %}

後述するDocker上でビルドする際、`lib`ディレクトリをそのままArduino LibのおいてあるDirectoryにマウントして参照・ビルドしている。

これで通常のビルドと遜色なく運用できるようになった。

### 3. Arduino IDEに依存しないビルド環境

[Arduino CLI](https://github.com/arduino/arduino-cli) なるものがあるらしい。これを使えばDocker上ですべて解決できるのでは...ということでやってみた。

#### Dockerfile作成

配布済を使っても良かったがビルドする際に、任意のBSP/Libraryを入れたかったので書いた。
正規の手順でArduino CLIを導入し、必要なパッケージ群をインストールしているだけである。

[Dockerfile - kamiyaowl/wfh_monitor](https://github.com/kamiyaowl/wfh_monitor/blob/master/Dockerfile)

{% highlight c %}
FROM ubuntu:20.04

# install from apt package

RUN apt-get update && apt-get install -y curl

# install arduino-cli

RUN mkdir -p /tool
RUN curl -fsSL https://raw.githubusercontent.com/arduino/arduino-cli/master/install.sh | BINDIR=/tool sh
ENV PATH $PATH:/tool/
RUN echo $PATH

# refresh platform indexes

RUN arduino-cli core update-index
RUN arduino-cli core update-index --additional-urls https://files.seeedstudio.com/arduino/package_seeeduino_boards_index.json

# add arduino core and libraries

RUN arduino-cli core install arduino:samd
RUN arduino-cli core install Seeeduino:samd --additional-urls https://files.seeedstudio.com/arduino/package_seeeduino_boards_index.json
# RUN arduino-cli core list # for debug


# create work directory

RUN mkdir /work
WORKDIR /work

CMD ["/bin/sh"]
{% endhighlight %}

私はコマンドを打つのが面倒なタイプなのでdocker-composeを使う。
ただ、`docker-compose.yaml`にコマンド直書きするにはちょっと取り回しが面倒なので、実行するscriptは分離した(linux環境だったらそのまま使えますし)

[build.sh - kamiyaowl/wfh_monitor](https://github.com/kamiyaowl/wfh_monitor/blob/master/build.sh)
{% highlight text %}
#!/bin/sh
arduino-cli compile -b Seeeduino:samd:seeed_wio_terminal ./wfh_monitor.ino --verbose --log-level trace
{% endhighlight %}

小テクだが、checkout時点でscriptに実行権限がないと積むのでchmod +xしている。

[docker-compose.yml - kamiyaowl/wfh_monitor](https://github.com/kamiyaowl/wfh_monitor/blob/master/docker-compose.yml)
{% highlight yaml %}
version: "3"
services:
    build:
      build: .
      volumes:
        - ./:/work
        - ./lib:/root/Arduino/libraries:ro
      command: >
        bash -c "chmod +x ./build.sh && ./build.sh"
{% endhighlight %}

`./lib`にcheckoutしてあるライブラリも、ここでRead Onlyでマウントしておくことで参照できる。

これでDocker/Docker Composeさえ入っていればArduino CLIでビルドできるようになった。めでたい。

{% highlight yaml %}
$ docker-compose run build
{% endhighlight %}

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">Docker上でarduino-cli使ってWio Terminalのビルド環境作った(後でどこかにまとめます <a href="https://t.co/FTCbkMrlbJ">pic.twitter.com/FTCbkMrlbJ</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1261942881335406594?ref_src=twsrc%5Etfw">May 17, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

### おまけ: Github Actionでビルドさせる

やっとタイトル回収。Dockerでビルドできるなら、Github Actionでもすぐ動かせるはず。

注意点としてactions/checkoutではsubmoduleを引っ張ってきてくれないので自分であとから引っ張ってきている(git cloneも--recursiveオプションがないとだめなはず)

また、`actions/upload-artifact`で生成物を指定することでzipで固めてDL可能な状態にすることができる。

[.github/workflows/build.yml - kamiyaowl/wfh_monitor](https://github.com/kamiyaowl/wfh_monitor/blob/master/.github/workflows/build.yml)
{% highlight yaml %}
name: Build

on:
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - run: git submodule update --init --recursive
    - run: docker-compose run build
    - uses: actions/upload-artifact@v2
      with:
        name: wfh_monitor.zip
        path: wfh_monitor.ino.Seeeduino.samd.seeed_wio_terminal.
{% endhighlight %}

[これでpushするだけでビルド済バイナリを作れるようになった。](https://github.com/kamiyaowl/wfh_monitor/actions/runs/107319698)

workflowにDockerfile相当を書いても行けるはずだが両方保守するのは面倒なのでこの方法が手軽で良い。

## 終わりに

全然関係ないけど、Rust+Arduinoをさっさと試したい人向けに作ったが需要がニッチ過ぎて微妙だった。興味があったら試してほしい。

手軽さのArduinoと堅牢さのRustってものすごく相性悪い気がしてきた...?私は堅牢なところだけRustで書くつもりだったのだが。

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">A minimal sample of running Rust on an Arduino.✌ <a href="https://twitter.com/hashtag/WioTerminal?src=hash&amp;ref_src=twsrc%5Etfw">#WioTerminal</a> <a href="https://twitter.com/hashtag/Arduino?src=hash&amp;ref_src=twsrc%5Etfw">#Arduino</a><a href="https://t.co/wQThswGhgm">https://t.co/wQThswGhgm</a> <a href="https://t.co/1odQ7SwNvE">pic.twitter.com/1odQ7SwNvE</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1260276824484896774?ref_src=twsrc%5Etfw">May 12, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>