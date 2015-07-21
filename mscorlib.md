Mscorlib and Calling Into the Runtime
mscorlib とランタイムの呼び出し

===
Author: Brian Grunkemeyer ([@briangru](https://github.com/briangru)) - 2006

# Introduction
# はじめに

Mscorlib is the assembly for defining the core parts of the type system, and a good portion of the Base Class Library. Base data types live in this assembly, and it  has a tight coupling with the CLR. Here you will learn exactly how & why mscorlib.dll is special, and the basics about calling into the CLR from managed code via QCall and FCall methods. It also discusses calling from within the CLR into managed code. 
mscorlib は型システムの中核部分を定義するアセンブリであり、基本クラスライブラリ（BCL：Base Class Library）の重要なパーツです。基本データ型はこのアセンブリに存在し、かつ CLR と密に結合しています。ここでは、mscorlib.dll が正確にはどのように、また何故特別なのか、さらにマネージドコードから QCall と FCall メソッドを通じた CLR の呼び出し方についての基本を学びます。さらに、CLR からのマネージドコード呼び出しについても説明します。


## Dependencies
## 依存関係

Since mscorlib defines base data types like Object, Int32, and String, mscorlib cannot depend on other managed assemblies. However, there is a strong dependency between mscorlib and the CLR. Many of the types in mscorlib need to be accessed from native code, so the layout of many managed types is defined both in managed code and in native code inside the CLR. Additionally, some fields may be defined only in debug or checked builds, so typically mscorlib must be compiled separately for checked vs. retail builds. 

mscorlib は Object、Int32、String といった基本データ型を定義するので、mscorlib は他のマネージドアセンブリに依存できません。ただし、mscorlib と CLR の間には強い依存関係があります。mscorlib にある型の多くはネイティブコードからアクセスされる必要があるので、多くのマネージド型のレイアウトはマネージドコードと CLR 内部のネイティブコードの両方で定義されます。さらに、いくつかのフィールドはデバッグビルドまたはチェックトビルドでのみ定義される可能性があるので、通常、mscorlib はチェックトビルドとリテールビルドで別々にビルドされなければなりません。

For 64 bit platforms, some constants are also defined at compile time. So a 64 bit mscorlib.dll is slightly different from a 32 bit mscorlib.dll. Due to these constants, such as IntPtr.Size, most libraries above mscorlib should not need to build separately for 32 bit vs. 64 bit.

64 ビットプラットフォームでは、いくつかの定数もコンパイル時に定義されます。そのため、64 ビット版の mscorlib.dll は 32 ビット版の mscorlib.dll とは若干異なっています。これらの定数、たとえば IntPtr.Size により、mscorlib.dll より上位のほとんどのライブラリは 32 ビット版と 64 ビット版を別々にビルドしなくても済むはずです。

## What Makes Mscorlib Special?
## mscorlib を特別にしているものは何か？

Mscorlib has several unique properties, many of which are due to its tight coupling to the CLR. 
mscorlib には他にはない特徴がいくつかあり、それらの多くは CLR との密結合によるものです。

- Mscorlib defines the core types necessary to implement the CLR's Virtual Object System, such as the base data types (Object, Int32, String, etc).
- mscorlib は基本データ型（Object、Int32、String など）のような、CLR の仮想オブジェクトシステムを実装するために必要な中核型を定義しています。
- The CLR must load mscorlib on startup to load certain system types.
- CLR は特定のシステム型を読み込むために、起動時に mscorlib を読み込まなければなりません。
- Can only have one mscorlib loaded in the process at a time, due to layout issues. Loading multiple mscorlibs would require formalizing a contract of behavior, FCall methods, and datatype layout between CLR & mscorlib, and keeping that contract relatively stable across versions. 
- レイアウト上の問題により、あるプロセスに一度に読み込める mscorlib は 1 つだけです。複数の mscorlib を読み込むには、CLR と mscorlib の間で振る舞い、FCall メソッド、データ型のレイアウトのコントラクトを形式化し、そのコントラクトをバージョン間で今より安定させ続ける必要があるでしょう。
- Mscorlib's types will be used heavily for native interop, and managed exceptions should map correctly to native error codes/formats.
- mscorlib の型はネイティブ相互運用のために大いに使用され、かつマネージド例外はネイティブのエラーコードとフォーマットに正しく対応付けされるべきです。
- The CLR's multiple JIT compilers may special case a small group of certain methods in mscorlib for performance reasons, both in terms of optimizing away the method (such as Math.Cos(double)), or calling a method in peculiar ways (such as Array.Length, or some implementation details on StringBuilder for getting the current thread). 
- CLR の複数の JIT コンパイラは、mscorlib 内のごく一部のメソッドについて、パフォーマンス上の理由から特別なケースとして扱い、最適化してメソッド呼び出しをなかったことにする（Math.Cos(double) など）か、メソッド呼び出しを特殊な方法で行う（Array.Length や現在のスレッド用の StringBuilder に対する実装詳細など）かを行います。
- Mscorlib will need to call into native code, via P/Invoke where appropriate, primarily into the underlying operating system or occasionally a platform adaptation layer.
- mscorlib は、主に、基になるオペレーティングシステムや、場合によってはプラットフォーム抽象化レイヤーに対して、適切な場合には P/Invoke を使用して、ネイティブコードを呼び出す必要があります。
- Mscorlib will require calling into the CLR to expose some CLR-specific functionality, such as triggering a garbage collection, to load classes, or to interact with the type system in a non-trivial way. This requires a bridge between managed code and native, "manually managed" code within the CLR.
- mscorlib は、ガベージコレクションの開始、クラスの読み込み、複雑な方法での型システムとの対話といった、何らかの CLR 固有の機能を公開するために、CLR を呼び出す必要があります。
- The CLR will need to call into managed code to call managed methods, and to get at certain functionality that is only implemented in managed code.
- CLR は、マネージドメソッドを呼び出すため、およびマネージドコード出の未実装されている特定の機能を使用するためにマネージドコードを呼び出す必要があります。

# Interface between managed & CLR code
# マネージドコードと CLR コードの間のインターフェイス

To reiterate, the needs of managed code in mscorlib include:
繰り返しになりますが、mscorlib でのマネージドコードが必要とするものには次のものがあります。

- The ability to access fields of some managed data structures in both managed code and "manually managed" code within the CLR
- マネージドコードと CLR 内の「手動マネージド」コードの両方にある何らかのマネージド構造体のフィールドにアクセスする機能
- Managed code must be able to call into the CLR
- マネージドコードは CLR を呼び出せなければならない
- The CLR must be able to call managed code.
- CLR はマネージドコードを呼び出せなければならない

To implement these, we need a way for the CLR to specify and optionally verify the layout of a managed object in native code, a managed mechanism for calling into native code, and a native mechanism for calling into managed code. 
これらを実装するために、CLR に対して、ネイティブコードでマネージドオブジェクトのレイアウトを指定、場合によっては検証する方法、ネイティブコードを呼び出すマネージド側の仕組み、そしてマネージドコードを呼び出すためのネイティブ側の仕組みが必要です。

The managed mechanism for calling into native code must also support the special managed calling convention used by String's constructors, where the constructor allocates the memory used by the object (instead of the typical convention where the constructor is called after the GC allocates memory).
ネイティブコードを呼び出すマネージド側の仕組みは、コンストラクターがオブジェクトによって使用されるメモリを割り付けるという、String のコンストラクターによって使用される特別なマネージド呼び出し規約（一般的な規約では、GC がメモリを割り付けた後にコンストラクターが呼び出されます）もサポートしなければなりません。

The CLR provides a [mscorlib binder](https://github.com/dotnet/coreclr/blob/master/src/vm/binder.cpp) internally, providing a mapping between unmanaged types and fields to managed types & fields. The binder will look up & load classes, allow you to call managed methods. It also does some simple verification to ensure the correctness of any layout information specified in both managed & native code. The binder ensures that the managed class you're attempting to use exists in mscorlib, has been loaded, and the field offsets are correct. It also needs the ability to differentiate between method overloads with different signatures.
CLR は内部的に [mscorlib バインダー](https://github.com/dotnet/coreclr/blob/master/src/vm/binder.cpp) を提供し、これはアンマネージド型とそのフィールドと、マネージド型とそのフィールドの間の対応付けを提供します。mscorlib バインダーはクラスを検索して読み込み、マネージドメソッドを呼び出せるようにします。同様に、マネージドコードとネイティブコードの両方で指定されたあらゆるレイアウト情報の正しさを確認するための単純な検証も行います。mscorlib バインダーは、使おうとしているマネージドクラスが mscorlib に存在し、読み込み済みで、そのフィールドのオフセットが正しいことを確認します。異なるシグネチャ間のメソッドのオーバーロードを区別する機能も必要です。

# Calling from managed to native code
# マネージドコードからのネイティブコードの呼び出し

We have two techniques for calling into the CLR from managed code. FCall allows you to call directly into the CLR code, and provides a lot of flexibility in terms of manipulating objects, though it is easy to cause GC holes by not tracking object references correctly. QCall allows you to call into the CLR via the P/Invoke, and is much harder to accidentally mis-use than FCall. FCalls are identified in managed code as extern methods with the MethodImplOptions.InternalCall bit set. QCalls are _static_ extern methods that look like regular P/Invokes, but to a library called "QCall".
マネージドコードから CLR を呼び出す技術は 2 つあります。FCall は CLR のコードを直接呼び出せるようにするもので、オブジェクトの操作と言う点からは大きな柔軟性をもたらしますが、オブジェクト参照を正しく追跡しないことによる GC ホールを簡単に引き起こします。QCall は P/Invoke 経由で CLR を呼び出せるようにするもので、FCall に比べてうっかり使い方を間違う可能性は大幅に低くなっています。FCall は、マネージドコード内では、MethodImplOptions.InternalCall ビットが設定された extern メソッドとして識別されます。QCall は、一般的な P/Invoke のような _static_ な extern メソッドですが、呼び出し先のライブラリ名は「QCall」になります。

There is a small variant of FCall called HCall (for Helper call) for implementing JIT helpers, for doing things like accessing multi-dimensional array elements, range checks, etc. The only difference between HCall and FCall is that HCall methods won't show up in an exception stack trace.
他次元配列の要素へのアクセス、範囲チェックと言った JIT ヘルパーを実装するための、HCall（Helper Call）という FCall のちょっとした変化形があります。HCall と FCall の間の違いは、HCall のメソッドは例外スタックトレースに表れないというところです。

### Choosing between FCall, QCall, P/Invoke, and writing in managed code
### FCall、QCall、P/Invoke、マネージドコード実装の選択

First, remember that you should be writing as much as possible in managed code. You avoid a raft of potential GC hole issues, you get a good debugging experience, and the code is often simpler. It also is preparation for ongoing refactoring of mscorlib into smaller layered fully managed libraries in [corefx](https://github.com/dotnet/corefx/).
まず、可能な限りマネージドコードで記述すべきと言うことを覚えておいてください。GC ホール問題を引き起こすかもしれないという綱渡りを避けられ、より優れたデバッギングエクスペリエンスを得られ、コードは多くの場合より単純になります。また、[corefx](https://github.com/dotnet/corefx/) では、mscorlib をリファクタリングして、レイヤー化された振るマネージドなライブラリにしようと準備を進めています。

Reasons to write FCalls in the past generally fell into three camps: missing language features, better performance, or implementing unique interactions with the runtime. C# now has almost every useful language feature that you could get from C++, including unsafe code & stack-allocated buffers, and this eliminates the first two reasons for FCalls. We have ported some parts of the CLR that were heavily reliant on FCalls to managed code in the past (such as Reflection and some Encoding & String operations), and we want to continue this momentum. We may port our number formatting & String comparison code to managed in the future.
過去、FCall を記述した理由は、一般的には 3 つの陣営に分けられます。すなわち、言語機能の不足、より良いパフォーマンス、ランタイムとの特殊な対話の実装です。今や、C# は、案セーフコードとスタック割り付けバッファーのような、 C++ で得られる有用な言語機能をほぼ全て持っており、FCall を使用する理由のうち最初の 2 つはなくなっています。我々は、これまでに FCall に大いに依存していた CLR の一部分をマネージドコードに移植してきており（たとえばリフレクションやいくつかのエンコーディングと文字列処理）、この勢いを続けたいと思っています。我々は、数値の書式指定と文字列比較のコードも将来的にマネージドコードに移植できるでしょう。

If the only reason you're defining a FCall method is to call a native Win32 method, you should be using P/Invoke to call Win32 directly. P/Invoke is the public native method interface, and should be doing everything you need in a correct manner.
FCall メソッドを定義する唯一の理由がネイティブの Win32 メソッドを呼び出すことだけならば、Win32 を直接呼び出すために P/Invoke を使用すべきです。P/Invoke はパブリックなネイティブメソッドインターフェイスであり、正しい方法で呼び出しを行うために必要なことをすべて行ってくれるはずです。

If you still need to implement a feature inside the runtime, now consider if there is a way to reduce the frequency of transitioning to native code. Can you write the common case in managed, and only call into native for some rare corner cases? You're usually best off keeping as much as possible in managed code. 
それでもランタイムの内部に機能を実装する必要があるならば、ネイティブコードへの遷移の頻度を減らす方法がないか検討してください。一般的なケースをマネージドコードで対処し、何らかのまれなコーナーケースのためだけにネイティブコードを呼び出すようにできないでしょうか？　一般的に、可能な限りマネージドコードに居続けるのが最良です。

QCalls are the preferred mechanism going forward. You should only use FCalls when you are "forced" to. This happens when there is common "short path" through the code that is important to optimize. This short path should not be more than a few hundred instructions, cannot allocate GC memory, take locks or throw exceptions (GC_NOTRIGGER, NOTHROWS). In all other circumstances (and especially when you enter a FCall and then simply erect HelperMethodFrame), you should be using QCall.
将来的には、QCall の方が好ましい仕組みです。FCall は「そうせざるを得ない」場合にのみ使用すべきです。これは、最適化のために重要なコードでの共通「短縮パス」であり得ます。この短縮パスでは、2～300 程度の命令に収めるべきで、GC メモリを割り付けることも、ロックを取得することも、例外をスローすることもできません（GC_NOTRIGGER、NOTHROWS）。その他のあらゆる状況では（特に FCall に入り、単に HelperMethodFrame を組み立てる場合には）、QCall を使用すべきです。

FCalls were specifically designed for short paths of code that must be optimized. They allowed you to take explicit control over when erecting a frame was done.  However it is error prone and is not worth it for many APIs. QCalls are essentially P/Invokes into CLR. 
FCall は、最適化せざるを得ないコードの短縮パス専用に設計されました。FCall では、フレームの組み立てが終わるタイミングを明示的に制御できます。ただし、これは間違いやす上に多くの API ではやる価値がありません。QCall は本質的に CLR への P/Invoke です。

As a result, QCalls give you some advantageous marshaling for SafeHandles automatically – your native method just takes a HANDLE type, and can use it without worrying whether someone will free the handle while you are in that method body. The resulting FCall method would need to use a SafeHandleHolder, and may need to protect the SafeHandle, etc. Leveraging the P/Invoke marshaler can avoid this additional plumbing code.
結果として、QCall には SafeHandle を自動的にマーシャリングしてくれるという優位性があります。ネイティブ側のメソッドは単に HANDLE 型を受け取るだけで、ネイティブ側のメソッドの本体にいる間に、そのハンドルが誰かに解放されることを心配せずに使用できます。結果として生じる FCall メソッドは SafeHandleHolder を使用する必要があり、SafeHandle を保護する必要がある可能性がある、等々あります。P/Invoke マーシャラーの活用によって、余計な配管工事コードを避けることができます。

## QCall Functional Behavior
## QCall の機能的な振る舞い

QCalls are very much like a normal P/Invoke from mscorlib.dll to CLR. Unlike FCalls, QCalls will marshal all arguments as unmanaged types like a normal P/Invoke. QCall also switch to preemptive GC mode like a normal P/Invoke. These two features should make QCalls easier to write reliably compared to FCalls. QCalls are not prone to GC holes and GC starvation bugs that are common with FCalls.
QCall は mscorlib.dll から CLR への通常の P/Invoke に非常によく似ています。FCall と異なり、QCall はすべての実引数を通常の P/Invoke のようにアンマネージド型にマーシャリングします。QCall は通常の P/Invoke のように割り込み型 GC モードへの切り替えも行います。これら 2 つの機能により、QCall では FCall よりも簡単に信頼性のある実装を行うことができるはずです。QCall は、FCall でありがちな GC ホールバグと GC スタベーションバグを生みづらくなっています。

QCalls perform better than FCalls that erect a HelperMethodFrame. The overhead is about 1.4x less compared to FCall w/ HelperMethodFrame overhead on x86 and x64.
QCall は、HelperMethodFrame を組み立てる FCall よりも優れたパフォーマンスが得られます。オーバーヘッドは、x86 と x64 ともに、HelperMethodFrame のオーバーヘッドのある FCall に比べて 1.4 倍少なくなっています。

The preferred types for QCall arguments are primitive types that are efficiently handled by the P/Invoke marshaler (INT32, LPCWSTR, BOOL). Notice that BOOL is the correct boolean flavor for QCall arguments. On the other hand, CLR_BOOL is the correct boolean flavor for FCall arguments.
QCall の実引数として好ましい型は、P/Invoke マーシャラーによって効率的に処理されるプリミティブ型（INT32、LPCWSTR、BOOL）です。BOOL は QCall の実引数において正しい真偽値として振る舞うことに注意してください。それに対し、FCall の実引数で正しい真偽値として振る舞うのは CLR_BOOL 型です。

The pointers to common unmanaged EE structures should be wrapped into handle types. This is to make the managed implementation type safe and avoid falling into unsafe C# everywhere. See AssemblyHandle in [vm\qcall.h][qcall] for an example.
一般的なアンマネージド EE 構造体へのポインターは、ハンドル型としてラップされるべきです。これはマネージドの実装を型安全にし、アンセーフ C# コードがあちらこちらに散らばることを避けます。例として、[vm\qcall.h][qcall] にある AssemblyHandle を参照してください。

[qcall]: https://github.com/dotnet/coreclr/blob/master/src/vm/qcall.h

There is a way to pass a raw object references in and out of QCalls. It is done by wrapping a pointer to a local variable in a handle. It is intentionally cumbersome and should be avoided if reasonably possible. See the StringHandleOnStack in the example below. Returning objects, especially strings, from QCalls is the only common pattern where passing the raw objects is widely acceptable. (For reasoning on why this set of restrictions helps make QCalls less prone to GC holes, read the "GC Holes, FCall, and QCall" section below.)
QCall で生のオブジェクト参照の入出力を行う方法が 1 つあります。それは、ローカル変数へのポインターをハンドルでラップすることです。これは、意図的に面倒になっており、合理的に避けられるならば避けるべきです。以下の StringHandleOnStack を参照してください。QCall からオブジェクト、特に文字列を返すことは、唯一通常あり得る生のオブジェクトを渡すパターンとして広く受け入れられています。（この一連の制約が QCall で GC ホールが起こり辛くできる理由については、「GC ホール、FCall、QCall」セクションを参照してください）

### QCall Example - Managed Part
### QCall の例（マネージド側）

Do not replicate the comments into your actual QCall implementation. This is for illustrative purposes.
実際の QCall の実装で、サンプルのコメントをコピーしないでください。これらのコメントは説明用のものです。

    class Foo
    {
        // All QCalls should have the following DllImport and 
        // SuppressUnmanagedCodeSecurity attributes
        // すべての QCall には以下の DllImport 属性と SuppressUnmanaedCodeSecurity 属性を付けるべき
        [DllImport(JitHelpers.QCall, CharSet = CharSet.Unicode)]
        [SuppressUnmanagedCodeSecurity]
        // QCalls should always be static extern.
        // QCall は常に static extern のはず。
        private static extern bool Bar(int flags, string inString, StringHandleOnStack retString);

        // Many QCalls have a thin managed wrapper around them to expose them to 
        // the world in more meaningful way.
        // より意味のある方法で公開するために、QCall を薄いマネージドラッパーで囲んでもよい
        public string Bar(int flags)
        {
            string retString = null;

            // The strings are returned from QCalls by taking address
            // of a local variable using JitHelpers.GetStringHandle method 
            // QCall から返される文字列は JitHelpers.GetStringHandle メソッドを使用してローカル変数からアドレスを取得する
            if (!Bar(flags, this.Id, JitHelpers.GetStringHandle(ref retString)))
             FatalError();

            return retString;
        }
    }

### QCall Example - Unmanaged Part
### QCall の例（アンマネージド側）

Do not replicate the comments into your actual QCall implementation.
実際の QCall の実装で、サンプルのコメントをコピーしないでください。

The QCall entrypoint has to be registered in tables in [vm\ecalllist.h][ecalllist] using QCFuncEntry macro. See "Registering your QCall or FCall Method" below.
QCall のエントリポイントは、QCFuncEntry マクロを使用して [vm\ecalllist.h][ecalllist] 内のテーブルに登録しておかなければなりません。「QCall または FCall メソッドを登録する」を参照してください。

[ecalllist]: https://github.com/dotnet/coreclr/blob/master/src/vm/ecalllist.h

    class FooNative
    {
    public:
         // All QCalls should be static and should be tagged with QCALLTYPE
         // すべての QCall は static で、かつ QCALLTYPE というタグをつけるべきです
         static
         BOOL QCALLTYPE Bar(int flags, LPCWSTR wszString, QCall::StringHandleOnStack retString);
    };
    
    BOOL QCALLTYPE FooNative::Bar(int flags, LPCWSTR wszString, QCall::StringHandleOnStack retString)
    {
        // All QCalls should have QCALL_CONTRACT.
        // It is alias for THROWS; GC_TRIGGERS; MODE_PREEMPTIVE; SO_TOLERANT.
        // すべての QCall は QCALL_CONTRACT を記述すべきです。
        // これは THROWS; GC_TRIGGERS; MODE_PREEMPTIVE; SO_TOLERANT のエイリアスです。
        QCALL_CONTRACT;

        // Optionally, use QCALL_CHECK instead and the expanded form of the contract
        // if you want to specify preconditions:
        // オプションで、事前条件を指定したい場合には、QCALL_CONTRACT の代わりに QCALL_CHECK とコントラクトの拡張形式を使用します。
        // CONTRACTL { 
        //     QCALL_CHECK; 
        //     PRECONDITION(wszString != NULL);
        // } CONTRACTL_END;

        // The only line between QCALL_CONTRACT and BEGIN_QCALL
        // should be the return value declaration if there is one.
        // QCALL_CONTRACT と BEGIN_QCALL の間に存在し得る行は、戻り値がある場合の、戻り値の宣言のみであるべきです。
        BOOL retVal = FALSE;

        // The body has to be enclosed in BEGIN_QCALL/END_QCALL macro. It is necessary 
        // to make the exception handling work.
        // 本体は BEGIN_QCALL と END_QCALL マクロで囲まなければなりません。これは例外処理を有効にするために必要です。
        BEGIN_QCALL;

        // Validate arguments if necessary and throw exceptions.
        // There is no convention currently on whether the argument validation should be 
        // done in managed or unmanaged code.
        // 必要ならば引数のバリデーションを行い、例外をスローします。
        // 現在のところ、引数のバリデーションをマネージド側とアンマネージド側のどちらで行うべきかの規約はありません。
        if (flags != 0)
         COMPlusThrow(kArgumentException, L"InvalidFlags");

        // No need to worry about GC moving strings passed into QCall.
        // Marshalling pins them for us.
        // QCall 内で、渡された文字列が GC によって移動されることを心配する必要はありません。
        // マーシャリング処理が文字列をピニングしてくれます。
        printf("%S", wszString);

        // This is most the efficient way to return strings back 
        // to managed code. No need to use StringBuilder.
        // これがマネージドコードに文字列を返す最も効率的な方法です。
        // StringBuilder を使用する必要はありません。
        retString.Set(L"Hello");

        // You can not return from inside of BEGIN_QCALL/END_QCALL. 
        // The return value has to be passed out in helper variable.
        // BEGIN_QCALL と END_QCALL で囲まれた内部で return することはできません。
        // 戻り値はヘルパー変数に渡しておかなければなりません。
        retVal = TRUE;

        END_QCALL;

        return retVal;
    }

## FCall Functional Behavior
## FCall の機能的な振る舞い

FCalls allow more flexibility in terms of passing object references around, with a higher code complexity and more opportunities to hang yourself. Additionally, FCall methods must either erect a helper method frame along their common code paths, or for any FCall of non-trivial length, explicitly poll for whether a garbage collection must occur. Failing to do so will lead to starvation issues if managed code repeatedly calls the FCall method in a tight loop, because FCalls execute while the thread only allows the GC to run in a cooperative manner.
FCall はオブジェクト参照の渡し方についてより柔軟にすることができますが、コードがかなり複雑になり、プログラマがハングを引き起こする可能性が高まります。さらに、FCall メソッドはその共通コードパスの周りでヘルパーメソッドフレームを組み立てるか、または相当な行数の FCall ではガベージコレクションを発生させなければならないかの明示的なポーリングを行うかのいずれかを行わなければなりません。これを怠ると、マネージドコードから FCall をタイトなループで繰り返し呼び出した場合に、スタベーションの問題を引き起こすことになります。FCall はスレッドが GC を協調的な方法（cooperative manner）でのみ実行できる状態で実行されるためです。

FCalls require a lot of glue, too much to describe here. Look at [fcall.h][fcall] for details.
FCall には多数のグル―コードが必要で、それはここで書くには大きすぎます。詳細については [fcall.h][fcall] を参照してください。

[fcall]: https://github.com/dotnet/coreclr/blob/master/src/vm/fcall.h

### GC Holes, FCall, and QCall
### GC ホール、FCall、QCall

A much more complete discussion on GC holes can be found in the [CLR Code Guide](clr-code-guide.md). Look for ["Is your code GC-safe?"](clr-code-guide.md#is-your-code-gc-safe). This tailored discussion motivates some of the reasons why FCall and QCall have some of their strange conventions.
GC ホールについて、ここよりもかなり完成度の高い説明が [CLRコーディングガイド](clr-code-guide.md) にあります。["そのコードGC安全ですか？"]を参照してください。この簡略版の説明は、FCall と QCall がいくつかの奇妙な規約を持っている、いくつかの理由を説明することを目的としています。

Object references passed as parameters to FCall methods are not GC-protected, meaning that if a GC occurs, those references will point to the old location in memory of an object, not the new location. For this reason, FCalls usually follow the discipline of accepting something like "StringObject*" as their parameter type, then explicitly converting that to a STRINGREF before doing operations that may trigger a GC. You must GC protect object references before triggering a GC, if you expect to be able to use that object reference later.
FCall メソッドにパラメーターとして渡されるオブジェクト参照は GC プロテクトされていない、つまり GC が発生すると、それらの参照はオブジェクトのメモリの古い位置を指し示すことになり、新しい位置を指し示しません。そのため、FCall は一般的にパラメーターの型に「StringObject*」のようなものを受け取るという規律に従い、GC を引き起こす可能性のある操作を行う前に、STRINGREF に明示的に変換します。オブジェクト参照を GC 後も使用可能にしたい場合には、GC を起動する前にオブジェクト参照を GC プロテクトしなければなりません。

All GC heap allocations within an FCall method must happen within a helper method frame. If you allocate memory on the GC's heap, the GC may collect dead objects & move objects around in unpredictable ways, with some low probability. For this reason, you must manually report any object references in your method to the GC, so that if a garbage collection occurs, your object reference will be updated to refer to the new location in memory. Any pointers into managed objects (like arrays or Strings) within your code will not be updated automatically, and must be re-fetched after any operation that may allocate memory and before your first usage. Reporting a reference can be done via the GCPROTECT macros, or as parameters when you erect a helper method frame. 
FCall メソッド内でのあらゆる GC ヒープ割り付けは、ヘルパーメソッドフレーム内で行わなければなりません。GC ヒープ上のメモリを割り付ける場合、GC は何らかの、発生する確率が低い、予測不能な方法でデッドオブジェクトの回収とオブジェクトの移動を行う可能性があります。そのため、ガベージコレクションが発生した場合にオブジェクト参照がメモリの新しい位置を参照するよう更新されるように、メソッド内のあらゆるオブジェクト参照について GC に手動で報告しなければなりません。コードにあるマネージドオブジェクト内のあらゆるポインター（配列や文字列等）は自動的に更新されないので、メモリ割り付けを行うかもしれない操作を行ってから最初に使用するまでの間に、差異フェッチを行わなければなりません。参照の報告は GCPROTECT マクロか、ヘルパーメソッドフレームを組み立てる場合のパラメーターとして実行できます。

Failing to properly report an OBJECTREF or to update an interior pointer is commonly referred to as a "GC hole", because the OBJECTREF class will do some validation that it points to a valid object every time you dereference it in checked builds. When an OBJECTREF pointing to an invalid object is dereferenced, you'll get an assert saying something like "Detected an invalid object reference. Possible GC hole?". This assert is unfortunately easy to hit when writing "manually managed" code. 
OBJECTREF の正しい報告、または内部のポインターの更新の漏れは、チェックトビルドでは OBJECTREF クラスが逆参照を行うたびに正しいオブジェクトを指し示しているかどうかのバリデーションを実行するために、一般的に「GC ホール」と言います。不正なオブジェクトを指し示す OBJECTREF が逆参照された場合、「Detected an invalid object reference. Possible GC hole?」（不正なオブジェクト参照が検出されました。GC ホールではありませんか？）というアサーションが発生します。残念ながら、「手動マネージド」コードを記述していると、このアサーションはすぐに見ることができます。

Note that QCall's programming model is restrictive to sidestep GC holes most of the time, by forcing you to pass in the address of an object reference on the stack. This guarantees that the object reference is GC protected by the JIT's reporting logic, and that the actual object reference will not move because it is not allocated in the GC heap. QCall is our recommended approach, precisely because it makes GC holes harder to write.
QCall のプログラミングモデルは、スタック上のオブジェクト参照のアドレスを渡すことを強制することで、ほとんどの場合の sidestep GC ホールについて restrictive であることに注意してください。これは、JIT のレポーティングロジックによってオブジェクト参照が GC プロテクトされ、かつ GC ヒープに割り付けられていないために実際のオブジェクト参照が移動しないことを保証します。まさに GC ホールを書いてしまう可能性が低いという理由により、QCall が推奨される手法です。

### FCall Epilogue Walker for x86
### FCall エピローグウォーカー（x86 用）

The managed stack walker needs to be able to find its way from FCalls. It is relative easy on newer platforms that define conventions for stack unwinding as part of the ABI. The stack unwinding conventions are not defined by ABI for x86. The runtime works around it by implementing a epilog walker. The epilog walker computes the FCall return address and callee save registers by simulating the FCall execution. This imposes limits on what constructs are allowed in the FCall implementation.
マネージドスタックウォーカーは、FCall からでも自力で動作しなければなりません。これは、ABI の一部としてスタックのアンワインド処理の規約を定義している新しめのプラットフォームでは比較的簡単です。スタックのアンワインド処理の規約は x86 用の ABI では定義されていません。CLR はエピローグウォーカー（epilog walker）を実装することでこれを回避します。エピローグウォーカーは FCall の戻りアドレスを計算し、呼び出された側は FCall の実行をシミュレートすることでレジスタを保存します。これは、FCall の実装で許可されるコンストラクトに制限をかけることを無理強いします。

Complex constructs like stack allocated objects with destructors or exception handling in the FCall implementation may confuse the epilog walker. It leads to GC holes or crashes during stack walking. There is no exact list of what constructs should be avoided to prevent this class of bugs. An FCall implementation that is fine one day may break with the next C++ compiler update. We depend on stress runs & code coverage to find bugs in this area.
FCall の実装において、デストラクタ―や例外処理を伴うスタック割り付けオブジェクトのような複雑なコンストラクトはエピローグウォーカーを混乱させる可能性があります。これにより、GC ホールを引き起こしたり、スタックウォーク処理中のクラッシュを引き起こしたりします。この種類のバグを防ぐために割けるべきコンストラクトの正確な一覧はありません。ある時点で問題のない FCall 実装は、次回の C++ コンパイラの更新によって壊れるかもしれません。この領域のバグの発見は、ストレステストの実行とコード網羅率に依存しています。

Setting a breakpoint inside an FCall implementation may confuse the epilog walker. It leads to an "Invalid breakpoint in a helpermethod frame epilog" assert inside [vm\i386\gmsx86.cpp](https://github.com/dotnet/coreclr/blob/master/src/vm/i386/gmsx86.cpp).
FCall 実装内へのブレークポイントの設定は、エピローグウォーカーを混乱させる可能性があります。ブレークポイントによって、[vm\i386\gmsx86.cpp](https://github.com/dotnet/coreclr/blob/master/src/vm/i386/gmsx86.cpp) 内で「Invalid breakpoint in a helpermethod frame epilog」（ヘルパーメソッドフレームのエピローグでの不正なブレークポイント」というアサーションが引き起こされます。

### FCall Example – Managed Part
### FCall の例（マネージド側）

Here's a real-world example from the String class:
これは String クラスでの実際の例です。

    public partial sealed class String
    {
        // Replaces all instances of oldChar with newChar.
        // すべての oldChar を newChar で置き換える
        [MethodImplAttribute(MethodImplOptions.InternalCall)]
        public extern String Replace (char oldChar, char newChar);
    }

### FCall Example – Native Part
### FCall の例（ネイティブ側）

The FCall entrypoint has to be registered in tables in [vm\ecalllist.h][ecalllist] using FCFuncEntry macro. See "Registering your QCall or FCall Method".
FCall のエントリポイントは、FCFuncEntry マクロを使用して [vm\ecalllist.h][ecalllist] 内のテーブルに登録しておかなければなりません。「QCall または FCall メソッドを登録する」を参照してください。

Notice how oldBuffer and newBuffer (interior pointers into String instances) are re-fetched after allocating memory. Also, this method is an instance method in managed code, with the "this" parameter passed as the first argument. We use StringObject* as the argument type, then copy it into a STRINGREF so we get some error checking when we use it. 
oldBuffer と newBuffer（String インスタンスへの内部ポインター）をメモリ割り付け後に再フェッチする方法に注意してください。さらに、このメソッドはマネージドコード内ではインスタンスメソッドであり、第 1 引数として「this」パラメーターが渡されています。その引数の型には StringObject* を使用しており、そしてそれを STRINGREF にコピーすることで、使用するときにエラーチェックを行っています。

	FCIMPL3(LPVOID, COMString::Replace, StringObject* thisRefUNSAFE, CLR_CHAR oldChar, CLR_CHAR newChar)
	{
	    FCALL_CONTRACT;
	
	    int length = 0;
	    int firstFoundIndex = -1;
	    WCHAR *oldBuffer = NULL;
	    WCHAR *newBuffer;
	
	    STRINGREF   newString   = NULL;
	    STRINGREF   thisRef     = (STRINGREF)thisRefUNSAFE;
	
	    if (thisRef==NULL) {
	        FCThrowRes(kNullReferenceException, L"NullReference_This");
	    }
	
	    [… Removed some uninteresting code here for illustrative purposes…]
        [… 説明のために重要でないコードは割愛 …]
	
	    HELPER_METHOD_FRAME_BEGIN_RET_ATTRIB_2(Frame::FRAME_ATTR_RETURNOBJ, newString, thisRef);
	
	    //Get the length and allocate a new String
	    //We will definitely do an allocation here.
        // 文字列の長さを取得し、新しい String を割り付けます
        // ここで、決定的に割り付けを実行します
	    newString = NewString(length);
	
	    //After allocation, thisRef may have moved
        // 割り付け後に、thisRef は移動されている可能性があります
	    oldBuffer = thisRef->GetBuffer();
	
	    //Get the buffers in both of the Strings.
        // 両方の String からバッファを取得します
	    newBuffer = newString->GetBuffer();
	
	    //Copy the characters, doing the replacement as we go.
        // 置き換えを行いつつ、文字をコピーします
	    for (int i=0; i<firstFoundIndex; i++) {
	        newBuffer[i]=oldBuffer[i];
	    }
	    for (int i=firstFoundIndex; i<length; i++) {
	        newBuffer[i]=(oldBuffer[i]==((WCHAR)oldChar))?
	                      ((WCHAR)newChar):oldBuffer[i];
	    }
	
	    HELPER_METHOD_FRAME_END();
	
	    return OBJECTREFToObject(newString);
	}
	FCIMPLEND


## Registering your QCall or FCall Method
## QCall または FCall メソッドを登録する

The CLR must know the name of your QCall and FCall methods, both in terms of the managed class & method names, as well as which native methods to call. That is done in [ecalllist.h][ecalllist], with two arrays. The first array maps namespace & class names to an array of function elements. That array of function elements then maps individual method names & signatures to function pointers.
CLR は、マネージドクラスとメソッドの名前だけでなく、呼び出し先のネイティブメソッドについても、QCall メソッドと FCall メソッドの名前を知らなければなりません。これは [ecalllist.h][ecalllist] にある 2 つの配列で行われます。1 番目の配列は関数要素の配列に名前空間とクラス名を対応付けます。関数要素の配列は、個々のメソッド名とシグネチャを関数ポインターに対応付けます。

Say we defined an FCall method for String.Replace(char, char), in the example above. First, we need to ensure that we have an array of function elements for the String class.
ここで、上記の例の String.Replace(char, char) 用の FCall メソッドを定義するとしましょう。まず、String クラス用の関数要素の配列があるようにする必要があります。

	// Note these have to remain sorted by name:namespace pair (Assert will wack you if you 
	    …
	    FCClassElement("String", "System", gStringFuncs)
	    …

Second, we must then ensure that gStringFuncs contains a proper entry for Replace. Note that if a method name has multiple overloads (such as String.Replace(String, String)), then we can specify a signature:
次に、gStringFuncs が Replace 用の正しいエントリを含むようにしなければなりません。メソッド名に複数のオーバーロードがある場合（たとえば、String.Replace(String, String)）、シグネチャを指定できることに注意してください。

	FCFuncStart(gStringFuncs)
	    …
	    FCFuncElement("IndexOf", COMString::IndexOfChar)
	    FCFuncElementSig("Replace", &gsig_IM_Char_Char_RetStr, COMString::Replace)
	    FCFuncElementSig("Replace", &gsig_IM_Str_Str_RetStr, COMString::ReplaceString)
	    …
	FCFuncEnd()

There is a parallel QCFuncElement macro.
対応する QCFuncElement マクロもあります。

## Naming convention
## 命名規約

Try to use normal name (e.g. no “_”, “n” or “native” prefix) for all FCalls and QCalls. It is not good idea to embed that the function is implemented in VM in the name of the function for the following reasons:
すべての FCall と QCall に通常の名前（たとえば、「_」、「n」または「native」といったプレフィックスをつけない）を使用するようにします。その関数が VM 内部で実装されていることを関数の名前に埋め込むがことが良くない理由は以下のとおりです。

- There are directly exposed public FCalls. These FCalls have to follow the naming convention for public APIs.
- パブリックな FCall によって直接公開されています。これらの FCall はパブリックな API の命名規約に従わなければなりません。
- The implementation of functions do move between CLR and mscorlib.dll. It is painful to change the name of the function in all call sites when this happens.
- 関数の実装が CLR と mscorlib.dll の間で移ります。このときに、すべての呼び出し元で関数の名前を変更するのは大変です。

When necessary you can use "Internal" prefix to disambiguate the name of the FCall or QCall from public entry point (e.g. the public entry point does error checking and then calls shared worker function with exactly same signature). This is no different from how you would deal with this situation in pure managed code in BCL.
必要な場合、FCall または QCall の名前をパブリックなエントリポイント（たとえば、エラーチェックを行い、全く同じシグネチャの共有ワーカー関数を呼び出すパブリックなエントリポイント）と区別するために「Internal」プレフィックスを使用して構いません。これについて、BCL 内の純粋なマネージドコードにおける場合とやり方に違いはありません。

# Types with a Managed/Unmanaged Duality
# マネージドとアンマネージドの双対性（duality）を持つ型

Certain managed types must have a representation available in both managed & native code. You could ask whether the canonical definition of a type is in managed code or native code within the CLR, but the answer doesn't matter – the key thing is they must both be identical. This will allow the CLR's native code to access fields within a managed object in a very fast, easy to use manner. There is a more complex way of using essentially the CLR's equivalent of Reflection over MethodTables & FieldDescs to retrieve field values, but this probably doesn't perform as well as you'd like, and it isn't very usable. For commonly used types, it makes sense to declare a data structure in native code & attempt to keep the two in sync.
いくつかのマネージド型はマネージドコードとネイティブコードの両方で使用可能な表現を持たなければなりません。マネージドコードと CLR 内のネイティブコードで、どちらが型の正式な定義なのか疑問に思ったかもしれませんが、その答えは重要ではありません。鍵となるは、両方が一致していなければならないということです。これによって、CLR のネイティブコードがマネージドオブジェクト内のフィールドに対し、非常に高速で、簡単に使用できる方法でアクセスできるようになります。フィールドの値を取得するために MethodTable と FieldDesc を使用するという、本質的には CLR のリフレクションと同じ方法を使用するというより複雑な方法がありますが、これはおそらく思ったようには動きませんし、あまり有用ではありません。一般的に使用される型については、ネイティブコードにデータ構造体を宣言して、マネージドコード型と同期し続けようとするのがよいでしょう。

The CLR provides a binder for this purpose. After you define your managed & native classes, you should provide some clues to the binder to help ensure that the field offsets remain the same, to quickly spot when someone accidentally adds a field to only one definition of a type.
CLR はこのためのバインダーを提供します。マネージドクラスととネイティブクラスを定義した後、誰かがうっかり型のいずれかの定義にだけフィールドを足してしまった場合にそれが素早くわかるように、フィールドのオフセットが同じであり続けるようにしてくれるバインダーにいくつかの手がかりを提供すべきです。

In [mscorlib.h][mscorlib.h], you can use macros ending in "_U" to describe a type, the name of fields in managed code, and the name of fields in a corresponding native data structure. Additionally, you can specify a list of methods, and reference them by name when you attempt to call them later.
[mscorlib.h][mscorlib.h] では、「_U」で終わるマクロを使用して、型、マネージドコード内のフィールド名、対応するネイティブデータ構造体内のフィールド名を記述できます。さらに、メソッドのリストを指定して、後から呼び出そうとするときに名前でそのメソッドを参照できます。

[mscorlib.h]: https://github.com/dotnet/coreclr/blob/master/src/vm/mscorlib.h

	DEFINE_CLASS_U(SAFE_HANDLE,         Interop,                SafeHandle,         SafeHandle)
	DEFINE_FIELD(SAFE_HANDLE,           HANDLE,                 handle)
	DEFINE_FIELD_U(SAFE_HANDLE,         STATE,                  _state,                     SafeHandle,            m_state)
	DEFINE_FIELD_U(SAFE_HANDLE,         OWNS_HANDLE,            _ownsHandle,                SafeHandle,            m_ownsHandle)
	DEFINE_FIELD_U(SAFE_HANDLE,         INITIALIZED,            _fullyInitialized,          SafeHandle,            m_fullyInitialized)
	DEFINE_METHOD(SAFE_HANDLE,          GET_IS_INVALID,         get_IsInvalid,              IM_RetBool)
	DEFINE_METHOD(SAFE_HANDLE,          RELEASE_HANDLE,         ReleaseHandle,              IM_RetBool)
	DEFINE_METHOD(SAFE_HANDLE,          DISPOSE,                Dispose,                    IM_RetVoid)
	DEFINE_METHOD(SAFE_HANDLE,          DISPOSE_BOOL,           Dispose,                    IM_Bool_RetVoid)


Then, you can use the REF<T> template to create a type name like SAFEHANDLEREF. All the error checking from OBJECTREF is built into the REF<T> macro, and you can freely dereference this SAFEHANDLEREF & use fields off of it in native code. You still must GC protect these references.
それから、SAFEHANDLEREF のような型名を作成するために、REF<T> テンプレートを使用できます。OBJECTREF 由来のあらゆるエラーチェックは REF<T> マクロに組込まれており、気兼ねなくこの SAFEHANDLEREF を逆参照したり、ネイティブコードでそのフィールドを使用したりできます。ただし、これらの参照は GC プロテクトしなければなりません。

#Calling Into Managed Code From Native
# ネイティブコードからマネージドコードを呼び出す

Clearly there are places where the CLR must call into managed code from native. For this purpose, we have added a MethodDescCallSite class to handle a lot of plumbing for you. Conceptually, all you need to do is find the MethodDesc\* for the method you want to call, find a managed object for the "this" pointer (if you're calling an instance method), pass in an array of arguments, and deal with the return value. Internally, you'll need to potentially toggle your thread's state to allow the GC to run in preemptive mode, etc. 
明らかに、CLR がネイティブコードからマネージドコードを呼び出さなくてはならない場所が存在します。このために、多数の配管工事（定型コード記述）を処理してくれる MethodDescCallSite クラスを追加しました。考え方としては、やる必要があるのは、呼び出したいメソッドの MethodDesc\* を見つけること、「this」ポインターに対するマネージドオブジェクトを見つけること（インスタンスメソッドを呼び出す場合）、実引数の配列を渡すこと、そして戻り値を処理することだけです。内部的に、スレッドの状態を GC が割り込みモードで実行できるように切り替える必要がある場合などがあります。

Here's a simplified example. Note how this instance uses the binder described in the previous section to call SafeHandle's virtual ReleaseHandle method.
以下は単純化した例です。このインスタンスが SafeHandle の ReleaseHandle 仮想メソッドを呼び出すために前の節で説明したバインダーを使用しているそのやり方に注意してください。

	void SafeHandle::RunReleaseMethod(SafeHandle* psh)
	{
	    CONTRACTL {
	        THROWS;
	        GC_TRIGGERS;
	        MODE_COOPERATIVE;
	    } CONTRACTL_END;
	
	    SAFEHANDLEREF sh(psh);
	
	    GCPROTECT_BEGIN(sh);
	
	    MethodDescCallSite releaseHandle(s_pReleaseHandleMethod, METHOD__SAFE_HANDLE__RELEASE_HANDLE, (OBJECTREF*)&sh, TypeHandle(), TRUE);
	
	    ARG_SLOT releaseArgs[] = { ObjToArgSlot(sh) };
	    if (!(BOOL)releaseHandle.Call_RetBool(releaseArgs)) {
	        MDA_TRIGGER_ASSISTANT(ReleaseHandleFailed, ReportViolation)(sh->GetTypeHandle(), sh->m_handle);
	    }
	
	    GCPROTECT_END();
	}


# Interactions with Other Subsystems
# その他のサブシステムとの対話

## Debugger
## デバッガー

One limitation of FCalls today is that you cannot easily debug both managed code and FCalls easily in Visual Studio's Interop (or mixed mode) debugging. Setting a breakpoint today in an FCall and debugging with Interop debugging just doesn't work. This most likely won't be fixed.
今日での FCall の制限の一つは、Visual Studio の相互運用（または混在モード）のデバッグ機能で、マネージドコードと FCall の両方を簡単にデバッグできないことです。FCall 内にブレークポイントを設定しての相互運用デバッグは、単に動作しません。これはおそらく修正されないでしょう。

# Physical Architecture
# 物理アーキテクチャ

When the CLR starts up, mscorlib is loaded by a method called LoadBaseSystemClasses. Here, the base data types & other similar classes (like Exception) are loaded, and appropriate global pointers are set up to refer to mscorlib's types.
CLR の開始時に、mscorlib は LoadBaseSystemClasses というメソッドによって読み込まれます。ここで、基本データ型とその他の同類のクラス群（Exception など）が読み込まれ、mscorlib の型を参照するように適切なグローバルポインターがセットアップされます。

For FCalls, look in [fcall.h][fcall] for infrastructure, and [ecalllist.h][ecalllist] to properly inform the runtime about your FCall method.
FCall について、そのインフラストラクチャについては、[fcall.h][fcall] を、FCall メソッドについてランタイムに正しく伝える方法については [ecalllist.h][ecalllist] を参照してください。

For QCalls, look in [qcall.h][qcall] for associated infrastructure, and [ecalllist.h][ecalllist] to properly inform the runtime about your QCall method.
QCall について、関連するインフラストラクチャについては [qcall.h][qcall] を、QCall メソッドについてランタイムに正しく伝える方法については [ecalllist.h][ecalllist] を参照してください。

More general infrastructure and some native type definitions can be found in [object.h][object.h]. The binder uses mscorlib.h to associate managed & native classes.
より汎用なインフラストラクチャといくつかのネイティブ型定義は [object.h][object.h] にあります。バインダーはマネージドクラスとネイティブクラスを関連付けるために mscorlib.h を使用します。

[object.h]: https://github.com/dotnet/coreclr/blob/master/src/vm/object.h
