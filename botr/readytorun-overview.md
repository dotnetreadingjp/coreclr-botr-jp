<!--
Managed Executables with Native Code
-->
ネイティブコード付きのマネージド実行可能ファイル
===

（これは https://github.com/dotnet/coreclr/blob/master/Documentation/botr/readytorun-overview.md の日本語訳です。対象rev.は ce99897）

<!--
Since shipping the .NET Runtime over 10 years ago, there has only been one file format which can be used to distribute and deploy managed code components: the CLI file format. This format expresses all execution as machine independent intermediate language (IL) which must either be interpreted or compiled to native code sometime before the code is run. This lack of an efficient, directly executable file format is a very significant difference between unmanaged and managed code, and has become more and more problematic over time. Problems include:
-->
.NETランタイムが出て10年以上経ちましたが、ファイルフォーマットとしては、配布・配置可能なマネージドコードのコンポーネントであるCLIファイルフォーマットしかありません。このフォーマットは全ての実行コードをマシンから独立した中間言語（IL：Intermediate Language）として表現し、このILコードを実行するには、インタープリターまたはコンパイラによってネイティブコードに変換されなければなりません。この、効率的で、直接実行可能なファイルフォーマットがないことが、アンマネージドコードとマネージドコードの非常に大きな違いとなっており、次第により大きな問題になってきました。その問題とは次のようなものです：

<!--
- Native code generation takes a relatively long time and consumes power.
- For security / tamper-resistance, there is a very strong desire to validate any native code that gets run (e.g. code is signed).
- Existing native codegen strategies produce brittle code such that when the runtime or low level framework is updated, all native code is invalidated, which forces the need for recompilation of all that code.
-->
- ネイティブコード生成には、比較的時間とCPUパワーがかかる。
- セキュリティ上の理由や改ざん耐性のために、実行されるネイティブコードがすべて検証されていること（たとえば、コードが署名されているなど）が強く望まれている。
- 既存のネイティブコード生成戦略が生成するコードは脆いところがあり、ランタイムや低レベルのフレームワークを更新すると、全てのネイティブコードが無効になり、ネイティブコード全体の再コンパイルが必要とされる。

<!--
All of these problems and complexity are things that unmanaged code simply avoids. They are avoided because unmanaged code has a format with the following characteristics:
-->
これら全ての問題と複雑さは、アンマネージドコードであれば単純に避けられるものです。アンマネージドコードは以下のような特性のフォーマットを持つからです：

<!--
- The executable format can be efficiently executed directly. Very little needs to be updated at runtime (binding _some_ external references) to prepare for execution. What does need to be updated can be done lazily.
- As long as a set of known versioning rules are followed, version compatible changes in one executable do not affect any other executable (you can update your executables independently of one another).
- The format is clearly defined, which allows variety of compilers to produce it.
-->
- 実行可能ファイルのフォーマットは、効率的に、直接実行できる。実行できるようにするために、実行時に更新する必要があるものはごくわずか（_いくつかの_外部参照へのバインディング）である。必要な更新も遅延実行できる。
- 既知の、一連のバージョン管理のルールに従っている限り、ある実行可能ファイルのバージョン互換性を保った変更が、他の実行可能ファイルに影響することはない（それぞれ独立して実行可能ファイルを更新できる）。
- フォーマットは明確に定義されているため、さまざまなコンパイラが実行可能ファイルを生成できる。

<!--
In this proposal we attack this discrepancy between managed and unmanaged code head on: by giving managed code a file format that has the characteristics of unmanaged code listed above. Having such a format brings managed up to at least parity with unmanaged code with respect to deployment characteristics. This is a huge win!
-->
この提案では、上記のアンマネージドコードの特性を持つファイルフォーマットをマネージドコードに与えることで、マネージドコードとアンマネージドコードの間の相違に正面から取り組みます。そのようなフォーマットを持つことによって、少なくとも配置に関する特性について、マネージコードとアンマネージコードを同等にします。これなら大成功です！

<!--
## Problem Constraints
-->
## 問題の制約事項

<!--
The .NET Runtime has had a native code story (NGEN) for a long time. However what is being proposed here is architecturally different than NGEN. NGEN is fundamentally a cache (it is optional and only affects the performance of the app) and thus the fragility of the images was simply not a concern. If anything changes, the NGEN image is discarded and regenerated. On the other hand:
-->
.NETランタイムには、ずっと前からネイティブコード用の戦略（NGEN）がありました。ただし、ここで提案するものはNGENとはアーキテクチャ的に異なるものです。NGENは基本的にキャッシュであり（使用しないこともでき、アプリケーションのパフォーマンスにのみ影響します）、そのため、イメージの脆さについては単に関わりを持っていません。何らかの変更があると、NGENイメージは破棄、再生成されます。その一方で、
<!--
**A native file format carries a strong guarantee that the file will continue to run despite updates and improvements to the runtime or framework.**
-->
**ネイティブファイルフォーマットは、ランタイムやフレームワークの更新や改善があっても、そのファイルが実行し続けることについての強い保証をもたらします。**

<!--
Most of this proposal is the details of achieving this guarantee while giving up as little performance as possible.
-->
この提案のほとんどは、可能な限りパフォーマンスをあきらめることなく、この保証を達成する方法についての詳細説明です。

<!--
This compatibility guarantee means that, unlike NGEN, anything you place in the file is a _liability_ because you will have to support it in all future runtimes. This drives a desire to be 'minimalist' and only place things into the format that really need to be there. For everything we place into the format we have to believe either:
-->
この互換性保証が意味することは、NGENとは異なり、ファイルに入れるものがすべて_債務_になるということです。将来の全てのランタイムについて、サポートしなければならないからです。これにより、「ミニマリスト」になることと、本当に必要なものだけをそのフォーマットに配置しようという強い動機が生じます。このフォーマットに入れるものは、全て次を満たしていなければなりません。

<!--
1. It is very unlikely to change (in particular we have not changed it over the current life of CLR)
2. We have a scheme in which we can create future runtimes that could support both old and new format efficiently (both in terms of runtime efficiency and engineering complexity).
-->
1. 変更の可能性が非常に低い（特に、現在のCLRのサポート期間中には変更されない）。
2. 新旧両方のフォーマットを効率的にサポートできるランタイムを将来作成できるスキームがある（ここでいう効率的とは、実行効率と工学的な複雑さの両方の観点）。

<!--
Each feature of the file format needs to have an answer to the question of how it versions, and we will be trying to be as 'minimalist' as possible. 
-->
このファイルフォーマットの各機能は、どのようにバージョン管理するかという質問に対する答えを持たねばならず、さらに、我々は可能な限り「ミニマリスト」であろうとします。

<!--
## Solution Outline
-->
## 解決策の概要

<!--
As mentioned, while NGEN is a native file format, it is not an appropriate starting point for this proposal because it is too fragile. 
-->
前述のように、NGENはネイティブファイルフォーマットですが、あまりにも脆いために、この提案のスタート地点として適切ではありません。

<!--
Looking carefully at the CLI file format shows that it is really 'not that bad' as a starting point. At its heart CLI is a set of database-like tables (one for types, methods, fields, etc.), which have entries that point at variable-length things (e.g. method names, signatures, method bodies). Thus CLI is 'pay for play' and since it is already public and version resilient, there is very little downside to including it in the format. By including it we also get the following useful properties:
-->
CLIファイルフォーマットを注意深く見てみると、スタート地点として「それほど悪くない」ことがわかります。本来、CLIは、データベースのような（型、メソッド、フィールド等それぞれに対しての）表のセットであり、それぞれのエントリが可変長のデータ（メソッド名、シグネチャ、メソッド本体など）を指し示すものです。したがって、CLIは「賭ける価値がある」もので、さらに既に公開済みでバージョン耐性を持つので、新しいファイルフォーマットに含めることによる欠点はごくわずかです。含めることによって、次のような有効な特性も得られます：

<!--
- Immediate support for _all_ features of the runtime (at least for files that include complete CLI within them)
- The option to only add the 'most important' data required to support fast, direct execution. Everything else can be left in CLI format and use the CLI code paths. This is quite valuable given our desire to be minimalist in augmenting the format.
-->
- （少なくとも、完全なCLIが含まれているファイルについては、）CLRの_全ての_機能をすぐにサポートできる。
- 高速な、直接実行をサポートするために必要な「最重要」データのみを追加するという選択肢。それ以外は全てCLIフォーマットのままにして、CLIのコードパスを使用するようにできる。これは、フォーマットの増補に対してミニマリストであろうという思いにおいて非常に価値のあることです。
 
<!--
Moreover there is an 'obvious' way of extending the CIL file to include the additional data we need. A CLI file has a well-defined header structure, and that header already has a field that can point of to 'additional information'. This is used today in NGEN images. We would use this same technique to allow the existing CLI format to include a new 'Native Header' that would then point at any additional information needed to support fast, direct execution. 
-->
さらに、我々が必要としている追加のデータを含めるようCLIファイルを拡張する「明らかな」方法があります。CLIファイルには既知の定義済みヘッダー構造があり、そのヘッダーには「追加情報」を指し示すことができるフィールドが既にあります。このフィールドは現在NGENイメージで使われています。我々は、高速な、直接実行をサポートするのに必要な一切の追加情報を指し示す、新しい「ネイティブヘッダー」を既存のCLIフォーマットに含めるのに、NGENと同じテクニックを使用することにしました。

<!--
The most important parts of this extra information include:
-->
この追加情報の最も重要な部分は次のとおりです：

<!--
1. Native code for the methods (as well as a way of referencing things outside the module)
2. Garbage Collection (GC) information for each method that allows you to know what values in registers and on the stack are pointers to the GC heap wherever a GC is allowed. 
3. Exception handling (EH) tables that allow an exception handler to be found when an exception is thrown. 
4. A table that allows the GC and EH to be found given just the current instruction pointer (IP) within the code. (IP map).
5. A table that links the information in the metadata to the corresponding native structure. 
-->
1. メソッドのネイティブコード（モジュールの外部にあるものを参照する方法を含む）
2. GCが実行可能なあらゆる地点において、レジスタとスタック上の値がGCヒープへのポインターかどうかわかるようにするための、各メソッドについてのガベージコレクション（GC：Garbage Collection）情報。
3. 例外がスローされたときに例外ハンドラーを見つけられるようにするための例外処理（EH：Exception Handling）テーブル。
4. コード内の命令ポインタ（IP：Instruction Pointer）のみでGCとEHを見つけられるようにするためのテーブル（IPマップ）。
5. メタデータ内の情報から対応するネイティブ構造体をリンクするテーブル。

<!--
That is, we need something to link the world of metadata to the world of native. We can't eliminate meta-data completely because we want to support existing functionality. In particular we need to be able to support having other CLI images refer to types, methods and fields in this image. They will do so by referencing the information in the metadata, but once they find the target in the metadata, we will need to find the actual native code or type information corresponding to that meta-data entry. This is the purpose of the additional table. Effectively, this table is the 'export' mechanism for managed references.
-->
つまり、メタデータの世界からネイティブの世界への何らかのリンクが必要なのです。既存の機能をサポートしたいので、メタデータを完全に切り捨てることはできません。特に、他のイメージからこのイメージにある型、メソッド、フィールドへの参照をサポートできる必要があります。これはメタデータにある情報を参照することで行いますが、一度メタデータ内で対象を見つけたならば、そのメタデータエントリに対応する実際のネイティブコードや型情報を必要とします。これが、追加のテーブルの存在理由です。事実上、このテーブルはマネージド参照に対する「エクスポート」の仕組みです。

<!--
Some of this information can be omitted or stored in more efficient form, e.g.:
-->
次に示すように、この情報のいくつかは省略するか、またはより効率的な形式で保存できます：

<!--
- The garbage collection information can be omitted for environments with conservative garbage collection, such as IL2CPP.
- The full metadata information is not strictly required for 'private' methods or types so it is possible to strip it from the CLI image.
- The metadata can be stored in more efficient form, such as the .NET Native metadata format.
- The platform native executable format (ELF, Mach-O) can be used as envelope instead of PE to take advantage of platform OS loader.
-->
- ガベージコレクション情報は、IL2CPPのような保守的なガベージコレクションを持つ環境では省略できる。
- CLIイメージから取り除くことができる「private」メソッドや型について、完全なメタデータ情報が厳密には要らない。
- メタデータは、.NET Nativeメタデータフォーマットのようなより効率的な形式で保存できる。
- プラットフォームとなるOSのローダーを活用するために、プラットフォームネイティブな実行可能形式（ELFやMach-O）をエンベロープとして使用できる。

<!--
## Definition of Version Compatibility for Native Code
-->
## ネイティブコードにおけるバージョン互換性の定義

<!--
Even for IL or unmanaged native code, there are limits to what compatible changes can be made. For example, deleting a public method is sure to be an incompatible change for any extern code using that method.
-->
ILやアンマネージドネイティブコードについて考えても、互換性のある変更を実施できるケースには限界があります。たとえば、publicなメソッドの削除は、そのメソッドを使用する外部コードに対する互換性のない変更であることは間違いありません。
 
<!--
Since CIL already has a set of [compatibility rules](https://github.com/dotnet/corefx/blob/master/Documentation/coding-guidelines/breaking-changes.md), ideally the native format would have the same set of compatibility rules as CIL. Unfortunately, that is difficult to do efficiently in all cases. In those cases we have multiple choices:
-->
CILには既に [互換性規則(https://github.com/dotnet/corefx/blob/master/Documentation/coding-guidelines/breaking-changes.md)] のセットがあるので、理想的には、このネイティブフォーマットはCILと同じ互換性規則のセットを持つことになります。残念ながら、全てのケースにおいてこれを効率的に行うことは困難です。それらのケースについては、複数の選択肢があります。

<!--
1. Change the compatibility rules to disallow some changes
2. Never generate native structures for the problematic cases (fall back to CIL techniques)
3. Generate native structures for the problematic cases, but use them only if there was no incompatible change made
4. Generate less efficient native code that is resilient
-->
1. いくつかの変更を許可しないよう互換性規則を変更する。
2. 問題のあるケースについてネイティブ構造体の生成を行わない（CILのテクニックにフォールバックする）。
3. 問題のあるケースについてネイティブ構造体を生成するが、非互換の変更が行われない場合にのみ使用する。
4. バージョン耐性があるがあまり効率的でないネイティブコードを生成する。
 
<!--
Generally the hardest versioning issues revolve around:
-->
一般的に、最も難しいバージョン管理問題は次に関するものです：

<!--
- Value types (structs)
- Generic methods over value types (structs)
-->
- 値型（構造体）
- 値型（構造体）に対するジェネリックメソッド

<!--
These are problematic because value classes are valuable precisely _because_ they have less overhead than classes. They achieve this value by being 'inlined' where they are used. This makes the code generated for value classes very fragile with respect to any changes to the value class's layout, which is bad for resilience. Generics over structs have a similar issue.
-->
値型はクラスよりもオーバーヘッドが少ないが_ゆえに_、明らかに有益であると言えるため、これらが問題となるのです。この有益さは、使用されている場所で値が「インライン化」されるために達成されるものです。これによって、値型のために生成されるコードは、値型のレイアウトへの一切の変更に対して非常に脆いものとなり、バージョン耐性については良くないことです。構造体に対するジェネリクスにも同様の問題があります。

<!--
Thus this proposal does _not_ suggest that we try to solve the problem of having version resilience in the presence of layout changes to value types. Instead we suggest creating a new compatibility rule:
-->
そのため、この提案は値型のレイアウト変更がある場合におけるバージョン耐性（version resilience）の問題を解決しようとするための提案を_しません_。その代わりに、我々は新しい互換性規則を作成することを提案します。

<!--
**It is a breaking change to change the number or type of any (including private) fields of a public value type (struct). However if the struct is non-public (that is internal), and not reachable from any nesting of value type fields in any public value type, then the restriction does not apply.**
-->
**publicな値型（構造体）のフィールド（publicでないものを含む）の数または型の変更は、すべて互換性のない変更である。ただし、その構造体が非public（つまりinternal）であり、publicな値型にある値型のフィールドのネスト構造から到達不能な場合、この制約は適用されない。**
 
<!--
This is a compatibility that is not present for CIL. All other changes allowed by CIL can be allowed by native code without prohibitive penalty. In particular the following changes are allowed:
-->
これはCILには存在しない互換性規則です。CILで許可される他の全ての変更は、法外なペナルティを伴うことなく、ネイティブコードでも許可することができます。特に、次の変更は問題ありません：

<!--
1. Adding instance and static fields to reference classes
2. Adding static fields to a value class. 
3. Adding virtual, instance or static methods to a reference or value class
4. Changing existing methods (assuming the semantics is compatible).
5. Adding new classes.
-->
1. 参照型に対するインスタンスまたは静的フィールドの追加。
2. 値型に対する静的フィールドの追加。
3. 参照型または値型に対する仮想、インスタンス、または静的メソッドの追加。
4. 既存のメソッドの変更（前提として、セマンティクスに互換性がある限り）。
5. 新しい型の追加。

<!--
## Version Bubbles
-->
## バージョンバブル

<!--
When changes to managed code are made, we have to make sure that all the artifacts in a native code image _only_ depend on information in other modules that _cannot_ _change_ without breaking the compatibility rules. What is interesting about this problem is that the constraints only come into play when you _cross_ module boundaries. 
-->
マネージドコードに対して変更が行われるときには、ネイティブコードイメージ内の全ての成果物が、他のモジュール内にある、互換性規則を破ることなく_変更できない_情報に_のみ_依存するようにしなければなりません。この問題に対して興味深い点は、モジュール境界を_越える_場合にのみこの制約が当てはまるということです。
<!--
As an example, consider the issue of inlining of method bodies. If module A would inline a method from Module B, that would break our desired versioning property because now if that method in module B changes, there is code in Module A that would need to be updated (which we do not wish to do). Thus inlining is illegal across modules. Inlining _within_ a module, however, is still perfectly fine. 
-->
たとえば、メソッド本体のインライン化に関する問題を考えます。モジュールAがモジュールBにあるメソッドをインライン化する場合、我々が望ましいと考えているバージョン管理特性を破壊することになります。この場合、モジュールBのメソッドが変更されると、モジュールAの中にあるコードも更新しなければなりません（これは望ましくありません）。したがって、モジュールを超えたインライン化は不正です。ただし、モジュール_内_でのインライン化は全く問題ありません。

<!--
Thus in general the performance impact of versioning decreases as module size increases because there are fewer cross-module references. We can take advantage of this observation by defining something called a version bubble. **A version bubble is a set of DLLs that we are willing to update as a set.** From a versioning perspective, this set of DLLs is a single module. Inlining and other cross-module optimizations are allowed within a version bubble. 
-->
したがって、一般的に言うと、モジュール越えの参照が少なくなるため、モジュールのサイズが大きくなるにつれて、バージョン管理のパフォーマンスへの悪影響は軽くなります。我々は、バージョンバブル（version bubble）という考え方を定義することで、この観察結果を活用できます。**バージョンバブルは、1つのセットとして更新しようとするだろうDLLのセットです。** バージョン管理の見地からは、このDLLセットが単一のモジュールです。インライン化やその他のモジュール越えの最適化は、バージョンバブル内では許可されます。

<!--
It is worth reiterating the general principle covered in this section
-->
このセクションで触れる一般的な原則を振り返ります。

<!--
**Code of methods and types that do NOT span version bubbles does NOT pay a performance penalty.**
-->
**バージョンバブルを越え_ない_メソッドや型のコードはパフォーマンスペナルティを払わ_ない_。**

<!--
This principle is important because it means that only a fraction (for most apps a small fraction) of all code will pay any performance penalties we discuss in the sections that follow.
-->
コード全体のごく一部（ほとんどのアプリでは本当にごく一部）のみが、この後に述べる何らかのパフォーマンスペナルティを払うということなので、この原則は重要です。

<!--
The extreme case is where the entire application is a single version bubble. This configuration does not need to pay any performance penalty for respecting versioning rules. It still benefits from a clearly defined file format and runtime contract that are the essential part of this proposal.
-->
極端なケースでは、アプリケーション全体が単一のバージョンバブルになります。この構成では、バージョン管理規則に従うことによるパフォーマンスペナルティを払う必要は一切ありません。依然として、このアプリは明確に定義されたファイルフォーマットと、この提案の基本部分をなす実行時コントラクトのメリットを享受できます。

<!--
## Runtime Versioning
-->
## ランタイムのバージョン管理

<!--
The runtime versioning is solved using different techniques because the runtime is responsible for interpretation of the binary format.
-->
ランタイムはバイナリフォーマットを解釈する役目を持つため、ランタイムのバージョン管理は異なるテクニックを用いて解決されます。

<!--
To allow changes in the runtime, we simply require that the new runtime handle all old formats as well as the new format. The 'main defense' in the design of the file format is having version numbers on important structures so that the runtime has the option of supporting a new version of that structure as well as the old version unambiguously by checking the version number. Fundamentally, we are forcing the developers of the runtime to be aware of this constraint and code and test accordingly.
-->
ランタイムに対する変更を可能にするため、我々は、新しいランタイムが新しいフォーマットだけでなく全ての旧式フォーマットを処理することを要求します。ファイルフォーマットの設計における「メインディフェンス」は、重要な構造体に対してバージョン番号を持たせることにより、その番号をチェックすることで、構造体の新しいバージョンだけでなく古いバージョンについても、ランタイムが正確にサポートする選択肢を持つようにすることです。根本的に、我々は、ランタイムの開発者に対して、この制約を認識し、適切にコーディングおよびテストすることを強制します。
 
<!--
### Restrictions on Runtime Evolution
-->
### ランタイムの変化に関する制約

<!--
As mentioned previously, when designing for version compatibility we have the choice of either simply disallowing a change (by changing the breaking change rules), or insuring that the format is sufficiently flexible to allow evolution. For example, for managed code we have opted to disallow changes to value type (struct) layout so that codegen for structs can be efficient. In addition, the design also includes a small number of restrictions that affect the flexibility of evolving the runtime itself. They are:
-->
前述のように、バージョン互換性を設計するときに、我々には、（互換性のない変更の規則を変更することで）単に変更を許可しないか、評価できるようにするために十分な柔軟性をフォーマットに保証させるかという選択肢があります。たとえば、マネージドコードについて、値型（構造体）のレイアウトに対する変更を許可しないことを選択して、構造体に対するコード生成を効率的にしました。さらに、この設計には、ランタイム自身の進化の柔軟性に影響を与える、いくつかの制約も含まれています：

<!--
- The field layout of `System.Object` cannot change. (First, there is a pointer sized field for type information and then the other fields.)
- The field layout of arrays cannot change. (First, there is a pointer sized field for type information, and then a pointer sized field for the length. After these fields is the array data, packed using existing alignment rules.)
- The field layout of `System.String` cannot change. (First, there is a pointer sized field for type information, and then a int32 sized field for the length. After these fields is the zero terminated string data in UTF16 encoding.)
-->
- `System.Object` のフィールドレイアウトは変更できない。（1番目は型情報用のポインターサイズのフィールドで、その後に他のフィールドが続く）
- 配列のフィールドレイアウトは変更できない。（1番目は型情報用のポインターサイズのフィールドで、その後に長さ用のポインターサイズのフィールドが続く。これらのフィールドの後ろが配列のデータであり、既存のアライメント規則を使用してパッキングされる）
- `System.String` のフィールドレイアウトは変更できない。（1番目は型情報用のポインターサイズのフィールドで、その後に長さ用のint32サイズのフィールドが続く。これらのフィールドの後はUTF16で符号化されたゼロ終端の文字列データである）

<!--
These restrictions were made because the likelihood of ever wanting to change these restrictions is low, and the performance cost _not_ having these assumptions is high. If we did not assume the field layout of `System.Object` never changes, then _every_ field fetch object outside the framework itself would span a version bubble and pay a penalty. Similarly if we don't assume the field layout for arrays or strings, then every access will pay a versioning penalty.
-->
これらの制約は、これらの制約を変更したいと思う可能性が低く、これらの前提が_ない_ことによるパフォーマンスコストが高いために作られました。`System.Object` のフィールドレイアウトが決して変更されないことを前提にできないとすると、フレームワーク自身以外の、フィールドをフェッチする_全ての_オブジェクトがバージョンバブルをまたぐことになり、ペナルティを支払うことになります。同様に、配列や文字列のフィールドレイアウトを前提にできないとすると、これらへの全てのアクセスがバージョン管理ペナルティを支払うことになります。

<!--
## Selective use of the JIT
-->
## JITの選択的な使用

<!--
One final point that is worth making is that selective use of the JIT compiler is another tool that can be used to avoid code quality penalties associated with version resilience, in environments where JITing is permitted. For example, assume that there is a hot user method that calls across a version bubble to a method that would a good candidate for inlining, but is not inlined because of versioning constraints. For such cases, we could have an attribute that indicates that a particular method should be compiled at runtime. Since the JIT compiler is free to generate fragile code, it can perform this inlining and thus the program steady-state performance improves. It is true that a startup time cost has been paid, but if the number of such 'hot' methods is small, the amount of JIT compilation (and thus its penalty) is not great. The point is that application developers can make this determination on a case by case basis. It is very easy for the runtime to support this capability.
-->
注目すべき最後の点は、JITコンパイルが許されている環境において、選択的なJITコンパイラの使用が、バージョン耐性に関連するコード品質ペナルティを回避するために使用可能なもう一つのツールであるということです。たとえば、よく使われるメソッドがバージョンバブルを超えて別のメソッドを呼び出すことを考えると、そのメソッドはインライン化の有力候補となりますが、バージョン管理の制約のためにそのメソッド呼び出しはインライン化されません。そのようなケースでは、特定のメソッドが実行時にコンパイルされるべきことを示す属性を付けることができます。JITコンパイラが脆いコードを生成するのは自由なので、この種のインライン化を行って、プログラムの安定状態におけるパフォーマンスを向上できます。起動時間のコストが必要になることは間違いありませんが、そのような「よく使われる」メソッドの数が少なければ、JITコンパイルの量（とそのペナルティ）は大したことになりません。ポイントは、アプリケーション開発者は、ケースバイケースでこの判断を行えるということです。この機能をランタイムがサポートさせるのはとても簡単です。

<!--
# Version Resilient Native Code Generation
-->
# バージョン耐性のあるネイティブコード生成

<!--
Because our new native format starts with the current CLI format, we have the option of falling back to it whenever we wish to. Thus we can choose to add new parts to the format in chunks. In this section we talk about the 'native code' chunk. Here we discuss the parts of the format needed to emit native code for the bodies of 'ordinary' methods. Native images that have this addition information will not need to call the JIT compiler, but will still need to call the type loader to create types.
-->
新しいネイティブフォーマットは現在のCLIフォーマットを発展させるので、そうしたいときにはいつでもフォールバックするという選択肢があります。そのため、新しいフォーマットを、いくつかの塊に分けた新しいパーツとして追加することができます。このセクションでは、「ネイティブコード」チャンクについて説明します。ここでは、「通常の」メソッドの本体用のネイティブコードを生成するのに必要なフォーマットパーツを説明します。この追加情報を持つネイティブイメージはJITコンパイラを呼び出す必要はありませんが、依然として型を作成するためにタイプローダーを呼び出す必要があるでしょう。

<!--
It is useful to break the problem of generating version resilient native code by CIL instruction. Many CIL instructions (e.g. `ADD`, `MUL`, `LDLOC` ... naturally translate to native code in a version resilient ways. However CIL that deals with object model (e.g. `NEWOBJ`, `LDFLD`, etc) need special care as explained below. The descriptions below are roughly ordered in the performance priority in typical applications. Typically, each section will describe what code generation looks like when all information is within the version bubble, and then when the information crosses version bubbles. We use x64 as our native instruction set, applying the same strategy to other processor architectures is straightforward. We use the following trivial example to demonstrate the concepts
-->
CIL命令からのバージョン耐性のあるネイティブコード生成についての問題を分解するとわかりやすくなります。多くのCIL命令（`add`、`mul`、`ldloc`など）は、バージョン耐性のある方法で自然にネイティブコードに変換されます。ただし、オブジェクトモデルを扱うCIL（`newobj`、`ldfld`、など）は、後述するような特別な対処が必要です。以降の説明は、典型的なアプリケーションにおけるパフォーマンスペナルティの順で大雑把に並べています。一般的に、各セクションは、全ての情報があるバージョンバブル内に収まっている場合にコード生成がどのように動作するのかについて述べ、それから情報がバージョンバブルをまたぐ場合について述べています。ネイティブ命令セットにはx64を使用しており、別のプロセッサーアーキテクチャに同じ戦略を用いることは簡単です。コンセプトを説明するために、以下に示すちょっとした例を使用します。

```C#
    interface Intf
    {
        void intfMethod();
    }
    
    class BaseClass
    {
        static int sField;
        int iField;

        public void iMethod()
        {
        }

        public virtual void vMethod(BaseClass aC)
        {
        }
    }
    
    class SubClass : BaseClass, Intf
    {
        int subField;
    
        public override void vMethod(BaseClass aC)
        {
        }
    
        virtual void intfMethod()
        {
        }
    }
```

<!--
## Instance Field access - LDFLD / STFLD
-->
## インスタンスフィールドアクセス（ldfldとstfld）

<!--
The CLR stores fields in the 'standard' way, so if RCX holds a BaseClass then
-->
CLRはフィールドを「標準的な」方法で保存するので、RCXが`BaseClass`を保持する場合、

```Assembly
        MOV RAX, [RCX + iField_Offset]
```

<!--
will fetch `iField` from this object. `iField_Offset` is a constant known at native code generation time. This is known at compile time only because we mandated that the field layout of `System.Object` is fixed, and thus the entire inheritance chain of `BaseClass` is in the version bubble. It's also true even when fields in `BaseClass` contain structs (even from outside the version bubble), because we have made it a breaking change to modify the field layout of any public value type. Thus for types whose inheritance hierarchy does not span a version bubble, field fetch is as it always was.
-->
はこのオブジェクトから`iField`をフェッチします。`iField_Offset`はネイティブコードの生成時に既知の定数です。`System.Object`のフィールドレイアウトが固定されていることを必須要件としており、かつ`BaseClass`の継承の連鎖全体がバージョンバブル内にとどまるため、コンパイル時に既知となるのです。これは、`BaseClass`内のフィールドに構造体が含まれていても（それがバージョンバブル外の構造体であっても）当てはまります。あらゆるpublicな値型のフィールドレイアウトの変更は互換性のない変更としているためです。

<!--
To consider the inter-bubble case, assume that `SubClass` is defined in a different version bubble than BaseClass and we are fetching `subField`. The normal layout rules for classes require `subField` to come after all the fields of `BaseClass`. However `BaseClass` could change over time, so we can't wire in a literal constant anymore. Instead we require the following code
-->
バブル間のケースを考えるために、`SubClass`が`BaseClass`とは異なるバージョンバブルで定義されており、`subField`をフェッチするものとします。クラスの通常のレイアウト規則では、`subField`は`BaseClass`の全てのフィールドの後に来ることを要求します。ところが、`BaseClass`はそのうち変わる可能性があるので、もはやリテラル定数で結ぶことはできません。その代わりに、次のコードが必要となります。

```Assembly
        MOV TMP, [SIZE_OF_BASECLASS]
        MOV EAX, [RCX + TMP + subfield_OffsetInSubClass]

    .data // dataセクション内
        SIZE_OF_BASECLASS: UINT32 // 派生クラスが作成され得るEXTERN CLASSごとに1つ
```
 
<!--
Which simply assumes that a uint32 sized location has been reserved in the module and that it will be filled in with the size of `BaseClass` before this code is executed. Now a field fetch has one extra instruction, which fetches this size and that dynamic value is used to compute the field. This sequence is a great candidate for CSE (common sub-expression elimination) optimization when multiple fields of the same class are accessed by single method.
-->
これは、モジュール内にuint32サイズの場所が予約され、かつコードの実行前にその領域に`BaseClass`のサイズが書き込まれることを前提にしています。ここで、フィールドのフェッチは1つの追加命令を持ち、その命令はこのサイズをフェッチし、フィールドの位置の計算にその動的な値が使用されます。このシーケンスは、単一のメソッド内で同じクラスのフィールドに複数回アクセスする場合に、CSE（Common Sub-expression Elimination：共通部分式除去）最適化の有力候補になります。

<!--
A special attention needs to be given to alignment requirements of `SubClass`.
-->
`SubClass`のアライメント要件には特別な注意が必要になります。

<!--
### GC Write Barrier
-->
### GC書き込みバリア

<!--
The .NET GC is generational, which means that most GCs do not collect the whole heap, and instead only collect the 'new' part (which is much more likely to contain garbage). To do this it needs to know the set of roots that point into this 'new' part. This is what the GC write barrier does. Every time an object reference that lives in the GC heap is updated, bookkeeping code needs to be called to log that fact. Any fields whose values were updated are used as potential roots on these partial GCs. The important part here is that any field update of a GC reference must do this extra bookkeeping. 
-->
.NETのGCは世代別GC、つまりはほとんどのGCはヒープ全体を回収せず、（ごみを含む可能性が最も高い）「新しい」世代のみを回収します。このために、ルートのセットが「新しい」世代を指しているかどうかわかるようにする必要があります。これこそ、GC書き込みバリア（GC write barrier）が行うことです。GCヒープ内で生存するオブジェクト参照が更新されるたびに、その事実を記録するためにブックキーピングコードが呼び出されなければなりません。値が更新されたあらゆるフィールドは、これらの部分GCにおける潜在的なルートとして使用されます。ここで重要なことは、GC参照を持つあらゆるフィールドの更新では、この追加のブックキーピングを行わなければならないということです。

<!--
The write barrier is implemented as a set of helper functions in the runtime. These functions have special calling conventions (they do not trash any registers). Thus these helpers act more like instructions than calls. The write barrier logic does not need to be changed to support versioning (it works fine the way it is).
-->
書き込みバリアはランタイム内のヘルパー関数のセットとして実装されています。これらの関数には特別な呼び出し規約があります（これらはレジスタを一切壊しません）。そのため、これらのヘルパーは呼び出し（call）というよりも命令（instruction）のように振る舞います。書き込みバリアロジックをバージョン管理をサポートするために変更する必要はありません（そのままで動く動きます）。

<!--
### Initializing the field size information
-->
### フィールドサイズ情報の初期化

<!--
A key observation is that you only need this overhead for each distinct class that inherits across a version bubble. Thus there is unlikely to be many slots like `SIZE_OF_BASECLASS`. Because there are likely to be few of them, the compiler can choose to simply initialize them at module load.
-->
ポイントは、このオーバーヘッドは、バージョンバブルをまたがって継承するクラスに対してのみ必要となるということです。つまり、`SIZE_OF_BASECLASS`のようなスロットが多数生じる可能性はほとんどありません。数が少なくなるであろうことから、コンパイラは単純にモジュールのロード時に初期化することを選べます。

<!--
Note that if you accessed an instance field of a class that was defined in another module, it is not the size that you need but the offset of a particular field. The code generated will be the same (in fact it will be simpler as no displacement is needed in the second instruction). Our coding guidelines strongly discourage public instance fields so this scenario is not particularly likely in practice (it will end up being a property call) but we can handle it in a natural way. Note also that even complex inheritance hierarchies that span multiple version bubbles are not a problem. In the end all you need is the final size of the base type. It might take a bit longer to compute during one time initialization, but that is the extent of the extra cost.
-->
別のモジュールに定義されたクラスのインスタンスフィールドにアクセスする場合、必要なのはサイズではなくそのフィールドのオフセットです。生成されるコードは同一です（実際のところ、2番目の命令でdisplacementが不要になるので、より単純になります）。我々のコーディングガイドラインではpublicなインスタンスフィールドは強く非推奨としているため、このシナリオは実際にはほとんどないでしょう（最終的にプロパティ呼び出しになるでしょう）が、我々はこれを自然な方法で処理できます。複数のバージョンバブルをまたがる複雑な継承階層であっても問題にはならないことに注意してください。結局、必要なものは基底型のサイズのみです。一度限りの初期化の計算時間はほんの少し長くなるかもしれませんが、追加コストの範囲内です。

<!--
### Performance Impact
-->
### パフォーマンスへの影響

<!--
Clearly we have added an instruction and thus made the code bigger and more expensive to run. However what is also true is that the additional cost is small. The 'worst' case would be if this field fetch was in a tight loop. To measure this we created a linked list element which inherited across a version bubble. The list was long (1K) but small enough to fit in the L1 cache. Even for this extreme example (which by the way is contrived, linked list nodes do not normally inherit in such a way), the extra cost was small (< 1%).
-->
明らかに、1命令足したため、コードは大きくなり、実行コストも上がっています。ただし、追加のコストが小さいことも確かです。「ワースト」ケースは、小さなループ内でこの種類のフィールドフェッチが行われる場合です。これを計測するために、バージョンバブルを超えて継承されるリンクリスト要素を作成しました。このリストは長い（1K）ものでありつつ、L1キャッシュに収まるサイズです。この極端な例（いずれにせよわざとらしいもので、リンクリストは通常このように継承しません）でさえ、追加のコストは小さなもの（1%未満）でした。

<!--
### Null checks 
-->
### nullチェック

<!--
The managed runtime requires any field access on null instance pointer to generate null reference exception. To avoid inserting explicit null checks, the code generator assumes that memory access at addresses smaller than certain threshold (64k on Windows NT) will generate null reference exception. If we allowed unlimited growth of the base class for cross-version bubble inheritance hierarchies, this optimization would be no longer possible.
-->
マネージドランタイムはnullインスタンスポインターへのあらゆるフィールドアクセスがnull参照例外を生じるようにしなければなりません。明示的なnullチェックを挿入せずに済むように、コードジェネレーターは特定のしきい値（Windows NTでは64k）よりも小さいアドレスへのメモリアクセスがnull参照例外を生じるものと仮定します。バージョンバブル越えの継承階層について、基底クラスの増大を無制限に認めると、この最適化が使えなくなります。

<!--
To make this optimization possible, we will limit growth of the base class size for cross-module inheritance hierarchies. It is a new versioning restriction that does not exist in IL today.
-->
この最適化を使用可能にするために、モジュール越え継承階層における基底クラスのサイズの増大を制限します。これは、現在のILには存在しない新しいバージョン管理上の制約です。

<!--
## Non-Virtual Method Calls - CALL
-->
## 非仮想メソッド呼び出し（call）

<!--
### Intra-module call
-->
### モジュール内呼び出し
 
<!--
If RCX holds a `BaseClass` and the caller of `iMethod` is in the same module as BaseClass then a method call is simple machine call instruction
-->
RCXが`BaseClass`を保持し、`iMethod`の呼び出し元が`BaseClass`と同一のモジュールである場合、メソッド呼び出しは単純なcall機械語命令です。

```Assembly
        CALL ENTRY_IMETHOD
```

<!--
### Inter-module call
-->
### モジュール間呼び出し
 
<!--
However if the caller is outside the module of BaseClass (even if it is in the same version bubble) we need to call it using an indirection
-->
ところが、呼び出し元が`BaseClass`のモジュールの外にある場合、（同一のバージョンバブル内であったとしても）間接呼び出しを使用しなければなりません。

```Assembly
        CALL [PTR_IMETHOD]

    .data // dataセクション内
        PTR_IMETHOD: PTR = RUNTIME_ENTRY_FIXUP_METHOD // TARGET呼び出しごとに1つ。 
```

<!--
Just like the field case, the pointer sized data slot `PTR_IMETHOD` must be fixed up to point at the entry point of `BaseClass.iMethod`. However unlike the field case, because we are fixing up a call (and not a MOV), we can have the call fix itself up lazily via standard delay loading mechanism.
The delay loading mechanism often uses low-level tricks for maximum efficiency. Any low-level implementation of delay loading can be used as long as the resolution of the call target is left to the runtime.
-->
まさにフィールドのケースのように、ポインターサイズの`PTR_IMETHOD`というデータスロットが、`BaseClass.iMethod`のエントリポイントを指し示すように修正されなければなりません。ただし、フィールドのケースとは異なり、呼び出しを修正しており（さらに`MOV`がなく）、標準的な遅延読み込みの仕組みによって呼び出しを遅延修正できます。
遅延読み込みの仕組みは、効率の最大化ｊのために、しばしば低レベルのトリックを使用します。呼び出し先の解決に使用できる限り、遅延読み込みについては、任意の低レベル実装余地がランタイムに残されています。

<!--
### Retained Flexibility for runtime innovation
-->
### ランタイムの刷新のための柔軟性確保

<!--
Note that it might seem that we have forever removed the possibility of innovating in the way we do SLOT fixup, since we 'burn' these details into the code generation and runtime helpers. However this is not true. What we have done is require that we support the _current_ mechanism for doing such fixup. Thus we must always support a `RUNTIME_ENTRY_FIXUP_METHOD` helper. However we could devise a completely different scheme. All that would be required is that you use a _new_ helper and _keep_ the old one. Thus you can have a mix of old and new native code in the same process without issue. 
-->
注意点として、これらの詳細事項をコード生成とランタイムヘルパーの中に「焼き付けて」いるので、SLOT修正を行う方法についての刷新の可能性を永遠になくしたように見えるかもしれません。ところが、そうではありません。ここで行ったことは、そのような修正を行うために_現在の_仕組みをサポートするのに必要なことです。したがって、私たちは常に`RUNTIME_ENTRY_FIXUP_METHOD`ヘルパーをサポートしなければなりません。ただし、全く異なるやり方を考案することができます。必要となるであろうことは、_新しい_ヘルパーを使用しつつ、古い仕組みを_そのままにする_ことだけです。それによって、同一プロセス内に、古いネイティブコードと新しいネイティブコードを問題なく混在させることができます。

<!--
### Calling Convention
-->
### 呼び出し規約

<!--
The examples above did not have arguments and the issue of calling convention was not obvious. However it is certainly true that the native code at the call site does depend heavily on the calling convention and that convention must be agreed to between the caller and the callee at least for any particular caller-callee pair. 
-->
前述の例は引数を持たず、呼び出し規約の問題が明らかではありません。ただし、呼び出し元のネイティブコードが呼び出し規約に大いに依存しており、その規約は呼び出し元と呼び出し先の間、少なくとも任意の特定の呼び出し元と呼び出し先の間で、合意されていなければなりません。

<!--
The issue of calling convention is not specific to managed code and thus hardware manufacturers typically define a calling convention that tends to be used by all languages on the system (thus allowing interoperability). In fact for all platforms except x86, CLR attempts to follow the platform calling convention.
-->
呼び出し規約の問題は、マネージドコード固有の問題ではないため、一般的にはハードウェア製造者がそのシステムでの全ての言語で使用される傾向にある呼び出し規約を定義します（したがって、相互運用可能です）。実際に、x86を除く全てのプラットフォームにおいて、CLRはプラットフォームの呼び出し規約に従おうとします。

<!--
Our understanding of the most appropriate managed convention evolved over time. Our experience tells us that it is worthwhile for implementation simplicity to always pass managed `this` pointer in the fixed register, even if the platform standard calling convention says otherwise.
-->
最も適切なマネージドの規約についての我々の理解は徐々に進化しました。我々の経験から、たとえプラットフォームの標準的な呼び出し規約が他の方法をとれと言っていたとしても、実装を単純にするために、マネージド`this`ポインターを常に固定のレジスタで渡すことに価値があります。

<!--
#### Managed Code Specific Conventions
-->
#### マネージドコード固有の呼び出し規約

<!--
In addition the normal conventions for passing parameters as well as the normal convention of having a hidden byref parameter for returning value types, CLR has a few managed code specific argument conventions:
-->
パラメーターの渡し方についての標準規約と値型を返すための隠しbyrefパラメーターを持つことについての標準規約に加え、CLRにはいくつかマネージドコード固有の引数規約があります：
 
<!--
1. Shared generic code has a hidden parameter that represents the type parameters in some cases for methods on generic types and for generic methods.
2. GC interactions with hidden return buffer. The convention for whether the hidden return buffer can be allocated in the GC heap, and thus needs to be written to using write barrier.
-->
1. 共有されたジェネリックコードは、ジェネリック型のメソッドとジェネリックメソッドのために、いくつかのケースにおいて型パラメーターを表す隠しパラメーターを持つ。
2. 隠しリターンバッファーを伴うGCインタラクション。隠しリターンバッファをGCヒープに割り当て可能かどうか、それによって書き込みバリアを使用した書き込みが必要かどうかについての規約。

<!--
These conventions would be codified as well.
-->
これらの規約も文書化されることでしょう。

<!--
### Performance Impact
-->
### パフォーマンスへの影響

<!--
Because it was already the case that methods outside the current module had to use an indirect call, versionability does not introduce more overhead for non-virtual method calls if inlining was not done. Thus the main cost of  making the native code version resilient is the requirement that no cross version bubble inlining can happen.
-->
現在のモジュールの外にあるメソッドのケースでは既に間接呼び出しを行わなければならなかったので、バージョン管理を可能とすることで、インライン化が行われない場合の非仮想メソッド呼び出しについて、追加のオーバーヘッドはありません。そのため、ネイティブコードのバージョン耐性のための主要なコストは、バージョンバブルを越えたインライン化が起こらないという要件です。
 
<!--
The best solution to this problem is to avoid 'chatty' library designs (Unfortunately, `IEnumerable`, is such a chatty design, where each iteration does a `MoveNext` and `Current` property fetch). Another mitigation is the one mentioned previously: to allow clients of the library to selectively JIT compile some methods that make these chatty calls. Finally you can also use new custom `NonVersionableAttribute` attribute, which effectively changes the versioning contract to indicate that the library supplier has given up his right to change that method's body and thus it would be legal to inline.
-->
この問題に対する最良の解決策は、「チャッティ」なライブラリ設計を避けることです（残念ながら、`IEnumerable`はそのようなチャッティ設計になっていて、イテレーションごとに`MoveNext`メソッドと`Current`プロパティのフェッチを行います）。別の緩和策は、以前に述べたものです。すなわち、このようなチャッティな呼び出しを行うメソッドについて、そのライブラリの使用者が選択的にJITコンパイルできるようにすることです。最終的に、新しいカスタム属性である`NonVersionableAttribute`を使用することもでき、ライブラリ供給側がメソッドの本体を変更する権利をあきらめることでインライン化を正当化することを示すように、バージョン管理のコントラクトを実質的に変更します。

<!--
The proposal is to disallow cross-version bubble inlining by default, and selectively allow inlining for critical methods (by giving up the right to change the method).
-->
提案は、既定ではバージョンバブルを越えたインライン化を許可せず、重要なメソッドについて（メソッドの変更権をあきらめることで）インライン化を選択的に許可するというものです。

<!--
Experiments with disabled cross-module inlining with the selectively enabled inlining of critical methods showed no visible regression in ASP.NET throughput.
-->
重要なメソッドのインライン化を選択的に有効化しつつモジュール越えインライン化を無効化する実験では、ASP.NETのスループットで観測可能な退行は見られませんでした。

<!--
## Non-Virtual calls as the baseline solution to all other versioning issues
-->
## 他の全てのバージョン管理問題に対する解決策のベースラインとしての非仮想呼び出し

<!--
It is important to observe that once you have a mechanism for doing non-virtual function calls in a version resilient way (by having an indirect CALL through a slot that that can be fixed lazily at runtime, all other versioning problems _can_ be solved in that way by calling back to the 'definer' module, and having the operation occur there instead. Issues associated with this technique
-->
バージョン耐性のある方法（実行時に遅延修正できるスロット経由での間接CALLを持つこと）で非仮想関数呼び出しを行うための仕組みを持つと、その方法で「定義元」モジュールを呼び返し、そこで処理を実行させることで、その他の全てのバージョン管理の問題を解決_できる_ことに気付くことが大切です。このテクニックに関する問題は次のとおりです：

<!--
1. You will pay the cost of a true indirection function call and return, as well as any argument setup cost. This cost may be visible in constructs that do not contain a call naturally, like fetching string literals or other constants. You may be able to get better performance from another technique (for example, we did so with instance field access).
2. It introduces a lot of indirect calls. It is not friendly to systems that disallow on the fly code generation. A small helper stub has to be created at runtime in the most straightforward implementation, or there has to be a scheme how to pre-create or recycle the stubs.
3. It requires that the defining assembly 'know' the operations that it is responsible for defining. In general this could be fixed by JIT compiling whatever is needed at runtime (where the needed operations are known), but JIT compiling is the kind of expensive operation that we are trying to avoid at runtime.
-->
1. 真の間接関数呼び出しと戻りのコストと、さらに任意の引数セットアップのコストを払うことになる。このコストは本来呼び出しを含まない構造、たとえば文字列リテラルやその他の定数のフェッチにおいて目に見えるものとなる可能性がある。別のテクニック（たとえば、インスタンスフィールドアクセスで行った方法）によってより優れたパフォーマンスを得られるかもしれない。
2. 多数の間接呼び出しがもたらされる。これは都度（on-the-fly）コード生成を許可しないシステムにはなじまない。最も簡単な実装においては実行時に小さなヘルパースタブを作成しなければならず、そうでなければスタブを事前作成するか再利用するスキームが必要となる。
3. 定義元のアセンブリが、定義に対して責任を負う操作を「知る」必要がある。一般的に、実行時に必要となった場所で（必要な操作がわかった場所で）、JITコンパイル処理によって修正可能であるが、JITコンパイル処理は、我々が実行時に避けようとしているコストのかかる処理である。
 
<!--
So while there are limitations to the technique, it works very well on a broad class of issues, and is conceptually simple. Moreover, it has very nice simplicity on the caller side (a single indirect call). It is hard to get simpler than this. This simplicity means that you have wired very few assumptions into the caller which maximizes the versioning flexibility, which is another very nice attribute. Finally, this technique also allows generation of optimal code once the indirect call was made. This makes for a very flexible technique that we will use again and again.
-->
そのため、このテクニックには制限がありますが、さまざまな問題に対してとても上手く対処でき、考え方としてシンプルです。さらに、呼び出し元では非常に良い単純さを持ちます（単一の間接呼び出し）。これよりも単純にすることは難しいです。この単純さは、バージョン管理の柔軟性を最大化する呼び出し元に、ごくわずかな前提事項を埋め込むだけで済むということで、これはもう一つの非常に優れた特性です。最後に、このテクニックでは、間接呼び出しが行われるたびに最適化されたコードを生成することもできます。これは、我々が繰り返し使用するであろう非常に柔軟なテクニックです。

<!--
The runtime currently supports two mechanisms for virtual dispatch. One mechanism is called virtual stub dispatch (VSD). It is used when calling interface methods. The other is a variation on traditional vtable-based dispatch and it is used when a non-interface virtual is called. We first discuss the VSD approach. 
-->
現在、ランタイムは仮想ディスパッチ用の仕組みを2つサポートします。1番目は仮想スタブディスパッチ（VSD：Virtual Stub Dispatch）と呼ばれるものです。VSDはインターフェイスメソッドを呼び出すときに使用されます。もう1つは伝統的なvtableベースのディスパッチの派生形で、非インターフェイス仮想メソッドが呼び出されるときに使用されます。まず、VSDのやり方について議論します。

<!--
Assume that RCX holds a `Intf` then the call to `intfMethod()` would look like
-->
RCXが`Intf`を保持し、それから`intfMethod()`を呼び出すとすると、次のようになります

```Assembly
        CALL [PTR_CALLSITE]
    .data // dataセクション内
        PTR_CALLSITE: INT_PTR = RUNTIME_ENTRY_FIXUP_METHOD // 呼び出し*元*ごとに1つ。
```

<!--
This looks same as the cross-module, non-virtual case, but there are important differences. Like the non-virtual case there is an indirect call through a pointer that lives in the module. However unlike the non-virtual case, there is one such slot per call site (not per target). What is in this slot is always guaranteed to get to the target (in this case to `Intf.intfMethod()`), but it is expected to change over time. It starts out pointing to a 'dumb' stub which simply calls a runtime helper that does the lookup (in likely a slow way). However, it can update the `PTR_CALLSITE` slot to a stub that efficiently dispatches to the interface for the type that actually occurred (the remaining details of stubbed based interface dispatch are not relevant to versioning). 
-->
これはモジュール越えの、非仮想のケースと同じように見えますが、重要な違いがあります。非仮想のケースのように、モジュール内に存在するポインター経由の間接呼び出しがあります。ただし、非仮想のケースとは異なり、（呼び出し先ごとではなく）呼び出し元ごとに1つのスロットがあります。このスロットにあるものは、常に呼び出し先（この場合には`Intf.intfMethod()`）に到達することが保証されていますが、時間がたつにつれて変化していくことが期待されます。最初は、検索を（おそらく遅い方法で）行うランタイムヘルパーを単純い呼び出す「ダム（dumb）」スタブを指し示しています。ところが、ダムスタブは、実際に出現した型に対するインターフェイスに効率的にディスパッチするスタブを指し示すように、`PTR_CALLSITE`スロットを更新します（スタブベースのインターフェイスディスパッチに関する残りの詳細事項は、バージョン管理とは無関係です）。

<!--
The above description is accurate for the current CLR implementation for interface dispatch. What's more, is that nothing needs to be changed about the code generation to make it version resilient. It 'just works' today. Thus interface dispatch is version resilient with no performance penalty. 
-->
上記の説明はインターフェイスディスパッチに対する現在のCLR実装については正確です。さらに、バージョン耐性を持つようにコード生成を変更することについて、必要なことは何もありません。今のところ「そのまま動く」のです。したがって、インターフェイスディスパッチはパフォーマンスペナルティなしでバージョン耐性を持ちます。

<!--
What's more, we can actually see VSD is really just a modification of the basic 'indirect call through updateable slot' technique that was used for non-virtual method dispatch. The main difference is that because the target depends on values that are not known until runtime (the type of the 'this' pointer), the 'fixup' function can never remove itself completely but must always check this runtime value and react accordingly (which might include fixing up the slot again). To make as likely as possible that the value in the fixup slot stabilizes, we create a fixup slot per call site (rather than per target).
-->
さらに、我々は、実際のところVSDは非仮想メソッドディスパッチに使用されていた基本的な「更新可能なスロット経由の間接呼び出し」テクニックの修正版に過ぎないとみることができます。主な違いは、呼び出し先が実行時までわからない値（「this」ポインターの型）に依存しているために、「修正」関数が自分自身を完全に取り除くことができないが、常にこの実行時の値を検査して適切に対応しなければならない（スロットを再び修正するかもしれない）ことです。修正スロット内の値を可能な限り安定させるために、（呼び出し先ごとではなく）呼び出し元ごとに1つの修正スロットを作成します。

<!--
### Vtable Dispatch
-->
### Vtableディスパッチ

<!--
The CLR current also supports doing virtual dispatch through function tables (vtables). Unfortunately, vtables have the same version resilience problem as fields. This problem can be fixed in a similar way, however unlike fields, the likelihood of having many cross bubble fixups is higher for methods than for instance fields. Further, unlike fields we already have a version resilient mechanism that works (VSD), so it would have to be better than that to be worth investing in. Vtable dispatch is only better than VSD for polymorphic call sites (where VSD needs to resort to a hash lookup). If we find we need to improve dispatch for this case we have some possible mitigations to try:
-->
CLRは現在、関数テーブル（vtable）経由の仮想ディスパッチもサポートしています。残念ながら、vtableにはフィールドと同じバージョン耐性の問題があります。この問題は同じ方法で修正できますが、フィールドとは異なり、バージョンバブル越えの修正が多数存在する可能性は、インスタンスフィールドよりもメソッドの方が高くなります。さらに、フィールドとは異なり、既に動作するバージョン耐性のある仕組み（VSD）があるので、vtableディスパッチに投資するよりもそちらのほうが良いに違いありません。vtableディスパッチは、ポリモーフィックな呼び出し元においてのみVSDよりも優れています（VSDではハッシュ検索に頼らざるを得ないためです）。このケースに対するディスパッチを改善する必要性が明らかになったならば、挑戦する緩和策としては次のものがあります：

<!--
1. If the polymorphism is limited, simply trying more cases before falling back to the hash table has been prototyped and seems to be a useful optimization.
2. For high polymorphism case, we can explore the idea of dynamic vtable slots (where over time the virtual method a particular vtable slot holds can change). Before falling back to the hash table a virtual method could claim a vtable slot and now the dispatch of that method for _any_ type will be fast.
-->
1. ポリモーフィズムが限定的であれば、ハッシュテーブルにフォールバックする前に、単により多くのケースに挑戦するというプロトタイプがあり、それは有用な最適化に見える。
2. ポリモーフィズムのバリエーションが多い場合、動的vtableスロット（特定のvtableスロットが保持する仮想メソッドが常に変化可能）のアイディアを調査できる。ハッシュテーブルにフォールバックする前に、仮想メソッドはvtableスロットを殺し、_任意の_型についてそのメソッドのディスパッチを高速になるようにできる。

<!--
In short, because of the flexibility and natural version resilience of VSD, we propose determining if VSD can be 'fixed' before investing in making vtables version resilient and use VSD for all cross version bubble interface dispatch. This does not preclude using vtables within a version bubble, nor adding support for vtable based dispatch in the future if we determine that VSD dispatch can't be fixed. 
-->
まとめると、VSDの柔軟性と本質的なバージョン耐性により、vtableをバージョン耐性のあるものにすることに投資する前にVSDを「修正」できるかどうかを判断しすること、および全てのバージョンバブル越えのインターフェイスディスパッチにVSDを使用することを提案します。これは、バージョンバブル内でのvtableの使用を除外するものでも、VSDディスパッチを修正できないと判断した場合にvtableベースのディスパッチのサポートを追加するというものでもありません。

<!--
## Object Creation - NEWOBJ / NEWARR
-->
## オブジェクトの作成（newobjとnewarr）

<!--
Object allocation is always done by a helper call that allocates the uninitialized object memory (but does initialize the type information `MethodTable` pointer), followed by calling the class constructor. There are a number of different helpers depending on the characteristics of the type (does it have a finalizer, is it smaller than a certain size, ...).
-->
オブジェクトの割り当ては、常に、未初期化のオブジェクトメモリを割り当て（つつ型情報`MethodTable`ポインターを初期化する）ヘルパー呼び出しと、それに続くクラスコンストラクターの呼び出しによって行われます。型の特性（ファイナライザーの有無、特定のサイズより小さいか否か、など）に応じた様々なヘルパーが存在します。

<!--
We will defer the choice of the helper to use to allocate the object to runtime. For example, to create an instance of `SubClass` the code would be:
-->
オブジェクトを割り当てるために使用するヘルパーの選択は、実行時まで遅延させます。たとえば、`SubClass`のインスタンスを作成するコードは次のようになります：

```Assembly
        CALL [NEWOBJ_SUBCLASS]
        MOV RCX, RAX  // EAXは新しいオブジェクトを保持する
        // コンストラクターにパラメーターがある場合、それを設定する
        CALL SUBCLASS_CONSTRUCTOR

    .data // dataセクション内
        NEWOBJ_SUBCLASS: RUNTIME_ENTRY_FIXUP // 型ごとに1つ
```

<!--
where the `NEWOBJ_SUBCLASS` would be fixed up using the standard lazy technique.
-->
ここで、`NEWOBJ_SUBCLASS`は標準の遅延テクニックを使用して修正されます。

<!--
The same technique works for creating new arrays (NEWARR instruction). 
-->
同じテクニックが、新しい配列の作成（`newarr`命令）でも動作します。

<!--
## Type Casting - ISINST / CASTCLASS
-->
## 型のキャスト（isinstとcastclass）

<!--
The proposal is to use the same technique as for object creation. Note that type casting could easily be a case where VSD techniques would be helpful (as any particular call might be monomorphic), and thus caching the result of the last type cast would be a performance win. However this optimization is not necessary for version resilience. 
-->
提案するのは、オブジェクトの作成と同じテクニックです。型のキャストは、容易にVSDテクニックが役に立つケース（特定の呼び出しがモノモーフィックかもしれない場合）になり得、そのために最後の型キャストの結果のキャッシュがパフォーマンス向上に役立つであろうことに注意してください。ただし、この最適化はバージョン耐性について必要ではありません。

<!--
## GC Information for Types
-->
## 型についてのGC情報

<!--
To do its job the garbage collector must be able to take an arbitrary object in the GC heap and find all the GC references in that object. It is also necessary for the GC to 'scan' the GC from start to end, which means it needs to know the size of every object. Fast access to two pieces of information is what is needed. 
From a versioning perspective, the fundamental problem with GC information is that (like field offsets) it incorporates information from the entire inheritance hierarchy in general case. This means that the information is not version resilient.
-->
ガベージコレクターが動作するには、GCヒープ内の任意のオブジェクトを処理可能でなければならず、かつそのオブジェクトに対する全てのGC参照を発見できなければなりません。GCが初めから終わりまで「スキャン」できる必要もあります。つまり、全てのオブジェクトのサイズを知る必要があります。必要なものは、2つの情報ピースへの高速なアクセスです。バージョン管理の観点からは、GC情報における基本的な問題は、（フィールドオフセットと同様に）、一般的なケースにおいて、継承階層全体からの情報を合わせたものであることです。つまり、この情報にはバージョン耐性がありません。

<!--
While it is possible to make the GC information resilient and have the GC use this resilient data, GC happens frequently and type loading happens infrequently, so arguably you should trade type loading speed for GC speed if given the choice. Moreover the size of the GC information is typically quite small (e.g. 12-32 bytes) and will only occur for those types that cross version bubbles. Thus forming the GC information on the fly (from a version resilient form) is a reasonable starting point. 
-->
GC情報をバージョン耐性のあるものにし、GCがこのバージョン耐性のあるデータを使用するようにできたとしても、GCは頻繁に発生し、型の読み込みはめったに発生しないため、選択肢がある場合には、型の読み込みの速さよりもGCの速さを優先すべきであるのは間違いありません。さらに、GC情報のサイズは、一般的に非常に小さなもの（12から32バイト程度）で、バージョンバブル越えがある型でのみ出現します。そのため、GC情報を都度（バージョン耐性のある形式で）作成するというのが合理的なスタート地点です。

<!--
Another important observation is that `MethodTable` contains other very frequently accessed data, like flags indicating whether the `MethodTable` represents an array, or pointer to parent type. This data tends to change a lot with the evolution of the runtime. Thus, generating method tables at runtime will solve a number of other versioning issues in addition to the GC information versioning.
-->
もう一つの重要な観察結果は、`MethodTable`には、`MethodTable`が配列であることを示すフラグや、基底型へのポインターのような、非常に頻繁にアクセスされるデータも含むことです。このデータはランタイムの進化に応じて何回も変更される傾向にあります。したがって、実行時のメソッドテーブル生成が、GC情報のバージョン管理に加えて、他の多数のバージョン管理の問題を解決するでしょう。

<!--
# Current State
-->
# 現在の状態

<!--
The design and implementation is a work in progress under code name ReadyToRun (`FEATURE_READYTORUN`). RyuJIT is used as the code generator to produce the ReadyToRun images currently.
-->
設計と実装は、コードネームReadyToRun（`FEATURE_READYTORUN`）の元で作業中です。現在のところ、ReadyToRunイメージ生成用のコードジェネレーターとしてRyuJITが使用されています。
