# 9. TileLinkとDiplomacyリファレンス

TileLinkはRocketChipおよび他のChipyardジェネレータで使用されているキャッシュコヒーレンスとメモリプロトコルである。これらは、キャッシュ、メモリ、周辺機器、DMAデバイスなどの異なるモジュールとどのように互いに通信を行うかを規定している。

RocketChipのTilelinkの実装はDiplomacyとよばれる、Chiselジェネレータ内で構成情報を交換して行われる2段階のエラボレーション方式を使用している。Diplomacyの詳細に関しては、[Cook, Terpstra, Leeの論文](https://carrv.github.io/2017/papers/cook-diplomacy-carrv2017.pdf)を参考にすること。

シンプルなTileLinkのウィジェットを追加する方法の概要については、[アクセラレータデバイスを追加するセクション](https://chipyard.readthedocs.io/en/latest/Customization/Adding-An-Accelerator.html#adding-an-accelerator)を参考にすること。このセクションでは、RocketChipにより提供されているTilelinkおよびDiplomacyの機能について説明する。

TileLink 1.7プロトコルの詳細な仕様については[SiFiveのウェブサイト](https://sifive.cdn.prismic.io/sifive%2F57f93ecf-2c42-46f7-9818-bcdd7d39400a_tilelink-spec-1.7.1.pdf)を参考にすること。

# 9.1. TileLinkのノードタイプ

DiplomacyはSoC内の異なるコンポーネントをDirected Acyclic Graphのノードとして表現する。TileLinkのノードは異なるタイプを持つことができる。

## 9.1.1. Clientノード


TileLinkのClientはTileLinkのトランザクションを、まずはAチャネルを通じてリクエストを発行し、Dチャネルを通じてレスポンスを受け取る。Clientの種類がTL-Cであれば、ProbeリクエストをBチャネルを通じて受け取り、ReleaseコマンドをCチャネルを通じて発行し、Eチャネルを通じてGrantメッセージを発行する。

RocketChip/ChipyardのL1キャッシュ及びDMAデバイスはClientノードである。

TileLinkClientを追加するためには、testchippipから、LazyModuleにTLHelperオブジェクトを以下のように追加することで実現できる:

```scala
class MyClient(implicit p: Parameters) extends LazyModule {
  val node = TLHelper.makeClientNode(TLClientParameters(
    name = "my-client",
    sourceId = IdRange(0, 4),
    requestFifo = true,
    visibility = Seq(AddressSet(0x10000, 0xffff))))

  lazy val module = new LazyModuleImp(this) {
    val (tl, edge) = node.out(0)

    // 残りのコードをここに追加する。
  }
}
```

`name`引数はDiplomacyグラフの中でのノードの名前をIdentityする。この引数はTLClientParametersの中で唯一必須の引数である。

`SourceI`引数はこのClientが使用するソースIDの範囲を指定する。ここでは[0, 4)までのIDを使用しているので、このClientは同時に4つまでのリクエストをインフライトに発行することができる。各リクエストはsourceフィールドにおいてそれぞれ異なる値を設定される。このフィールドのデフォルト値は`IdRange(0, 1)`であり、これは単一のリクエストのみがインフライトすることができることを意味する。

`requestFifo`引数はBooleanのオプションでありデフォルトではfalseである。この引数をtrueに設定すると、ClientノードはダウンストリームのManagerに対してレスポンスをFIFOの順序で送ることを要求する(つまり、当該リクエストが送信されるのと同じ順番にリクエストが返ってくることを期待する)

`visibility` 引数はClientがアクセスするアドレス領域を指定する。デフォルトの値はすべてのアドレスを含んでいる。例えば、1つのアドレス領域`AddressSet(0x10000, 0xffff)`を指定した場合、Clientは0x10000から0x1fffffまでの領域のみアクセスできる。通常はこの設定は使用しないが、ダウンストリームのクロスバージェネレータがハードウェアを最適化するために使用することができる。つまり、ClientとManagerのアービトレーションにおいて、visibilityが重ならないアドレス領域におけるクロスバーの最適化を実施することができる。

Lazyモジュールの実装において、`node.out`という表記はbundle/edgeのペアのリストを取得するために使用する。TLHelperを使用しているならば、単一のClientエッジのみが指定され、単一のペアのみが返される。

`tl`バンドルはChiselのハードウェアバンドルであり、このモジュールのIOポートを接続する。このバンドルにはTL-ULおよびTL-UHの場合2種類のTileLinkチャネルのDecoupleバンドルが含まれており、TL-Cの場合は5種類である。ここまでで、TileLinkにメッセージを送受信するために必要なハードウェアを接続するための準備が整った。

`edge`オブジェクトには、Diplomacyグラフのエッジが示されている。エッジには[TileLinkのエッジオブジェクトメソッド](https://chipyard.readthedocs.io/en/latest/TileLink-Diplomacy-Reference/EdgeFunctions.html#tilelink-edge-object-methods)に示された様々な便利な関数が含まれている。

## 9.1.2. Managerノード

TileLinkのManagerはClientのAチャネルからリクエストを受け取り、Dチャネルにレスポンスを返す。ManagerノードはTLHelperを使用して以下のようにして作成できる。


```scala
class MyManager(implicit p: Parameters) extends LazyModule {
  val device = new SimpleDevice("my-device", Seq("tutorial,my-device0"))
  val beatBytes = 8
  val node = TLHelper.makeManagerNode(beatBytes, TLManagerParameters(
    address = Seq(AddressSet(0x20000, 0xfff)),
    resources = device.reg,
    regionType = RegionType.UNCACHED,
    executable = true,
    supportsArithmetic = TransferSizes(1, beatBytes),
    supportsLogical = TransferSizes(1, beatBytes),
    supportsGet = TransferSizes(1, beatBytes),
    supportsPutFull = TransferSizes(1, beatBytes),
    supportsPutPartial = TransferSizes(1, beatBytes),
    supportsHint = TransferSizes(1, beatBytes),
    fifoId = Some(0)))

  lazy val module = new LazyModuleImp(this) {
    val (tl, edge) = node.in(0)
  }
}
```

The `makeManagerNode` method takes two arguments. The first is `beatBytes`, which is the physical width of the TileLink interface in bytes. The second is a TLManagerParameters object.

`makeManagerNode` メソッドは2つの引数を取る。1つ目は`beatBytes`であり、これはTileLinkインタフェースの物理的な幅をバイト単位で返す。2番目の引数はTLManagerParametersオブジェクトである。

`TLManagerParameters`における必須の引数は`address`のみであり、これはManagerが管理するアドレスの領域を指定する。この情報はClientからのリクエストをルーティングするために使用される。この例では、Managerは0x20000から0x20fffまでの領域からのリクエストを受け取る。`AddressSet` の2番目の引数はマスクであり、サイズではない。したがってこのアドレス領域の設定は2の累乗の範囲内で行わなければならない。そうでなければ、アドレッシングの設定は設定の意図とは異なるように動作してしまう。

2番目の引数は`resources`であり、`Device`オブジェクトから値を設定する。この場合には`SimpleDevice`を指定している。この引数はエントリをBootROM上に書き込まれるデバイスツリーに追加したい場合には必須である。デバイスツリーの情報はLinuxのドライバにより読み込まれる。`SimpleDevice`に設定されている2つの引数は名前およびデバイスツリーのエントリの互換性に関する情報のリストである。このManagerでは、デバイスツリーの情報は以下のようになる:

```
L12: my-device@20000 {
    compatible = "tutorial,my-device0";
    reg = <0x20000 0x1000>;
};
```

次の引数は`regionType`である。この引数はManagerのキャッシュの動作について示している。この引数は、以下の7つの内どれかの値を設定できる:

1. `CACHED` - 中間エージェントはその領域におけるキャッシュされたコピーを保持する。
2. `TRACKED` - この領域は別のマスターによりキャッシュされるが、コヒーレンスの機能は提供される。
3. `UNCACHED` - この領域はキャッシュされない。しかし可能ならばキャッシュされるべきである。
4. `IDEMOTENT` - 最近Putされた内容を返すが、内容はキャッシュされるべきではない。
5. `VOLATILE` - 内容はPut操作無しで変更される可能性があるが、PutおよびGetでは副作用は発生しない。
6. `PUT_EFFECTS` - Putにより副作用が発生するため、コマンドを複合したり、遅延させるべきではない。
7. `GET_EFFECTS` - Getは副作用を発生させるため、投機的に発行させるべきではない。

次は`executable`引数であり、CPUがフェッチ命令をこのManagerから発行することができるかを示している。デフォルトではfalseであり、MMIOなどの周辺機器では設定すべきである。

次の6つの引数は`support`から始まり、異なるAチャネルのメッセージタイプについて、受付可能かを決めている。メッセージタイプの定義については、[TileLinkエッジオブジェクトのメソッド](https://chipyard.readthedocs.io/en/latest/TileLink-Diplomacy-Reference/EdgeFunctions.html#tilelink-edge-object-methods)を参照のこと。`TransferSizes` のケースクラスには、各メッセージタイプについてManagerが受け付けることのできる論理サイズがバイト単位で指定されている。これは内包的な領域であり、すべての論理サイズは2の累乗である必要がある。したがってこの場合には、Managerは1, 2, 4, 8バイトのリクエストを受け付けることができる。

最後の引数は`fifoId` 設定であり、FIFOのドメインにManagerが入っているかを示している。この引数をNoneに設定(デフォルトである)すると、Managerはレスポンスの順序を保証しない。`fifoId`を設定すると、同じ`fifoId` が設定された他のすべてのManagerと共有され、つまりそのFIFOドメイン内のClientからのリクエストはすべて同じ順序で帰ってくることを意味する。

## 9.1.3. Registerノード

Managerノードを直接指定してTileLinkリクエストを処理する論理をすべて書いたとしても、通常はRegisterノードを使用した方が簡単である。このタイプのノードは`regmap`メソッドを提供し制御/状態Registerを自動的に生成しTileLinkプロトコルを処理するための論理も生成する。Registerノードの使用方法について野依詳細については[Registerノード](https://chipyard.readthedocs.io/en/latest/TileLink-Diplomacy-Reference/Register-Router.html#register-router)を参照のこと。

## 9.1.4. Identityノード


これまでのノードは入力もしくは出力が定義されていたが、Identityノードは両方を持っている。名前が示しているように、入力と出力を単純に接続するだけである。このノードは複数のノードを複数のエッジをもつ単一のノードに接続するために使用される。例えば、2つのClientLazyモジュールがあったとして、これを1つのClientノードに接続している。

```scala
class MyClient1(implicit p: Parameters) extends LazyModule {
  val node = TLHelper.makeClientNode("my-client1", IdRange(0, 1))

  lazy val module = new LazyModuleImp(this) {
    // ...
  }
}

class MyClient2(implicit p: Parameters) extends LazyModule {
  val node = TLHelper.makeClientNode("my-client2", IdRange(0, 1))

  lazy val module = new LazyModuleImp(this) {
    // ...
  }
}
```

2つの異なるLazyモジュールをインスタンスして、この2つのノードを1つのノードに見せている。

```scala
class MyClientGroup(implicit p: Parameters) extends LazyModule {
  val client1 = LazyModule(new MyClient1)
  val client2 = LazyModule(new MyClient2)
  val node = TLIdentityNode()

  node := client1.node
  node := client2.node

  lazy val module = new LazyModuleImp(this) {
    // Nothing to do here
  }
}
```

Managerに対しても同様の処理が可能である。

```scala
class MyManager1(beatBytes: Int)(implicit p: Parameters) extends LazyModule {
  val node = TLHelper.makeManagerNode(beatBytes, TLManagerParameters(
    address = Seq(AddressSet(0x0, 0xfff))))

  lazy val module = new LazyModuleImp(this) {
    // ...
  }
}

class MyManager2(beatBytes: Int)(implicit p: Parameters) extends LazyModule {
  val node = TLHelper.makeManagerNode(beatBytes, TLManagerParameters(
    address = Seq(AddressSet(0x1000, 0xfff))))

  lazy val module = new LazyModuleImp(this) {
    // ...
  }
}

class MyManagerGroup(beatBytes: Int)(implicit p: Parameters) extends LazyModule {
  val man1 = LazyModule(new MyManager1(beatBytes))
  val man2 = LazyModule(new MyManager2(beatBytes))
  val node = TLIdentityNode()

  man1.node := node
  man2.node := node

  lazy val module = new LazyModuleImp(this) {
    // Nothing to do here
  }
}
```

ClientとManagerのグループを接続したい場合、以下のように接続できる。

```scala
class MyClientManagerComplex(implicit p: Parameters) extends LazyModule {
  val client = LazyModule(new MyClientGroup)
  val manager = LazyModule(new MyManagerGroup(8))

  manager.node :=* client.node

  lazy val module = new LazyModuleImp(this) {
    // Nothing to do here
  }
}
```

`:=*`演算子の意味については[Diplomacy コネクタ](https://chipyard.readthedocs.io/en/latest/TileLink-Diplomacy-Reference/Diplomacy-Connectors.html#diplomacy-connectors)の章を確認してほしい。つまり、複数のエッジを持つノードをたがいに接続したのである。Identityノードのエッジは順番に接続され、従ってこの場合は`client1.node`は`manager1.node`に接続され、`client2.node`は`manager2.node`に接続される。

Identityノードの入力ノードと出力ノードの数は一致していなければならない。そうでなければ、エラボレーション時にエラーが発生する。

## 9.1.5. Adapterノード


Identityノードと同様に、Adapterノードは複数の入力を受け付け、同じだけの出力を生成する。しかしIdentityノードと異なるのはAdapterノードは単純にノードを接続するわけではない。入力と出力の論理と物理的なインタフェースを変更し、メッセージを変更することができる。RocketChipはAdapterのライブラリを[Diplomaticウィジェット](https://chipyard.readthedocs.io/en/latest/TileLink-Diplomacy-Reference/Widgets.html#diplomatic-widgets)で提供している。

Adapterノードを自分で作成する必要性はあまりないが、必要であれば以下のようにして作成できる。

```scala
val node = TLAdapterNode(
  clientFn = { cp =>
    // ..
  },
  managerFn = { mp =>
    // ..
  })
```

`clientFn` は入力の`TLClientPortParameters` を受け付ける関数であり、出力のための該当するパラメータを生成する。`managerFn` は出力の`TLManagerPortParameters` を受け付ける関数であり、入力のための該当するパラメータを生成する。

## 9.1.6. Nexusノード


NexusノードはAdapterノードと似ているが出力インタフェースが異なる。しかし同様に出力ノードの数と入力ノードの数は異なっていても良い。このノードタイプはTileLinkのクロスバージェネレータを提供している`TLXbar`ウィジェットに使用される。このノードを手動で作成する必要性はあまりないが、必要に応じて以下のように生成できる。

```scala
val node = TLNexusNode(
  clientFn = { seq =>
    // ..
  },
  managerFn = { seq =>
    // ..
  })
```

Adapterノードのコンストラクタと似ているが、単一のパラメータオブジェクトを渡して単一の結果を返すのではなく、複数のパラメータを受け取る関数を取る。さらに、入力シーケンスの数と出力シーケンスの数は同一でなくても良い。

# Diplomacyコネクタ

Diplomacyのグラフは互いにエッジを使って接続されている。Diplomacyのライブラリは4つの演算子を使ってノードを接続している。

- `:=`

基本的な演算子である。Chiselの単方向の接続コネクタと同じ記法であるが、意味が異なる。この演算子はDiplomacyノードを接続するための演算子であり、バンドルを接続する演算子ではない。

基本的な接続演算子は、2つのノードを接続するための1つのエッジを生成する。

- `:=*`

「クエリ」タイプの接続演算子である。ノード間の複数のエッジにおいて、Clientノード(演算子の右側)によりエッジの数が決められている場合に、エッジの接続する。

- `:*=`

「スター」タイプの接続演算子である。複数のエッジを作成するが、エッジの数はClientではなくManager(演算子の左側)により決めらる。複数エッジのManagerノードに対してNexusノードを接続する場合に有益である。

- `:*=*`

「フレックス」タイプの接続演算子である。 演算子のどちらの側に既知の数のエッジがあるかに基づいて、複数のエッジを作成する。 これは、どちらかの側のノードのタイプが実行時までわからないジェネレーターで使用できる。

# Diplomaticウィジェット

RocketChipはDiplomaticなTileLinkとAXI4のウィジェットをライブラリとして提供している。共通部品として最も多く使用されるウィジェットを以下に示す。TileLinkのウィジェットは``freechips.rocketchip.tilelink``から入手できる。また、AXI4のウィジェットは``freechips.rocketchip.amba.axi4``から入手できる。

TLBuffer
--------

TileLinkのトランザクションをバッファリングするためのウィジェットである。TL-C以外の場合は2つのキューをインスタンス化し、TL-Cの場合は5つのキューをインスタンス化するだけである。各チャンネルのキューを柔軟に設定するために、``freechips.rocketchip.diplomacy.BufferParams``をコンストラクタに渡すことができる。case classの引数は以下のとおりである。

 - `depth: Int` - キューのエントリ数
 - ``flow: Boolean`` - Trueの場合、Valid信号は組み合わせ回路で構成されるためエンキューと同じサイクルで消費される。
 - ``pipe: Boolean`` - Trueの場合、Ready信号は組み合わせ回路で構成されるため、1エントリのキューは最大のレートで使用される。

コンストラクタには`Int`型の整数のみを渡すことができる。`BufferParams`オブジェクトの代わりに整数を渡すことによって、整数で渡されたdepthのキューが生成され、`flow`と`pipe`はfalseに設定される。

また、以下のあらかじめ定義された`BufferParams`オブジェクトを使用することができる。

 - ``BufferParams.default`` = ``BufferParams(2, false, false)``
 - ``BufferParams.none`` = ``BufferParams(0, false, false)``
 - ``BufferParams.flow`` = ``BufferParams(1, true, false)``
 - ``BufferParams.pipe`` = ``BufferParams(1, false, true)``

**引数:**

4種類のコンストラクタがある。引数が0, 1, 2, 5つ取るバリエーションである。

0個の引数を持つコストラクタはすべてのチャネルで``BufferParams.default``を使用する。

1つの引数を持つコンストラクタはすべてのチャネルで`BufferParams`オブジェクトを使用する。

2つの引数を持つコンストラクタは、それぞれ以下のように引数が適用される。

- `ace: BufferParams` - A, C, Eチャネルに使用されるパラメータである。
- `bd: BufferParams`  - B, Dチャネルに使用されるパラメータである。

5つの引数を持つコンストラクタはそれぞれ以下のように引数が適用される。

 - ``a: BufferParams`` - Aチャネルに使用されるパラメータである。
 - ``b: BufferParams`` - Bチャネルに使用されるパラメータである。
 - ``c: BufferParams`` - Cチャネルに使用されるパラメータである。
 - ``d: BufferParams`` - Dチャネルに使用されるパラメータである。
 - ``e: BufferParams`` - Eチャネルに使用されるパラメータである。

**使用例:**

```scala
// デフォルトの設定
manager0.node := TLBuffer() := client0.node

// 暗黙的にチャネル当たり8エントリのキューを挿入する。
manager1.node := TLBuffer(8) := client1.node

// Aチャネルではデフォルト設定を使用し、Dチャネルではパイプを使用する。
manager2.node := TLBuffer(BufferParams.default, BufferParams.pipe) := client2.node

// AチャネルとDチャネルにキューを挿入する。
manager3.node := TLBuffer(
  BufferParams.default,
  BufferParams.none,
  BufferParams.none,
  BufferParams.default,
  BufferParams.none) := client3.node
```

AXI4Buffer
----------

`TLBuffer`と同様だがAXI4用である。`BufferParams`オブジェクトを引数に取る。

**引数:**

TLBufferと同様に、AXI4Bufferにも4種類のコンストラクタがある。引数が0, 1, 2, 5つ取るバリエーションである。

0個の引数を持つコストラクタはすべてのチャネルで``BufferParams.default``を使用する。

1つの引数を持つコンストラクタはすべてのチャネルで`BufferParams`オブジェクトを使用する。

2つの引数を持つコンストラクタは、それぞれ以下のように引数が適用される。

- `aw: BufferParams` - AR, AW, Wチャネルに使用されるパラメータである。
- `br: BufferParams`  - B, Rチャネルに使用されるパラメータである。

5つの引数を持つコンストラクタはそれぞれ以下のように引数が適用される。

 - ``aw: BufferParams`` - AWチャネルに使用されるパラメータである。
 - ``w: BufferParams`` - Wチャネルに使用されるパラメータである。
 - ``b: BufferParams`` - Bチャネルに使用されるパラメータである。
 - ``ar: BufferParams`` - ARチャネルに使用されるパラメータである。
 - ``r: BufferParams`` - Rチャネルに使用されるパラメータである。

**使用例:**

```scala
// デフォルト設定
slave0.node := AXI4Buffer() := master0.node

// 暗黙的にチャネル当たり8エントリのキューを挿入する
slave1.node := AXI4Buffer(8) := master1.node

// AW/W/ARチャネルではデフォルト設定を使用し、B/Rチャネルではパイプを使用する。
slave2.node := AXI4Buffer(BufferParams.default, BufferParams.pipe) := master2.node

// AW/B/ARチャネルに1エントリのキューを挿入し、W/Rチャネルにキューを挿入する。
// Single-entry queues for aw, b, and ar but two-entry queues for w and r
slave3.node := AXI4Buffer(1, 2, 1, 1, 2) := master3.node
```

AXI4UserYanker
--------------

このウィジェットはユーザフィールドを持つAXIポートを受け取り、そのユーザフィールドを取り払う。ARとAWリクエストチャネルのユーザフィールドは内部のキューによりARID/AWIDに関連付けられて保持される。このユーザフィールドはレスポンス時に付け加えられる。

**引数:**

 - `capMaxFlight: Option[Int]` - (任意) インフライトなIDを保持することのできるリクエストの数。None(デフォルト)の場合、UserYankerはインフライトなリクエストの最大数までサポートする。

**使用例:**

```scala
nouser.node := AXI4UserYanker(Some(1)) := hasuser.node
```

AXI4Deinterleaver
-----------------

異なるIDを持つ複数ビートのAXIリードレスポンスはインタリーブされる可能性がある。このウィジェットはスレーブからのリードレスポンスを並び替え、すべてのビートが単一のトランザクションになるように調整する。

**引数:**

 - ``maxReadBytes: Int`` - 単一のトランザクションでの最大バイト数。

**使用例:**

```scala
interleaved.node := AXI4Deinterleaver() := consecutive.node
```

TLFragmenter
------------

TLFragmenterウィジェットはTileLinkインタフェースの最大論理転送サイズを縮小するために、大きな大きなトランザクションをより小さなトランザクションに分割する。

**引数:**

 - ``minSize: Int`` - 接続されるManagerの最小転送サイズ
 - ``maxSize: Int`` - Fragmenterが適用された後の最大転送サイズ
 - ``alwaysMin: Boolean`` - (オプション)すべてのリクエストをminSizeに分割するか(そうでない場合、Managerでサポートされる最大サイズに分割する) (デフォルトではfalse)。
 - ``earlyAck: EarlyAck.T`` - (オプション)複数ビートのPutコマンドにおいて、最初のビートでAcknowledgeするか、最後のビットでAckowledgeするか？取りうる値は以下である(デフォルト : ``EarlyAck.None``)
    - ``EarlyAck.AllPuts`` - 常に最初のビートでAckowledgeを返す。
    - ``EarlyAck.PutFulls`` - PutFullの場合は最初のビートでAckowledgeを返す。そうでなければ最後のビートでAckowledgeを返す。
    - ``EarlyAck.None``  - 常に最後のビートでAckowledgeを返す。
- ``holdFirstDenied: Boolean`` - (optional) Allow the Fragmenter to unsafely combine multibeat Gets by taking the first denied for the whole burst. (default: false)

**使用例:**

```scala
val beatBytes = 8
val blockBytes = 64

single.node := TLFragmenter(beatBytes, blockBytes) := multi.node

axi4lite.node := AXI4Fragmenter() := axi4full.node
```

**Additional Notes**

 - TLFragmenterは PutFull, PutPartial, LogicalData, Get, Hint に対して変更を加える。
 - TLFragmenterはArithmeticDataはそのまま通過させる(alwaysMinが有効の場合は`minSize`まで縮小させる)
 - TLFragmenterはacqureコマンドを変更できない(livelockを発生させる可能性がある)。したがって、両サイドにキャッシュを配置するのは危険である。

AXI4Fragmenter
--------------

AXI4Fragmenterは`TLFragmenter`と似ているが、TileLinkではなく複数ビートのAXI4トランザクションを分割する。これはAXI4からAXI4-Liteへの効率的な変換の機能も持っている。このウィジェットのコンストラクタは引数を取らない。

**使用例:**

```scala
axi4lite.node := AXI4Fragmenter() := axi4full.node
```

TLSourceShrinker
----------------

Managerが管理するソースIDは、通常は接続されるClientの情報により計算される。いくつかの場合には、ソースのIDの数を固定したいときがある。例えばTileLinkのポートをVerilogのブラックボックスにエクスポートしたいときなどが挙げられる。このような問題を解決するために、Clientがより多くのソースIDを必要とする場合にはTLSourceShrinkerを使用できる。

**引数:**

 - ``maxInFlight: Int`` - The maximum number of source IDs that will be sent from the TLSourceShrinker to the manager.
 - ``maxInFlight: Int``  - TLSourceShrinkerからManagerに転送されるソースIDの最大数。

**使用例:**

```scala
// ClientノードはおそらくソースIDを16以上持っている。
// ManagerノードはIDを16個までしか管理できない。
manager.node := TLSourceShrinker(16) := client.node
```

AXI4IdIndexer
-------------

`TLSourceShrinker`のAXI4番である。スレーブAXI4インタフェースのAWID/ARIDのビット数を制限する。AXI4ポートを外部インタフェース・もしくはブラックボックスに接続する場合に有益である。

**引数:**

 - ``idBits: Int`` - スレーブインタフェース上のIDビット数

**使用例:**

```scala
// マスターノードはおおそらく16以上のIDを持っている。
// スレーブノードは4つのIDまでしか管理できない。
slave.node := AXI4IdIndexer(4) := master.node
```

**注意:**

AXI4IDIndexerはスレーブインタフェースに`user`フィールドを追加し、マスターリクエストのIDをフィールドにストアする。`user`フィールドを持たないAXI4インタフェースに接続する場合、`AXI4UserYanker`を使用する必要がある。

TLWidthWidget
-------------

TileLinkの物理的なインタフェースビット幅を変更する。TileLinkインタフェースのビット幅はManagerにより構成されるが、Client側が特定のビット幅を持っていたい場合がある。

**引数:**

 - ``innerBeatBytes: Int`` - Client側から見た物理的なビット幅(バイト単位)

**使用例:**

```scala
// ManagerノードがbeatBytesを8に設定している。
// WidthWidgetにより、ClientはbeatByetsが4に設定されている。
manager.node := TLWidthWidget(4) := client.node
```

TLFIFOFixer
-----------

FIFOドメインを宣言したTileLinkManagerは、Clientから到達する、そのドメインへのすべてのFIFOオーダリングなリクエストがリクエストの順番通りに返されることを保証しなければならない。しかし、そのレスポンスの順序を制御するだけでは、同じFIFOドメイン内の他のManagerからのインタリーブされたレスポンスを制御することができない。FIFOの順序を保証するための責任はTLFIFOFixerにより達成される。

**引数:**

 - ``policy: TLFIFOFixer.Policy`` - (オプション) どのManagerがTLFIFOFixerにオーダリングを行わせるか？(デフォルト: `TLFIFOFixer.all`) 

`policy`が取ることのできる引数は以下のとおりである。

 - `TLFIFOFixer.all` - すべてのManager(FIFOドメイン以外のManagerも含む)のオーダリングが保証される。
 - ``TLFIFOFixer.allFIFO`` - All managers that define a FIFO domain will have ordering guaranteed
 - ``TLFIFOFixer.allFIFO`` - FIFOドメインを定義するすべてのManagerがオーダリングを保証される。
 - ``TLFIFOFixer.allVolatile`` - `VOLATILE`, `PUT_EFFECTS`, `GET_EFFECTS`のリージョンタイプを持つすべてのManagerがオーダリングを保証させる(リージョンタイプについては`Manager Node`を参照すること)。

TLXbar と AXI4Xbar
-------------------

TileLinkおよびAXI4のクロスバージェネレータであり、TL/AXIClientからのリクエストを、Manager・スレーブのアドレス定義に基づいてTL/AXIのスレーブに転送する。通常はこのウィジェットは引数無しで生成される。しかし、アービタ内でどのClientポートが優先権を手に入れるかなどのアービトレーションのポリシーを変更することができる。デフォルトのポリシーは`TLArbiter.roundRobin`であるが、優先権洗濯ポリシーを変更したい場合は`TLArbiter.lowestIndexFirst`に変更することができる。

**引数:**

All arguments are optional.

すべての引数はオプションである。

 - ``arbitrationPolicy: TLArbiter.Policy`` - 使用するアービトレーションのポリシー
 - ``maxFlightPerId: Int`` - (AXI4のみ) 同じIDにおいて同時にインフライトになれるIDの数(デフォルト: 7)。
 - ``awQueueDepth: Int`` - (AXI4のみ) ライトアドレスキューのサイズ(デフォルト: 2)

**使用例:**

```scala
// lazyモジュールでクロスバーをインスタンス化する。
val tlBus = LazyModule(new TLXbar)

// 単一の入力エッジの接続。
tlBus.node := tlClient0.node
// 複数の入力エッジの接続。
tlBus.node :=* tlClient1.node

// 単一の出力エッジの接続。
tlManager0.node := tlBus.node
// 複数の出力エッジの接続。
tlManager1.node :*= tlBus.node

// クロスバーをlowestIndexFirstのアービトレーションポリシーで宣言する。
// TLArbiterのsingletonを使用しているが、実際にはAXI4である。
val axiBus = LazyModule(new AXI4Xbar(TLArbiter.lowestIndexFirst))

// TLと同様に接続される。
axiBus.node := axiClient0.node
axiBus.node :=* axiClient1.node
axiManager0.node := axiBus.node
axiManager1.node :*= axiBus.node
```



TLToAXI4 と AXI4ToTL
---------------------

TileLinkとAXI4プロトコルのコンバータである。TLToAXI4はClientにTileLinkを持ち、AXI4スレーブに接続する。AXI4ToTLはAXI4マスターをTileLinkのManagerに接続する。通常はデフォルトの引数をオーバライドすることはない。

**使用例:**

```scala
axi4slave.node :=
    AXI4UserYanker() :=
    AXI4Deinterleaver(64) :=
    TLToAXI4() :=
    tlclient.node

tlmanager.node :=
    AXI4ToTL() :=
    AXI4UserYanker() :=
    AXI4Fragmenter() :=
    axi4master.node
```

TLToAXI4コンバータの後には、`AXI4Deinterleaver` を挿入する必要がある。なぜなら、TLToAXI4コンバータはインタリーブされたリードレスポンスを取り扱うことができないからだ。TLToAXI4コンバータはAXI4のユーザフィールドを使用していくつかの情報を格納する。したがってユーザフィールドの存在しないAXI4ポートに接続する場合には`AXI4UserYanker`を接続する必要がある。

AXI4ポートをAXI4ToTLウィジェットに接続する場合、`AXI4Fragmenter`と`AXI4UserYanker`を接続する必要がある。なぜならば、コンバータはユーザフィールドおよび複数ビートのトランザクションを扱うことができないからである。

TLROM
------

The TLROM widget provides a read-only memory that can be accessed using
TileLink. Note: this widget is in the ``freechips.rocketchip.devices.tilelink``
package, not the ``freechips.rocketchip.tilelink`` package like the others.

TLROMウィジェットはTileLink経由でアクセスすることのできるRead-Onlyのメモリである。このウィジェットは`freechips.rocketchip.devices.tilelink`パッケージに含まれており、``freechips.rocketchip.tilelink``ではないことに注意が必要である。

**引数:**

 - ``base: BigInt`` - メモリのベースアドレス
 - ``size: Int`` - バイト単位でのメモリサイズ
 - ``contentsDelayed: => Seq[Byte]``  - ROMのバイト内容を生成するための関数。
 - ``executable: Boolean`` - (オプション) CPUがこのROMの内容をフェッチすることができるかを示す(デフォルト: `true`)
 - ``beatBytes: Int`` - (オプション) バイト単位でのインタフェースのビット幅。(デフォルト: 4)
 - ``resources: Seq[Resource]`` - (オプション) デバイスツリーに接続されるリソースのシーケンス

**使用例:**

```scala
val rom = LazyModule(new TLROM(
  base = 0x100A0000,
  size = 64,
  contentsDelayed = Seq.tabulate(64) { i => i.toByte },
  beatBytes = 8))
rom.node := TLFragmenter(8, 64) := client.node
```

**サポートされる操作:**

TLROMは単一ビートの読み込みのみサポートされる。複数ビートの読み込みを行う場合、TLFragmenterをROMの前に配置する必要がある。

TLRAM と AXI4RAM
-----------------

The TLRAM and AXI4RAM widgets provide read-write memories implemented as SRAMs.

TLRAMとAXI4RAMウィジェットはSRAMとして実装されるRead-Writeメモリである。

**引数:**

 - ``address: AddressSet`` - RAMがカバーするアドレス範囲。
 - ``cacheable: Boolean`` - (オプション) RAMのコンテンツをキャッシュできるかを示す(デフォルト: `true`)。
 - ``executable: Boolean`` - (オプション) RAMのコンテンツが命令としてフェッチできるかを示す(デフォルト: `true`)
 - ``beatBytes: Int`` - (オプション) TL/AXI4インタフェースの幅をバイト単位で示す(デフォルト: 4)。
 - ``atomics: Boolean`` - (オプション, TileLinkのみ) RAMがアトミック操作をサポートするかを示す(デフォルト: `false`)。

**使用例:**

```scala
val xbar = LazyModule(new TLXbar)

val tlram = LazyModule(new TLRAM(
  address = AddressSet(0x1000, 0xfff)))

val axiram = LazyModule(new AXI4RAM(
  address = AddressSet(0x2000, 0xfff)))

tlram.node := xbar.node
axiram := TLToAXI4() := xbar.node
```

**サポートされている操作:**

TLRAMはTL-ULリクエストの単一ビートのみをサポートしている。`atomics`をtrueに設定すると、LogicalとArithmeticの操作をサポートする。複数ビートのRead/Writeを行いたい場合は`TLFragmenter`を接続すること。

AXI4RAMはAXI4-Liteの操作のみをサポートするため、複数ビートのRead/Writeおよび最大サイズよりも小さなRead/Writeはサポートされない。フルのAXI4プロトコルを使用したい場合は `AXI4Fragmenter`を使用すること。

