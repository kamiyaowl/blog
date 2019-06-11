---
layout: post
title: "Animation PNGを自力でデコードする"
date: 2019-06-11
description: '私的にはgo言語入門'
main-class: 'jekyll'
image: 
color: '#B31917'
tags:
- apng
- golang
categories:
---

今回はPNGファイルを自力で読み込んで表示することができたので紹介する。ただ本命はPNGをデコードしたことではなく、これをgo言語で実装したことにある。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">やったーー！！！golangで作った自作Animation PNG Decoderでアニメーションが表示できた！（背景の水色は合成）<br>ぞうさんは <a href="https://t.co/4Eyy5zvsJ8">https://t.co/4Eyy5zvsJ8</a> からお借りしました。 <a href="https://t.co/qZWa4VoDCa">pic.twitter.com/qZWa4VoDCa</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1138108992150855681?ref_src=twsrc%5Etfw">June 10, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

上記ツイートを見てもらえばわかるが動いている。これはAnimated PNGというアニメGIFの上位互換を目指したようなPNGの拡張規格である。せっかくGUIを出しているのでこのAnimation PNG(apng)をデコードしてみる。

## Go言語、今更入門する

モチベーションは以下にある。実は前にbackendで使ったことがあったが理解がなくinterface地獄になったこともあって個人的印象はあまり良くない。

偏見は良くないので以下の点を触って確かめるべくという具合だ。触って確かめたのであまり文書にできる感想はない。どちらかといえば良かったと思う。

1. 複数人開発におけるコーディングへの有用性
2. 組み込み等低レイヤでの有用性が知りたい
3. 開発環境の手軽さ

## 関係ドキュメント

先に示しておく

1. [W3C Recommendation Portable Network Graphics Specification](https://www.w3.org/TR/PNG/)
2. [MDN web docs Animated PNG graphics](https://developer.mozilla.org/ja/docs/Animated_PNG_graphics)
3. [Github kamiyaowl/Animation-PNG-Viewer](https://github.com/kamiyaowl/animation-png-viewer)

## バイナリの読み込み

`os.File`を利用する。見ればわかる単純明快さだ。(あとから知ったが`io`パッケージがあったらしい)


{% highlight text %}
func (self *Apng) Parse(src string) (err error) {
	f, err := os.Open(src)
	if err != nil {
		fmt.Println(err)
		return
	}
	defer f.Close()

    // ...
{% endhighlight %}

defer句はリソース破棄時に実行されるようだ。ここではfileハンドルを捨てている。c#でいうところの`using(){}`らしい

## PNGファイルのデコード

golangはさておき早速実装を始めていく。詳細は記載しないので興味のある人やちゃんと定義を知りたい人はW3Cの資料を参照してほしい。

まず最小限の構成を示す。golangでの実装については[Github - kamiyaowl/Animation-PNG-Viewer apng.go](https://github.com/kamiyaowl/animation-png-viewer/blob/master/apng/apng.go)のParse()を参考にしてほしい。

| 名前 | 内容 | 補足 |
| ---- | --- | ---- | 
| header | {0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a} | マジックナンバー |
| IHDR | Width, Height, BitDepth, ColorType等 | 画像自体の構成情報 |
| IDAT | zlib圧縮を施された画像データ | 先頭1byteはフィルタの種類を示す(後述)。またIDATは分割可能 |
| IEND | - | 内容なし | 

ヘッダはファイルを識別するためのものなので単純一致を調べれば良い。

他についてだが、PNGファイルはchunkと呼ばれる単位で分割されている。その中身は以下のようになっている。

| 名前 | 長さ | 内容 | 
| ---- | --- | -- |
| Length | 4byte | Chunk Dataのbyte数 |
| Chunk Type | 4byte | acsii charでchunk名(`IHDR`, `IDAT`等) |
| Chunk Data | 0~Length byte | chunkのデータ本体 |
| CRC | 4byte | **ChunkType~ChunkDataのCRC32(生成多項式はISO-3309)** |

ここで言うChunkTypeは、先頭二文字が大文字は必須chunkで小文字だと補助(必須ではない)chunkである。また、PNGの2byte,4byteデータはBigEndianなので気をつけてほしい。

手順としてはこうだ。

1. Length, ChunkTypeを読み出す(4byte)
2. Length分だけChunkDataを読み出す(Length byte)
3. CRCを読み出す(4byte)
4. ChunkType+ChunkDataのCRC32を計算してデータ検査
5. 問題なければChunkDataの中身をデコード

これをループさせて、`IHDR`, `IDAT`が読み出すことができれば元通りの画像を作ることができる。

## IHDR

ChunkDataをIhdr構造体にばらした。

{% highlight text %}
func (self *Apng) parseIHDR(data []uint8) (err error) {
	if len(data) != 13 {
		return errors.New("IHDRのヘッダサイズは13でなければならない")
	}
	self.Ihdr.Width = int(binary.BigEndian.Uint32(data[0:4]))
	self.Ihdr.Height = int(binary.BigEndian.Uint32(data[4:8]))
	self.Ihdr.BitDepth = data[8]
	self.Ihdr.ColorType = data[9]
	self.Ihdr.Compress = data[10]
	self.Ihdr.Filter = data[11]
	self.Ihdr.Interlace = data[12]

	return nil
}
{% endhighlight %}

## IEND

特にやることはない

## IDAT

Interlace対応をひとまずおいておくなら(全部読み切ってから表示するなら)以下の実装で構わない。Idatのデータを連結しているだけである。

{% highlight text %}
func (self *Apng) parseIDAT(data []uint8) (err error) {
	self.Idat = append(self.Idat, data...)
	return nil
}
{% endhighlight %}

### IDATの画像化

IHDR, IDAT, IENDを読み出したところでデータをもとに戻す。手順は以下の通り。

1. IDATのデータ列をzlib展開する
2. 各行ごとに、フィルタリング処理を解除する
3. IHDRのBitDepth, ColorTypeに従い色情報を復元する

1.についてはgoのライブラリを使って展開した。自力でやることも当然可能だ。
2.はフィルタリングについて解説する。

zlib圧縮(しいてはDeflate)は、いわゆるハフマン符号化なので同じ符号列が頻出したほうが圧縮率が良い。(いろいろ語弊があって怒られそう)
何がしたいかというと、

1. 例えば0行目と1行目に同じ色パターンが並んでいるとする
2. 1行目の定義を0行目の色データとの差分と定義する
3. 定義に従うと1行目のデータはすべて0が並ぶ
   
こうなると圧縮に有利である。今の例はUpフィルタと呼ばれている。実際には以下の種類が定義されている。

| 種類 | 説明 |
| --- | --- |
| None | フィルタリングなし |
| Sub | 左Pixelとの差分 |
| Up | 上Pixelとの差分 |
| Average | (上Pixel+左Pixel)/2との差分 |
| Paeth | 割愛 |

Paethフィルタはさておき割と簡単に実装できる。(上Pixelと左Pixelで近い方の色を使う、ような実装になっている)
各行の先頭1byteにフィルタの種類が定義されているのでこれを読み出して、もとのbitmapに戻せば完了である。

ここまでで元通りの画像が表示できる。ちなみにguiには[faiface/pixel](https://github.com/faiface/pixel)を使っている。非常に良い

![image](https://pbs.twimg.com/media/D8OrC_hVsAEUrVi.jpg:large)

## Animated PNG対応

実はここまで実装できていればもう一息である。`acTL`, `fcTL`, `fdAT`の補助チャンクが追加される。

| 名前 | 内容 | 補足 |
| ---- | --- | ---- | 
| header | {0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a} | マジックナンバー |
| IHDR | Width, Height, BitDepth, ColorType等 | 画像自体の構成情報 |
| acTL | アニメーションのフレーム数, 再生回数 | IHDR-IDAT間にあることが必須 |
| fcTL | Width, Height, OffsetX, OffsetY, 表示時間、描画方法等 | fdATの画像情報を持つ。IDAT前にある場合はIDATがfdATの最初の画像とする |
| IDAT | zlib圧縮を施された画像データ | 先頭1byteはフィルタの種類を示す(後述)。またIDATは分割可能 |
| fcTL | SequenceNumber, Width, Height, OffsetX, OffsetY, 表示時間、描画方法等 | - |
| fdAT | SequenceNumber, IDATと同様にデータ | 必ず手前にfcTLがある必要がある |
| fcTL | ... | ... |
| fdAT | ... | ... |
| ... | ... | ... |
| IEND | - | 内容なし | 

差分はこうだ。

1. acTLは画像に含まれるフレーム数、再生回数がある。これがあったらAnimated PNGとして処理する
2. fcTLは直後のfdAT(またはIDAT)の画像情報を定義する
3. fdATはsequence_numが定義されたIDATと同じ。直前のsequence_numを持つfcTLを参照して画像を復元する

IDATと同じフォーマットの画像データがfdATという補助チャンクで繰り返し出現するだけである。
シンプルな機能追加だと思わせているが、以下の特徴に注意する。

1. IDATの画像サイズはIHDRに必ず一致する
2. fcTLで画像サイズとオフセットが定義されており、これはIHDRの画像サイズとは一致しない
3. fcTLで画像の合成方法が定義されておりDisposeOp, BlendOpで決定される
   1. DisposeOpによって、元画像に直前のフレームをそのまま使うか、透明黒画像を使うか決定する
   2. BlendOpによって、元画像に対しPixel値の上書きをするか、アルファブレンディングするか決定する

これらを失敗すると、おもしろ画像が生成される。


![preview](https://pbs.twimg.com/media/D8tSTHzUwAA8NUC.jpg:large)

正しく実装できれば、冒頭のアニメーションが再生できるようになる。

## 所管

Animated PNGもだが、PNGのフォーマット自身がかなり扱いやすいものだと実感した。

Go言語に関しては多相型がなかったり三項演算子がなかったり、今どきの言語に限らず目につくところはいろいろあるが、あとからコードを見返したときになんとなく何をしているのか読めるのが良い点だと感じる。
少なくともバイナリデコードするようなbetter cの用途においては十分すぎる簡潔さであると感じる。(Rustのほうが...とも思うが)

組み込みに関してはもう少し調査が必要である。

