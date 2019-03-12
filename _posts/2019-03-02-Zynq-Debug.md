---
layout: post
title: "ZynqのDebug時のエラー"
date: 2019-03-02
description: 'XSDKのデバッグ周りのエラーについて'
main-class: 'jekyll'
image: 
color: '#B31917'
tags:
- FPGA
- Zynq
categories:
introduction: 
---


# ZynqのDebug時によくわからないエラーが出る

こういうやつ

![dialog](https://pbs.twimg.com/media/D0lWF7eU8AAiIYR.png:large)


# 原因

PSのペリフェラル等々をリセットしていないことが問題。Block Design時にPSに対して行った設定はどうやらbspのps7_init.tclにいるようなので、デバッグ時には指定してあげよう。

![config](https://pbs.twimg.com/media/D0lWF7fU4AkwWUV.jpg:large)


# 確認項目

他にも躓く点がありそうだったのでメモ。

* Block DesignのIP Version(Validate Design時に怒られる)
    * Report IP Status -> Upgradeしよう
* ZyboのBoot設定をJTAGにしよう
    * SD/QSPIと設定があるので注意、デバッグ時はJTAGで
* Debug Configuration
    * Program FPGA - 未コンフィグではダメ
    * Run ps7_init/ps7_post_config - 必要(今回の記事)
    * Application - デバッグしたいコアのチェックは入れておこう
* 他
    * 電源 - 電流が少ないとちゃんとうごかないかも
    * Firewall - JTAGのやりとりにVivadoがhw_serverを立てているので遮断せぬよう