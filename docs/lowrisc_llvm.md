# RISC-V LLVMバックエンドのステップバイステップガイド

[toc]

### 序章

新しいコンパイラ・バックエンドを作成することは、時に乗り越えられないほどの困難に思えることがあります。複雑なコンパイラ・ツールチェーンにコードを統合するという課題だけでなく、不安定なアセンブラ、リンカ、ランタイム・ライブラリ、さらにはシミュレータにも悩まされることがあります。私の考えでは、バックエンド開発にインクリメンタルなアプローチを取ることが、この複雑さにうまく対処する唯一の方法です。他のバックエンドからコードをコピーして、納得のいく出力が得られるまでFixupすることで、初期の開発を迅速に進めることができますが、問題を切り分けてデバッグすることは非常に困難な場合があります。

このドキュメントでは、RISC-Vバックエンドを実装するためのアプローチを説明します。LLVMバックエンドの開発を検討している方の参考になれば幸いです。

注意: このドキュメントは作業中のものです。HTMLレンダリングされた "v1 "は、まもなくlowrisc.orgに掲載される予定です。

### LLVMの構築
他のことをする前に、変更されていない LLVM をチェックアウトしてビルドできることを確認しておきましょう。

```sh
git clone https://git.llvm.org/git/llvm.git
cd llvm
```

この説明書は、LLVMを構築するための非常に凝縮されたレシピを提供しています。必要であれば、公式のドキュメントに詳細が記載されています。

まず、必要な依存関係を確認してください。LLVMのドキュメントには、より完全なリストがありますが、ninja、CMake、最近のホストコンパイラ(Clangなど)があることを確認するのが良い出発点です。

```sh
sudo apt-get install clang ninja cmake
```

LLVM開発では、膨大な数のインクリメンタルリビルド（小さな変更を加えた後にコンパイルすること）が必要になります。この時間を短縮するのに役立つコンパイラオプションを選択することは、生産性の向上に非常に役立ちます。このブログ記事では、いくつかの有用なガイダンスを提供しています。

```sh
mkdir build
cd build
cmake -G Ninja -DCMAKE_BUILD_TYPE="Debug" \
  -DBUILD_SHARED_LIBS=True -DLLVM_USE_SPLIT_DWARF=True \
  -DLLVM_OPTIMIZED_TABLEGEN=True \
  -DLLVM_BUILD_TESTS=True \
  -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ \
  -DLLVM_TARGETS_TO_BUILD="X86" ../
cmake --build .
```

これらのオプションのいくつかを説明します。

- `cmake -G Ninja`はCMakeがNinjaビルドシステムを使用し、Makefilesではなくbuild.ninjaファイルを生成します。
- `-DBUILD_SHARED_LIBS=True`および`-DLLVM_USE_SPLIT_DWARF=True`は、最終的にファイルI/Oを削減することで、ビルド時間のインクリメンタルな短縮を実現します。
- `-DLLVM_TARGETS_TO_BUILD="X86"`は、X86ターゲットのみをビルドすることを指定します。これはビルドを高速化するための一つの方法です。
- `-DCMAKE_BUILD_TYPE="Debug"`は、アサーションとバックトレースを有効にしたデバッグビルドを生成することを意味します。バイナリはリリースビルドよりも大きく、遅くなります。
- `-DLLVM_OPTIMIZED_TABLEGEN=True は`、tblgen のリリースバイナリがビルドされて使用されることを意味します。これにより、デバッグビルドのビルド時間を短縮することができます。
- `-DLLVM_BUILD_TESTS=True`は、オプション名が示すように、デフォルトでユニットテストがビルドされます。
  `-DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++`の場合、システムのClangコンパイラが使用されます。これは一般的にGCCよりも高速で、より良いエラーメッセージを出力することができます。

### LLVMテストスイートの実行
LLVMには、大規模なテストインフラストラクチャがあります。コーディングに没頭する前に、基本的なことをよく理解し、テストスイートを呼び出すことができることを確認する価値があります。

LLVMのリグレッションテストは、`test/`と`unittest/`のサブディレクトリにあります。`llvm-lit`ツールを使って、これらすべてのテストを呼び出すことができます。ビルドディレクトリから `./bin/llvm-lit test -s` を実行してください。これはすべてのテストを実行し、素敵なプログレスバーを表示します。`./bin/llvm-lit --show-suites test` で自分自身で確認できるように、lit ツールは両方のテストセットを発見します。完全なスイートは合理的に迅速に実行されますが、テストのサブセットだけを実行するのも便利です。例えば、`./bin/llvm-lit -s --filter 'ScalarEvolution' test` は、名前に ScalarEvolution が含まれるテストのみを実行します。edit-compile-testサイクルを高速化するのに役立つもう一つの便利なパラメータは `-i` で、これはFixupされたテストと最近失敗したテストを最初に実行する。これは、以前に失敗したテストをFixupしたのか、最近Fixupしたテストがまだ合格しているのかを迅速に判断したい場合によく使われます。

### レポの設定
これでLLVMをビルドしてテストスイートを実行できるようになりました。あとはGitリポジトリをセットアップして、変更をコミットしたり、アップストリームからの変更を簡単に取り込むことができるようにするだけです。これにはいろいろな方法がありますが、私が最も有用だと感じた方法を以下に示します。

まず、'origin' の名前を 'upstream' に変更します。

```sh
git remote rename origin upstream
git config --unset branch.master.remote
```

そして、新しい 'origin' を git remote で設定し、コードをホストするようにします (GitHub アカウントのリポジトリなど)。

```sh
git remote add origin your@gitrepo.org:here.git
git push -u origin master
```

今後この新しい 'origin' からのチェックアウトでは、上流のリモートを明示的に追加する必要があります。

```sh
git remote add upstream http://llvm.org/git/llvm.git
```

上流からの変更をいつでもリベースするには

```sh
git fetch upstream
git rebase upstream/master
```

## 新しいバックエンドの開始
### Target tripleのサポートを追加

最初のステップは、ターゲットに依存しない"triple"解析コードでアーキテクチャ名を認識するサポートを追加することです。tripleは、アーキテクチャ、ベンダ、オペレーティングシステム、場合によっては環境を識別するために使用されます。RISC-Vでは、認識するアーキテクチャ名は2つあります - riscv32とriscv64です。

必要な変更には、riscv32 と riscv64 の追加が含まれます。

- `Triple::ArchType` enum
- `Triple::getArchTypeName`、`Triple::getArchTypePrefix`、`Triple::getArchTypeForLLVMName`、`Triple::parseArch`
- `Triple::getDefaultFormat`（`Triple::ELF`を返す）、`Triple::getArchPointerBitWidth`、`Triple::isLittleEndian`などのヘルパー関数。
- 最も重要なのは、`unittests/ADT/TripleTest.cpp` をFixupして上記のすべてをテストすることです。
  完全なリストはパッチを参照してください。

{{% showpatch "recognise riscv32 riscv64 triple parsing" %}}

これで、(このチュートリアルの最初の部分で説明したように) lit を使ってテストを実行し、パッチをコミットできるようになりました。

### RISC-V ELF ファイルのサポート

バックエンド自体を始める前の最後のステップは、RISC-V ELFファイルに必要な定義を追加することです。これには以下が含まれます。

- リロケーションタイプを`include/llvm/BinaryFormat/ELFRelocs/RISCV.def`に追加します。
- `EM_RISCV`マシン定義の追加（参考リストは[こちら](http://www.sco.com/developers/gabi/latest/ch4.eheader.html)を参照
- `include/llvm/Object/ELFObjectFile.h`の様々なcase文に`EM_RISCV`を追加しました。いくつかのアーキテクチャでは、32ビット対64ビットで異なる`EM_*`定数を持っていますが、RISC-V（MIPSのような）では、オブジェクトファイルが64ビットマシンの32ビット用であるかどうかを判断するために`EI_CLASS` ELFフィールドを使用しています。
- ELFYAML、llvm-objdump、llvm-readobjのcase文に`EM_RISCV`を追加しました。
  詳細はパッチを参照してください。

{{% showpatch "add RISC-V ELF defines" %}}

### スケルトンバックエンドの追加
次のステップは，「スケルトン」バックエンドを追加することです。これには以下が含まれます。

- LLVM ビルドシステムに RISC-V バックエンドを登録します。
- スタブ `RISCVTargetMachine.{cpp,h}` を追加します。
- ターゲットを登録するスタブ `RISCVTargetInfo.cpp` を追加します。

`RISCVTargetMachine` の最も興味深い部分は，おそらく `computeDataLayout` ヘルパーでしょう。これは，マシン（riscv32 または riscv64）に適したデータレイアウト文字列を返します。例えば、riscv32のデータレイアウト文字列は `"e-m:e-p:32:32-i64:64-n32-S128 "`です。これを引き離すと、この文字列は次のような意味になります。

- `e`: リトルエンディアン
- `m:e`: ELF マングリングモード
- `p:32:32`: ポインタは32ビット幅で、32ビットのアラインメントを持っています。
- `i64:64`: ビット整数は 64 ビットのアラインメントを持つ
- `n32`: ネイティブの整数型は32ビットです。
- `S128`: スタックのナチュラルアライメントは128ビット(16バイト)です。

これでバックエンドをビルドできるようになりました。LLVM_EXPERIMENTAL_TARGETS_TO_BUILDでRISCVを指定して、CMakeを再実行する必要があります。

```sh
cmake -G Ninja -DCMAKE_BUILD_TYPE="Debug" \
  -DBUILD_SHARED_LIBS=True -DLLVM_USE_SPLIT_DWARF=True \
  -DLLVM_OPTIMIZED_TABLEGEN=True \
  -DLLVM_BUILD_TESTS=True \
  -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ \
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD="RISCV" ../
cmake --build .
```

`./bin/llvm/as --version` のようなコマンドの出力を確認することで、ターゲットが正しく登録されているかどうかを確認することができ、登録されたターゲットとして riscv32 と riscv64 がリストアップされているはずです。もちろん、RISC-Vバックエンドはこの時点では単なるスタブなので、`./bin/llc -march=riscv64 ../test/Object/Inputs/trivial.ll`のようなことを試しても、アサーションですぐに失敗します。



## RISC-V アセンブリの解析サポートの実装
### 概要

警告。残念ながら、アセンブラがテスト可能になる前に書かなければならないコードがかなり多いです。

必要なステップは以下の通りです。

- RISC-Vのレジスタ、命令フォーマット、命令を記述した最小限のTableGenファイルを追加します。
- MC層(マシンコード層)を正しく初期化するために、RISC-VのMCTargetDescを追加しました。
- 単純なRISC-Vアセンブリファイルを解析できるRISCVAsmParserを追加しました。
- 全体をテスト可能にするために，RISCVInstPrinter を追加します（これはこのシリーズの次のパートで行います）。

既に動作するアセンブラを持っているアーキテクチャをターゲットにしている場合、MC層のサポートを完全に実装することを避けて、外部のアセンブラに頼ることができます。しかし、MC層から始めることは、バックエンドを段階的に構築するのに有効な方法です。

### TableGenファイルの追加
[TableGen](http://llvm.org/docs/TableGen/)は、レコードを指定するための簡潔で読みやすい構文を提供するドメイン固有の言語です。LLVMバックエンドの場合、これらのレコードにはレジスタ、命令フォーマット、命令定義が含まれます。[RISC-Vの仕様書](https://riscv.org/specifications/)は，このための参考文献として有用です。

#### RISCVRegisterInfo.tdの追加

RISCVRegisterInfo.tdは，各レジスタを`x1`のような名前とABIの名前`ra`で定義します。各レジスタの定義は，`include/llvm/Target/Target.td`で定義されている`Register`からサブクラス化されています。代替名をサポートするために、一意の `RegAltNameIndex` を定義する必要があります。

```
def ABIRegAltName : RegAltNameIndex
```

次に、各レジスタについて、有効なRegAltNameIndicesと同様に代替名のリストを指定します。例えば、以下の例では、"zero"は`ABIRegAltName`です。

```
let RegAltNameIndices = [ABIRegAltName] in {.
  def X0_32 : RISCVReg32<0, "x0", ["ゼロ"]>, DwarfRegNum<[0]>.
```

レジスタは，適切な`RegisterClass`に集められます。クラス内のレジスタの順序は，好ましいレジスタ割り当て順序を反映しています。しかし、ここではMC層だけに焦点を当てているので、今のところは昇順に追加しても構いません。

```
def GPR : RegisterClass<"RISCV", [i32], 32, (add
  (sequence "X%u_32", 0, 31)
)>;
```

#### RISCVInstrFormats.tdの追加

各命令フォーマットの定義は、個々のRISC-V命令のスーパークラスとして機能します。命令フォーマットは，その型の命令がどのようにエンコードされるべきかを指定します。例えば、'I' フォーマットの命令は、3 ビットの funct フィールド、7 ビットの opcode フィールド、2 つのレジスタ、12 ビットの immediate フィールドを持っています。

```
class FI<bits<3> funct3, bits<7> opcode, dag outs, dag ins, string asmstr, 
    list<dag> pattern>
    : RISCVInst<outs, ins, asmstr, pattern>
{
  bits<12> imm12;
  bits<5> rs1;
  bits<5> rd;

  let Inst{31-20} = imm12;
  let Inst{19-15} = rs1;
  let Inst{14-12} = funct3;
  let Inst{11-7} = rd;
  let Opcode = opcode;
}
```

#### RISCVInstrInfo.td の追加
フォーマットが定義されているので，命令を追加するのは比較的簡単です。例えば，以下のようになります。

```
def simm12 : Operand<i32>;

class ALU_ri<bits<3> funct3, string OpcodeStr> :
      FI<funct3, 0b0010011, (outs GPR:$rd), (ins GPR:$rs1, simm12:$imm12),
         OpcodeStr#"\t$rd, $rs1, $imm12", []> {}

def ADDI  : ALU_ri<0b000, "addi">;
```

`ALU_ri`クラスを定義することで、同様に定義されたreg-imm ALU命令群を指定する際に、繰り返しを避けることができます。

文字列`"\t$rd, $rs1, $imm12"`は、命令のオペランドがテキストアセンブリでどのように表現されるかを指定します。

#### RISCV.tdの追加と記述の確認

RISCV.tdは，最上位のTableGenファイルです。インクルードを介してレジスタや命令の定義を組み込むだけでなく，プロセッサのモデルや機能の定義，その他のバックエンドの雑多な設定が含まれています。

生成されたTableGenレコードは、以下のように実行することで確認することができます。

```sh
./bin/llvm-tblgen -I ../lib/Target/RISCV/ -I ../include/ -I ../lib/Target/ ../lib/Target/RISCV/RISCV.td
```

そして、`lib/Target/RISCV/CMakeLists.txt`にtablegen呼び出しを追加して、.tdの内容に基づいてバックエンドのC++コードを生成することができます。

```
set(LLVM_TARGET_DEFINITIONS RISCV.td)

tablegen(LLVM RISCVGenRegisterInfo.inc -gen-register-info)
tablegen(LLVM RISCVGenInstrInfo.inc -gen-instr-info)

add_public_tablegen_target(RISCVCommonTableGen)
```

### RISC-V MCTargetDescの追加
次のステップは，RISCVMCTargetDescサブディレクトリに最小限のクラス群を追加することです。これにより、RISC-V ELFを出力するのに十分な機能が提供されます。

ボトムアップではなくトップダウンで始めた方が簡単なので，`RISCVMCTargetDesc.cpp`から始めてください。これは主に必要なMC層のクラスを登録するためのものです。

```cpp
extern "C" void LLVMInitializeRISCVTargetMC() {
  for (Target *T : {&getTheRISCV32Target(), &getTheRISCV64Target()}) {
    TargetRegistry::RegisterMCAsmInfo(*T, createRISCVMCAsmInfo);
    TargetRegistry::RegisterMCInstrInfo(*T, createRISCVMCInstrInfo);
    TargetRegistry::RegisterMCRegInfo(*T, createRISCVMCRegisterInfo);
    TargetRegistry::RegisterMCAsmBackend(*T, createRISCVAsmBackend);
    TargetRegistry::RegisterMCCodeEmitter(*T, createRISCVMCCodeEmitter);
  }
}
```

この登録プロセスを理解するために、LLVM `TargetRegistry`がどのように動作するかを見る価値があります。典型的なLLVMフロントエンドやツール（llvm-mcなど）は、Targetのインスタンスを取得し、与えられた型のインスタンスを取得するために`createMCRegInfo`などのメソッドを呼び出します。`LLVMInitializeRISCVTargetMC`は，後で取得できるように，これらのインスタンスを登録する役割を果たします。実装すべき最小限のMCクラスを見つける最も簡単な方法は，それらのクラスを1つずつ調べて，`./bin/llvm-mc -arch=riscv64 -filetype=obj foo.s`を実行したときに何が失敗するかを確認することです。

- `RISCVMCAsmInfo`
  - MCAsmInfoELFを継承しています。コメント文字列のようないくつかの重要な詳細を指定する必要があります。
  - 'anchor'メソッドは [LLVM のコーディング標準](http://llvm.org/docs/CodingStandards.html#provide-a-virtual-method-anchor-for-classes-in-headers)に従って提供されています。
    この[StackOverflowの回答](https://stackoverflow.com/questions/16801222/out-of-line-virtual-method)で説明したように、このようなダミーの仮想メソッドを作成することで、vtableが1つの.oファイルだけで出力されるようになります。
- `RISCVMCInstrInfo`
    - tablegenによって生成される`InitRISCVMCInstrInfo`を呼び出す必要があるだけです。
- RISCVAsmBackend
  - `RISCVAsmBackend::writeNopData`を除いて、すべてのメソッドはスタブアウトされています。RISC-V の標準的な NOP は `addi x0, x0, 0` (0x13) です。
- RISCVELFObjectWriter
    - `MCELFObjectTargetWriter`からの派生で、メソッドはすべてスタブアウトされています。
- RISCVMCCodeEmitter
     - RISC-V命令(MCInst)をエンコードされた命令に変換します。ほとんどの作業はTableGenによって生成される`getBinaryCodeForInstr`によって行われます。

最後に、`lib/Target/RISCV/CMakeLists.txt`に`MCTargetDesc`を追加し、`lib/Target/RISCV/MCTargetDesc/CMakeLists.txt`を作成します。

この時点で、`./bin/llvm-mc -arch=riscv64 -filetype=obj foo.s` を実行してみてください。進歩しました。

### RISCVAsmParserの実装

次は，`lib/Target/RISCV/AsmParser/RISCVAsmParser.cpp`の実装です。`MatchRegisterName`, `MatchRegisterAltName`, `MatchInstructionImpl`など、多くの関数がRISCVInstrInfo.tdから自動生成されます。

RISCVAsmParserには3つの主要なコンポーネントがあります。

- 命令オペランド（トークン、レジスタ、または即時）を表す`RISCVOperand`。
- トップレベルの `MatchAndEmitInstruction`です。これは主に`MatchEmitInstructionImpl`を呼び出します。しかし、適切な診断を発しながら、様々な障害状態を処理するコードを書く必要があります。
- 命令やオペランドを解析するためのメソッド。

これらのメソッドの実装の中には、混乱を招く可能性があります。bool を返すものもあれば、MatchResult を返すものもあります。通常、戻り値が false の場合は成功を示します。

この時点で、`llvm-mc`を使って簡単なファイルを組み立てることができるはずです。`stats` と `-as-lex` オプションは、何が起こっているのかを追跡するのに便利です。

## Instruction Printingとテストの実装

必要なインフラストラクチャの大部分が整ったので、アセンブラのテストに必要な最後の部品を追加することができます。

### 命令プリンタの追加

RISCVInstPrinter は，`MCInst` を文字列に変換するために必要です。このクラスを実装すると，`llvm-m`c の `-show-encoding` オプションを利用して，解析された命令の印刷方法が期待通りであるか，エンコーディングが正しいかをチェックすることができます。

MC レイヤーの多くと同様に，「作業」の大部分は TableGen を用いて生成されたコードによって行われます。

```cpp
void RISCVInstPrinter::printInst(const MCInst *MI, raw_ostream &O,
                                 StringRef Annot, const MCSubtargetInfo &ST
  printInstruction(MI, O);
  printAnnotation(O, Annot);
}

void RISCVInstPrinter::printRegName(raw_ostream &O, unsigned RegNo) const {
  O << getRegisterName(RegNo);
}

void RISCVInstPrinter::printOperand(const MCInst *MI, unsigned OpNo,
                                    raw_ostream &O, const char *Modifier) {
  assert((Modifier == 0 || Modifier[0] == 0) && "No modifiers supported");
  const MCOperand &MO = MI->getOperand(OpNo);

  if (MO.isReg()) {
    printRegName(O, MO.getReg());
    return;
  }

  if (MO.isImm()) {
    O << MO.getImm();
    return;
  }

  assert(MO.isExpr() && "Unknown operand kind in printOperand");
  MO.getExpr()->print(O, &MAI);
}
```

### テストの実装

`test/MC/RISCV` ディレクトリを作成した後、以下の内容を含む `lit.local.cfg` ファイルを作成する必要があります。

```
if not 'RISCV' in config.root.targets:
    config.unsupported = True
```

これにより，LLVMがRISCVをサポートしてビルドされている場合にのみ，テストが実行されるようになります。

LLVMのテストは、通常、[FileCheck](http://llvm.org/docs/CommandGuide/FileCheck.html)ユーティリティを利用して、ツール出力のパターンをチェックします。テストの構成方法には柔軟性がありますが、有効なRISC-V命令が正しく組み立てられているかどうかをチェックするために`rv32i-valid.s`を作成し、無効な命令の診断をチェックするために`rv32i-invalid.s`を作成することをお勧めします。

例えば、`rv32i-valid.s`には以下のようなコードが含まれているかもしれません。

```
# RUN: llvm-mc %s -triple=riscv32 -show-encoding \
# RUN:     | FileCheck -check-prefixes=CHECK,CHECK-INST %s
# RUN: llvm-mc %s -triple=riscv64 -show-encoding \
# RUN:     | FileCheck -check-prefixes=CHECK,CHECK-INST %s

# CHECK-INST: addi ra, sp, 2
# CHECK: encoding: [0x93,0x00,0x21,0x00]
addi ra, sp, 2
# CHECK-INST: slti a0, a2, -20
# CHECK: encoding: [0x13,0x25,0xc6,0xfe]
slti a0, a2, -20
```

一方で`rv32i-invalid.s`は以下のようなコードが含まれているかもしれません。

```
# RUN: not llvm-mc -triple riscv32 < %s 2>&1 | FileCheck %s

# Out of range immediates
ori a0, a1, -2049 # CHECK: :[[@LINE]]:13: error: immediate must be an integer in the range [-2048, 2047]
andi ra, sp, 2048 # CHECK: :[[@LINE]]:14: error: immediate must be an integer in the range [-2048, 2047]
```

この時点で、すべてのRV32I命令が`RISCVInstrInfo.td`で定義されており、正しく組み立てることができることを確認することができます。

## ディスアセンブリのサポート

RISCVDisassemblerは、適切な`MCOperand`インスタンスにイミディエイトとレジスタをデコードするメソッドを提供します。これは`getInstruction`によって駆動され、TableGenが生成する`decodeInstruction`に依存します。

例えば、以下のようになります。

```cpp
template <unsigned N>
static DecodeStatus decodeSImmOperand(MCInst &Inst, uint64_t Imm,
                                      int64_t Address, const void *Decoder)
  assert(isUInt<N>(Imm) && "Invalid immediate");
  // Sign-extend the number in the bottom N bits of Imm
  Inst.addOperand(MCOperand::createImm(SignExtend64<N>(Imm)));
  return MCDisassembler::Success;
}
```

これらの`decode*`メソッドは `RISCVInstrInfo.td` で指定されています。

つまり、`llvm-mc -show-encoding` と `llvm-objdump -d` を使用してオブジェクトファイルを分解する際に、`CHECK-INST` 行が正しいかどうかをチェックします。

このディスアセンブリは`rv32i-valid.s`を拡張してすべてのテストをサポートすることができます。つまり、`llvm-mc -show-encoding` と `llvm-objdump -d` を使ってオブジェクトファイルを分解する際に `CHECK-INST` 行が正しいかどうかをチェックすることです。

## リロケーション・Fixup対応の実施

リロケーションは、未解決のシンボル参照（例えば、別のコンパイルユニット内の関数への参照）がある場合に発生します。LLVMでは、未知の情報が参照された場合(例えば、未解決のシンボル参照)に「Fixup」が生成されます。Fixupの中には、ELFが生成される前に解決できるものもあれば、リロケーションに変換しなければならないものもあります。すべてのリロケーションはかつてはFixupであったが、すべてのFixupがリロケーションになるわけではない。

これの実装が完了すると、MC層が実装され、コード生成に移る準備が整います。

### Fixupの導入

まず，`RISCVFixupKinds.h` を導入し，`RISCVAsmBackend::getFixupKindInfo` を実装します。これは，サポートされているFixupと，各Fixupに関するメタデータ（命令内のオフセット，長さ，プログラムカウンタに対する相対値の有無）を定義します。次に、`RISCVAsmBackend::applyFixup` を実装します。`applyFixup`が呼ばれた時点でFixup値がわかっている場合、このメソッドはエンコードされた命令をFixupする役割を担っています。必要に応じて値をFixupするヘルパー関数、例えば `adjustFixupValue` を導入すると便利です。これは、ビットレイアウトがターゲットのRISC-V命令に必要なものと一致するように値を操作するために必要です。

このようなロジックができたので、次のステップは、1) アセンブリパーサからのFixup生成のサポートを実装し、2) Fixupが解決されていない場合のリロケーションのサポートを実装することです。

### アセンブラの変更点

アセンブリパーサからFixupを生成するには，`RISCVMCExpr` の実装が必要です。アセンブリパーサが `%pcrel_hi()` のような修飾子を持つオペランドを記録できるようにするために，このインスタンスが作成されます。`RISCVMCExpr`は，もう一つの概念，`VariantKinds`を導入しています。これは、オペランド修飾子がFixupされない場合、例えば`%lo()`が定数値に適用される場合などに必要となります。

RISCVAsmParserはシンボルが解析できるようにFixupされなければなりません。さらに、オペランド修飾子を正常に解析しなければなりません。通常のLLVMアセンブリパーサと同様に、正しい形式の修飾子を認識し、その後、オペランド修飾子がRISC-Vで有効かどうかを検証します。

VariantKindからfixupに変換する場合、RISC-V命令フォーマットの中には12ビットのimmediateのエンコーディングが2つあることを考えると、`RISCVMCExpr::VK_RISCV_LO`をどのように変換するかという問題があります（I-format命令とS-format命令ではimmediateビットの配置が異なります）。これは、I型オペコードのリストとS型オペコードのリストをハードコーディングすることで対応できますが、このようなハードコーディングは避けた方が良いでしょう。命令フォーマットを各命令のプロパティとして追加することで、現在の命令のフォーマットを確認して適切なFixupを選択することができます。

### リロケーション

最後に、`RISCVELFObjectWriter::getRelocType`を変更して、Fixupタイプを適切なリロケーションタイプにマッピングすることができます。

## codegen サポートへの最初のステップ

次の実装課題は、基本的なコード生成のサポートを追加することです。これを行うために必要なボイラプレートコードはそれなりの量がありますが、これはほとんどの困難さが必要なサポートコードの最小セットを特定することから来ていることを意味します。本当に最低限の出発点は、引数なしで void を返す関数の codegen を実装することでしょう。しかし、単純な算術演算のためのcodegenをサポートすることの難易度は非常に低く、余分な作業を行うことで実装が正しいことをはるかに保証してくれます。したがって、RV32I ALU演算のサポートを実装することから始めてください。

### 新たに追加されたサポートコードの概要

- `RISCVAsmPrinter`と`RISCVMCInstLower`
  
  - これは，単にオペコードを設定し，入力`MachineInstr`のレジスタまたは即時オペランドに対応する`MCOperands`を追加するだけの`llvm::LowerRISCVMachineInstrToMCInst`によって処理されます。
- `RISCVInstrInfo`
  
    - この段階では，手書きのコードは必要ありません。RISCVInstrInfo.{h,cpp}には，RISCVInstrInfo.tdから生成されるRISCVGenInstrInfo.incが含まれています。これは，`RISCV::ADDI`などの定義されたすべてのRISCV命令を含むenumを定義します。
- `RISCVRegisterInfo`

    - `RISCVInstrInfo` と同様に，テーブル生成コードに大きく依存します。
    - 戻り値のレジスタを `RISCVGenRegisterInfo` コンストラクタに渡す必要があります。
    - `getReservedRegs` は，予約されているとみなされ，レジスタアロケータによって無視されるべきすべてのレジスタを示さなければなりません。
    - `getFrameRegister` は，単にフレームインデックスに使用されるレジスタを返します。まだフレームポインタの消去が実装されていないので，常に RISCV::X8 を返してください。
    - `eliminateFrameIndex` は存在する必要がありますが，今のところは `report_fatal_error` を呼び出すだけのスタブです。
- `RISCVSubtarget`
    - 実装したクラスのインスタンスに対する多数のアクセサが含まれています。
- `RISCVPassConfig`
- `RISCVISelDAGToDAG`
    - これは、各SelectionDAGノードに対して適切なRISC-V命令を選択する処理を行います。`RISCVDAGToDAGISel::Select`はこの処理のメインエントリーポイントであり、テーブル生成された`SelectCode`関数を呼び出すだけです。この処理を制御するパターンの指定については，以下を参照してください。
- `RISCVISelLowering`
    - 最終的にLLVMコードをSelectionDAGに落とします。この変換を実行するコードの大部分はターゲットに依存しないため、このプロセスに影響を与えるためにいくつかのフックを実装する必要があります。詳細は以下を参照してください。
### RISCVInstrInfo.tdでのパターンの指定

RISC-V バックエンドは RISCVInstrInfo.td を構造化しているので，命令選択パターンは命令定義とは別に指定されます。add のパターンは簡単に指定できます。

```
def : Pat<(add GPR:$rs1, GPR:$rs2), (ADD GPR:$rs1, GPR:$rs2)>;
```

Patへの最初のパラメータは、マッチするSelectionDAGノードを指定します。そのようなノードの完全なリストは`include/llvm/Target/TargetSelectionDAG.td`を参照してください。2番目のパラメータは、それをLoweringするRISC-V命令を指定します。

add (`addi`) のregiste-immediateフォーマットは以下のようにマッチします。

```
def : Pat<(add GPR:$rs1, simm12:$imm12), (ADDI GPR:$rs1, simm12:$imm12)>;
```

ADDIをLoweringするために、同じSelectionDAGノード(`add`)を、異なる制約（即時オペランド）でマッチングしていることに注意してください。LLVM SelectionDAGインフラストラクチャは、各パターンの前提条件のチェックを処理します。上記のパターンが動作するためには、`simm12`もまた、SelectionDAGが理解できる方法で制約を与えなければなりません。以前のように、`Operand`に加えて`ImmLeaf`から派生させます。

```
def simm12 : Operand<XLenVT>, ImmLeaf<XLenVT, [{return isInt<12>(Imm);}]> {
...
}
```

### RISCVISelLowering

コンストラクタは，Loweringに影響を与えるいくつかのヘルパー関数を呼び出します。重要なのは，`addRegisterClass(XLenVT, &RISCV::GPRRegClass);` によって，`XLenVT`（RV32の場合は`i32`）が正当な型とみなされるようになります。将来のパッチでは，例えば，どの操作を直接下げるのではなく，意味的に等価なものに展開すべきかを指定するなど，より多くのコードが追加される予定です。

現段階で実装すべき2つの主要なメソッドは、`LowerFormalArguments`と`LowerReturn`です。前者は、各引数がどのレジスタに渡されるかを判断し、適切な仮想レジスタを作成します。`LowerReturn`は戻り値を、ターゲットの呼び出し規約に適したレジスタにコピーします。

### Calling Conventionの指定

この段階では、[RISC-Vの呼び出し規約](https://github.com/riscv/riscv-elf-psabi-doc/blob/master/riscv-elf.md)の完全な詳細をサポートしようとはしません。代わりに，RISCVCallingConv.td で基本的なレジスタの割り当てを指定します：

```
// RISCV 32-bit C return-value convention.
def RetCC_RISCV32 : CallingConv<[CCIfType<[i32], CCAssignToReg<[X10, X11]>>]>;

// RISCV 32-bit C Calling convention.
def CC_RISCV32 : CallingConv<[
	// Promote i8/i16 args to i32
	CCIfType<[ i8, i16 ], CCPromoteToType<i32>>,

	// All arguments get passed in integer registers if there is space.
	CCIfType<[i32], CCAssignToReg<[ X10, X11, X12, X13, X14, X15, X16, X17]>>,

	// Could be assigned to the stack in 8-byte aligned units, but unsupported
	CCAssignToStack<8, 8>
]>;
```

これにより，RISCVISelLowering.cpp で利用できる関数 `CC_RISCV32` と `RetCC_RISCV32` が生成されます。

また，コール保存されているレジスタもリストアップしておきます。

```
def CSR : CalleeSavedRegs<(add X1, X3, X4, X8, X9, (sequence "X%u", 18, 27))>;
```

### テストの指定

MC層のテストは `test/MC/RISCV` に書き、`FileCheck` を利用しています。 テストは `test/CodeGen/RISCV` に配置することで，再び `FileCheck` を用いて codegen 用のテストを書くことができます。 これらのテストはLLVM IRで書かれた単純な関数の形をしており、`llc`でコンパイルした後に生成されたアセンブリのチェックを含んでいます。 チェック行の生成を自動化するには、`update_llc_test_checks.py`を使うことを強くお勧めします。

## メモリ操作のためのCodegenサポート

### 前提条件: 定数の具体化

このドキュメントのセットでは、各実装タスクが必要とされた順にリストアップされています。LLVMバックエンドを自分で書く場合、次のステップが何であるかは必ずしも明らかではありません。多くの場合、新しい機能を実装しようとし始め、その時点で前提条件を発見することになります。この場合、メモリ演算の実装を開始すると、定数を具現化する機能（レジスタに定数をロードする）を必要とするテストケースがすぐに見つかるでしょう。

良いニュースは，定数を実体化する機能を実装するのは簡単で，`RISCVInstrInfo.td`を修正するだけで実現できるということです。まず，符号付き12ビットのイミディエイトにマッチするパターンを指定します。

```
def : Pat<(simm12:$imm), (ADDI X0, simm12:$imm)>;
```

これは簡単です - 12 ビットのイミディエイトは `ADDI` への有効なオペランドです。より大きなイミディエイトの場合は、より多くの作業が必要です。addiとluiの組み合わせを生成したいのですが、addiは低い12ビットを取り、luiは20ビットを取ります。`ADDI`オペランドが符号拡張されるという事実を補償する必要があるので、実際にはそれよりも少し複雑です。パターンは次のようになります。

```
def : Pat<(simm32:$imm), (ADDI (LUI (HI20 imm:$imm)), (LO12Sext imm:$imm))>;
```

ここで、HI20とLO12Sextは、正しいオペランドを生成するための即値の変換です。

	// 即値から最下位の12ビットを抽出し、符号を拡張し、使用しています。
		def LO12Sext : SDNodeXForm<imm, [{
		  return CurDAG->getTargetConstant(SignExtend64<12>(N->getZExtValue()),
		                                   SDLoc(N), N->getValueType(0));
		}]>;
	
	// 即値から最上位の20ビットを抽出します。ビット 11 が 1 の場合は 1 を加算し、一致する即値の下位 12 ビットが負の値であることを補正します。
		def HI20 : SDNodeXForm<imm, [{
		  return CurDAG->getTargetConstant(((N->getZExtValue()+0x800) >> 12) & 0xfffff,
		                                   SDLoc(N), N->getValueType(0));
		}]>;
### メモリ操作のためのCodegenサポート
1)オフセットのないアドレスのロード/ストアと、2)オフセットのあるアドレスのロード/ストアの2つのパターンを一致させるために、ロードとストアには2つのパターンが必要です。後者は、ロード/ストアへのオペランドとしてaddノードを持つ`SelectionDAG`サブツリーで表現されます。以下のmulticlassはこれを示しています。

```
multiclass LdPat<PatFrag LoadOp, RVInst Inst> {
  def : Pat<(LoadOp GPR:$rs1), (Inst GPR:$rs1, 0)>;
  def : Pat<(LoadOp (add GPR:$rs1, simm12:$imm12)),
            (Inst GPR:$rs1, simm12:$imm12)>;
}
```

RISC-V の特徴は、ワードサイズよりも小さいデータ値のシグネクストおよびゼロエクステンディングロード（「符号なし」）です。シグネクスト、ゼロエクステンド、anyext（気にしない）ロードのパターンを指定しなければなりません。以下のようなテストは、これら3つのケースをカバーしています。

```
; RV32I-LABEL: lh:
; RV32I:       # %bb.0:
; RV32I-NEXT:    lh a1, 0(a0)
; RV32I-NEXT:    lh a0, 4(a0)
; RV32I-NEXT:    jalr zero, ra, 0
  %1 = getelementptr i16, i16* %a, i32 2
  %2 = load i16, i16* %1
  %3 = sext i16 %2 to i32
  ; the unused load will produce an anyext for selection
  %4 = load volatile i16, i16* %a
  ret i32 %3
}
```

## ブランチと関数呼び出しのためのCodegen
### 分岐のLowering
`include/llvm/CodeGen/ISDOpcodes.h`を見直すと、条件分岐の2つの表現があることがわかります。`BR_CC`は、条件コード、比較するノード、そして条件が真であれば分岐するブロックを取るcompare+branchです。`BR_CC`はRISC-Vのcompare+branch命令に近いですが、`BRCOND`形式の方がテーブルゲンパターンを書きやすくマッチします。そのため、BR_CCの拡張を要求します。

```cpp
setOperationAction(ISD::BR_CC, XLenVT, Expand);
```

以下のパターンを定義します。

```
// Match `(brcond (CondOp ..), ..)` and lower to the appropriate RISC-V branch
// instruction.
class BccPat<PatFrag CondOp, RVInstB Inst>
    : Pat<(brcond (i32 (CondOp GPR:$rs1, GPR:$rs2)), bb:$imm12),
          (Inst GPR:$rs1, GPR:$rs2, simm13_lsb0:$imm12)>;

def : BccPat<seteq, BEQ>;
```

多くの`setcc`条件コードは、直接一致するRISC-V命令を持っていませんが、入力オペランドをスワップすることでサポートすることができます。

```
class BccSwapPat<PatFrag CondOp, RVInst InstBcc>
    : Pat<(brcond (i32 (CondOp GPR:$rs1, GPR:$rs2)), bb:$imm12),
          (InstBcc GPR:$rs2, GPR:$rs1, bb:$imm12)>;

// マッチするRISC-V分岐命令を持たない条件コードは、2つの入力オペランドを入れ替えることで
// 簡単にサポートされます。
def : BccSwapPat<setgt, BLT>;
def : BccSwapPat<setle, BGE>;
def : BccSwapPat<setugt, BLTU>;
def : BccSwapPat<setule, BGEU>;
```

分岐テスト関数が動作するためには，生成されたレジスタのこぼれが必要な` RISCVInstrInfo::storeRegToStackSlot` と `RISCVInstrInfo::loadRegFromStackSlot` が実装されている必要があります。これらの関数は，単に適切なロードまたはストアを生成する必要があります。

### 関数コールのCodegen

`JALR` に展開する call の疑似命令を定義します。

```
let isCall = 1, Defs=[X1] in
def PseudoCALL : Pseudo<(outs), (ins GPR:$rs1), [(Call GPR:$rs1)]>,
                 PseudoInstExpansion<(JALR X1, GPR:$rs1, 0)>;
```

実装する必要のある主なコンポーネントが`RISCVTargetLowering::LowerCall` です。これは，適切なSelectionDAG ノードである `call_start`, `CALL`, `callseq_end` への呼び出しをLoweringします。これは，引数のコピーと結果のコピーを行います。

## フレーム処理のためのCodeGen
RISC-Vバックエンドは、関数呼び出しをLoweringすることができるようになりましたが、関数prolog/epilogを生成したり、FrameIndex参照を正しく下げたりすることはまだできません。

### フレームインデックスのLowering

まず、ダミーの「アドレッシングモード」を定義します。これはFrameIndexにマッチするもので、パターンではFrameIndexを直接マッチさせることができないので必要です。

```
def AddrFI : ComplexPattern<iPTR, 1, "SelectAddrFI", [frameindex], []>;

def : Pat<(LoadOp (add AddrFI:$rs1, simm12:$imm12)),
          (Inst AddrFI:$rs1, simm12:$imm12)>;
def : Pat<(LoadOp (IsOrAdd AddrFI:$rs1, simm12:$imm12)),
          (Inst AddrFI:$rs1, simm12:$imm12)>;
```

`ISD::FrameIndex`を適切なオペランドを持つ`ADDI`にLoweringするように`RISCVDAGToDAGISel::Select`を変更します。

```cpp
if (Opcode == ISD::FrameIndex) {
  SDLoc DL(Node);
  SDValue Imm = CurDAG->getTargetConstant(0, DL, XLenVT);
  int FI = dyn_cast<FrameIndexSDNode>(Node)->getIndex();
  EVT VT = Node->getValueType(0);
  SDValue TFI = CurDAG->getTargetFrameIndex(FI, VT);
  ReplaceNode(Node, CurDAG->getMachineNode(RISCV::ADDI, DL, VT, TFI, Imm));
  return;
}
```

最後に，`RISCVFrameLowering::getFrameIndexReference` を実装する必要があります．これは正しいフレームレジスタとオフセットを選択しなければなりません。Callee-saveレジスタはスタックポインタから相対的に参照され、それ以外の場合はフレームポインタが使用されます。

### PrologとEpilogの挿入

`RISCVFrameLowering::emitPrologue` と `RISCVFrameLowering::emitEpilogue` は，以前はスタブアウトされていましたが，スタック上のスペースを確保・解放し，フレームポインタを適切に操作しなければなりません．

## RISC-Vの呼び出し規約をサポート
tablegen指定の呼び出し規則は，単純なケースでは動作しますが，[RISC-Vの呼び出し規則](https://github.com/riscv/riscv-elf-psabi-doc/blob/master/riscv-elf.md)を記述する規則の完全なセットを表すものではありません．

`CC_RISCV`は，RISC-Vの呼び出し規則を実装しています．`CC_RISCV`は、各引数が正規化された後、断片的に渡されます。例えば，`i64`は`i32`のペアに分割されます．したがって，`CC_RISCV` は，そのような分割値を追跡して，元の引数のサイズがレジスタに渡すべきか，スタックに渡すべきかを判断するロジックを含んでいます．

フロントエンドのABI低下に対する期待値は、ターゲットによって異なります。理想的には、LLVMフロントエンドが多くのABIの詳細を気にせずに済むようにすることができますが、これは長期的な目標です。今のところは、フロントエンドの役割をできる限りシンプルかつ明確に定義しておくようにしています。ルールは以下のようにまとめることができます。

- 大きなスカラー引数は決して分割しないこと。ここではそれらを扱います。
- ハードフロートの呼び出し規約を使用していて、構造体がレジスタのペア(fp+fp, int+fp)で渡され、両方のレジスタが利用可能な場合は、2つの別々の引数として渡します。GPR または FPR のいずれかが枯渇している場合は、以下のルールに従って渡してください。
- 構造体がレジスタやスタックスロットに直接渡すことができなかった場合（`2*XLEN`よりも大きく、浮動小数点ルールが適用されないため）、byval属性を持つポインタを使用して渡します。
- 構造体が `2*XLEN` より小さい場合は、2 要素のワードサイズの配列か `2*XLEN` スカラ（アライメントに応じて）のどちらかに強制的に渡します。
- フロントエンドは、構造体が参照によって返されるかどうかを、そのサイズとフィールドに基づいて判断することができます。構造体が参照で返される場合、フロントエンドはプロトタイプを修正して、第一引数に sret アノテーションを持つポインタを渡すようにしなければなりません。これは大きなスカラ値を返す場合には必要ありません。
- 構造体の戻り値と varargs は、固定引数の場合と同じ状況で、レジスタサイズのフィールドを含む構造体に強制されるべきです。



