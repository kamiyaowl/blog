---
layout: post
title: "Dockerで簡易httpサーバを立てる"
date: 2019-03-11
description: 'htmlなど静的ファイルホスティング'
main-class: 'jekyll'
image: 
color: '#B31917'
tags:
- Docker
- Nginx
categories:
introduction: 
---

最近、こんなクソゲーを作った。いつも単品のhtmlで開発する際に、index.htmlを開いていると、例えば特定のファイルをうまく読んでくれなかったり、パスが不正だったりと本番のDeployingと差分があり不都合になる場合があったりする。

![sashimi](https://user-images.githubusercontent.com/4300987/54084179-da565580-4370-11e9-83c1-cfb83bb7326f.png)

[kamiyaowl/sashimi - github](https://github.com/kamiyaowl/sashimi)


## 簡単にhttpサーバを立てて、ホスティングしたい

Dockerは最強だった。nginxのイメージを持ってきて、ホストしたいディレクトリを`usr/share/nginx/html`にマウントしてあげれば完了。

自分はdockerコマンド直打ちが嫌いなので、次のdocker-compose.ymlを示す。直接実行したい場合は適宜。


{% highlight yaml %}
version: "3"
services:
  nginx:
    image: nginx:latest
    ports:
      - "<local server port>:80"
    volumes:
      - <mount directory>:/usr/share/nginx/html
{% endhighlight %}

例えば、カレントディレクトリのファイル一式をport:4444で公開したい場合は以下のようになる。


{% highlight yaml %}
version: "3"
services:
  nginx:
    image: nginx:latest
    ports:
      - "4444:80"
    volumes:
      - ./:/usr/share/nginx/html
{% endhighlight %}

もしマウントするディレクトリに公開したくないファイルがある場合、`.dockerignore`に記載するとマウントされません。

{% highlight text %}
secret-file.json
{% endhighlight %}

## まとめ

dockerは最近欠かせないツールになった。