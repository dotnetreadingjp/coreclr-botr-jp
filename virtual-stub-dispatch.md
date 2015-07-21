Virtual Stub Dispatch
仮想スタブディスパッチ
=====================

Author: Simon Hall ([@snwbrdwndsrf](https://github.com/snwbrdwndsrf)) - 2006

Introduction
はじめに
============

Virtual stub dispatching (VSD) is the technique of using stubs for virtual method invocations instead of the traditional virtual method table. In the past, interface dispatch required that interfaces had process-unique identifiers, and that every loaded interface was added to a global interface virtual table map. This requirement meant that all interfaces and all classes that implemented interfaces had to be restored at runtime in NGEN scenarios, causing significant startup working set increases. The motivation for stub dispatching was to eliminate much of the related working set, as well as distribute the remaining work throughout the lifetime of the process.
仮想スタブディスパッチ処理（VSD：Virtual Stub Dispatching）は、仮想メソッド呼び出しについて、伝統的な仮想メソッドテーブルの代わりにスタブを使用する技術です。過去には、インターフェイスディスパッチはインターフェイスがそれぞれプロセス内で一意な識別子を持つことと、読み込まれたすべてのインターフェイスがグローバルなインターフェイス仮想テーブルマップに追加されることを必要としていました。この要件は、NGEN のシナリオにおいて、すべてのインターフェイスとインターフェイスを実装するすべてのクラスが実行時に復元されなければならないことを意味し、起動時のワーキングセットを著しく増加させていました。スタブディスパッチ処理の動機は、関連するワーキングセットの多くを削減することと、さらにプロセスのライフタイムの残り全体に作業を分散させることでした。

Although it is possible for VSD to dispatch both virtual instance and interface method calls, it is currently used only for interface dispatch.
VSD は仮想インスタンスメソッド呼び出しとインターフェイスメソッド呼び出しの両方に使用可能ですが、現在はインターフェイスディスパッチにのみ使用されています。

Dependencies
依存関係
------------

### Component Dependencies
### コンポーネントの依存関係

The stub dispatching code exists relatively independently of the rest of the runtime. It provides an API that allows dependent components to use it, and the dependencies listed below comprise a relatively small surface area.
スタブディスパッチ処理のコードはランタイムの残りの部分からは比較的独立して存在しています。スタブディスパッチ処理は依存するコンポーネントから使用できる API を提供しており、さらに以降に述べる依存関係は比較的小さなサーフェイス領域を comprise します。★

#### Code Manager
#### コードマネージャー

VSD effectively relies on the code manager to provide information about state of a method, in particular, whether or not any particular method has transitioned to its final state in order that VSD may decide on details such as stub generation and target caching.
VSD はメソッドの状態、特に、VSD がスタブ生成とターゲットキャッシュ処理のような詳細について判断できるようにするために特定のメソッドが最終状態に遷移済かどうかについての情報を提供するために、実質的にコードマネージャーに依存しています。

#### Types and Methods
#### 型とメソッド

MethodTables hold pointers to the dispatch maps used to determine the target code address for any given VSD call site.
メソッドテーブルは、任意の VSD コールサイトに対するターゲットのコードアドレスを判定するために使用されるディスパッチマップへのポインターを保持します。

#### Special Types
#### 特別な型

Calls on COM interop types must be custom dispatched, as they both have specialized target resolution.
as they both have 特殊化されたターゲット解決、COM 相互運用型への呼び出しはカスタムのディスパッチを実行しなければなりません。★

### Components Dependent on this Component
### このコンポーネントに依存するコンポーネント

#### Code Manager
#### コードマネージャー

The code manager relies on VSD for providing the JIT compiler with call site targets for interface calls.
コードマネージャーは、インターフェイス呼び出しに対するコールサイトターゲットを持つ JIT コンパイラを提供するために VSD に依存します。

#### Class Builder
#### クラスビルダー

The class builder uses the API exposed by the dispatch mapping code to create dispatch maps during type building that will be used at dispatch type by the VSD code.
クラスビルダーは、VSD コードによって型のディスパッチで使用される型構築の間にディスパッチマップを作成するために、ディスパッチマップ処理によって公開される API を使用します。

Design Goals and Non-goals
設計の目標と目標でないもの
--------------------------

### Goals
### 目標

#### Working Set Reduction
#### ワーキングセットの軽減

Interface dispatch was previously implemented using a large, somewhat sparse vtable lookup map dealing with process-wide interface identifiers. The goal was to reduce the amount of cold working set by generating dispatch stubs as they were required, in theory keeping related call sites and their dispatch stubs close to each other and increasing the working set density.
インターフェイスディスパッチは、以前は、巨大で、プロセス全体のインターフェイス識別子を取り扱うそれなりに疎な vtable ルックアップマップを使用して実装されていました。目標は、理論的には関連するコールサイトとそのディスパッチスタブを互いに隣接するようにし、ワーキングセットの密度を上げるように、必要なディスパッチスタブを生成することで、コールドワーキングセットの量を削減することでした。

It is important to note that the initial working set involved with VSD is higher per call site due to the data structures required to track the various stubs that are created and collected as the system runs; however, as an application reaches  steady state, these data structures are not needed for simple dispatching and so gets paged out. Unfortunately, for client applications this equated to a slower startup time, which is one of the factors that led to disabling VSD for virtual methods.
注意すべき重要なポイントがあります。システムの実行に従って作成および回収されるさまざまなスタブを追跡するために必要なデータ構造のために、コールサイトごとに VSD によって導入される初期ワーキングセットは以前よりも多くなっています。ただし、アプリケーションが安定的な状態に近づくにつれて、これらのデータ構造は単純なディスパッチ処理には不要であるために、ページアウトされていきます。残念ながら、クライアントアプリケーションでは起動時間が遅くなることを equate します。これは、仮想メソッドでは VSD を無効にした要因の一つです。★

#### Throughput Parity
#### スループット parity ★

It was important to keep interface and virtual method dispatch at an amortized parity with the previous vtable dispatch mechanism.
インターフェイスメソッドと仮想メソッドのディスパッチが、以前の vtable ディスパッチ機構と amortized parity であり続けることが重要でした。★

While it was immediately obvious that this was achievable with interface dispatch, it turned out to be somewhat slower with virtual method dispatch, one of the factors that led to disabling VSD for virtual methods.
これがインターフェイスディスパッチでは達成されることはすぐに明らかになりましたが、仮想メソッドのディスパッチではいくらか遅くなることが明らかになり、これは仮想メソッドの場合に VSD を無効にした要因の一つです。

Design of Token Representation and Dispatch Map
トークン表現とディスパッチマップの設計
-----------------------------------------------

Dispatch tokens are native word-sized values that are allocated at runtime, consisting internally of a tuple that represents an interface and slot.
ディスパッチトークンは、実行時に割り付けられる、ネイティブワードサイズの値で、内部的にはインターフェイスとスロットを表すタプルで構成されています。

The design uses a combination of assigned type identifier values and slot numbers. Dispatch tokens consist of a combination of these two values. To facilitate integration with the runtime, the implementation also assigns slot numbers in the same way as the classic v-table layout. This means that the runtime can still deal with MethodTables, MethodDescs, and slot numbers in exactly the same way, except that the v-table must be accessed via helper methods instead of being directly accessed in order to handle this abstraction.
今の設計では、割り当てられた型の ID 値とスロット番号の組み合わせを使用します。ディスパッチトークンはこれら 2 つの値の組み合わせで構成されています。ランタイムとの統合の折り合いをつけるために、今の実装では従来の v-table のレイアウトと同じ方法でスロット番号を割り当てています。つまり、ランタイムは、この抽象構造を処理するために、直接アクセスするのではなくヘルパーメソッド経由で v-table にアクセスしなければならない点を除き、依然として MethodTable、MethodDesc、スロット番号をまったく同じ方法で取り扱うことができます。

The term _slot_ will always be used in the context of a slot index value in the classic v-table layout world and as created and interpreted by the mapping mechanism. What this means is that this is the slot number if you were to picture the classic method table layout of virtual method slots followed by non-virtual method slots, as previously implemented in the runtime. It's important to understand this distinction because within the runtime code, slot means both an index into the classic v-table structure, as well as the address of the pointer in the v-table itself. The change is that slot is now only an index value, and the code pointer addresses are contained in the implementation table (discussed below).
_スロット_ と言う用語は、常に従来型の v-table のレイアウトにおけるスロットのインデックス値という文脈で使用され、マッピング機構によって作成および解釈されるものです。これが何を意味するかと言うと、以前ランタイム内に実装されていた、仮想メソッドのスロットの後に非仮想メソッドのスロットが続くという従来型のメソッドテーブルのレイアウトを描く場合のスロット番号だということです。ランタイムのコードにおいて、スロットは従来型の v-table 構造におけるインデックスでありつつ v-table 自身におけるポインターのアドレスでもあるため、この違いを理解することが重要です。この変更は、スロットは今やインデックス値のみであり、コードポインターのアドレスは実装テーブル（後述）に保持されます。★

The dynamically assigned type identifier values will be discussed later on.
動的割り当てられる型 ID 値については後述します。

### Method Table
### メソッドテーブル

#### Implementation Table
#### 実装テーブル

This is an array that, for each method body introduced by the type, has a pointer to the entrypoint to that method. Its members are arranged in the following order:
これは、型によって導入される各メソッドの本体に対し、そのメソッドのエントリポイントへのポインターを保持する配列です。このテーブルのメンバーは以下の順序になっています。

- Introduced (newslot) virtual methods.
- 導入された（新しいスロットの）仮想メソッド群。
- Introduced non-virtual (instance and static) methods.
- 導入された非仮想（インスタンスと静的）メソッド群。
- Overriding virtual methods.
- オーバーライドする仮想メソッド群。

The reason for this format is that it provides a natural extension to the classic v-table layout. As a result many entries in the slot map (described below) can be inferred by this order and other details such as the total number of virtuals and non-virtuals for the class.
この形式になっている理由は、これが従来型の v-table レイアウトの自然な拡張になるためです。結果として、スロットマップ内の多くのエントリ（後述）はこの順序およびクラスの仮想メソッドと非仮想メソッドの総数のようなその他の詳細によって推論できます。

When stub dispatch for virtual instance methods is disabled (as it is currently), the implementation table is non-existant and is substituted with a true vtable. All mapping results are expressed as slots for the vtable rather than an implementation table. Keep this in mind when implementation tables are mentioned throughout this document.
仮想インスタンスメソッドに対するスタブディスパッチを無効にすると（現在の状態です）、実装テーブルは存在しなくなり、本物の vtable によってその実体が置き換えられます。すべてのマッピング結果は、実装テーブルではなく vtable に対するスロットとして表されます。本書の残りの部分で実装テーブルについて言及されているときには、このことを思い出してください。

#### Slot Map
#### スロットマップ

The slot map is a table of zero or more <_type_, [<_slot_, _scope_, (_index | slot_)>]> entries. _type_ is the dynamically assigned identification number mentioned above, and is either a sentinel value to indicate the current class (a call to a virtual instance method), or is an identifier for an interface implemented by the current class (or implicitly by one if its parents). The sub-map (contained in brackets) has one or more entries. Within each entry, the first element always indicates a slot within _type_. The second element, _scope_, specifies whether or not the third element is an implementation _index_ or a _slot_ number. _scope_ can be a known sentinel value that indicates that the next number is to be interpreted as a virtual slot number, and should be resolved virtually as _this.slot_. _scope_ can also identify a particular class in the inheritance hierarchy of the current class, and in such a case the third argument is an _index_ into the implementation table of the class indicated by _scope_, and is the final method implementation for _type.slot_.
スロットマップは 0 以上の <_type_, [<_slot_, _scope_, (_index | slot_)>]> エントリのテーブルです。 _type_ は前述の動的に割り当てられる ID 番号 で、現在のクラスを示すための番兵値（仮想インスタンスメソッドの呼び出し）か、現在のクラスによって実装される（かその親によって暗黙的に実装される）インターフェイスに対する IDです。（bracket に含まれる）サブマップは、1 つ以上のエントリがあります。その各エントリについて、1 番目の要素は常に _type_ のスロットを示します。2 番目の要素である _scope_ は 3 番目の要素が実装 _index_ なのか _slot_ 番号なのかを示します。 _scope_ は次の番号が仮想スロット番号として解釈されることになることを示す既知の番兵値となる可能性があるため、 _this.slot_ として仮想的に解決されるべきです。 _scope_ は現在のクラスの継承階層における特定のクラスを識別することもでき、そのような場合、第 3 引数は _scope_ によって示されるクラスの実装テーブルの _index_ であり、かつ _type.slot_ に対する最終的なメソッド実装です。★

#### Example
#### 例

The following is a small class structure (modeled in C#), and what the resulting implementation table and slot map would be for each class.
下の図は小さなクラス構造（C# によるモデル）と、その結果として生成される実装テーブルとスロットマップが各クラスに対してどのようなものになるかです。

![Figure 1](images/virtualstubdispatch-fig1.png)

Thus, looking at this map, we see that the first column of the sub-maps of the slot maps correspond to the slot number in the classic virtual table view (remember that System.Object contributes four virtual methods of its own, which are omitted for clarity). Searches for method implementations are always bottom-up. Thus, if I had an object of type _B_ and I wished to invoke _I.Foo_, I would look for a mapping of _I.Foo_ starting at _B_'s slot map. Not finding it there, I would look in _A_'s slot map and find it there. It states that virtual slot 0 of _I_ (corresponding to _I.Foo_) is implemented by virtual slot 0. Then I return to _B_'s slot map and search for an implementation for slot 0, and find that it is implemented by slot 1 in its own implementation table.
つまり、このマップを見ると、スロットマップの 1 番目の列が従来の仮想テーブルビュー（System.Object 自身が 4 つの仮想メソッドを提供していることを思い出してください。ただし、わかりやすくするために省略しています★）のスロット番号に対応しているのが分かります。メソッドの実装の検索は常にボトムアップです。つまり、型 _B_ のオブジェクトがあり、 _I.Foo_ を呼び出したいとすると、 _B_ のスロットマップを起点として _I.Foo_ のマッピングを探すことになります。そこで見つからないので、 _A_ のスロットマップに行って検索し、そこで見つけます。そこでは、 _I_ の仮想スロット 0（ _I.Foo_ に対応します）は仮想スロット 0 で実装されていると記述されています。そして、 _B_ のスロットマップに戻り、スロット 0 の実装を検索し、それ自身の実装テーブルのスロット 1 で実装されていることを見つけます。

### Additional Uses
### 追加の使用方法

It is important to note that this mapping technique can be used to implement methodimpl re-mapping of virtual slots (i.e., a virtual slot mapping in the map for the current class, similar to how an interface slot is mapped to a virtual slot). Because of the scoping capabilities of the map, non-virtual methods may also be referenced. This may be useful if ever the runtime wants to support the implementation of interfaces with non-virtual methods.
このマッピングテクニックを、仮想スロットのメソッド実装再マッピングの実装に使用できる（つまり、現在のクラスに対するスロットマップ内の仮想スロットマッピングは、インターフェイススロットが仮想スロットにマップされる方法によく似ている）ことに注意することが重要です。スロットマップのスコープ設定機能により、非仮想メソッドも参照されるようにできます。これは、ランタイムが非仮想メソッドでのインターフェイスの実装をサポートしたい場合であっても有用なことでしょう。

### Optimizations
### 最適化

The slot maps are bit-encoded and take advantage of typical interface implementation patterns using delta values, thus reducing the map size significantly. In addition, new slots (both virtual and non-) can be implied by their order in the implementation table. If the table contains new virtual slots followed by new instance slots, then followed by overrides, then the appropriate slot map entries can be implied by their index in the implementation table combined with the number of virtuals inherited by the parent class. All such implied map entries have been indicated with a (\*). The current layout of data structures uses the following pattern, where the DispatchMap is only present when mappings cannot be fully implied by ordering in the implementation table.
スロットマップはビットエンコードされており、デルタ値を使用して典型的なインターフェイス実装パターンを活用するので、スロットマップのサイズは顕著に小さくなっています。さらに、新しいスロット（仮想と非仮想の両方）は実装テーブル内の順序によって黙示できます。実装テーブルが新しい仮想スロットに続けて新しいインスタンススロット、さらにその後にオーバーライドを保持する場合、親クラスによって継承された仮想メソッドの数と組み合わせた実装テーブル内のインデックスから、適切なスロットマップエントリを黙示できます。そのように暗黙的に示されるマップエントリはすべて (\*) で示されます。データ構造の現在のレイアウトは以下のパターンを使用します。ここで、DispatchMap は実装テーブルの順序によってマッピングを完全に黙示できない場合にのみ存在します。

	MethodTable -> [DispatchMap ->] ImplementationTable

Type ID Map
型 ID マップ
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