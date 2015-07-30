Mscorlibおよびランタイム呼び出し
===

（これは https://github.com/dotnet/coreclr/blob/8d3936bff7ae46a5a964b15b5f0bc3eb8d4e32db/Documentation/botr/mscorlib.md の日本語訳です。対象rev.は 8d3936b）

著者: Brian Grunkemeyer ([briangru](https://github.com/briangru)) - 2006

# 導入

Mscorlibは型システムの中心的部分を定義するアセンブリであり、基本クラスライブラリのけっこうな部分を占めるものです。基本データ型はこのアセンブリの中にあり、CLRと密に結合しています。ここでは、mscorlib.dllがなぜ、いかに特別であるか、そしてマネージドコードからQCallおよびFCallメソッドを経由したCLR呼び出しの基本について、知ることができるでしょう。CLRからのマネージドコードの呼び出しについても議論します。

## 依存関係

mscorlibはObject、Int32、Stringといった基本データ型を定義するため、mscorlibは他のマネージド アセンブリに依存することはできません。一方で、mscorlibとCLRの間には厳密な依存関係があります。mscorlibに含まれる型の多くは、ネイティブコードからアクセスできる必要があり、したがって、多くのマネージド型のレイアウトは、マネージドコードとCLR内部のネイティブコードの両方で定義されます。さらに、いくつかのフィールドは、デバッグおよび検査ビルドでのみ定義されるため、mscorlibは、通常、検査ビルドとリテールビルドでは、別々にコンパイルされる必要があります。

64ビット環境においては、定数のうちのいくつかは、コンパイル時にも定義されます。そのため、64ビットのmscorlib.dllは32ビットのmscorlib.dllと多少異なるものになります。IntPtr.Sizeなどを定数とするなどによって、mscorlibより上位の大半のライブラリが、32ビットと46ビットで別々にビルドすることを求められてはなりません。

## なぜMscorlibが特別なのか?

Mscorlibには、さまざまな独自の特性があり、その多くはCLRと密に結合しているがゆえのものです。

- Mscorlibでは基本データ型など(Object, Int32, Stringなど)、CLRの仮想オブジェクトシステムを実装する中心的な型を定義しています。
- CLRは特定のシステム型をロードするために、実行開始時にmscorlibをロードしなければなりません。
- オブジェクト レイアウトの都合上、プロセス実行時には、mscorlibはただ1つのみがロードできます。複数のmscorlibがロードできるようになると、振る舞いに関する要求事項、FCallメソッド、CLRとmscorlib間でのデータ型のレイアウト、といった部分に関する形式的仕様が必要になり、それを（CLRの）バージョン間で安定したものにすることが求められてしまいます。
- Mscorlibの型はネイティブ相互運用で多用されており、マネージド例外はそれらのネイティブエラーコードおよびエラーフォーマットに正しく対応していなければなりません。
- CLR上の複数のJITコンパイラでは、mscorlib中のある種のメソッドの集合について、パフォーマンス上の理由で特別扱いで処理することがあります。これは、メソッドを最適化して省略する場合（Math.Cos(double)など）としても、独特のメソッド呼び出しが行われる場合（たとえばArray.Lengthや、StringBuilderが現在のスレッドを取得するといった実装の詳細）としてもあり得ます。
- Mscorlibでは、必要な場合は、P/Invokeを使用して、主に、動作しているオペレーティング システムあるいはプラットフォーム適合レイヤー（platform adaptation layer, PAL）のネイティブコードを呼び出す必要があります。
- Mscorlibは、CLRに対して、CLR固有の機能、たとえばガベージコレクションの起動、クラスのロード、型システムとの複雑な相互運用などを、呼び出せるように公開することを要求します。このために、マネージドコードと、CLRに含まれるネイティブの「マニュアル マネージド」コードの間に、橋渡しが求められます。
- CLRは、マネージドメソッドを呼び出すことで、マネージドコードの中でのみ実装されている機能を使用する必要もあります。

# マネージドコードとCLRコード間のインターフェース

繰り返しになりますが、mscorlibに含まれるマネージドコードの必要性は以下の通りです:

- いくつかの、マネージドコードとCLR中の「マニュアル マネージド」コードの両方にある、マネージドデータ構造のフィールドに、アクセスできること
- マネージドコードがCLRの中身を呼び出せること
- CLRがマネージドコードを呼び出せること

これらを実装するためには、CLRがネイティブコード中でマネージドオブジェクトのレイアウトを指定し、状況に応じて検証する方法、ネイティブコードを呼び出すマネージドの機構、マネージドコードを呼び出すネイティブの機構、が必要となります。

ネイティブコードを呼び出すためのマネージドの機構では、Stringのコンストラクターで使用される特別なマネージド呼び出し規約もサポートしなければなりません。このコンストラクターは、そのオブジェクトによって使用されるメモリを確保します（通常の規約では、GCがメモリを確保した後でコンストラクターが呼び出されます）。

CLRでは、内部的に、[mscorlib バインダー](https://github.com/dotnet/coreclr/blob/master/src/vm/binder.cpp)が、アンマネージド型とそのフィールド群から、マネージド型とそのフィールド群への、マッピングを提供します。このバインダーはクラス群をルックアップおよびロードして、あなたがマネージドメソッドを呼び出せるようにします。これは、いくつかの簡単な検証も行い、マネージドとネイティブのコード間で指定されたレイアウト情報が正しいかどうかも確認します。このバインダーは、あなたが使用しようとしているマネージドクラスがmscorlib内に存在し、ロードされ、フィールドオフセットが正しいことを確認します。また、メソッドのオーバーロードの間で、それぞれシグネチャーが異なるものとして区別できる能力も必要となります。

# マネージドコードからネイティブコードを呼び出す

マネージドコードからCLRを呼び出す方法は2つあります。FCallでは、CLRコードを直接呼び出すことができます。これは、フレキシブルなオブジェクトの操作を多大に行うことができますが、オブジェクト参照を正しくトラッキングしないことで簡単にGCの抜け穴を作ることが出来てしまいます。
QCallでは、P/InvokeによってCLRを呼び出すことができます。これはFCallよりもはるかにミスユースの可能性が小さくなります。FCallは、MethodImplOptions.InternalCallを指定されたexternメソッドとして識別できます。QCallは、通常のP/Invokeと同様に見える_staticな_externメソッドですが、"QCall"というライブラリを呼び出すものです。

FCallには、HCall（Helper Call）と呼ばれる小規模な亜種があります。これは、JITのヘルパーを実装するもので、多次元配列要素へのアクセスや範囲チェックなどを行うためのものです。HCallとFCallの違いは、HCallのメソッドは例外スタックトレース上に出現しない、ということです。

### FCallで書くか、QCallで書くか、P/Invokeで書くか、マネージドコードで書くか

まず、可能な限りマネージドコードで書くべきです。GCの抜け穴問題をあらかた回避でき、まともなデバッグ作業が期待でき、コードは往々にして単純になります。また、これによって、[corefx](https://github.com/dotnet/corefx/)で現在進んでいる、mscorlibを小さい階層的な完全マネージドライブラリとする、リファクタリング作業の準備にもなります。

FCallを書く理由は、かつては主に次の3つの集団のどれかに当てはまるものでした。言語機能の不足、より良いパフォーマンス、ランタイムと独自の相互運用の実装です。C#は今や、unsafeコードやスタック アロケーション バッファなどを含む、C++で得られる便利な機能の大半を実現しており、上記理由の最初の2つは排除されました。私たちは以前に、、CLRの一部分をFCall依存からマネージドコードに移植しており（たとえばReflection、EncodingとString操作のいくつか）、この作業は継続して行われていきます。数値のフォーマッティングと文字列比較のコードは、将来的にマネージドに移植されるかもしれません。

もしあなたがFCallメソッドを定義する理由が、単純にWin32メソッドを呼び出すためであれば、P/Invokeを使用して、直接Win32を呼び出すべきです。P/Invokeはネイティブメソッドの公開されたインターフェースであり、あなたにとって必要な作業を全て適切に行ってくれます。

もしこれでもまだランタイム内部に機能を実装する必要があるなら、ネイティブコードへの遷移の頻度を可能な限り減らす方法がないか、検討して下さい。共通の処理はマネージドで書けないか、ネイティブを呼び出す必要があるのは一部の重箱の隅のケースだけではないか? 可能な限りマネージドコードで行う方法が多くの場合はあることでしょう。

以降は、QCallが望ましいメカニズムであることを説明しましょう。FCallはそれを「強制される」場合にのみ使用すべきです。FCallは、最適化の際に重要である、コードの一般的な「短いコードパス」がある場合に発生します。この短いコードパスは、数百命令以上にはせず、GCメモリを確保せず、ロックを確保したり例外を投げたりしないことです (GC_NOTRIGGER, NOTHROWS)。それに当てはまらないなら、いかなる状況下でも（特に、FCallに入って単にHelperMethodFrameを準備するだけでも）、QCallを使用すべきです。

FCallは、最適化される必要がある短いコードパスのために、特別に設計されています。これを使うと、フレームの準備が完了した時に明示的に制御を渡すことが可能でした。ただし、これは間違いを起こしやすいものであり、多くのAPIにおいては、使用に値しないものです。QCallこそが、本質的に、CLRに特化したP/Invokeです。

結局のところ、QCallはSafeHandleを自動的にマーシャリングする際のアドバンテージを与えてくれます - あなたのネイティブメソッドは、HANDLE型を受け取って、メソッド本体の実行中に誰かがそれを解放したりしないか、などと心配することなく、使用することができるようになるのです。FCallメソッドでは、SafeHandleHolderを使用する必要があり、おそらくそのSafeHandle等を保護する必要があるでしょう。P/Invokeのマーシャラーを活用することで、そのような余計なコードを書かずに済むようになるのです。

## QCallの機能的な振る舞い

QCallは、ほぼ、通常のP/Invokeの、mscorli.dllからCLRを呼び出す版のようなものです。FCallとは異なり、QCallは通常のP/Invokeと同様に、引数を全てアンマネージド型としてマーシャリングします。QCallはまた、通常のP/Invokeと同様に、プリエンプティブGCモードへの切り替えを行います。これらの2つの機能が、QCallがFCallに比べて信頼性の高いコードを書けるようにしています。QCallは、FCallでよく起こる、GCの抜け穴を作り出したり、GC不足のバグを起こすこともありません。

QCallは、HelperMethodFrameを準備するFCallよりも、パフォーマンスが良いです。そのオーバーヘッドは、x86およびx64では、HelperMethodFrameのあるFCallの1.4分の1ですみます。

QCall引数の望ましい型は。P/Invokeマーシャラーで効率的に扱われるプリミティブ型（INT32、LPCWSTR、BOOL）です。注意すべきは、QCall引数における正しいbooleanのフレーバーはBOOLである、ということです。一方、FCall引数における正しいbooleanのフレーバーはCLR_BOOLとなります。

一般的なアンマネージドEE構造体へのポインタは、ハンドル型でラップするべきです。これによって、マネージド実装が型安全になり、至るところでunsafeなC#を書くような事態が回避できます。例として、[vm\qcall.h][qcall] に含まれるAssemblyHandleを見てみて下さい。

[qcall]: https://github.com/dotnet/coreclr/blob/master/src/vm/qcall.h

生のオブジェクト参照をQCallの入出力に渡すことは可能です。ポインターをラップしてハンドルとしてローカル変数にすれば良いのです。これは意図的に中途半端に設計されたものであり、可能な限り避けるべきやり方です。以下の例に示すStringHandleOnStackを見て下さい。QCallからの戻り値となるオブジェクトとして、特に文字列のみが、生のオブジェクトとして渡される、というのが広く容認されるパターンです。（なぜこの制約の集合が、QCallがGCの抜け穴を作らないことに繋がるのか、その理由付けについては、「GCの抜け穴とFCallとQCall」の節を読んでください。）

### QCallの使用例 - マネージド部分

以下のコメントをあなたの実際のQCall実装に使い回さないでください。これは説明のために入れたものです。

	class Foo
	{
	    // 全てのQCallは、以下のDllImportと
	    // SuppressUnmanagedCodeSecurity 属性を使用すべきです。
	    [DllImport(JitHelpers.QCall, CharSet = CharSet.Unicode)]
	    [SuppressUnmanagedCodeSecurity]
	    // QCallsは常にstatic externである必要があります。
	    private static extern bool Bar(int flags, string inString, StringHandleOnStack retString);

	    // 多くのQCallには、それを意味のあるやり方で世界に公開するための、
	    // 薄いマネージドラッパーがあります。
	    public string Bar(int flags)
	    {
	        string retString = null;

	        // QCallでは、文字列は、ローカル変数のアドレスを 
	        // JitHelpers.GetStringHandleメソッドに渡して返されます。
	        if (!Bar(flags, this.Id, JitHelpers.GetStringHandle(ref retString)))
	            FatalError();

	        return retString;
	    }
	}

### QCallの使用例 - アンマネージド部分

以下のコメントをあなたの実際のQCall実装に使い回さないでください。

このQCallのエントリポイントは、[vm\ecalllist.h][ecalllist]で、QCFuncEntryマクロを使って、テーブルに登録されています。以下の「QCallあるいはFCallのメソッドを登録する」を見て下さい。

[ecalllist]: https://github.com/dotnet/coreclr/blob/master/src/vm/ecalllist.h

	class FooNative
	{
	public:
	    // 全てのQCallは、staticで、QCALLTYPEでタグ付けされるべきです。
	    static
	    BOOL QCALLTYPE Bar(int flags, LPCWSTR wszString, QCall::StringHandleOnStack retString);
	};

	BOOL QCALLTYPE FooNative::Bar(int flags, LPCWSTR wszString, QCall::StringHandleOnStack retString)
	{
	    // 全てのQCallsはQCALL_CONTRACTとなるべきです。これは、THROWS; 
	    // GC_TRIGGERS; MODE_PREEMPTIVE; SO_TOLERANT のエイリアスです。
	    QCALL_CONTRACT;

	    // あるいは、特別な条件を指定したい場合は、QCALL_CHECKを代わりに使用して、
	    // 拡張されたコントラクトの形式を使用します:
	    // CONTRACTL {
	    //     QCALL_CHECK;
	    //     PRECONDITION(wszString != NULL);
	    // } CONTRACTL_END;

	    // QCALL_CONTRACTとBEGIN_QCALLの行の間に含まれるべきは、戻り値宣言のみです。
	    // （戻り値がある場合）
	    BOOL retVal = FALSE;

	    // メソッド本体は、BEGIN_QCALLとEND_QCALL のマクロの中に囲まれるべきです。
	    // これは、例外処理を適切に行うために必要です。
	    BEGIN_QCALL;

	    // 必要に応じて、引数を検証し、例外を投げます。
	    // 引数の検証がマネージドでなされるべきか、アンマネージドでなされるべきか、
	    // については、現在のところ合意はありません。
	    if (flags != 0)
	        COMPlusThrow(kArgumentException, L"InvalidFlags");

	    // QCallに渡した文字列がGCされる心配はありません。
	    // マーシャリングの過程でピン止めされます。
	    printf("%S", wszString);

	    // 戻り値の文字列をマネージドコードに返す、最も効率の良いやり方はこれです。
	    // StringBuilderを使う必要はありません。
	    retString.Set(L"Hello");

	    // BEGIN_QCALLとEND_QCALLの間でreturnすることはできません。
	    // 戻り値はヘルパー変数を経由して渡されなければなりません。
	    retVal = TRUE;

	    END_QCALL;

	    return retVal;
	}

## FCallの機能的な振る舞い

FCallを使うと、より複雑なコードによって、柔軟にオブジェクト参照を渡して回り、そしてあなたの首を絞めることができます。また、FCallメソッドは、一般的なコードパスと一緒にヘルパーメソッドフレームを立ち上げるか、あるいは短くないFCallについては、ガベージコレクションが発生するかどうかを明示的にポーリングする必要があります。それに失敗すると、もしマネージドコードがそのFCallメソッドを緊密なループの中で何度も呼び出すと、GC不足の問題が生じるでしょう。というのは、FCallは、そのスレッドがGCに対して協調モードで実行することのみを許している間に、実行されるのです。

FCallは、さまざまなglueを要求します。それらについてはここでは到底説明できません。詳細は [fcall.h][fcall] を見て下さい。

[fcall]: https://github.com/dotnet/coreclr/blob/master/src/vm/fcall.h

### GCの抜け穴とFCallとQCall

GCの抜け穴についての、より完全な議論については、[CLR Code Guide](../coding-guidelines/clr-code-guide.md)で見ることができます。["Is your code GC-safe?"](../coding-guidelines/clr-code-guide.md#is-your-code-gc-safe)を探して下さい。ここでは各論として、なぜFCallとQCallについて奇妙な規約があるのか、その理由をいくらか説明することを目的としています。

（TODO: coding-guidelinesも取り込む?）

FCallメソッドのパラメータに渡されるオブジェクト参照は、GC保護されていません。すなわち、もしGCが発生すると、それらの参照はそのオブジェクトのメモリの、新しい位置ではなく、古い位置を指し示しているかもしれない、ということです。この理由から、FCallは通常、パラメーター型としては "StringObject*" のようなものを受け入れた上で、それをGC発生の可能性がある操作より前に STRINGREF に明示的に変換する、といった規律に従っています。もしあなたが、オブジェクト参照を後でも利用できるようにしたいのであれば、GCをトリガーする前に、そのオブジェクト参照を保護しなければなりません。

FCallメソッド内部の全てのGCヒープ アロケーションは、ヘルパーメソッドフレームの中で起こらなければなりません。蓋然性は低いですが、もしGCヒープ上にメモリを確保したら、GCは予測不能なやり方で、死んだオブジェクトを回収したり、オブジェクトを移動する場合があります。そのため、そのメソッドの中で使用する全てのオブジェクト参照について、もしGCが発生した場合でも、それらのオブジェクト参照がそのメモリ上の新しい位置を指し示すように更新するよう、GCに手動でレポートしなければなりません。あなたのコードに含まれるマネージドオブジェクト（配列やStringなど）に対するあらゆる参照も、自動的に更新されることは無いので、メモリ確保したかもしれないあらゆる操作の後で、最初に使用する前に再度取得しなければなりません。参照のレポートは、GCPROTECTマクロによって、あるいはヘルパーメソッドフレームを立ち上げる時にそのパラメータとして渡すことで、行えます。

OBJECTREFクラスは、そのオブジェクトをcheckedビルドで参照解決する際に、毎回そのOBJECTREFが有効なオブジェクトを指し示していることをある程度確認するため、OBJECTREFのレポートや内部ポインターのアップデートを適切に行わない状態は、「GCの抜け穴」と呼ばれます。OBJECTREFが参照解決されたときに無効なオブジェクトを指し示していた場合は、 「無効なオブジェクト参照を検出しました。GCの抜け穴でしょうか?」（"Detected an invalid object reference. Possible GC hole?"）といったアサーション メッセージを見ることになるでしょう。

このアサーションは、残念ながら「マニュアル マネージド」コードを書いていると、簡単に引き起こすことになります。

ちなみに、QCallのプログラミング モデルは、オブジェクト参照のアドレスをスタックに積んで渡すことが強制されるため、ほとんどの場合においてGCの抜け穴を回避できるような、制限的なものです。これは、そのオブジェクト参照がJITのレポートのロジックによってGC保護され、またそのオブジェクトはGCヒープ上に確保されていないことから、実際のオブジェクト参照によって移動されることは無いためです。QCallを使うことをおすすめします。GCの抜け穴を発生させることが難しくなるためです。


### x86のFCallエピローグ ウォーカー

The managed stack walker needs to be able to find its way from FCalls. It is relative easy on newer platforms that define conventions for stack unwinding as part of the ABI. The stack unwinding conventions are not defined by ABI for x86. The runtime works around it by implementing a epilog walker. The epilog walker computes the FCall return address and callee save registers by simulating the FCall execution. This imposes limits on what constructs are allowed in the FCall implementation.

Complex constructs like stack allocated objects with destructors or exception handling in the FCall implementation may confuse the epilog walker. It leads to GC holes or crashes during stack walking. There is no exact list of what constructs should be avoided to prevent this class of bugs. An FCall implementation that is fine one day may break with the next C++ compiler update. We depend on stress runs & code coverage to find bugs in this area.

Setting a breakpoint inside an FCall implementation may confuse the epilog walker. It leads to an "Invalid breakpoint in a helpermethod frame epilog" assert inside [vm\i386\gmsx86.cpp](https://github.com/dotnet/coreclr/blob/master/src/vm/i386/gmsx86.cpp).

### FCall Example – Managed Part

Here's a real-world example from the String class:

	public partial sealed class String
	{
	    // Replaces all instances of oldChar with newChar.
	    [MethodImplAttribute(MethodImplOptions.InternalCall)]
	    public extern String Replace (char oldChar, char newChar);
	}

### FCall Example – Native Part

The FCall entrypoint has to be registered in tables in [vm\ecalllist.h][ecalllist] using FCFuncEntry macro. See "Registering your QCall or FCall Method".

Notice how oldBuffer and newBuffer (interior pointers into String instances) are re-fetched after allocating memory. Also, this method is an instance method in managed code, with the "this" parameter passed as the first argument. We use StringObject* as the argument type, then copy it into a STRINGREF so we get some error checking when we use it.

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

	    [... Removed some uninteresting code here for illustrative purposes...]

	    HELPER_METHOD_FRAME_BEGIN_RET_ATTRIB_2(Frame::FRAME_ATTR_RETURNOBJ, newString, thisRef);

	    //Get the length and allocate a new String
	    //We will definitely do an allocation here.
	    newString = NewString(length);

	    //After allocation, thisRef may have moved
	    oldBuffer = thisRef->GetBuffer();

	    //Get the buffers in both of the Strings.
	    newBuffer = newString->GetBuffer();

	    //Copy the characters, doing the replacement as we go.
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

The CLR must know the name of your QCall and FCall methods, both in terms of the managed class & method names, as well as which native methods to call. That is done in [ecalllist.h][ecalllist], with two arrays. The first array maps namespace & class names to an array of function elements. That array of function elements then maps individual method names & signatures to function pointers.

Say we defined an FCall method for String.Replace(char, char), in the example above. First, we need to ensure that we have an array of function elements for the String class.

	// Note these have to remain sorted by name:namespace pair (Assert will wack you if you
	    ...
	    FCClassElement("String", "System", gStringFuncs)
	    ...

Second, we must then ensure that gStringFuncs contains a proper entry for Replace. Note that if a method name has multiple overloads (such as String.Replace(String, String)), then we can specify a signature:

	FCFuncStart(gStringFuncs)
	    ...
	    FCFuncElement("IndexOf", COMString::IndexOfChar)
	    FCFuncElementSig("Replace", &gsig_IM_Char_Char_RetStr, COMString::Replace)
	    FCFuncElementSig("Replace", &gsig_IM_Str_Str_RetStr, COMString::ReplaceString)
	    ...
	FCFuncEnd()

There is a parallel QCFuncElement macro.

## Naming convention

Try to use normal name (e.g. no "_", "n" or "native" prefix) for all FCalls and QCalls. It is not good idea to embed that the function is implemented in VM in the name of the function for the following reasons:

- There are directly exposed public FCalls. These FCalls have to follow the naming convention for public APIs.
- The implementation of functions do move between CLR and mscorlib.dll. It is painful to change the name of the function in all call sites when this happens.

When necessary you can use "Internal" prefix to disambiguate the name of the FCall or QCall from public entry point (e.g. the public entry point does error checking and then calls shared worker function with exactly same signature). This is no different from how you would deal with this situation in pure managed code in BCL.

# Types with a Managed/Unmanaged Duality

Certain managed types must have a representation available in both managed & native code. You could ask whether the canonical definition of a type is in managed code or native code within the CLR, but the answer doesn't matter – the key thing is they must both be identical. This will allow the CLR's native code to access fields within a managed object in a very fast, easy to use manner. There is a more complex way of using essentially the CLR's equivalent of Reflection over MethodTables & FieldDescs to retrieve field values, but this probably doesn't perform as well as you'd like, and it isn't very usable. For commonly used types, it makes sense to declare a data structure in native code & attempt to keep the two in sync.

The CLR provides a binder for this purpose. After you define your managed & native classes, you should provide some clues to the binder to help ensure that the field offsets remain the same, to quickly spot when someone accidentally adds a field to only one definition of a type.

In [mscorlib.h][mscorlib.h], you can use macros ending in "_U" to describe a type, the name of fields in managed code, and the name of fields in a corresponding native data structure. Additionally, you can specify a list of methods, and reference them by name when you attempt to call them later.

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

# Calling Into Managed Code From Native

Clearly there are places where the CLR must call into managed code from native. For this purpose, we have added a MethodDescCallSite class to handle a lot of plumbing for you. Conceptually, all you need to do is find the MethodDesc\* for the method you want to call, find a managed object for the "this" pointer (if you're calling an instance method), pass in an array of arguments, and deal with the return value. Internally, you'll need to potentially toggle your thread's state to allow the GC to run in preemptive mode, etc.

Here's a simplified example. Note how this instance uses the binder described in the previous section to call SafeHandle's virtual ReleaseHandle method.

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

## Debugger

One limitation of FCalls today is that you cannot easily debug both managed code and FCalls easily in Visual Studio's Interop (or mixed mode) debugging. Setting a breakpoint today in an FCall and debugging with Interop debugging just doesn't work. This most likely won't be fixed.

# Physical Architecture

When the CLR starts up, mscorlib is loaded by a method called LoadBaseSystemClasses. Here, the base data types & other similar classes (like Exception) are loaded, and appropriate global pointers are set up to refer to mscorlib's types.

For FCalls, look in [fcall.h][fcall] for infrastructure, and [ecalllist.h][ecalllist] to properly inform the runtime about your FCall method.

For QCalls, look in [qcall.h][qcall] for associated infrastructure, and [ecalllist.h][ecalllist] to properly inform the runtime about your QCall method.

More general infrastructure and some native type definitions can be found in [object.h][object.h]. The binder uses mscorlib.h to associate managed & native classes.

[object.h]: https://github.com/dotnet/coreclr/blob/master/src/vm/object.h

