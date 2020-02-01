---
layout: post
title: "PYNQ-Z2で自作高位合成IPで音声処理をするまで"
date: 2020-02-02
description: 'ただ音をBypassするだけのデザインで小手調べ'
main-class: 'jekyll'
image: 
color: '#B31917'
tags:
- FPGA
- PYNQ
- HLS
categories:
---

最近ふとネットサーフィンをしていたら、PYNQ-Z2にAudio Codecが乗っていることに気がついた。

[http://www.tul.com.tw/ProductsPYNQ-Z2.html](http://www.tul.com.tw/ProductsPYNQ-Z2.html)

PYNQ-Z1が出たときはかなりオーディオはチープというイメージを受けていたので、これには感動してつい購入してしまった。
これを使いこなすために調べた内容と、音をBypassするイメージをPYNQのベースデザインに追加でインプリして動かした備忘録である。
ググれば比較的ある情報にはあまり触れてないので適宜調べるかdocを参照してほしい。


<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">やったーー！！！PYNQ-Z2にVivado HLSで作った音をバイパスするだけのサンプル（後々エフェクトとか実装したい思い）をインプリして、Pythonから開始停止できるようになった！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！ <a href="https://t.co/ukwhZ5pleR">pic.twitter.com/ukwhZ5pleR</a></p>&mdash; かみや (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1222908633396072453?ref_src=twsrc%5Etfw">January 30, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## PYNQを立ち上げる

おおよそは以下の手順通りで行けた。

[https://pynq.readthedocs.io/en/latest/getting_started/pynq_z2_setup.html](https://pynq.readthedocs.io/en/latest/getting_started/pynq_z2_setup.html)


気になる点は以下の通り。

### SD Cardにそこら編に落ちてた怪しいやつを使ったらBootしなかった

SanDiskの速そうなパッケージのやつにしたら動いた。Boot後はデフォルトイメージがコンフィグされてそこらへんのLEDが一斉に点滅するので確認に使うと良い。

以下の写真の状態ではコンフィグされていない。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">リングフィットアドベンチャー <a href="https://t.co/382EZ8S2bQ">pic.twitter.com/382EZ8S2bQ</a></p>&mdash; かみや (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1220994326450196480?ref_src=twsrc%5Etfw">January 25, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

### ネットワーク解決

DHCPも標準で探してくれるようになっており、netbiosでの名前解決ができるのでhttp://pynq でもアクセスできた。

## PYNQ Boot Imageの構成

おおよそ以下の構成のようだった。

### Pynqライブラリの編集

`~/pynq` は`/usr/local/lib/python3.6/dist-packages/`のpynqからシンボリックリンクがはられているので弄ると反映される。
C++で実装された部分も`~/pynq/lib/_pynq`にある。おいてあるmakefileでビルドできるので出来上がった`*.so`で既存の`*.so`を上書きすれば良い

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">なるほどね、完全に理解した(なにもわかってない <a href="https://t.co/eOJASRo8jr">pic.twitter.com/eOJASRo8jr</a></p>&mdash; かみや (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1221065786413830144?ref_src=twsrc%5Etfw">January 25, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Jupyterの自動起動はsystemdに登録されているだけ

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">/usr/bin/jupyter-notebookが登録されていたか <a href="https://t.co/rTorSt9Qa1">pic.twitter.com/rTorSt9Qa1</a></p>&mdash; かみや (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1221062123364577280?ref_src=twsrc%5Etfw">January 25, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

自分でいじったライブラリrepoに差し替えたりが少しやりづらいなぁと感じたり感じなかったり...

### Audio Codecの実装

ADAU1761というADC/DACの乗った俗に言うCODECが実装されており、I2SとI2CがFPGAと直結されていた。
`/boards/ip/audio_codec_ctrl`に実装があるが、AXI4経由で先頭から4byteずつ`RX_L`, `RX_R`, `TX_L`, `TX_R`, `Status`が公開されていた。
`Status`には受信データがReadyになっているとビットが立つようだった。

その他I2C経由の設定はC++の`audio_adau1761.cpp`の実装で各種設定しているようだった。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">xi_lite_ipifから叩かれているのはここか、やっぱりdata_rdy_bitだけアサインされてそう <a href="https://t.co/8O4rQnBC5K">pic.twitter.com/8O4rQnBC5K</a></p>&mdash; かみや (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1221453916887384064?ref_src=twsrc%5Etfw">January 26, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

少し罠なのはpynqライブラリのC++実装にある`audio_adau1761.cpp`を見ればわかるのだが、Bypass関数を呼んだとき以外はADAU1761にI2Sでデータを送ってもIC内臓のMixer3/4とボリュームによって結局ミュートされてしまうよう実装されていた。

新しい関数I/Fを生やすのも面倒なので、Line入力に設定した時点で上記設定をするように修正した。
これでPythonからでも受信レジスタの値を送信レジスタに書いてあげればループバックが実現できる。

<script src="https://gist.github.com/kamiyaowl/8d87cd1386e08b2ef9adb54d76a8bdb9.js"></script>

## PYNQ-Z2のベースデザインを手動でビルドする

まずは既存のデザインを自力で論理合成してみる。現在時点ではVivado 2019.1向けに書かれたtclなので2019.1を入れた。

適当なprojectを作って`/boards/Pynq-Z2/base/base.tcl`を実行するのだが、私の環境かWindowsかわからないが作業Directoryが`~/AppData/....`あたりに飛ばされて解決できなかったのでIP Packageの登録だけ手動でやった。

以下の通りbase.tclをいじって、`/boards/ip`にいるIPは事前にVivadoのGUIから手動で追加しておいた。

```diff
################################################################
# START
################################################################

# To test this script, run the following commands from Vivado Tcl console:
# source system_script.tcl

# If there is no project opened, this script will create a
# project, but make sure you do not have an existing project
# <./<overlay_name>/<overlay_name>.xpr> in the current working folder.

set overlay_name base
set list_projs [get_projects -quiet]
if { $list_projs eq "" } {
   create_project ${overlay_name} ${overlay_name} -part xc7z020clg400-1
}

-set_property  ip_repo_paths  ../../ip [current_project]
+# set_property  ip_repo_paths  ../../ip [current_project]
-update_ip_catalog
+# update_ip_catalog

# CHANGE DESIGN NAME HERE
variable design_name
set design_name base
```

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">☑base.tclのご機嫌をとった <a href="https://t.co/Qhpl487OmB">pic.twitter.com/Qhpl487OmB</a></p>&mdash; かみや (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1221300155925716992?ref_src=twsrc%5Etfw">January 26, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

あとはBlock Designのwrapperを作って合成を進める正規の手順でbitstreamが生成できた。

### ビルド生成物

今のPynqのOverlayライブラリは`.bit`, `.hwh`, `.dtbo`(DeviceTreeが変わる場合のみ)を必要としているようだった。

`<project-root>/<project>.runs/impl_1/<top_file_name>.bit`と`<project-root>/<project>.src/source_1/bd/base/hw_handoff/base.hwh`に配置されていたのでこれを利用した。

### PynqでのOverlay

先の生成物をPynqにコピーして、以下のコードをJupyterあたりで実行すれば無事同じように動作できた。
overlay.pyとか周辺を読む限り、bitファイルのファイルパスをもじってhwhを取得しているようだった。

```python
from pynq.overlays.base import BaseOverlay
base = BaseOverlay("~/path/to/<top_file_name>.bit")
```

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">わーい、自前で生成し直したbitstreamでも昨日のADCの入力を取れるようになった！<a href="https://t.co/F985jGegW0">https://t.co/F985jGegW0</a> <a href="https://t.co/FJYQq5fW34">pic.twitter.com/FJYQq5fW34</a></p>&mdash; かみや (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1221396938722926592?ref_src=twsrc%5Etfw">January 26, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
