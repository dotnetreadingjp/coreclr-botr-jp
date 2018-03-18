（これは https://github.com/dotnet/coreclr/commits/master/Documentation/botr/xplat-minidump-generation.md の日本語訳です。対象rev.は 901e0d1）

<!-- # Introduction # -->
# はじめに #

<!-- Core dump generation on Linux and other non-Windows platforms has several challenges. Dumps can be very large and the default name/location of a dump is not consistent across all our supported platforms.  The size of a full core dumps can be controlled somewhat with the "coredump_filter" file/flags but even with the smallest settings may be still too large and may not contain all the managed state needed for debugging. By default, some platforms use _core_ as the name and place the core dump in the current directory from where the program is launched; others add the _pid_ to the name. Configuring the core name and location requires superuser permission. Requiring superuser to make this consistent is not a satisfactory option. -->
Linuxとその他Windows以外のプラットフォームにおけるコアダンプの生成にはいくつかの課題があります。ダンプのサイズは非常に大きなものとなる可能性があり、さらに、ダンプの既定のファイル名と出力先について、サポート対象の全てのプラットフォーム間での一貫性がありません。完全コアダンプのサイズは「coredump_filter」ファイルまたはフラグに類するもので制御可能ですが、サイズが最小となる設定であったとしても、依然としてサイズとしては大きすぎるかもしれず、かつデバッグに必要なマネージドの状態をすべて含んでいないかもしれません。既定では、いくつかのプラットフォームでは名前として _core_ を使用し、プログラムが起動した場所であるカレントディレクトリにコアダンプを出力します。それ以外のプラットフォームでは、ファイル名に _pid_ を追加します。コアダンプのファイル名と出力先を構成するには、スーパーユーザー権限が必要となります。一貫性を保つためにスーパーユーザーである必要があるという点で、選択肢として不十分となります。

<!-- Our goal is to generate core dumps that are on par with WER (Windows Error Reporting) crash dumps on any supported Linux platform. To the very least we want to enable the following:  -->
我々の目標は、サポート対象の任意のLinuxにおいて、WER（Windows Error Reporting）のクラッシュダンプと同等のコアダンプを生成することです。最低限、以下のことを可能にしたいと考えています：
<!-- - automatic generation of minimal size minidumps. The quality and quantity of the information contained in the dump should be on par with the information contained in a traditional Windows mini-dump.
- simple configurabilty by the user (not _su_!).  -->
- 最小限のサイズのミニダンプの自動生成。ミニダンプに含まれる情報の質と量は、従来のWindowsミニダンプに含まれる情報と同等であるべきです。
- ユーザーによってシンプルに構成できること（_su_ なしで！）

<!-- Our solution at this time is to intercept any unhandled exception in the PAL layer of the runtime and have coreclr itself trigger and generate a "mini" core dump.  -->
現時点での我々の解決策は、ランタイムのPALレイヤーであらゆる未処理例外に割り込み、coreclr自身に「ミニ」コアダンプを生成させることです。

<!-- Design -->
# 設計 #

<!-- We looked at the existing technologies like Breakpad and its derivatives (e.g.: an internal MS version called _msbreakpad_ from the SQL team....). Breakpad generates Windows minidumps but they are not compatible with existing tools like Windbg, etc. Msbreakpad even more so. There is a minidump to Linux core conversion utility but it seems like a wasted extra step. _Breakpad_ does allow the minidump to be generated in-process inside the signal handlers. It restricts the APIs to what was allowed in a "async" signal handler (like SIGSEGV) and has a small subset of the C++ runtime that was also similarly constrained. We also need to add the set of memory regions for the "managed" state which requires loading and using the _DAC_'s (*) enumerate memory interfaces. Loading modules is not allowed in an async signal handler but forking/execve is allowed so launching an utility that loads the _DAC_, enumerates the list of memory regions and writes the dump is the only reasonable option. It would also allow uploading the dump to a server too. -->
我々は、Breakpadやその派生（例：SQLチームが作成したMSの内部バージョンである _msbreakpad_）のような既存技術を調査しました。BreakpadはWindowsミニダンプを生成しますが、生成されるファイルはWindbg等の既存のツールと互換性がありません。msbreakpadも同様です。Linuxコアダンプ用のミニダンプ変換ユーティリティはありますが、変換は無駄な手順のように思えます。_Breakpad_ はシグナルハンドラー内でのプロセス内ミニダンプ生成を可能にします。「非同期」シグナルハンドラー（SIGSEGVなど）において実行可能なAPIには制限があり、かつ使用できるC++ランタイムは小さなサブセットに制限されます。<!--←ここ怪しい-->さらに、_DAC_ （*）のメモリ列挙インターフェイスの読み込みと使用に必要となる「マネージド」状態を持つ一連のメモリ領域を追加する必要もあります。非同期シグナルハンドラー内ではモジュールの読み込みは許可されていませんが、forkやexecveは許可されているので、_DAC_ を読み込むユーティリティを起動し、メモリ領域を列挙してダンプに書き込むというのが唯一の合理的な選択肢です。これによって、サーバーへのダンプのアップロードも可能になります。

<!-- \* The _DAC_ is a special build of parts of the coreclr runtime that allows inspection of the runtime's managed state (stacks, variables, GC state heaps) out of context. One of the many interfaces it provides is [ICLRDataEnumMemoryRegions](https://github.com/dotnet/coreclr/blob/master/src/debug/daccess/dacimpl.h) which enumerates all the managed state a minidump would require to enable a fuitful debugging experience. -->
\* _DAC_ はランタイムのマネージド状態（スタック、変数、GC状態ヒープ）をコンテキストの外部から調査可能にする、coreclrランタイムの特別なビルドの一部です。DACが提供する数多くのインターフェイスの一つに[ICLRDataEnumMemoryRegions](https://github.com/dotnet/coreclr/blob/master/src/debug/daccess/dacimpl.h)があり、これはミニダンプによる有益なデバッグエクスペリエンスを可能にするのに必須のマネージド状態をすべて列挙します。

<!-- _Breakpad_ could have still been used out of context in the generation utility but there seemed no value to their Windows-like minidump format when it would have to be converted to the native Linux core format away because in most scenarios using the platform tools like _lldb_ is necessary. It also adds a coreclr build dependency on Google's _Breakpad_ or SQL's _msbreakpad_ source repo. The only advantage is that the breakpad minidumps may be a little smaller because minidumps memory regions are byte granule and Linux core memory regions need to be page granule. -->
_Breakpad_ 生成ユーティリティにおいてコンテキストの外部で依然として使用される可能性がありますが、ほとんどのシナリオにおいては _lldb_ のようなプラットフォームツールを使用する必要があるため、Breakpadが生成するWindowsライクなミニダンプ形式をネイティブのLinuxコアダンプ形式に変換する必要があるので、Breakpadに価値はないように思えます。さらに、coreclrのビルド依存先としてGoogleの _Breakpad_ やSQLの _msbreakpad_ ソースリポジトリを加えることにもなります。唯一の利点は、Linuxコアダンプのメモリ領域がページの粒度であるのに対して、ミニダンプのメモリ領域はバイトの粒度であるために、Breakpadのミニダンプは少しばかり小さくなる可能性があることです。

<!-- # Implementation Details # -->
# 実装の詳細 #

### Linux ###

<!-- Core dump generation is triggered anytime coreclr is going to abort (via [PROCAbort()](https://github.com/dotnet/coreclr/blob/master/src/pal/src/include/pal/process.h)) the process because of an unhandled managed exception or an async signal like SIGSEGV, SIGILL, SIGFPE, etc. The _createdump_ utility is located in the same directory as libcoreclr.so and is launched with fork/execve. The child _createdump_ process is given permission to ptrace and access to the various special /proc files of the crashing process which waits until _createdump_ finishes. -->
コアダンプ生成は、coreclrが（[PROCAbort()](https://github.com/dotnet/coreclr/blob/master/src/pal/src/include/pal/process.h)経由で）プロセスのアボートをしようとした任意の時点で開始されます。プロセスのアボートは、未処理のマネージド例外か、SIGSEGV、SIGILL、SIGFPE等の非同期シグナルによって起こります。_createdump_ ユーティリティがlibcoreclr.soと同じディレクトリに配置されており、forkまたはexecveによって起動されます。子 _createdump_ プロセスはptraceに必要な権限と、_createdump_ が完了するまで待機することになるクラッシュ中のプロセスのさまざまな特殊な/procファイル群へのアクセス許可が与えられています。

<!-- The _createdump_ utility starts by using ptrace to enumerate and suspend all the threads in the target process. The process and thread info (status, registers, etc.) is gathered. The auxv entries and _DSO_ info is enumerated. _DSO_ is the in memory data structures that described the shared modules loaded by the target. This memory is needed in the dump by gdb and lldb to enumerate the shared modules loaded and access their symbols. The module memory mappings are gathered from /proc/$pid/maps. None of the program or shared modules memory regions are explicitly added to dump's memory regions. The _DAC_ is loaded and the enumerate memory region interfaces are used to build the memory regions list just like on Windows. The threads stacks and one page of code around the IP are added. The byte sized regions are rounded up to pages and then combined into contagious regions. -->
_createdump_ ユーティリティは、最初にptraceを使用して対象のプロセスの全てのスレッドを列挙・サスペンドします。そのプロセスとスレッドの情報（ステータスやレジスターなど）が収集されます。auxvエントリと _DSO_ 情報が列挙されます。_DSO_ は対象によって読み込まれた共有モジュールを記述するインメモリデータ構造です。このメモリは、gdbとlldbが読み込み済みの共有モジュールを列挙し、それらのシンボルにアクセスするために、ダンプに含まれている必要があります。_DAC_が読み込まれ、Windowsと同等のメモリ領域リストを構築するためにメモリ領域列挙インターフェイスが使用されます。スレッドのスタックと、IPの周囲にあるコードの1ページが追加されます。バイトサイズの領域はページサイズに切り上げられ、連続した領域に結合されます。

<!-- All the memory mappings from /proc/$pid/maps are in the PT_LOAD sections even though the memory is not actually in the dump. They have a file offset/size of 0.  -->
そのメモリがダンプには実際に含まれないにもかかわらず、/proc/$pid/maps由来の全てのメモリマッピングがPT\_LOADセクションに含まれます。それらのファイルオフセットとサイズは0です。

<!-- After all the process crash information has been gathered, the ELF core dump with written. The main ELF header created and written. The PT\_LOAD note section is written one entry for each memory region in the dump. The process info, auxv data and NT_FILE entries are written to core. The NT\_FILE entries are built from module memory mappings from /proc/$pid/maps. The threads state and registers are then written. Lastly all the memory regions gather above by the _DAC_, etc. are read from the target process and written to the core dump. All the threads in the target process are resumed and _createdump_ terminates. -->
全てのプロセスクラッシュ情報が収集されると、ELFコアダンプが書き込まれます。メインELFヘッダーが作成されて書き込まれます。ダンプファイルの各メモリ領域ごとにPT\_LOADノートセクションが1エントリ書き込まれます。プロセス情報、auxvデータ、NT\_FILEエントリ群がコアダンプファイルに書き込まれます。NT\_FILEエントリ群は/proc/$pid/maps由来のモジュールメモリマッピングから構築されます。それから、スレッド状態とレジスターが書き込まれます。最後に、_DAC_ 等によって収集された全てのメモリ領域が対象のプロセスから読み取られ、コアダンプファイルに書き込まれます。対象のプロセスの全てのスレッドがリジュームされ、_createdump_ が終了します。

<!-- **Severe memory corruption** -->
**厄介なメモリ破壊**

<!-- As long as control can making it to the signal/abort handler and the fork/execve of the utility succeeds then the _DAC_ memory enumeration interfaces can handle corruption to a point; the resulting dump just may not have enough managed state to be useful. We could investigate detecting this case and writing a full core dump. -->
制御機構がシグナルハンドラーまたはアボートハンドラーを作成することができ、かつユーティリティのforkとexecveが成功するならば、_DAC_ のメモリ列挙インターフェイスはそのメモリ破壊を処理できます。結果として生成されるダンプに、有用なマネージド状態が十分に保持されていないかもしれない、というだけです。我々は、このケースの検出方法を調査し、完全なコアダンプを書き込みたいと考えています。

<!-- **Stack overflow exception** -->
**スタックオーバーフロー例外**

<!-- Like the severe memory corruption case, if the signal handler (`SIGSEGV`) gets control it can detect most stack overflow cases and does trigger a core dump. There are still many cases where this doesn't happen and the OS just terminates the process. -->
厄介なメモリ破壊のケースと同様に、シグナルハンドラー（`SIGSEGV`）に制御が移ったならば、ハンドラーはほとんどのスタックオーバーフローのケースを検出できるので、コアダンプを開始します。ただし、ハンドラーに制御が移ることなく、OSが単にプロセスを強制終了するケースが多数あります。

### FreeBSD/OpenBSD/NetBSD ###

<!-- There will be some differences gathering the crash information but these platforms still use ELF format core dumps so that part of the utility should be much different. The mechanism used for Linux to give _createdump_ permission to use ptrace and access the /proc doesn't exists on these platforms. -->
クラッシュ情報の収集にいくつかの違いは生じますが、これらのプラットフォームはコアダンプにELF形式を使用しているので、ユーティリティのいくつかはそれほど違ってこないでしょう。Linuxで _createdump_ にptraceの使用権限と/procへのアクセス権を与えるために使用した仕組みは、これらのプラットフォームに存在しません。

### OS X ###

<!-- Gathering the crash information on OS X will be quite a bit different than Linux and the core dump will be written in the Mach-O format instead of ELF. The OS X support currently has not been implemented. -->
OS Xにおけるクラッシュ情報の収集はLinuxとは大きく異なるものとなり、コアダンプはELFではなくMach-O形式で書き込まれることでしょう。OS Xサポートは現在実装されていません。

<!-- # Configuration/Policy # -->
# 構成とポリシー #

<!-- Any configuration or policy is set with environment variables which are passed as options to the _createdump_ utility. -->
_createdump_ ユーティリティに対するオプションとして渡される環境変数を使用して、構成またはポリシーを設定できます。

<!-- Environment variables supported: -->
サポートされる環境変数：

<!-- - `COMPlus_DbgEnableMiniDump`: if set to "1", enables this core dump generation. The default is NOT to generate a dump.
- `COMPlus_DbgMiniDumpType`: if set to "1" generates _MiniDumpNormal_, "2" _MiniDumpWithPrivateReadWriteMemory_, "3" _MiniDumpFilterTriage_, "4" _MiniDumpWithFullMemory_. Default is _MiniDumpNormal_.
- `COMPlus_DbgMiniDumpName`: if set, use as the template to create the dump path and file name. The pid can be placed in the name with %d. The default is _/tmp/coredump.%d_.
- `COMPlus_CreateDumpDiagnostics`: if set to "1", enables the _createdump_ utilities diagnostic messages (TRACE macro). -->
- `COMPlus_DbgEnableMiniDump`："1" に設定すると、本書のコアダンプ生成が有効になります。既定の状態ではダンプを生成 **しません**。
- `COMPlus_DbgMiniDumpType`："1" に設定すると _MiniDumpNormal_ を、"2" に設定すると _MiniDumpWithPrivateReadWriteMemory_ を、"3" に設定すると _MiniDumpFilterTriage_ を、"4" に設定すると _MiniDumpWithFullMemory_ を生成します。既定では _MiniDumpNormal_ です。
- `COMPlus_DbgMiniDumpName`：設定されている場合、ダンプのパスとファイル名を作成するときのテンプレートとして使用されます。ファイル名に%dと記述することで、そこをpidで置換するようにできます。既定では _/tmp/coredump.%d_ です。
- `COMPlus_CreateDumpDiagnostics`："1" に設定すると、_createdump_ ユーティリティの診断メッセージ（TRACEマクロ）が有効になります。

<!-- (Please refer to MSDN for the meaning of the [minidump enum values](https://msdn.microsoft.com/en-us/library/windows/desktop/ms680519(v=vs.85).aspx) reported above) -->
（上記のミニダンプの種類の意味については、MSDNの[minidump enum values](https://msdn.microsoft.com/en-us/library/windows/desktop/ms680519(v=vs.85).aspx) を参照してください）

<!-- **Utility command line options**: -->
**ユーティリティコマンドラインオプション**：

    createdump [options] pid
    -f, --name - dump path and file name. The pid can be placed in the name with %d. The default is "/tmp/coredump.%d"
    -n, --normal - create minidump (default).
    -h, --withheap - create minidump with heap.
    -t, --triage - create triage minidump.
    -u, --full - create full core dump.
    -d, --diag - enable diagnostic messages.

    createdump [オプション] pid
    -f, --name - ダンプのパスとファイル名。ファイル名の%dはpidで置き換えられます。既定値は "/tmp/coredump.%d" です。
    -n, --normal - ミニダンプを作成します（既定）。
    -h, --withheap - ヒープ付きのミニダンプを作成します。
    -t, --triage - トリアージミニダンプを作成します。
    -u, --full - 完全コアダンプを作成します。
    -d, --diag - 診断メッセージを有効にします。

<!-- # Testing # -->
# テスト実施 #

<!-- The test plan is to modify the SOS tests in the (still) private debuggertests repo to trigger and use the core minidumps generated. Debugging managed core dumps on Linux is not supported by _mdbg_ at this time until we have a ELF core dump reader so only the SOS tests (which use _lldb_ on Linux) will be modified. -->
コアミニダンプを実行してその結果を使用するSOSのテストの修正についてのテスト計画は（まだ）プライベートのdebuggertestsリポジトリにあります。Linux上でのマネージドコアダンプのデバッグは、現時点で、ELFコアダンプリーダーができるまで _mdbg_ ではサポートされないので、SOSのテスト（Linuxでは _lldb_ を使用）のみが修正される予定です。

<!-- # Open Issues # -->
# オープン中のissue #

<!-- - May need more than just the pid for decorating dump names for docker containers because I think the _pid_ is always 1.
- Do we need all the memory mappings from `/proc/$pid/maps` in the PT\_LOAD sections even though the memory is not actually in the dump? They have a file offset/size of 0. Full dumps generated by the system or _gdb_ do have these un-backed regions.
- There is no way to get the signal number, etc. that causes the abort from the _createdump_ utility using _ptrace_ or a /proc file. It would have to be passed from CoreCLR on the command line.
- Do we need the "dynamic" sections of each shared module in the core dump? It is part of the "link_map" entry enumerated when gathering the _DSO_ information.
- There may be more versioning and/or build id information needed to be added to the dump.
- It is unclear exactly what cases stack overflow does not get control in the signal handler and when the OS just aborts the process. -->
- _pid_ が常に1になるはずなので、Dockerコンテナーようにpid以外のダンプファイル名を修飾する方法が必要と思われます。
- そのメモリがダンプには実際に含まれないにもかかわらず、/proc/$pid/maps由来の全てのメモリマッピングがPT\_LOADセクションに含まれる必要はあるのでしょうか？ それらのファイルオフセットとサイズは0です。システムまたは _gdb_ によって生成されるフルダンプにはこれらのun-backed領域が含まれます。
- シグナル番号等を取得する手段がないので、_createdump_ からのアボートに _ptrace_ または /proc ファイルを使用せざるを得ません。コマンドラインでCoreCLRから渡す必要があるでしょう。
- コアダンプに各共有モジュールの「dynamic」セクションは必要でしょうか？ これは _DSO_ 情報の収集時に列挙される「link_map」エントリの一部です。
- ダンプに対して、より詳細なバージョン情報と、ビルドID情報を加える必要があるように思えます。
- スタックオーバーフローでシグナルハンドラーに制御が移らずにOSが単にプロセスを強制終了するのが、正確にどういった場合なのかが明白ではありません。
