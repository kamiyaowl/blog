---
layout: post
title: "Chisel3を始めるにあたって(2/2)"
date: 2019-03-17
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

[前回までの記事](https://kamiyaowl.github.io/blog/start-chisel/)で、Compressorのscala上での実装について解説した。今回はscala上でのテスト方法、verilogの生成方法、verilogでの検証について解説する。

# scalaでのテスト

chiselにはテストをするためのツールセットも用意されている。今回はScalaTestと合わせて使って、Compressorが正しく振る舞っているのかテストしてみる。

## ScalaTestの導入

前回`build.sbt`を記載した際に一緒に追加しているため、sbt上で`test`と打つだけで実行できる。

## テストの記述

[/src/test/scala/audio/CompressorSpec.scala](https://github.com/kamiyaowl/chisel-practice/blob/master/src/test/scala/audio/CompressorSpec.scala)

※ 2019/03/19追記：ChiselFlatSpecを継承させるのが良い

{% highlight scala %}
package audio

import org.scalatest._
import chisel3._
import chisel3.util._
import chisel3.iotesters.{ChiselFlatSpec, Driver, PeekPokeTester}

class CompressorSpec extends ChiselFlatSpec {
  "Distortion" should "parametric full test" in {
    val result = Driver(() => new Compressor(32)) {
      c => new PeekPokeTester(c) {
        (-10000 to 10000) filter(_ % 1000 == 0) foreach { data_in =>
          (0 to 10000) filter(_ % 1000 == 0) foreach { data_point =>
            (0 to 100) foreach { data_rate => {
                poke(c.io.in, data_in)
                poke(c.io.point, data_point)
                poke(c.io.rate, data_rate)
                step(2) // 1cycで取り込み計算、2cyc目で出力

                if (Math.abs(data_in) <= data_point || data_rate <= 0) {
                  expect(c.io.out, data_in)
                } else {
                  expect(c.io.out, ((Math.abs(data_in) - data_point) / data_rate + data_point) * (if (data_in < 0) -1 else 1))
                }
              }
            }
          }
        }
      }
    }
    assert(result)
    true
  }
}
{% endhighlight %}

例によって解説していく

### ScalaTest

scalaはとにかくDSLがすごいので、scalaTestも文章のように書ける。詳しくはscalatestの仕様を参照。
FlatSpecに関しては以下のような雛形である。

{% highlight scala %}
class CompressorSpec extends FlatSpec with Matchers {
  "あたりまえテスト" should "出力が100になるよね" in {
    val myModule = new MyModule()
    myModule.func() == 100
  }
}
{% endhighlight %}

### PeekPokeTester

これはchiselで作成したモジュールに何かしらの入力やクロックを与え、出力値を観測するテストセットである。
使用にあたっては、Moduleを駆動するDriverとセットで使用するらしい。

3重ループを組んでややこしくしているだけなので、要所だけ抜き出した。入力を与え、クロックを駆動し、出力値を確認するだけである。

{% highlight scala %}
val result = Driver(() => new Compressor(32)) {
  c => new PeekPokeTester(c) {
      poke(c.io.in, 100)
      poke(c.io.point, 1000)
      poke(c.io.rate, 2)
      step(2) // 1cycで取り込み計算、2cyc目で出力

      expect(c.io.out, 100)
    }
  }
assert(result)
{% endhighlight %}

テストは可能な限りキチンと書くべきである。特にこの後動かなくなった場合に切り分けが大変になりことが想定できるからである。（最適化、使い方、変換後のverilogに問題がある、chisel上での設計に問題があるなど）

これで`$ sbt test`がきちんと通ることを確認したら、いよいよverilogを出力する。

# verilog出力

`Compressor.scala`の末尾に以下の記述を増やす。処理としては実行可能なプログラムにして、verilogを生成するプログラムを書く。

`class Compressor`とかぶっていると感じるかもしれないが、scalaではいわゆるstaticなものはobjectに定義する。

`extends App`はそのままエントリポイントになる。

`chisel3.Driver.execute`でverilogを生成。

{% highlight scala %}
object Compressor extends App {
  chisel3.Driver.execute(args,()=>new Compressor(32))
}
{% endhighlight %}

これで、プロジェクトルートに`Compressor.v`が生成される。

# verilogシミュレーション

生成されたverilogが正しく動作しているか、波形確認も含めて実施。

## テストベンチ作成

まずはテストベンチを記述。詳細は解説しないが、scalaでのテストと同じように入力を変化させながら出力の振る舞いを確認するようにしている。

`$dumpfile`と`$dumpvars`であとで波形確認したいデータをvcdファイルに出力しておく。


[`/verilog/Compressor_tb.v`](https://github.com/kamiyaowl/chisel-practice/blob/master/verilog/Compressor_tb.v)

{% highlight verilog %}
`timescale 1ns/1ns

module tb();
    localparam DELAY = 10;
    localparam POINT_DELAY = 10000;
    localparam RATE_DELAY = 100000;

    reg CLK = 0;
    reg RST = 0;
    reg signed [31:0] in_data = 32'sd0;
    wire signed [31:0] out_data;
    reg signed [31:0] point = 32'sd0;
    reg signed [31:0] rate = 32'sd0;

    Compressor d(
        CLK,
        RST,
        in_data,
        out_data,
        point,
        rate
    );

    initial begin
        CLK = 0;
        forever #DELAY CLK = !CLK;
    end

    initial begin
        point = 32'sd0;
        forever #POINT_DELAY point = $signed(point) + $signed(32'sd100);
    end
    initial begin
        rate = 32'sd0;
        forever #RATE_DELAY rate = $signed(rate) + $signed(32'sd1);
    end

    initial begin
        RST = 1;
        #100 RST = 0;
    end

    always @ (posedge CLK) begin
        if (RST == 1) begin
            in_data <= 32'sd0;
        end else begin
            in_data <= $signed(in_data) + $signed(32'sd11);
        end
    end

    initial begin
        $dumpfile("Compressor.vcd");
        $dumpvars(0, CLK);
        $dumpvars(0, RST);
        $dumpvars(0, in_data);
        $dumpvars(0, out_data);
        $dumpvars(0, point);
        $dumpvars(0, rate);

        #1000000 $finish();
    end
endmodule
{% endhighlight %}

## テストベンチ実行

icarus-verilogで実行してみる。

{% highlight bash %}
$ iverilog verilog/Compressor.v verilog/Compressor_tb.v 
{% endhighlight %}

これで`a.out`というシミュレーション実行ファイルが作成されるので、`vvp`で実行する。

{% highlight bash %}
$ vvp a.out 
{% endhighlight %}

シミュレーションを実行すると、テストベンチに記述したとおり.vcdファイルが作成される。

## 波形表示

.vcdファイルはgtkwaveでそのまま開ける。テストベンチで予期したとおりの振る舞いになっているか確認する。
なおmacOSの最新の場合、署名問題で起動できない場合があるので設定→セキュリティをイジる必要があった。

![wave](https://pbs.twimg.com/media/D1xSoRZU0AACTKT.jpg:large)


# まとめ

駆け足ながら一通りのフローを舐めた。おそらく自分で見返すことが訪れると思う。

ともあれ新規参入者にはかなりハードルが高いと感じる。HLSでさえもHDL設計能力が求められるのに、Chiselがそれから逃げられるツールではないことは自明である。（HLSの場合、最適化しやすいC言語を書くようなスキルが要求される）
しかしながらaltHDLとしては、scalaでそのままテストが書けることが何より設計を楽にするのではないかと感じる。
verilatorもあるので好みの問題かもしれないが...。個人的には書いた言語と同じ言語でテストをしたいのである。
