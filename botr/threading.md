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

Thread Lifetimes
================

A マネージドスレッド is created in the following situations:

1. マネージドコード explicitly asks the CLR to create a new thread via System.Threading.Thread.
2. The CLR creates the マネージドスレッド directly (see "special threads" below).
3. ネイティブコード calls マネージドコード on a ネイティブスレッド which is not yet associated with a マネージドスレッド (via "reverse p/invoke" or COM interop).
4. A managed process starts (invoking its Main method on the process' Main thread).

In cases #1 and #2, the CLR is responsible for creating a ネイティブスレッド to back the マネージドスレッド. This is not done until the thread is actually _started_. In such cases, the ネイティブスレッド is "owned" by the CLR; the CLR is responsible for the ネイティブスレッド's lifetime. In these cases, the CLR is aware of the existence of the thread by virtue of the fact that the CLR created it in the first place.

In cases #3 and #4, the ネイティブスレッド already existed prior to the creation of the マネージドスレッド, and is owned by code external to the CLR. The CLR is not responsible for the ネイティブスレッド's lifetime. The CLR becomes aware of these threads the first time they attempt to call マネージドコード.

When a ネイティブスレッド dies, the CLR is notified via its DllMain function. This happens inside of the OS "loader lock," so there is little that can be done (safely) while processing this notification. So rather than destroying the data structures associated with the マネージドスレッド, the thread is simply marked as "dead" and signals the finalizer thread to run. The finalizer thread then sweeps through the threads in the ThreadStore and destroys any that are both dead _and_ unreachable via マネージドコード.

Suspension
==========

The CLR must be able to find all references to managed objects in order to perform a GC. マネージドコード is constantly accessing the GC heap, and manipulating references stored on the stack and in registers. The CLR must ensure that all マネージドスレッドs are stopped (so they aren't modifying the heap) to safely and reliably find all managed objects. It only stops at _safe point_, when registers and stack locations can be inspected for live references.

Another way of putting this is that the GC heap, and every thread's stack and register state, is "shared state," accessed by multiple threads. As with most shared state, some sort of "lock" is required to protect it. マネージドコード must hold this lock while accessing the heap, and can only release the lock at safe points.

The CLR refers to this "lock" as the thread's "GC mode." A thread which is in "cooperative mode" holds its lock; it must "cooperate" with the GC (by releasing the lock) in order for a GC to proceed. A thread which is in "preemptive" mode does not hold its lock – the GC may proceed "preemptively" because the thread is known to not be accessing the GC heap.

A GC may only proceed when all マネージドスレッドs are in "preemptive" mode (not holding the lock). The process of moving all マネージドスレッドs to preemptive mode is known as "GC suspension" or "suspending the 実行 Engine (EE)."

A naïve implementation of this "lock" would be for each マネージドスレッド to actually acquire and release a real lock around each access to the GC heap. Then the GC would simply attempt to acquire the lock on each thread; once it had acquired all threads' locks, it would be safe to perform the GC.

However, this naïve approach is unsatisfactory for two reasons. First, it would require マネージドコード to spend a lot of time acquiring and releasing the lock (or at least checking whether the GC was attempting to acquire the lock – known as "GC polling.") Second, it would require the JIT to emit "GC info" describing the layout of the stack and registers for every point in JIT'd code; this information would consume large amounts of memory.

We refined this naïve approach by separating JIT'd マネージドコード into "partially interruptible" and "fully interruptible" code. In partially interruptible code, the only safe points are calls to other methods, and explicit "GC poll" locations where the JIT emits code to check whether a GC is pending. GC info need only be emitted for these locations. In fully interruptible code, every 命令 is a safe point, and the JIT emits GC info for every 命令 – but it does not emit GC polls. Instead, fully interruptible code may be "interrupted" by hijacking the thread (a process which is discussed later in this document). The JIT chooses whether to emit fully- or partially-interruptible code based on heuristics to find the best tradeoff between code quality, size of the GC info, and GC suspension latency.

Given the above, there are three fundamental operations to define: entering cooperative mode, leaving cooperative mode, and suspending the EE.

Entering Cooperative Mode
-------------------------

A thread enters cooperative mode by calling Thread::DisablePreemptiveGC. This acquires the "lock" for the current thread, as follows:

1. If a GC is in progress (the GC holds the lock) then block until the GC is complete.
2. Mark the thread as being in cooperative mode. No GC may proceed until the thread reenters preemptive mode.

These two steps proceed as if they were atomic.

Entering Preemptive Mode
------------------------

A thread enters preemptive mode (releases the lock) by calling Thread::EnablePreemptiveGC. This simply marks the thread as no longer being in cooperative mode, and informs the GC thread that it may be able to proceed.

Suspending the EE
-----------------

When a GC needs to occur, the first step is to suspend the EE. This is done by GCHeap::SuspendEE, which proceeds as follows:

1. Set a global flag (g\_fTrapReturningThreads) to indicate that a GC is in progress. Any threads that attempt to enter cooperative mode will block until the GC is complete.
2. Find all threads currently executing in cooperative mode. For each such thread, attempt to hijack the thread and force it to leave cooperative mode.
3. Repeat until no threads are running in cooperative mode.

Hijacking
---------

Hijacking for GC suspension is done by Thread::SysSuspendForGC. This method attempts to force any マネージドスレッド that is currently running in cooperative mode, to leave cooperative mode at a "safe point." It does this by enumerating all マネージドスレッドs (walking the ThreadStore), and for each マネージドスレッド currently running in cooperative mode.

1. Suspend the underlying ネイティブスレッド. This is done with the Win32 SuspendThread API. This API forcibly stops the thread from running, at some random point in its 実行 (not necessarily a safe point).
2. Get the current CONTEXT for the thread, via GetThreadContext. This is an OS concept; CONTEXT represents the current register state of the thread. This allows us to inspect its 命令ポインター, and thus determine what type of code it is currently executing.
3. Check again if the thread is in cooperative mode, as it may have already left cooperative mode before it could be suspended. If so, the thread is in dangerous territory: the thread may be executing arbitrary ネイティブコード, and must be resumed immediately to avoid deadlocks.
4. Check if the thread is running マネージドコード. It is possible that it is executing native VM code in cooperative mode (see Synchronization, below), in which case the thread must be immediately resumed as in the previous step.
5. Now the thread is suspended in マネージドコード. Depending on whether that code is fully- or partially-interruptable, one of the following is performed:
  * If fully interruptable, it is safe to perform a GC at any point, since the thread is, by definition, at a safe point. It is reasonable to leave the thread suspended at this point (because it's safe) but various historical OS bugs prevent this from working, because the CONTEXT retrieved earlier may be corrupt). Instead, the thread's 命令ポインター is overwritten, redirecting it to a stub that will capture a more complete CONTEXT, leave cooperative mode, wait for the GC to complete, reenter cooperative mode, and restore the thread to its previous state.
  * If partially-interruptable, the thread is, by definition, not at a safe point. However, the caller will be at a safe point (method transition). Using that knowledge, the CLR "hijacks" the top-most stack frame's return address (physically overwrite that location on the stack) with a stub similar to the one used for fully-interruptable code. When the method returns, it will no longer return to its actual caller, but rather to the stub (the method may also perform a GC poll, inserted by the JIT, before that point, which will cause it to leave cooperative mode and undo the hijack).

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
