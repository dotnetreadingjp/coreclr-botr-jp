Virtual Stub Dispatch
仮想スタブディスパッチ
=====================

(これはhttps://github.com/dotnet/coreclr/blob/master/Documentation/botr/virtual-stub-dispatch.mdの日本語訳です。対象rev.は 8d3936b ）

著者：Simon Hall ([@snwbrdwndsrf](https://github.com/snwbrdwndsrf)) - 2006

はじめに
============

仮想スタブディスパッチ処理（VSD：Virtual Stub Dispatching）は、仮想メソッド呼び出しについて、伝統的な仮想メソッドテーブルの代わりにスタブを使用する技術です。過去、インターフェイスディスパッチでは、それぞれのインターフェイスがプロセス内で一意な識別子を持つことと、読み込まれたすべてのインターフェイスがグローバルなインターフェイス仮想テーブルマップに追加される必要がありました。この要件により、NGENのシナリオにおいて、すべてのインターフェイスと、インターフェイスを実装するすべてのクラスが実行時に復元されなければならないこととなり、起動時のワーキングセットを著しく増加させていました。スタブディスパッチ処理の動機は、これに関連するワーキングセットの大半を減らすことと、プロセスのライフタイム全体に残りの作業を分散させることでした。

VSDは仮想インスタンスメソッド呼び出しとインターフェイスメソッド呼び出しの両方に使用可能なのですが、現在のところインターフェイスディスパッチにのみ使用されています。

依存関係
------------

### コンポーネントの依存関係

スタブディスパッチ処理のコードはランタイムの残りの部分から比較的独立しています。スタブディスパッチ処理は依存するコンポーネントから使用できるAPIを提供しており、さらに以下に述べる依存関係は比較的小さくなっています。

#### コードマネージャー

メソッドの状態、特に、VSDがスタブ生成とターゲットキャッシュ処理のような詳細事項について判定できるようにするための、特定のメソッドが最終状態に遷移済かどうかの情報提供について、VSDは実質的にコードマネージャーに依存しています。

#### 型とメソッド

メソッドテーブルは、任意のVSDコールサイトについて、その呼び出し先コードアドレスを判定するための、ディスパッチマップへのポインターを保持します。

#### 特別な型

COM相互運用型への呼び出しは、両方から特別なターゲット解決がなされるよう、カスタムディスパッチが実行されなければなりません。

### このコンポーネントに依存するコンポーネント

#### コードマネージャー

コードマネージャーは、インターフェイス呼び出しのコールサイトターゲットをJITコンパイラに提供するにあたってVSDに依存します。

#### クラスビルダー

クラスビルダーは、型の構築中、VSDのコードが型のディスパッチで使用することになるディスパッチマップを作成するために、ディスパッチマップ処理コードによって公開されるAPIを使用します。

設計の目標と目標でないもの
--------------------------

### 目標

#### ワーキングセットの削減

インターフェイスディスパッチは、以前は、プロセス内のすべてのインターフェイス識別子を取り扱う、巨大で、少々まばらなvtableルックアップマップを使用して実装されていました。VSDの目標は、理論上は関連するコールサイトとそのディスパッチスタブが互いに隣接するようにしてワーキングセットの密度を上げるように、必要とされた都度ディスパッチスタブを生成することで、コールドワーキングセットの量を削減することでした。

ここには注意すべき重要なポイントがあります。システムの実行に従って作成され、回収される個々のスタブを追跡するために必要となるデータ構造のために、VSDがコールサイトごとに導入する初期ワーキングセットvtableの場合よりも多くなっています。ただし、これらのデータ構造は単純なディスパッチ処理には不要であるため、アプリケーションが安定的な状態に近づくにしたがってページアウトされていきます。残念ながら、クライアントアプリケーションでは、これは起動時間が遅くなることと同じことです。これは、仮想メソッドディスパッチでVSDを無効にした要因の一つです。

#### 同等なスループット

インターフェイスメソッドと仮想メソッドのディスパッチが、以前のvtableディスパッチ機構と平均的には同等であり続けることが重要でした。

インターフェイスディスパッチでこれを達成できることはすぐにわかりましたが、仮想メソッドのディスパッチは少々遅くなることが明らかになりました。これは仮想メソッドの場合にVSDを無効にした要因の一つです。

トークン表現とディスパッチマップの設計
-----------------------------------------------

ディスパッチトークンは、実行時に割り付けられる、ネイティブワードサイズの値で、内部的には、インターフェイスとスロットを表すタプルでなります。

この設計では、割り当てられた型のID値とスロット番号の組み合わせを使用します。ディスパッチトークンはこれら2つの値の組み合わせでなります。ランタイムとの統合の折り合いをつけるために、この実装では従来のvtableのレイアウトと同じ方法でスロット番号を割り当てています。つまり、この抽象構造を取り扱うために、ランタイムは直接ではなくヘルパーメソッド経由でvtableにアクセスしなければならないという点を除き、メソッドテーブル、メソッド既述子、スロット番号をvtableとまったく同じ方法で処理できます。

_スロット_ と言う用語は、常に従来のvtableのレイアウトにおけるスロットのインデックス値であり、マッピング機構によって作成および解釈されるものという文脈で使用していきます。これが何を意味するかと言うと、ランタイム内に以前実装されていた、仮想メソッドのスロットの後に非仮想メソッドのスロットが続くという従来のメソッドテーブルのレイアウトを思い描いた場合の、そのスロット番号だということです。ランタイムのコードにおいて、スロットは従来のvtable構造におけるインデックスと、vtableそのものへのポインターのアドレスの両方の意味を持つため、この違いを理解することが重要です。変更点は、スロットは今やインデックス値のみであり、コードポインターのアドレスは実装テーブル（後述）に保持されるということです。

動的割り当てられる型ID値については後述します。

### メソッドテーブル

#### 実装テーブル

これは、型によって導入される各メソッドの本体に対し、そのメソッドのエントリポイントへのポインターを保持する配列です。このテーブルのメンバーは以下の順序で配置されています。

- 導入された（新しいスロットの）仮想メソッド群。
- 導入された非仮想（インスタンスと静的）メソッド群。
- オーバーライドする仮想メソッド群。

この形式になっている理由は、これが従来型のvtableレイアウトの自然な拡張になるためです。結果として、スロットマップ（後述）の多くのエントリは、上記の順序と、クラスの仮想メソッドと非仮想メソッドの総数のようなその他の詳細事項から推論することができます。

仮想インスタンスメソッドに対するスタブディスパッチが無効な場合（現在の状態）、実装テーブルは存在しなくなり、本物のvtableによってその実体が置き換えられます。すべてのマッピング結果は、実装テーブルではなくvtable用のスロットとして表されます。本書の残りの部分で実装テーブルについて言及されているときには、このことを思い出してください。

#### スロットマップ

スロットマップは0以上の< _型_, [<_スロット_, _スコープ_, (_インデックス|テーブル_)>]>エントリのテーブルです。 _型_ は、前述の動的に割り当てられるID番号で、現在のクラスを示すための番兵値（仮想インスタンスメソッドの呼び出し）か、現在のクラスによって実装される（かその親によって暗黙的に実装される）インターフェイス用のIDです。（角カッコ内の）サブマップには、1つ以上のエントリがあります。サブマップ各エントリについて、1番目の要素は常に _型_ のスロットを示します。2 番目の要素である _スコープ_ は、3番目の要素が実装 _インデックス_ なのか _スロット_ 番号なのかを示します。 _スコープ_ は次の要素の番号が仮想スロット番号として解釈され、 _this.スロット_ として仮想的に解決されるべきことを示す既知の番兵値にできます。 _スコープ_ は現在のクラス継承階層における特定のクラスを識別することもでき、そのような場合、第3引数は _スコープ_ によって示されるクラスの実装テーブルの _インデックス_ であり、かつ _型.スロット_ に対する最終的なメソッド実装です。

#### 例

下の図は小さなクラス構造（C#によるモデル）と、その結果として生成される各クラスの実装テーブルとスロットマップがどのようなものになるかを示しています。

![Figure 1](https://github.com/dotnet/coreclr/blob/master/Documentation/images/virtualstubdispatch-fig1.png)

つまり、このマップを見ると、スロットマップのサブマップの1番目の列が、従来の仮想テーブルビューのスロット番号に対応しているのが分かります（System.Object自身が4つの仮想メソッドを提供していることを思い出してください。ただし、わかりやすくするためにそれらは省略しています）。メソッドの実装の検索は常にボトムアップです。つまり、型 _B_ のオブジェクトがあり、 _I.Foo_ を呼び出したいとすると、 _B_ のスロットマップを起点として _I.Foo_ のマッピングを探すことになります。そこで見つからないので、 _A_ のスロットマップに行って検索し、そこで見つけます。そこでは、 _I_ の仮想スロット0（ _I.Foo_ に対応します）は仮想スロット0で実装されていると記述されています。そして、 _B_ のスロットマップに戻り、スロット0の実装を検索し、B自身の実装テーブルのスロット1で実装されていることがわかります。

### Additional Uses
<!--
### 追加の使用方法
-->

It is important to note that this mapping technique can be used to implement methodimpl re-mapping of virtual slots (i.e., a virtual slot mapping in the map for the current class, similar to how an interface slot is mapped to a virtual slot). Because of the scoping capabilities of the map, non-virtual methods may also be referenced. This may be useful if ever the runtime wants to support the implementation of interfaces with non-virtual methods.
<!--
このマッピングテクニックを、仮想スロットのメソッド実装再マッピングの実装に使用できる（つまり、現在のクラスに対するスロットマップ内の仮想スロットマッピングは、インターフェイススロットが仮想スロットにマップされる方法によく似ている）ことに注意することが重要です。スロットマップのスコープ設定機能により、非仮想メソッドも参照されるようにできます。これは、ランタイムが非仮想メソッドでのインターフェイスの実装をサポートしたい場合であっても有用なことでしょう。
-->

### Optimizations
<!--
### 最適化
-->

The slot maps are bit-encoded and take advantage of typical interface implementation patterns using delta values, thus reducing the map size significantly. In addition, new slots (both virtual and non-) can be implied by their order in the implementation table. If the table contains new virtual slots followed by new instance slots, then followed by overrides, then the appropriate slot map entries can be implied by their index in the implementation table combined with the number of virtuals inherited by the parent class. All such implied map entries have been indicated with a (\*). The current layout of data structures uses the following pattern, where the DispatchMap is only present when mappings cannot be fully implied by ordering in the implementation table.
<!--
スロットマップはビットエンコードされており、デルタ値を使用して典型的なインターフェイス実装パターンを活用するので、スロットマップのサイズは顕著に小さくなっています。さらに、新しいスロット（仮想と非仮想の両方）は実装テーブル内の順序によって黙示できます。実装テーブルが新しい仮想スロットに続けて新しいインスタンススロット、さらにその後にオーバーライドを保持する場合、親クラスによって継承された仮想メソッドの数と組み合わせた実装テーブル内のインデックスから、適切なスロットマップエントリを黙示できます。そのように暗黙的に示されるマップエントリはすべて (\*) で示されます。データ構造の現在のレイアウトは以下のパターンを使用します。ここで、DispatchMap は実装テーブルの順序によってマッピングを完全に黙示できない場合にのみ存在します。
-->

	MethodTable -> [DispatchMap ->] ImplementationTable

Type ID Map
-----------

This will map types to IDs, which are allocated as monotonically increasing values as each previously unmapped type is encountered. Currently, all such types are interfaces. 

Currently, this is implemented using a HashMap, and contains entries for both lookup directions.

Dispatch Tokens
---------------

Dispatch tokens will be <_typeID_,_slot_> tuples. For interfaces, the type will be the interface ID assigned to that type. For virtual methods, this will be a constant value to indicate that the slot should just be resolved virtually within the type to be dispatched on (a virtual method call on _this_). This value pair will in most cases fit into the platform's native word size. On x86, this will likely be the lower 16 bits of each value, concatenated. This can be generalized to handle overflow issues similar to how a _TypeHandle_ in the runtime can be either a _MethodTable_ pointer or a <_TypeHandle,TypeHandle_> pair, using a sentinel bit to differentiate the two cases. It has yet to be determined if this is necessary.

Design of Virtual Stub Dispatch
===============================

Dispatch Token to Implementation Resolution
-------------------------------------------

Given a token and type, the implementation is found by mapping the token to an implementation table index for the type. The implementation table is reachable from the type's MethodTable. This map is created in BuildMethodTable: it enumerates all interfaces implemented by the type for which it is building a MethodTable and determines every interface method that the type implements or overrides. By keeping track of this information, at interface dispatch time it is possible to determine the target code given the token and the target object (from which the MethodTable and token mapping can be obtained).

Stubs
-----

Interface dispatch calls go through stubs. These stubs are all generated on demand, and all have the ultimate purpose of matching a token and object with an implementation, and forwarding the call to that implementation.

There are currently three types of stubs. The below diagram shows the general control flow between these stubs, and will be explained below.

![Figure 2](images/virtualstubdispatch-fig2.png)

### Generic Resolver

This is in fact just a C function that serves as the final failure path for all stubs. It takes a <_token_, _type_> tuple and returns the target. The generic resolver is also responsible for creating dispatch and resolver stubs when they are required, patching indirection cells when better stubs become available, caching results, and all bookkeeping.

### Lookup Stubs

These stubs are the first to be assigned to an interface dispatch call site, and are created when the JIT compiles an interface call site. Since the JIT has no knowledge of the type being used to satisfy a token until the first call is made, this stub passes the token and type as arguments to the generic resolver. If necessary, the generic resolver will also create dispatch and resolve stubs, and will then back patch the call site to the dispatch stub so that the lookup stub is no longer used.

One lookup stub is created for each unique token (i.e., call sites for the same interface slot will use the same lookup stub).

### Dispatch Stubs

These stubs are used when a call site is believed to be monomorphic in behaviour. This means that the objects used at a particular call site are typically the same type (i.e. most of the time the object being invoked is the same as the last object invoked at the same site.) A dispatch stub takes the type (MethodTable) of the object being invoked and compares it with its cached type, and upon success jumps to its cached target. On x86, this is typically results in a "comparison, conditional failure jump, jump to target" sequence and provides the best performance of any stub. If a stub's type comparison fails, it jumps to its corresponding resolve stub (see below).

One dispatch stub is created for each unique <_token_,_type_> tuple, but only lazily when a call site's lookup stub is invoked.

### Resolve Stubs

Polymorphic call sites are handled by resolve stubs. These stubs use the key pair <_token_, _type_> to resolve the target in a global cache, where _token_ is known at JIT time and _type_ is determined at call time. If the global cache does not contain a match, then the final step of the resolve stub is to call the generic resolver and jump to the returned target. Since the generic resolver will insert the <_token_, _type_, _target_> tuple into the cache, a subsequent call with the same <_token_,_ type_> tuple will successfully find the target in the cache.

When a dispatch stub fails frequently enough, the call site is deemed to be polymorphic and the resolve stub will back patch the call site to point directly to the resolve stub to avoid the overhead of a consistently failing dispatch stub. At sync points (currently the end of a GC), polymorphic sites will be randomly promoted back to monomorphic call sites under the assumption that the polymorphic attribute of a call site is usually temporary. If this assumption is incorrect for any particular call site, it will quickly trigger a backpatch to demote it to polymorphic again.

One resolve stub is created per token, but they all use a global cache. A stub-per-token allows for a fast, effective hashing algorithm using a pre-calculated hash derived from the unchanging components of the <_token_, _type_> tuple.

### Code Sequences

The former interface virtual table dispatch mechanism results in a code sequence similar to this:

![Figure 3](images/virtualstubdispatch-fig3.png)

And the typical stub dispatch sequence is:

![Figure 1](images/virtualstubdispatch-fig4.png)

where expectedMT, failure and target are constants encoded in the stub.

The typical stub sequence has the same number of instructions as the former interface dispatch mechanism, and fewer memory indirections may allow it to execute faster with a smaller working set contribution. It also results in smaller JITed code, since the bulk of the work is in the stub instead of the call site. This is only advantageous if a callsite is rarely invoked. Note that the failure branch is arranged so that x86 branch prediction will follow the success case.

Current State
=============

Currently, VSD is enabled only for interface method calls but not virtual instance method calls. There were several reasons for this:

- **Startup:** Startup working set and speed were hindered because of the need to generate a great deal of initial stubs.
- **Throughput:** While interface dispatches are generally faster with VSD, virtual instance method calls suffer an unacceptable speed degradation.

As a result of disabling VSD for virtual instance method calls, every type has a vtable for virtual instance methods and the implementation table described above is disabled. Dispatch maps are still present to enable interface method dispatching.

Physical Architecture
=====================

For dispatch token and map implementation details, please see [clr/src/vm/contractImpl.h](https://github.com/dotnet/coreclr/blob/master/src/vm/contractimpl.h) and [clr/src/vm/contractImpl.cpp](https://github.com/dotnet/coreclr/blob/master/src/vm/contractimpl.cpp).

For virtual stub dispatch implementation details, please see [clr/src/vm/virtualcallstub.h](https://github.com/dotnet/coreclr/blob/master/src/vm/virtualcallstub.h) and [clr/src/vm/virtualcallstub.cpp](https://github.com/dotnet/coreclr/blob/master/src/vm/virtualcallstub.cpp).