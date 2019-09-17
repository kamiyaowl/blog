---
layout: post
title: "RustでNESエミュレータを作っている（備忘録）"
date: 2019-03-28
description: 'Twitterのモーメントが役に立たないので'
main-class: 'jekyll'
image: 
color: '#B31917'
tags:
- Rust
- NES
- エミュレータ
categories:
---

![preview](https://user-images.githubusercontent.com/4300987/64512802-1bc8bd00-d322-11e9-8a70-26df62bb5ee1.gif)

[rust-nes-emulator GitHub](https://github.com/kamiyaowl/rust-nes-emulator)

ここ一ヶ月ぐらいRust入門を兼ねて、ファミコンエミュレータを自作している。変遷についてまとめたかったのだがTwitterのモーメントがもう使わせる気がなさそうなのでブログにまとめておく。

誰がなんというとこれはモーメント。

## Rustを使うモチベーション

筆者の背景を紹介するが、単純に書いた量と馴染みで言えばC#, C/C++, JavaScript/TypeScript, Scalaの4つだと思う。

特定の言語を持ち上げたりすることはあまり得意ではないが、2015年頃から漠然と安全側に傾けた言語に興味が湧くようになっていた。書くと長くなるがscalaを書き始めたことと、LLVMについて調べだしたあたりが強く影響している。

その中でも組み込み開発での可用性が高そうで、パフォーマンスも既存の言語に引けを取らないという点でRustが私の中で候補に上がっている。（golangはOSレスno_stdな組込み向けではTinyGo次第なところがある）

## 自作にあたり

[NesDev wiki](http://wiki.nesdev.com/w/index.php/Nesdev_Wiki) が圧倒的な情報量を取り揃えている。幸いファミコンエミュレータについては日本でも作っている先輩方が多数おられるので、英語が苦手でも日本語情報がかなり得られるので照らしながら作ると良い。

細かい実装の話などは後日別の媒体にまとめようと思っているので、残りは実装の変遷を。

## CPU実装

はじめはwikiにあるCPU命令をすべて実装する作業を行った。きっと合っているだろうという気持ちで...。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">macro完全に理解した(何もわからない <a href="https://t.co/RD0TvtNuuJ">pic.twitter.com/RD0TvtNuuJ</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1162560042899390465?ref_src=twsrc%5Etfw">August 17, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

すぐに言語マクロを試したくなるのは趣味。

## カセット読み込みと割り込み実装

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">わーい、とりあえずRESET割り込みで命令先頭に飛んできてはじまるようになった <a href="https://t.co/EdhqazRvz4">pic.twitter.com/EdhqazRvz4</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1163070854730665984?ref_src=twsrc%5Etfw">August 18, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

CPU命令と.nes形式ファイルの展開を実装した頃で、実機起動時（もしくはResetボタン相当）の動きができるようになった。

Reset割り込みは0xfffc, 0xfffdに記されたアドレスに飛ぶような挙動なので、エントリポイントに飛ばせるようになったことに等しい。ここがスタート地点だ...?

## CPUのみでHello, World!挑戦

[NES研究所](http://hp.vector.co.jp/authors/VA042397/nes/sample.html) 様が公開しているHello, Worldを表示するサンプルを動かすことにした。対戦よろしくお願いします。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">rust emuの終了時レジスタを使った簡易リグレッションテスト、いいね <a href="https://t.co/E5nlVsvX8l">pic.twitter.com/E5nlVsvX8l</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1163823863215480832?ref_src=twsrc%5Etfw">August 20, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

バイナリエディタとオペコード一覧を見て、命令のフェッチがうまく行っているか見ながらステップ実行を繰り返している様子を眺めていた。

一部修正してPPUADDR, PPUDATA経由でCPUからBGの情報と"Hello, World!"の文字列をを流しているのがおおよそ確認できたので次に進むことにした。

[rust-nes-emulator Issue #3: Hello Worldの実行結果の正当性を確認する](https://github.com/kamiyaowl/rust-nes-emulator/issues/3)

## PPUでHello, World!

ファミコンにはCPUの他にPPU(Picture Processing Unit), APU(Audio Processing Unit)というサブシステムが乗っかっており、画面表示はこのPPUが担っている。

表示できるものにはいわゆる背景のBGと、マリオとかクリボーみたいに動かせるSpriteがある。Hello, WorldのサンプルではSpriteは使っていなかったのでBGを描画するコードの実装した。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">やったー！rustで書いてるファミコンエミュでHello Worldが出せるようになった～～～ <a href="https://t.co/VzdFz14g4x">pic.twitter.com/VzdFz14g4x</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1165602990469742592?ref_src=twsrc%5Etfw">August 25, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

頭がオカシイのか何故かここだけは一発で動いてしまった、多分ここで苦戦してたらエミュ自作にこれほどはまらなかったかもしれない...。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">GUIでグイグイ <a href="https://t.co/sp4heMW98p">pic.twitter.com/sp4heMW98p</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1165641917746446336?ref_src=twsrc%5Etfw">August 25, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

最初はbitmap画像にしていたが不便だったので、piston_windowというライブラリを使って表示制御することにした。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">ギャア（アスタリスクしかちゃんと出てねぇ <a href="https://t.co/oOk3VuU1k3">pic.twitter.com/oOk3VuU1k3</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1166023794688815104?ref_src=twsrc%5Etfw">August 26, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

残念ながらこの時点では他のROMはほぼ動かなかった。nestestでさえこの有様。Hello, Worldがいかに簡潔なサンプルなのか思い知らされる。

## えー！なるっちのCPU実装がバグだらけ！？

画像は省略します。nestest.nesという命令一式を試せる素晴らしいROMがあるのですが先の通り表示すらできない有様。

nestestの作者はこれはもう素晴らしくて、画面が表示できなくてもPCを0xC000に飛ばせば画面なくてもテストが進むよモードを実装していた...天才か。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">println散らかるし入れまくると遅いしでデバッグ大変だったので、ビルドオプション次第で抹消されるようなdebug printを導入した <a href="https://t.co/GzUltNpwli">pic.twitter.com/GzUltNpwli</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1167768946159677441?ref_src=twsrc%5Etfw">August 31, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

ただし確認方法は正しく動くROMのログとCompareすることだったため（8000行ぐらいある）、まずはログ出力を強化した。

<blockquote class="twitter-tweet"><p lang="und" dir="ltr"><a href="https://t.co/B7ehpDrHxq">pic.twitter.com/B7ehpDrHxq</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1167794508622221314?ref_src=twsrc%5Etfw">August 31, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

**がんばれかみ子、バグをしばいて立派なまぞくになるんだ。**

[rust-nes-emulator Issue #25: NESTESTのメニュー画面が正しく表示されるようにする](https://github.com/kamiyaowl/rust-nes-emulator/issues/25) に大体書いてあるが、ひたすらlogを見ていってミスった実装を直しての繰り返しをしていた。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">進捗です（だめです <a href="https://t.co/gCXKF1StRW">pic.twitter.com/gCXKF1StRW</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1167871230998827008?ref_src=twsrc%5Etfw">August 31, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

だんだん直って

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">わーい、nestestのメニューが出せるようになった <a href="https://t.co/iHdjgGL5SR">pic.twitter.com/iHdjgGL5SR</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1168127730451283968?ref_src=twsrc%5Etfw">September 1, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

起動するようになった。（下にCompareしてるスクショが残ってた）

ここで初めて気がついたがunofficial opcodeらしきものがあるらしい。それもすべて実装した。複数の命令の組み合わせか通常存在しないアドレッシングモードか、NOPかのいずれかな気がする。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">rustでファミコンエミュ、進捗です(うれしい <a href="https://t.co/jMh5d4wZ2M">pic.twitter.com/jMh5d4wZ2M</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1168135665923420160?ref_src=twsrc%5Etfw">September 1, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

ボタンが押せなくてテストが実行できなかったのでボタン入力を実装。

ここもまた一発で動いてしまった。ここまで死ぬほど動かなかったので、嬉しさのあまりまぞくになるところだった。

## リグレッションテスト導入

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">指定フレーム数進めて画像比較してボタン押させて～～～～のテスト環境整備した <a href="https://t.co/pFnnzPXXBT">pic.twitter.com/pFnnzPXXBT</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1168165217605308421?ref_src=twsrc%5Etfw">September 1, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

せっかく動いたのに、後の実装でぶっ壊れると嫌なのでテスト環境を整備した。

## 顔色が悪いPPU

nestestしか動かしていなかったので興味本位で動かしたら顔色が悪い。顔色が悪いけど嬉しかった。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">顔色が悪いｗｗｗｗｗｗｗｗｗｗｗｗｗｗｗｗｗｗｗｗｗｗｗｗ <a href="https://t.co/BHIMhMEqBg">pic.twitter.com/BHIMhMEqBg</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1168166542510477313?ref_src=twsrc%5Etfw">September 1, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">おおおお～～～～～～～～まじか！！！！！！！！！！！！顔色悪いし配管工来てないけど <a href="https://t.co/kclWNAxqck">pic.twitter.com/kclWNAxqck</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1168167098788405249?ref_src=twsrc%5Etfw">September 1, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

ファミコンには実際の表示領域4面分の描画空間(NameTable)が合って、その任意位置を変更できるスクロールレジスタが存在する。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">今日の進捗です（めげない心... <a href="https://t.co/EP85zkGJ0F">pic.twitter.com/EP85zkGJ0F</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1169280488885743616?ref_src=twsrc%5Etfw">September 4, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

実装したらこの有様...。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">マリオーーーーーー体がーーーーーーッッッッ！！！！！！！！（rust ファミコンエミュ進捗です <a href="https://t.co/ivnfDrIhmE">pic.twitter.com/ivnfDrIhmE</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1170358592500686849?ref_src=twsrc%5Etfw">September 7, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

スクロールがない1面だけのゲームに焦点を絞って先にSpriteを実装した。体がばらばらになってしまった...

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">スクロールもバグってるからマリオが死ぬほど面白い <a href="https://t.co/GdHVxQT3V8">pic.twitter.com/GdHVxQT3V8</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1170361949160239104?ref_src=twsrc%5Etfw">September 7, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

スクロールのバグと合わせて新しいゲームが完成した。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">よっしゃあ～～～～～～～～～～～～～～～～～～～～～～～～～～～～～！！！！！！！！！！！！！ <a href="https://t.co/8HxnyfzHw8">pic.twitter.com/8HxnyfzHw8</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1170372607574601728?ref_src=twsrc%5Etfw">September 7, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Spriteのバラバラ事件は表示反転の論理が逆になっていただけだった。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">やったー！rustで書いてるファミコンエミュでスプライトが表示できるようになったー！（<br>ドンキーコングのOPデモが正しく動くようになった！） <a href="https://t.co/zB1LjG6uXr">pic.twitter.com/zB1LjG6uXr</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1170374917579132928?ref_src=twsrc%5Etfw">September 7, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

ここまで来るとスクロールのないドンキーはそれっぽいゲームになっていた。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">Sprite背景透過できた <a href="https://t.co/PBTabChgUT">pic.twitter.com/PBTabChgUT</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1170550384978300928?ref_src=twsrc%5Etfw">September 8, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">ドンキーコングは、あとはBGの属性テーブルで表示は正しくなりそう <a href="https://t.co/AJZYl24caV">pic.twitter.com/AJZYl24caV</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1170551050702450694?ref_src=twsrc%5Etfw">September 8, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

背景色の判定をきちんと実装したのでそれっぽくなってきた。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">もう少し直した。続低テーブルの当て方を間違えていた <a href="https://t.co/w44MBt29Qt">pic.twitter.com/w44MBt29Qt</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1170560625128300544?ref_src=twsrc%5Etfw">September 8, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

属性テーブルの参照アドレスをミスっていたので修正したら、顔色はかなり良くなった。

透明色処理が正しく実装されていないと、タイトル表示がこうなるのは正しい。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">rustファミコンエミュ進捗です（けっこううれしい <a href="https://t.co/ySaYyMJD38">pic.twitter.com/ySaYyMJD38</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1170563567344480256?ref_src=twsrc%5Etfw">September 8, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

ドンキーコングはほぼ完動。

## マリオ完動に向けて修正

ここまで来たら自分の理解度が深まったこともあり、画面出力と実装をにらめっこしてどこがおかしいか順に潰していった。結構エスパーっぽいデバッグだった気もする。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">これが本当のSuper Mario Bros.です（Pad入力は正常に入るようになった。あとはめげない心 <a href="https://t.co/QSd1uDbqMt">pic.twitter.com/QSd1uDbqMt</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1170689415670198272?ref_src=twsrc%5Etfw">September 8, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

風を感じていた。これはスクロールの値が8倍に適用されている。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">rustファミコンエミュ進捗です（もう3声くらいで遊べそう？ <a href="https://t.co/B3wKd5s2dJ">pic.twitter.com/B3wKd5s2dJ</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1170735270871715840?ref_src=twsrc%5Etfw">September 8, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

これは属性テーブルがスクロールに追従していないので、背景と色がずれている。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">背景塗りしていないので隠しブロックが見えるの草。（rustファミコンエミュ進捗です、属性テーブル参照バグが治ったので色が正常化されつつある <a href="https://t.co/MbEmPEb8cc">pic.twitter.com/MbEmPEb8cc</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1170926784398815234?ref_src=twsrc%5Etfw">September 9, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

背景色処理をしていないので、隠しブロックが見えている。あとスクロールがカクカクしている。これはスクロール値の計算間違い。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">やったー！スクロールがちゃんと動くようになった！！！前のやつは8px単位でスクロールしててかくかくしてる <a href="https://t.co/ztvnCvqurJ">pic.twitter.com/ztvnCvqurJ</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1170935889066770438?ref_src=twsrc%5Etfw">September 9, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

動画の最後で変死しているが、これはパックンフラワーが見えていない。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">Sprite強制全面表示。完全に理解した <a href="https://t.co/64SDIaechX">pic.twitter.com/64SDIaechX</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1170939139518160896?ref_src=twsrc%5Etfw">September 9, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

これはわかりやすい。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">直した <a href="https://t.co/Cy8L8kxVe4">pic.twitter.com/Cy8L8kxVe4</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1170949923484815361?ref_src=twsrc%5Etfw">September 9, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

かなり現物に近づいた。あとは黒い部分だけ。これは[背景色, 背景Sprite, BG, 全面Sprite]とあって
透明色の黒色が選ばれていたら後ろの色（e.g.空の色）を出してあげるのが正解。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">やったー！！！！！！！！ウオオオオオーーーーーーーーーーー！！！！！！！！！！！！！！！！！ <a href="https://t.co/RUaa0l0t9R">pic.twitter.com/RUaa0l0t9R</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1170954266313281536?ref_src=twsrc%5Etfw">September 9, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">rustファミコンエミュ進捗です（すごくうれしい<a href="https://t.co/VvBct6k8EX">https://t.co/VvBct6k8EX</a> <a href="https://t.co/fdJ1xN1JUL">pic.twitter.com/fdJ1xN1JUL</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1170963720492576769?ref_src=twsrc%5Etfw">September 9, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

死ぬほど嬉しかった。

## 所感

~~いかがでしたか?~~ とりあえずマリオが遊べるまでのツイートをまとめた。最近WebAssemblyに吐いてブラウザでも遊べるようにしたので興味あれば遊んでみていただけると嬉しい限り。

[rust-nes-emulator.netlify.com](https://rust-nes-emulator.netlify.com/)

### Rustを使ってみて

WebAssembly版のポーティングするjavascriptを少し書いていただけでバグまみれになったので、私はRustがすごい良い言語だと感じる。以下紹介する

### 意図しない型変換、Overflow/Underflowなどはすぐに原因がわかった

異なる型の演算はそもそもコンパイルエラー、Overflow/Underflowは実装者が明示しないと実行時エラー。
なんかおかしい→調べたらOverflowだった。みたいなのがなかった。（すぐに場所がわかった）

### 書き換え可能な参照をばらまけないので、怪しい設計がコンパイル通らない

別の場所からフラグを書き換えるような行儀の悪いことはそもそもコンパイルエラー。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">wasmにしてなんか挙動がおかしかったので、console.logデバグしていたがROMを読み込む前にエミュを開始していたっぽい <a href="https://t.co/w91y01SxEG">pic.twitter.com/w91y01SxEG</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1173462900184190976?ref_src=twsrc%5Etfw">September 16, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

ありがとうRust...。私はか弱い凡才なのでコンパイラに守られたい。

### 速い、wasmにしても速い

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">これがwasmの本気ッッッーーー！！！！（計算ミスっだだけです。しかもまだ余裕ある <a href="https://t.co/DsjzwXuxVT">pic.twitter.com/DsjzwXuxVT</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1173572680106168320?ref_src=twsrc%5Etfw">September 16, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

流石にここまで早くなると思ってなかった。3.6msか...。

## 終わりに

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">最近の一連のアレ、そのうち本に書こうと思うよ</p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1173516903945359362?ref_src=twsrc%5Etfw">September 16, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">「この件、意外と反響がある。もともとやるつもりではいたがまともに考えないと。」の顔 <a href="https://t.co/TwAnSKHAbr">pic.twitter.com/TwAnSKHAbr</a></p>&mdash; kamiya (@kamiya_owl) <a href="https://twitter.com/kamiya_owl/status/1173586537444864003?ref_src=twsrc%5Etfw">September 16, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
