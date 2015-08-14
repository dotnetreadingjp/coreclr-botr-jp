CLRのスタックウォーキング
===

（これは https://github.com/dotnet/coreclr/blob/8d3936bff7ae46a5a964b15b5f0bc3eb8d4e32db/Documentation/botr/stackwalking.md の日本語訳です。対象rev.は 8d3936b）

Author: Rudi Martin ([Rudi-Martin](https://github.com/Rudi-Martin)) - 2008

CLRは、スタックウォーキング（あるいはスタッククローリング）として知られているテクニックを多用しています。これは、特定のスレッドにおける呼び出しフレームの列を、最も新しい位置（現在のスレッドの現在の関数）からそのスタックの起点までを巡回するものです。

このランタイムでは、いくつかの目的でスタック ウォークを行います:

- ランタイムは、ガベージコレクションの過程で、全てのスレッドのスタックを巡回し、マネージド ルート（マネージドメソッドのフレームの中でオブジェクト参照を保持しているローカル変数。そのオブジェクトを保持しておいて、GCがヒープのコンパクションを行う際にその移動をトラッキングするよう、GCに報告する必要があるものです）を探します。
- プラットフォームによっては、このスタックウォーカーが例外処理の過程で使用されます（まずハンドラーの探索が行われ、次にスタックのアンワインド（巻き戻し）が行われます）。
- デバッガーが、マネージドスタックトレースが生成された時に、この機能を使用します。
- その他のさまざまなメソッド、通常はパブリックのマネージドAPIに近いものが、スタックウォークを行って、その呼び出し元の情報（メソッド、クラス、アセンブリなど）を獲得します。

# スタックのモデル

ここでは、スレッドのスタックの典型的なレイアウトについて説明するために、いくつかの共通の語句を定義しておきます。

スタックは、論理的には、いくつかの_フレーム_に分割されます。それぞれのフレームは、現在実行中あるいは実行が完了しリターンを待っている状態にある、何らかの（マネージドあるいはアンマネージドの）関数をあらわします。ひとつのフレームには、関連付けられた関数の、1回の呼び出しに関する状態を保持します。一般的には、ここには、ローカル変数、他の関数の呼び出しのためにプッシュされた引数、保存された呼び出し元レジスターなどが含まれます。

フレームの正確な定義は、プラットフォームごとに異なり、多くのプラットフォームには、全ての機能が実現しているフレームのフォーマットのしっかりした定義がありません（x86がその一例です）。代わりに、フレームの正確なフォーマットを、コンパイラーが自由に最適化することが、よくあります。そのようなシステムでは、スタックウォークが100%正確かつ完全な結果を返すことは保証できません（デバッグ目的では、pdbのようなデバッグシンボルが、より正確なスタックトレースを生成するためのギャップを埋めています）。

ただし、私たちは完全に一般化したスタックウォークは必要ないため、これはCLRにとっては問題ではありません。その代わり、私たちにとっては、マネージドの（すなわち、マネージドメソッドを表す）フレームと、ランタイム自身の一部として実装されたアンマネージドコード由来のフレームの一部が、関心項目です。特に、サードパーティーのアンマネージドフレームの利用可能性は、ランタイム自身によって明示的に遷移可能であると分かっているもの（すなわち、私たちが実際に関心があるもの）以外では、何の保証もありません。

私たちは、自分たちの関心があるフレームのフォーマットを自由にコントロールしているので（その詳細は後述します）、それらのフレームがクロールできることは100%保証できます。これ以上の唯一の要求事項は、中間のアンマネージドの（そしてクロールできない）フレームをスキップできるような、分解したランタイムフレームのグループをリンクする機構だけです。

以下では、全てのフレーム種別を含むスタックを図解しています。（このドキュメントでは、スタックがページの上方に向かって伸びます。）

![image](../images/stack.png)

# フレームをクロール可能にする

## マネージドフレーム

このランタイムはJIT（ジャスト イン タイム コンパイラー）をコントロールするため、マネージドメソッド上に、常にクロール可能なフレームを設置することが可能です。ソリューションのひとつとしては、全てのメソッドに固定的なフレームフォーマット（例えばx86のEBPフレームフォーマット）を施行することです。ただし、これは実際には、特に小さなリーフメソッド（よくあるプロパティ アクセサーなど）には非効率的です。

メソッドは、通常はフレームがクロールされる回数よりも多く呼び出されるものです（スタッククロールはランタイム上では相対的に少ないものです。少なくとも通常のメソッドが呼び出される回数よりは）。そのため、メソッド呼び出しのパフォーマンスのために、クロール処理時間をいくらか犠牲にすることは、理にかなっています。結局、このJITでは、各メソッドに、スタッククローラーがメソッドに属するスタックフレームをデコードするために必要な情報を含む、追加のメタデータを生成します。

このメタデータは、あるハッシュテーブル中を、そのメソッド中の命令ポインターの位置をキーとしてルックアップすることで見つかるでしょう。JITでは、この、メソッド毎に生じる追加のメタデータを、最小限に抑える圧縮技術を利用しています。

このスタッククローラーは、いくつかの重要なレジスター（x86ベースのシステムならEIP、ESP、EBPなど）の初期値を与えられると、マネージドメソッドおよびそれに関連付けられたJITメタデータの位置を特定でき、その情報を用いてレジスターの値を現在のメソッドの呼び出し元の値にロールバックします。このやり方によって、マネージドメソッドの一連のフレームが、最新の呼び出し元から最古の呼び出し元まで辿ることができます。この操作は時々_仮想アンワインド_と呼ばれます（仮想というのは、私たちは実際にはESPなどの実際の値をアップデートしているわけではなく、スタックには触れないためです）。

## ランタイム アンマネージド フレーム

このランタイムは、部分的にアンマネージドコードで実装されています（coreclr.dllなど）。このコードの大部分は、「マニュアル マネージド」コードとして動作する点で特殊なものです。すなわち、それらはマネージドコードのルールとプロトコルの多くに従っているものの、明示的に制御されたものである、ということです。たとえば、そのようなコードでは、明示的にGCプリエンプティブモードを有効あるいは無効にでき、使用するオブジェクトの参照を適切に管理する必要があります。

マネージドコードとの相互作用を慎重に行う領域は、他にも、スタックウォークの過程でもあります。ランタイムのアンマネージドコードの大部分はC++で描かれているため、私たちには、マネージドコードと同じようにメソッドフレームのフォーマットをコントロールする能力はありません。一方で、ランタイムのアンマネージドフレームが、スタックウォークの過程で重要である情報を含むようなインスタンスも多数あります。これには、アンマネージド関数がローカル変数にオブジェクト参照を保持している場合（ガベージコレクションの際に報告されなければならないものです）や、例外処理などが含まれます。

スタッククロールにレポートする意味のあるデータを有するアンマネージド関数は、各アンマネージドフレームをクロール可能にしようとする代わりに、その情報を「フレーム」と呼ばれるデータ構造にバンドルします。この名前の選択は、スタックに関連する議論の中では、残念ながら曖昧なものです。このドキュメントでは、このデータ構造の類型のことを、括弧付きで「フレーム」と呼ぶことにします。<!-- 原文は capitalized Frame = 大文字表記で、固有名詞扱い -->

「フレーム」型の全体的な階層構造における抽象基底クラスは、実のところFrameです。Frameは、スタックウォークにおける有意な情報の種類に応じて、派生型が構築されます。しかし、このスタックウォーカーは、どうやってこれらの「フレーム」を発見し、それらがマネージドメソッドによって使用されるべくフレームに関連付くのでしょうか?

各「フレーム」は、単方向連結リストの一部であり、そのスレッドのスタック上で次に古い「フレーム」へのポインターを持っています（もしその「フレーム」が最古であればnullとなります）。CLRのThread構造体は、最新のフレームへのポインターを保持しています。アンマネージド ランタイムコードは、必要に応じて、Thread構造体と「フレーム」のリストを操作して、「フレーム」をプッシュあるいはポップします。

この方式では、スタックウォーカーは、アンマネージドの「フレーム」を、最新から最古の順に巡回することができます（マネージドフレームを巡回するのと同じ順序です）。しかし、マネージド メソッド群とアンマネージド メソッド群を混合させることはできませんし、全てのマネージドフレームを処理してからアンマネージド「フレーム」を処理する、あるいはその逆は、実際の呼び出し順序を正確に表していることにならないので、間違いです。

この問題を解決するため、「フレーム」にはさらに、自身を「フレーム」のリストにプッシュするメソッドのフレーム上にアロケートされなければならない、という制約が追加されました。スタックウォーカーは、各マネージドフレームのスタック境界を知っているため、単純なポインタ比較を行って、ある「フレーム」があるマネージドフレームより古いか新しいか、判断することができます。

本質的には、このスタックウォーカーは、現在のフレームをデコードした時は、次の（古い）フレームについて、常に2つの選択肢の可能性があります。レジスター集合の仮想アンワインドによって得られた次のマネージドフレームか、そのThreadの「フレーム」のリストにある次の（古い）「フレーム」です。どちらが適切であるかは、スタックのトップに近いスタックスペースを何が占領しているかを識別することで判断できます。実際に行われる計算はプラットフォーム依存ですが、通常は1回か2回のポインタ比較によって行えます。

When managed code calls into the unmanaged runtime one of several forms of transition Frame is often pushed by the unmanaged target method. This is needed both to record the register state of the calling managed method (so that the stack walker can resume virtual unwinding of managed frames once it has finished enumerating the unmanaged Frames) and in many cases because managed object references are passed as arguments to the unmanaged method and must be reported to the GC in the event of a garbage collection.

A full description of the available Frame types and their uses is beyond the scope of the document. Further details can be found in the [frames.h](https://github.com/dotnet/coreclr/blob/master/src/vm/frames.h) header file.

# Stackwalker Interface

The full stack walk interface is exposed to runtime unmanaged code only (a simplified subset is available to managed code via the System.Diagnostics.StackTrace class). The typical entrypoint is via the StackWalkFramesEx() method on the runtime Thread class.

The caller of this method provides three main inputs:

1. Some context indicating the starting point of the walk. This is either an initial register set (for instance if you've suspended the target thread and can call GetThreadContext() on it) or an initial Frame (in cases where you know the code in question is in runtime unmanaged code). Although most stack walks are made from the top of the stack it's possible to start lower down if you can determine the correct starting context.
2. A function pointer and associated context. The function provided is called by the stack walker for each interesting frame (in order from the newest to the oldest). The context value provided is passed to each invocation of the callback so that it can record or build up state during the walk.
3. Flags indicating what sort of frames should trigger a callback. This allows the caller to specify that only pure managed method frames should be reported for instance. For a full list see [threads.h](https://github.com/dotnet/coreclr/blob/master/src/vm/threads.h) (just above the declaration of StackWalkFramesEx()).

StackWalkFramesEx() returns an enum value that indicates whether the walk terminated normally (got to the stack base and ran out of methods to report), was aborted by one of the callbacks (the callbacks return an enum of the same type to the stack walk to control this) or suffered some other miscellaneous error.

Aside from the context value passed to StackWalkFramesEx(), stack callback functions are passed one other piece of context: the CrawlFrame. This class is defined in [stackwalk.h](https://github.com/dotnet/coreclr/blob/master/src/vm/stackwalk.h) and contains all sorts of context gathered as the stack walk proceeds.

For instance the CrawlFrame indicates the MethodDesc* for managed frames and the Frame* for unmanaged Frames. It also provides the current register set inferred by virtually unwinding frames up to that point.

# Stackwalk Implementation Details

Further low-level details of the stack walk implementation are currently outside the scope of this document. If you have knowledge of these and would care to share that knowledge please feel free to update this document.
