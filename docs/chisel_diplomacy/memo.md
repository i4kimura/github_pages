# Chisel v.s. Verilog

### （言われる前に）Chiselの悪口を書いておく

#### Q. Rocket-ChipのVerilog生成遅くない？

- A. Scalaに聞いて。
- 印象としては、Chisel → FIR はあまり遅くない印象。
- FIRからVerilogの変換フェーズが多くて生成が遅くなってるかな。後述するCIRCTが速く成熟してほしい。

#### Q. Verilogみたいに直感的に書けないんだよね...

- それはあなたがVerilog脳になっているから。
- というのはさておき、Verilogと違って「レジスタを生成して接続している」感は否めないよねー。

```scala
val rr = RegNext (rd)		# 私レジスタ作ってます！配線rdと接続してます！
```

Verilogを見たときのなんだろう...この安心感。

```verilog
logic [31: 0] rr;
always_ff @ (posedge clk) begin
  rr <= rd
end
```

