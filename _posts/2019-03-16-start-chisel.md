---
layout: post
title: "Chisel3を始めるにあたって(1/2)"
date: 2019-03-16
description: 'scalaでHDLが記述できるライブラリの導入'
main-class: 'jekyll'
image: 
color: '#B31917'
tags:
- scala
- FPGA
- Chisel
categories:
---

FPGAの論理回路設計にはHDLの習得が必須である。従来であればVHDL, Verilog HDL, SystemVerilogあたりが主流である。
しかしHDLで論理実装を行うのは敷居が高くまた時間がかかることもあって近年では高位合成なども流行っている（*要出典)

私はverilogで育ってきたのだが、確かにverilogの中途半端にゆるい成約は謎のバグの温床になっていたりすることがおおい。
かといってVHDLを書いてみても冗長という感想なので、altHDLを探していたところにscalaでHDLをなすChiselなるライブラリがある。

[Chisel - Constructing Hardware in a Scala Embedded Language](https://chisel.eecs.berkeley.edu/)

scalaとHDLを両方習得している人間は稀だろと様子を見ていたが、RISC-VやTPUに使われているらしく改めて触れてみることにする。

長々と書いたが、Chiselで作って見るところまでの手順まとめである。荒削りなメモなので誤り等があるかもしれないのでご容赦いただきたい。

# 導入

## Java

Scalaやsbtを動かすのに必要。JDKを公式ページよりインストール

インストール確認

{% highlight bash %}
$ java --version
{% endhighlight %}


## sbt

scalaのプロジェクト管理ツールのようなもの。plugin管理とビルドスクリプトを兼ねたものだと思ってもらえれば。

{% highlight bash %}
$ brew install sbt
{% endhighlight %}

動作確認

{% highlight bash %}
$ sbt about
{% endhighlight %}

## icarus-verilog

verilogファイルをシミュレーションするために必要。ベンダーツールのVivado(Vivado Simulator), Quartus Prime(ModelSim Intel)でもできる。

{% highlight bash %}
$ brew install icarus-verilog
{% endhighlight %}

動作確認

{% highlight bash %}
$ iverilog -v
{% endhighlight %}

## gtkwave

シミュレーション波形の表示用。

{% highlight bash %}
$ brew install caskroom/cask/gtkwave
{% endhighlight %}

# プロジェクト作成

`chisel`とテスト用の`chisel-iotesters`を追加するために`/build.sbt`に以下を記述。パッケージ名とか組織名は適宜。

{% highlight scala %}
import Dependencies._

ThisBuild / scalaVersion     := "2.12.8"
ThisBuild / version          := "0.1.0-SNAPSHOT"
ThisBuild / organization     := "kamiyaowl.chisel-practice"
ThisBuild / organizationName := "kamiyaowl"

lazy val root = (project in file("."))
  .settings(
    name := "chisel-practice",
    libraryDependencies += scalaTest % Test
  )

scalacOptions ++= Seq("-deprecation", "-feature", "-unchecked", "-language:reflectiveCalls")

val chiselGroupId = "edu.berkeley.cs"
libraryDependencies ++= Seq(
  chiselGroupId %% "chisel3" % "3.0.+",
  chiselGroupId %% "chisel-iotesters" % "1.1.+"
)
resolvers ++= Seq(
  Resolver.sonatypeRepo("snapshots"),
  Resolver.sonatypeRepo("releases")
)
{% endhighlight %}

この状態で`$ sbt`をすると、周辺ライブラリも引っ張ってきてくれた後に対話モードに移る。今後のビルド、テストはこのsbtの対話モードで行う。といっても`reload`, `test`, `run`あたりがメインである。

# 開発

## 作るもの

作りたいもの次第だが、今回はオーディオエフェクターやミキサーにあるコンプレッサを作ってみる。
音の粒度を整えるなどというが、司会者マイクなどですごく大きな音などを抑制するときなどに使う。

単純なものであればでかすぎる音をでかすぎないように傾きをいじっているだけである。
この傾きをratio、大きい音と判定するしきい値をthreshold(今回記載したコードではpoint)と呼ぶ。

aをratio、bをpointとした場合に、入力された音xに対し以下の処理を施すだけで良い。

$$
\begin{eqnarray}
    \left\{
        \begin{array}{l}
            y = x & \text{(|x| < b)} \\
            y = \frac{x}{a} & \text{(|x| ≥ b)} \\
        \end{array}
    \right.
\end{eqnarray}
$$

（まともなやつは大きな音が入ってから圧縮するまでのAttack、音のレベルを抑制する期間であるReleaseなどでトリガするものが主流。興味があれば調べてみてほしい）

## 方式

HDLを設計するにあたって以下を考える。

* Interface
    * in : 音声入力
    * out : 音声出力
    * point : 傾きの変化点、threashold
    * ratio : 傾き、今回は単純に入力値を割るだけにする
* Interfaceはすべてint32で扱う。表現は2-complementary
* Masterより供給されるクロックに同期して結果を出力
    * 途中計算をラッチすることも考慮し、total delayは1~5あたりで

## 実装

[`/src/main/scala/audio/Compressor.scala`](https://github.com/kamiyaowl/chisel-practice/blob/master/src/main/scala/audio/Compressor.scala)

{% highlight scala %}
package audio

import chisel3._
import chisel3.util._

class Compressor(width: Int) extends Module {
  val io = IO(new Bundle {
    val in = Input(SInt(width.W))
    val out = Output(SInt(width.W))
    // params
    
    val point = Input(SInt(width.W)) // 歪ませる変化点
    
    val rate = Input(SInt(width.W))  // pointより大きい値に対して除算する数値、0,1の場合は無効

  })
  val dst = RegInit(0.S)
  io.out := dst

  val is_negative = io.in < 0.S
  val abs_in = Mux(is_negative, -io.in, io.in)
  val is_valid = io.rate > 0.S
  when((abs_in > io.point) && is_valid) {
    val diff = (abs_in - io.point) / io.rate
    val next = io.point + diff.asInstanceOf[SInt]
    dst := RegNext(Mux(is_valid, Mux(is_negative, -next, next), io.in)) // rate:0->1に遷移させた直後のdstが不定になってしまうため

  } .otherwise {
    dst := RegNext(io.in)
  }
}
object Compressor extends App {
  chisel3.Driver.execute(args,()=>new Compressor(32 ))
}
{% endhighlight %}

流石に解説を簡単に書く。

### インポート

これはchiselのライブラリを使うためのおまじない。

{% highlight scala %}
import chisel3._
import chisel3.util._
{% endhighlight %}


### 入出力定義

自作モジュールの宣言と、IOポートの定義をしている。chiselのモジュールとして定義する場合`chisel3.Module`を継承する

`class Compressor(width: Int)`の`width`はコンストラクタであり、ビット幅可変で使えるようにしている。verilogで言うところのlocalparam。

`Bundle`というのはchisel上でユーザ定義の構造体を扱うような機構のこと。これを`IO`に渡すと、合成時に入出力ポートとして合成されるらしい。

`Input()`および`Output()`はポート方向の定義、その中にある`SInt(width.W)`は符号付きでビット幅がwidthビットの整数であることを示している。

`width.W`となっているが、chisel上ではビット幅指定するところの型が`chisel3.Width`のため`Int->chisel3.Width`変換を呼び出している具合である。
余談だが、これはchisel上のライブラリによってあたかも`Int`のメンバ関数(`.w`)が増えているかのように扱える。scalaの良い機能である。他にも`[Int].U`で`Int->chisel3.UInt`、`[Int].S`で`Int->chisel3.SInt`、`[Bool].B`で`Bool->chisel3.Boolean`などがある。

{% highlight scala %}
class Compressor(width: Int) extends Module {
  val io = IO(new Bundle {
    val in = Input(SInt(width.W))
    val out = Output(SInt(width.W))
    // params
    val point = Input(SInt(width.W)) // 歪ませる変化点

    val rate = Input(SInt(width.W))  // pointより大きい値に対して除算する数値、0,1の場合は無効

  })
  ...
}
{% endhighlight %}

### ロジック

※ここは自分の解釈の途中である部分も含まれるので、誤りがあったら指摘いただきたい。
statementの後ろに数字を付けたが、定義したものは大きく以下のパターンに分類される。


{% highlight scala %}
val dst = RegInit(0.S) // --- (1)


io.out := dst // --- (2)


val is_negative = io.in < 0.S // --- (3)


val abs_in = Mux(is_negative, -io.in, io.in) // --- (4)

...

dst := RegNext(io.in) // --- (5)


// --- (6)

when((abs_in > io.point) && is_valid) {
    ...
} .otherwise {
    ...
}

{% endhighlight %}


#### (1)レジスタ

`reg`として定義される。HDL設計の上では暗黙の了解になっているが、組み合わせ論理回路をひたすら連結した設計をしてしまうと、信号が変化してから確定するまでのパスが非常に長くなってしまう。これは周波数を上げるのにあまり向かなかったりする。理由はこれだけではないが、ある程度の計算ごとにRegすなわちFF（フリップフロップ）を挿入するなどしたりする。

{% highlight verilog %}
reg [32:0] dst;
{% endhighlight %}

dstが33bitの変数になっていることに気づいたらかなり鋭い。chiselはビット幅を明示しなかった場合に、行われている計算から推論してくれているように見えている(*要出典)

#### (2)`:=`演算子

レジスタやワイヤ、ポートを`:=`演算子でつなぐと、基本は`assign`に変換されるようだ。
基本といったのは、後述する`RegNext`などを使用した場合はFFが生成されるからである

{% highlight verilog %}
assign _GEN_2 = dst[31:0];
assign io_out = $signed(_GEN_2);
{% endhighlight %}


#### (3)scalaでの変数定義

これらは組み合わせ論理回路、すなわち`assign`に変換されるようだ。これはクロックには同期しない。

{% highlight verilog %}
wire  is_negative;
assign is_negative = $signed(io_in) < $signed(32'sh0);
{% endhighlight %}

#### (4)Mux

これらは組み合わせ論理回路だが、3項演算子と等価になるようだ。

{% highlight verilog %}
assign _T_10 = $signed(32'sh0) - $signed(io_in);
assign _T_11 = _T_10[31:0];
assign _T_12 = $signed(_T_11);

assign abs_in = is_negative ? $signed(_T_12) : $signed(io_in);
{% endhighlight %}


#### (5)同期代入

`RegNext`を使った場合、同期回路に変換される。このモジュールに供給されたクロックCLKの立ち上がりに同期して、入力値をキャプチャするような回路になる。

verilogで書き下すと以下の回路と等価になるのでそのまま変換される。

{% highlight verilog %}
always @(posedge clock) begin
    _T_32 <= io_in;
end
{% endhighlight %}


また、`reg`のように初期値が決まっている場合はリセット付きの以下の回路が生成される。

{% highlight verilog %}
always @(posedge clock) begin
    if (reset) begin
        dst <= 33'sh0;
    end else begin
        if (_T_15) begin
            dst <= _T_30;
        end else begin
            dst <= ...;
        end
    end
end
{% endhighlight %}

#### (6)when句

どうやら組み合わせ回路のみの場合はassignの条件として生成され、同期回路が含まれていた場合は(5)項にあるような`always`の内部にifが生成されるようだった。おそらく`switch`なども同様な気がする。


書いていたら意外とボリュームがあるので、ここで区切ることにする。

まだまだ理解がおぼつかない部分があるが、HLSのアプローチとは異なりHDLをより抽象度を高めて書けるところにメリットが有る、と感じる。

[Chisel3を始めるにあたって(2/2)へ]()