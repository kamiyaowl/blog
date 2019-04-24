---
layout: post
title: "riscv-gnu-toolchainをDockerコンテナにする"
date: 2019-04-25
description: '環境構築が面倒なときに'
main-class: 'jekyll'
image: 
color: '#B31917'
tags:
- Docker
- RISC-V
categories:
---

## Docker使ってますか？

実は環境構築がめちゃくちゃ面倒な人ほどおすすめできる。正直Vagrantでもいいですが、気軽にふっとばしたりBase Imageからカスタムしたりコマンドラインから手軽に扱ったりCloud(k8s等)に乗っけるときなど等々を考えたら、私はDockerが便利だと思っている。

個人的には中にログインしてしばらく作業するような使い方なら、Vagrantを推奨。

## Toolchainの構築

[riscv-gnu-toolchain](https://github.com/riscv/riscv-gnu-toolchain)を構築していたのですが、WinでもMacでもささっと使いたい+バージョンで悩みたくない。

そこでDockerで環境構築してみます。リポジトリにもあるようにUbuntu(apt)を使った例があるのでこれをそのまま構築してみる。

{% highlight text %}
FROM ubuntu:18.04

ENV RISCV=/opt/riscv
ENV PATH=$RISCV/bin:$PATH
WORKDIR $RISCV

RUN apt update
RUN apt install -y autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev
RUN apt install -y git

RUN git clone --recursive https://github.com/riscv/riscv-gnu-toolchain

RUN cd riscv-gnu-toolchain && ./configure --prefix=/opt/riscv && make

WORKDIR /work
{% endhighlight %}

完成！見れば何となく何をやっているか一目瞭然。

一応githubリポジトリとDocker Hubにもアップした。参考まで。

* [Github - kamiyaowl/riscv-gnu-toolchain-docker](https://github.com/kamiyaowl/riscv-gnu-toolchain-docker)
* [Docker Hub - kamiyaowl/riscv-gnu-toolchain-docker](https://hub.docker.com/r/kamiyaowl/riscv-gnu-toolchain-docker)