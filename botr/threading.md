CLRスレッディング概要
======================
(これは https://github.com/dotnet/coreclr/blob/8d3936bff7ae46a5a964b15b5f0bc3eb8d4e32db/Documentation/botr/threading.md の日本語訳です。対象rev.は 8d3936b）

マネージドスレッド対ネイティブスレッド
==========================

マネージドコードは"マネージドスレッド"上で動作します。これはオペレーティングシステムが提供するネイティブスレッドとは区別されます。
ネイティブスレッドは物理マシン上でネイティブコードを実行するスレッドですが、マネージドスレッドはCLRの仮想マシン上でコードを実行する仮想スレッドです。

JITコンパイラーがILの"仮想的な"命令を物理マシン上で動作するネイティブの命令に対応付けるのと同様に、
CLRのスレッド基盤は"仮想的な"マネージドスレッドをオペレーティングシステムが提供するネイティブスレッドに対応付けます。

ある1つのマネージドスレッドはどんなときでも、実行のために1つのネイティブスレッドが割り当てられているか、割り当てられていないかのどちらかです。
たとえば、（"new System.Threading.Thread"によって）作成されたマネージドスレッドが、まだ（System.Threading.Thread.Startによって）開始されていなければ、そのマネージドスレッドにはネイティブスレッドが割り当てられていません。
同様に、原理的には、ある1つのマネージドスレッドが実行中に複数のネイティブスレッド間を移動する可能性もあります。とはいえ実際には、現在のCLRはそのような動作をサポートしていません。

マネージドコードに対して公開されているThreadインターフェースは、背後にあるネイティブスレッドの詳細を意図的に隠しています。それは次の理由からです。

- マネージドスレッドは必ずしも1つのネイティブスレッドに対応付けられているわけではありません。（もしかしたらどのネイティブスレッドにも対応付いていないかもしれません。）
- オペレーティングシステムによって、ネイティブスレッドの抽象も異なります。
- 原理的には、マネージドスレッドとは"仮想化されている"ものです。

CLRはマネージドスレッドの抽象に相当するものを提供します。そしてそれはCLR自身に実装されているものです。
たとえば、CLRはオペレーティングシステムのスレッドローカルストレージ（TLS）機構を露出するのではなく、その代わりに管理された"スレッド静的"な変数を提供します。
同様に、CLRはネイティブスレッドの"スレッドID"を露出するのではなく、その代わりにOSとは独立して生成された"マネージドスレッドID"を提供します。
ただし、背後にあるネイティブスレッドの詳細情報の中には、診断に利用する目的のために、System.Diagnostics名前空間の型を通じて取得できるものもあります。

マネージドスレッドには、ネイティブスレッドでは通常必要とされない追加の機構も必要です。
まず、マネージドスレッドはGC参照を自身のスタック上に保持するので、CLRはGCが起こるたびにそれらの参照を列挙（して、おそらく修正）することができなければなりません。そのために、CLRは各マネージドスレッドを"サスペンド"する（すべてのGC参照を見つけられる時点で停止する）必要があります。
次に、AppDomainがアンロードされた場合、CLRはそのAppDomainにはコードを実行しているスレッドがないことを保証しなければなりません。そのために、スレッドをAppDomainからほどくことができる必要があります。CLRはそのようなスレッドにThreadAbortExceptionを注入します。

データ構造
===============

すべてのマネージドスレッドには関連づいたThreadオブジェクト（訳注：System.Threading.Threadクラスとは異なった、内部的なデータ構造）があります。
Threadオブジェクトは[threads.h][threads.h]で定義されています。
このオブジェクトは対応するマネージドスレッドについてVMが知っておくべきことをすべて追跡しています。
VMが知っておくべきことの中には _不可欠_ なもの、たとえばスレッドの現在のGCモードやFrameのチェインなどもあれば、
単に性能上の理由でスレッドごとに割り当てられていること（たとえばアリーナ形式の高速なメモリアロケーターなど）も多くあります。

ThreadオブジェクトはすべてThreadStore（こちらも[threads.h][threads.h]で定義されています）に格納されます。
これは既知のThreadオブジェクトをすべて格納する単純なリストです。
マネージドスレッドをすべて列挙するには、まずThreadStoreLockを獲得して、それからThreadStore::GetAllThreadListを使ってThreadオブジェクトをすべて列挙します。
このリストにはその時点でネイティブスレッドが割り当てられていないマネージドスレッド（たとえば、まだ開始していないとか、あるいは割り当てられたネイティブスレッドがすでに終了したなど）も含まれます。

[threads.h]: (https://github.com/dotnet/coreclr/blob/master/src/vm/threads.h)

現在ネイティブスレッドが割り当てられているマネージドスレッドは、ネイティブコードからアクセス可能です。
それには、割り当てられているネイティブスレッドがそれぞれ持っている、ネイティブスレッドのローカルストレージ（TLS）スロットを介します。
それにより、そのネイティブスレッド上で動作しているコードは、GetThread()を呼ぶことで対応するThreadオブジェクトを得ることができます。

それに加えて、多くのマネージドスレッドは、 _管理された_ Threadオブジェクト（System.Threading.Thread）を持っています。これはネイティブスレッドオブジェクトとは区別されるものです。
このマネージドスレッドオブジェクトは、マネージドコードがスレッドと相互作用するためのメソッド群を提供します。
また、このオブジェクトの大部分はネイティブスレッドオブジェクトが提供する機構のラッパーです。
現在のマネージドスレッドオブジェクトは、（マネージドコードから）Thread.CurrentThreadを介してアクセス可能です。

デバッガーでは、"!Threads"というSOS拡張コマンドを使用して、ThreadStore内のThreadオブジェクトをすべて列挙することができます。

スレッドの生存期間
================

マネージドスレッドは次の状況で生成されます。

1. マネージドコードがSystem.Threading.Threadを使って新しいスレッドを生成するようにCLRに明示的に要求するとき。
2. CLRが特定のマネージドスレッドを直接生成するとき（下記の「特別なスレッド」を参照のこと）。
3. まだマネージドスレッドに対応付けられていないネイティブスレッド上で、ネイティブコードが（「逆P/Invoke」またはCOM相互運用を使用して）マネージドコードを呼び出すとき。
4. あるマネージドプロセスが開始するとき（プロセスのメインスレッドがMainメソッドを呼び出します）。

上記の #1 と #2 の場合は、CLRにはマネージドスレッドの背後にあるネイティブスレッドを生成する責任があります。
これはマネージドスレッドが実際に _開始_ するまでは実施されません。
開始された後は、生成されたネイティブスレッドはCLRが「所有」し、CLRはネイティブスレッドの生存期間について責任があります。
これらの場合では、CLRは生成されたスレッドの存在を認識しています。そもそもスレッドを生成したのはCLRだという事実があるからです。

上記の #3 と #4 の場合は、マネージドスレッドが生成される前からネイティブスレッドは存在しており、CLRの外にあるコードによって所有されています。
CLRはそのネイティブスレッドの生存期間について責任がありません。
CLRはネイティブスレッドがマネージドコードを呼び出そうとした時に初めてそれらのスレッドの存在を認識します。

ネイティブスレッドが終了した時はDllMain関数を通じてCLRに通知されます。これはOSの「ローダーロック」の内側で起こるため、この通知を処理する際に（安全に）できることはほとんどありません。ですので、終了するスレッドは、マネージドスレッドに関連づいたデータ構造について、破棄するのではなく単に「終了」とマークした上で、ファイナライザースレッドに開始のシグナルを送ります。それを受けてファイナライザースレッドはThreadStore内のスレッドを一通り眺め、終了していて _かつ_ マネージドコードから到達不可能なスレッドがあれば破棄します。

一時中断
==========

CLRがGCを実行するためにはすべてのマネージドオブジェクトを見つけ出せる必要があります。
マネージドコードは定常的にGCヒープにアクセスして、スタックやレジスタに保持されている参照を操作しています。
CLRはすべてのマネージドスレッドが停止していることを保証しなければなりません。
それによって（ヒープが変更されないので）、安全に確実にすべてのマネージドオブジェクトを見つけ出すことができます。
スレッドは _セーフポイント_ でのみ停止します。その時に、生存している参照を探すためにレジスタとスタック位置を調査できます。

別の方法は、GCヒープ、および全スレッドのスタックとレジスターの状態が、複数のスレッドからアクセスされる「共有状態」になることです。
世のほとんどの共有状態と同様に、状態を守るにはある種の「ロック」が要求されます。
マネージドコードはヒープにアクセスする間そのロックを保持しなければなりませんし、ロックを開放できるのはセーフポイントに達したときだけです。

CLRはそのロックをスレッドの「GCモード」として参照します。
「協調（cooperative）モード」のスレッドは自身のロックを保持します。
GCを進行させるためには、そのスレッドは（ロックを解放することで）GCと「協調」しなければなりません。
「割り込み（preemptive）モード」のスレッドは自身のロックを保持しません――GCは「割り込み」によって進行する可能性があります。なぜならそのスレッドはGCヒープにアクセスしていないことが分かっているからです。

GCは全マネージドスレッドが「割り込み」モードにある（ロックを保持していない）時だけ進行します。全マネージドスレッドを割り込みモードに移行するプロセスは「GC一時中断」または「実行エンジン（EE）の中断」と呼ばれています。

この「ロック」をナイーブに実装するとすれば、個々のマネージドスレッドがGCヒープにアクセスするたびに本物のロックを実際に獲得・解放するというものになるでしょう。
その場合GCは単にそれぞれのスレッドについてロックを獲得しようと試みるだけでよくなります。
全スレッドのロックを獲得してしまえば、GCを実行しても安全ということになります。

しかしながら、このナイーブな手法は2つの理由で満足のいくものではありません。
1つは、マネージドコードがロックを獲得・解放するために（少なくとも、GCがロックを獲得しようとしていないかチェックする、いわゆる「GCポーリング」に）多くの時間を使わないとならないことです。もう1つは、JITが「GC情報」を出力しないとならないことです。それはJIT化されたコードのあらゆる点におけるスタックとレジスターのレイアウトを表すものです。この情報は多量のメモリを必要とするでしょう。

私たちはこのナイーブなアプローチを洗練させ、JIT化されたマネージコードを「部分的に割り込み可能」なコードと「完全に割り込み可能」なコードの2つに分離しました。
部分的に割り込み可能なコードでは、セーフポイントは他メソッドの呼び出しと、明示的な「GCポーリング」の場所だけとなります。
そしてそこでは、JITはGCが延期されているかチェックするコードを出力します。GC情報はそれらの場所においてだけ出力されればよくなります。
完全に割り込み可能なコードでは、すべての命令がセーフポイントであり、JITはすべての命令についてGC情報を出力します――ただしJITはGCポーリングを出力しません。
その代わり、完全に割り込み可能なコードはスレッドのハイジャック（本ドキュメントの後ろの方で説明します）によって「割り込まれる」可能性があります。
完全に割り込み可能なコードと部分的に割り込み可能なコードのどちらを出力するかをJITが判断する際にはヒューリスティックスが用いられます。
そのヒューリスティックスはコード品質、GC情報のサイズ、GC一時中断の遅延時間について最善のトレードオフを発見するものです。

ここまでを踏まえて、3つの基礎的な処理が定義されています。協調モードへの突入、協調モードからの脱出、そして実行エンジン（EE）の中断です。

協調モードへの突入
-------------------------

Thread::DisablePreemptiveGCを呼ぶことでスレッドは協調モードに入ります。これは現在のスレッドの「ロック」を獲得します。詳細は以下の通りです。

1. もしGCが処理中（GCがロックを保持している）なら、GCが完了するまでブロックします。
2. スレッドを協調モードにあるとマークします。スレッドが再び割り込みモードに入るまでGCは進行しません。

これらの2つのステップはアトミックに進行します。

協調モードからの脱出
------------------------

Thread::EnablePreemptiveGCを呼ぶことでスレッドは割り込みモードに入ります（ロックを解放します）。
これは単に、スレッドが協調モードではなくなったとマークして、GCスレッドに対して進行してもよいと伝えるだけです。

実行エンジン（EE）の中断
-----------------

GCが稼働しようとするときの最初のステップはEEを中断することです。これはGCHeap::SuspendEEで実行されます。プロセスは以下の通りです。

1. グローバルなフラグ（g\_fTrapReturningThreads）を立てて、GCが動作中であることを示します。協調モードに入ろうとするスレッドはすべて、GCが完了するまでブロックされます。
2. 現在協調モードで動作しているスレッドをすべて探し出します。見つかったスレッドのそれぞれについて、スレッドをハイジャックして協調モードから脱出させようとします。
3. 協調モードで動作するスレッドがなくなるまで繰り返します。

ハイジャック
---------

Thread::SysSuspendForGCによってGC一時中断のためのハイジャックが起こります。このメソッドは現在協調モードで動作しているすべてのスレッドについて、「セーフポイント」で協調モードから抜けるように強制しようとします。
これは（ThreadStoreを調査して）すべてのマネージドスレッドを列挙したうえで、現在協調モードで動作しているスレッドごとに以下のことを行います。

1. 裏側にあるネイティブスレッドを一時停止します。これにはWin32のSuspendThread APIが用いられます。このAPIはスレッドの実行を強制的に止めます。停止地点は実行中のランダムな場所です（セーフポイントである必要はありません）。
2. GetThreadContextでスレッドの現在のコンテキストを取得します。これはOSの概念で、コンテキストとはスレッドの現在のレジスター状態を表すものです。これによってスレッドの命令ポインターを調査できるので、したがって現在実行中のコードの種類を判別できます。
3. スレッドが協調モードか再確認します。そのスレッドが一時停止できるようになる前にすでに協調モードから抜け出ているかもしれないからです。もしそうなら、そのスレッドは危険な領域にいます。スレッドは任意のネイティブコードを実行している可能性があり、デッドロックを避けるには即座に再開されなければなりません。
4. スレッドがマネージドコードを実行しているかチェックします。可能性としてあり得るのは、スレッドが協調モードでネイティブのVMコードを実行しているという状況です（下記の「同期」の節を参照のこと）。その場合には上のステップ同様、スレッドは即座に再開されなければなりません。
5. このステップまで来れば、スレッドはマネージドコード内で中断しています。コードが完全に割り込み可能なのか部分的に割り込み可能なのかによって、次のどちらかが実行されます。
  * 完全に割り込み可能な場合は、どの地点でGCを実行しても安全です。なぜならスレッドは、定義上、セーフポイントにいるからです。このままスレッドを一時停止させることも（安全であれば）考えられますが、OSには多くの歴史的なバグがあるのでそのように動作させることはできません。すでに取得しているコンテキストが壊れる可能性があるためです。その代わりに、スレッドの命令ポインターは上書きされ、より完全なコンテキストをキャプチャするスタブにリダイレクトされます。それから協調モードから出て、GCが完了するのを待ち、再度協調モードに入り、スレッドを以前の状態に復元します。
  * 部分的に割り込み可能な場合は、スレッドは、定義上、セーフポイントにはいません。しかし、呼び出し元は（メソッドへの推移で）セーフポイントへと移動します。それを踏まえて、CLRは最上位のスタックフレームのリターンアドレスを「ハイジャック」（スタック上のその場所を物理的に上書き）し、完全に割り込み可能な場合と同様のスタブへと置き換えます。メソッドから制御が戻るときには、もはや実際の呼び出し元へと戻ることはなく、その代わりにスタブへと移動します（その地点より前に、メソッドはJITによって挿入されたGCポーリングを実行するかもしれません。その場合は協調モードから抜け、ハイジャックは取り消されることになります）。

ThreadAbort / AppDomain-Unload
==============================

In order to unload an AppDomain, the CLR must ensure that no thread is running in that AppDomain. To accomplish this, all マネージドスレッドs are enumerated, and "abort" any threads which have stack frames belonging to the AppDomain being unloaded. A ThreadAbortException is "injected" into the running thread, which  causes the thread to unwind (executing backout code along the way) until it is no longer executing in the AppDomain, at which point the ThreadAbortException is translated into an AppDomainUnloaded exception.

ThreadAbortException is a special type of exception. It can be caught by user code, but the CLR ensures that the exception will be rethrown after the user's exception handler is executed. Thus ThreadAbortException is sometimes referred to as "uncatchable," though this is not strictly true.

A ThreadAbortException is typically 'thrown' by simply setting a bit on the マネージドスレッド marking it as "aborting." This bit is checked by various parts of the CLR (most notably, every return from a p/invoke) and often times setting this bit is all that is needed to get the thread aborted in a timely manner.

However, if the thread is, for example, executing a long-running managed loop, it may never check this bit. To get such a thread to abort faster, the thread i "hijacked" and forced to raise a ThreadAbortException. This hijacking is done in the same way as GC suspension, except that the stubs that the thread is redirected to will cause a ThreadAbortException to be raised, rather than waiting for a GC to complete.

This hijacking means that a ThreadAbortException can be raised at essentially any arbitrary point in マネージドコード. This makes it extremely difficult for マネージドコード to deal successfully with a ThreadAbortException. It is therefore unwise to use this mechanism for any purpose other than AppDomain-Unload, which ensures that any state corrupted by the ThreadAbort will be cleaned up along with the AppDomain.

Synchronization: Managed
========================

マネージドコード has access to many synchronization primitives, collected within the System.Threading namespace. These include wrappers for native OS primitives like Mutex, Event, and Semaphore objects, as well as some abstractions such as Barriers and SpinLocks. However, the primary synchronization mechanism used by most マネージドコード is System.Threading.Monitor, which provides a high-performance locking facility on _any managed object_, and additionally provides "condition variable" semantics for signaling changes in the state protected by a lock.

Monitor is implemented as a "hybrid lock;" it has features of both a spin-lock and a kernel-based lock like a Mutex. The idea is that most locks are held only briefly, so it takes less time to simply spin-wait for the lock to be released, than it would to make a call into the kernel to block the thread. It is important not to waste CPU cycles spinning, so if the lock has not been acquired after a brief period of spinning, the implementation falls back to blocking in the kernel.

Because any object may potentially be used as a lock/condition variable, every object must have a location in which to store the lock information. This is done with "object headers" and "sync blocks."

The object header is a machine-word-sized field that precedes every managed object. It is used for many purposes, such as storing the object's hash code. One such purpose is holding the object's lock state. If more per-object data is needed than will fit in the object header, we "inflate" the object by creating a "sync block."

Sync blocks are stored in the Sync Block Table, and are addressed by sync block indexes. Each object with an associated sync block has the index of that index in the object's object header.

The details of object headers and sync blocks are defined in [syncblk.h][syncblk.h]/[.cpp][syncblk.cpp].

[syncblk.h]: https://github.com/dotnet/coreclr/blob/master/src/vm/syncblk.h
[syncblk.cpp]: https://github.com/dotnet/coreclr/blob/master/src/vm/syncblk.cpp

If there is room on the object header, Monitor stores the マネージドスレッド ID of the thread that currently holds the lock on the object (or zero (0) if no thread holds the lock). Acquiring the lock in this case is a simple matter of spin-waiting until the object header's thread ID is zero, and then atomically setting it to the current thread's マネージドスレッド ID.

If the lock cannot be acquired in this manner after some number of spins, or the object header is already being used for other purposes, a sync block must be created for the object. This has additional data, including an event that can be used to block the current thread, allowing us to stop spinning and efficiently wait for the lock to be released.

An object that is used as a condition variable (via Monitor.Wait and Monitor.Pulse) must always be inflated, as there is not enough room in the sync block to hold the required state.

Synchronization: Native
=======================

The native portion of the CLR must also be aware of threading, as it will be invoked by マネージドコード on multiple threads. This requires native synchronization mechanisms, such as locks, events, etc.

The ITaskHost API allows a host to override many aspects of マネージドスレッドing, including thread creation, destruction, and synchronization. The ability of a host to override native synchronization means that VM code can generally not use native synchronization primitives (Critical Sections, Mutexes, Events, etc.) directly, but rather must use the VM's wrappers over these.

Additionally, as described above, GC suspension is a special kind of "lock" that affects nearly every aspect of the CLR. ネイティブコード in the VM may enter "cooperative" mode if it must manipulate GC heap objects, and thus the "GC suspension lock" becomes one of the most important synchronization mechanisms in native VM code, as well as managed.

The major synchronization mechanisms used in native VM code are the GC mode, and Crst.

GC Mode
-------

As discussed above, all マネージドコード runs in cooperative mode, because it may manipulate the GC heap. Generally, ネイティブコード does not touch managed objects, and thus runs in preemptive mode. But some ネイティブコード in the VM must access the GC heap, and thus must run in cooperative mode.

ネイティブコード generally does not manipulate the GC mode directly, but rather uses two macros: GCX\_COOP and GCX\_PREEMP.  These enter the desired mode, and erect "holders" to cause the thread to revert to the previous mode when the scope is exited.

It is important to understand that GCX\_COOP effectively acquires a lock on the GC heap. No GC may proceed while the thread is in cooperative mode. And ネイティブコード cannot be "hijacked" as is done for マネージドコード, so the thread will remain in cooperative mode until it explicitly switches back to preemptive mode.

Thus entering cooperative mode in ネイティブコード is discouraged. In cases where  cooperative mode must be entered, it should be kept to as short a time as possible. The thread should not be blocked in this mode, and in particular cannot generally acquire locks safely.

Similarly, GCX\_PREEMP potentially _releases_ a lock that had been held by the thread. Great care must be taken to ensure that all GC references are properly protected before entering preemptive mode.

The [Rules of the Code](../coding-guidelines/clr-code-guide.md) document describes the disciplines needed to ensure safety around GC mode switches.

Crst
----

Just as Monitor is the preferred locking mechanism for マネージドコード, Crst is the preferred mechanism for VM code. Like Monitor, Crst is a hybrid lock that is aware of hosts and GC modes. Crst also implements deadlock avoidance via "lock leveling," described in the [Crst Leveling chapter of the BotR](../coding-guidelines/clr-code-guide.md#entering-and-leaving-crsts).

It is generally illegal to acquire a Crst while in cooperative mode, though exceptions are made where absolutely necessary.

Special Threads
===============

In addition to managing threads created by マネージドコード, the CLR creates several "special" threads for its own use.

Finalizer Thread
----------------

This thread is created in every process that runs マネージドコード. When the GC determines that a finalizable object is no longer reachable, it places that object on a finalization queue. At the end of a GC, the finalizer thread is signaled to process all finalizers currently in this queue. Each object is then dequeued, one by one, and its finalizer is executed.

This thread is also used to perform various CLR-internal housekeeping tasks, and to wait for notifications of some external events (such as a low-memory condition, which signals the GC to collect more aggressively). See GCHeap::FinalizerThreadStart for the details.

GC Threads
----------

When running in "concurrent" or "server" modes, the GC creates one or more background threads to perform various stages of garbage collection in parallel. These threads are wholly owned and managed by the GC, and never run マネージドコード.

Debugger Thread
---------------

The CLR maintains a single ネイティブスレッド in each managed process, which performs various tasks on behalf of attached managed debuggers.

AppDomain-Unload Thread
-----------------------

This thread is responsible for unloading AppDomains. This is done on a separate, CLR-internal thread, rather than the thread that requests the AD-unload, to a) provide guaranteed stack space for the unload logic, and b) allow the thread that requested the unload to be unwound out of the AD, if needed.

ThreadPool Threads
------------------

The CLR's ThreadPool maintains a collection of マネージドスレッドs for executing user "work items."  These マネージドスレッドs are bound to ネイティブスレッドs owned by the ThreadPool. The ThreadPool also maintains a small number of ネイティブスレッドs to handle functions like "thread injection," timers, and "registered waits."
