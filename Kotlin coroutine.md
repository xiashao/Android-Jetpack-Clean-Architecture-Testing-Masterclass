# Kotlin coroutine

## 什么是协程？

**Coroutines = Co + Routines**

co 是指协同， routine是指方法，顾名思义，Coroutines 就是协同方法。

怎么理解？

举个栗子，LOL或者说王者荣耀，我们总有一个双人路，而双人路中，有一个射手位作为主要输出，但是有一些操作是耗时的，比如插眼，这时候我们就需要一个辅助位帮助射手完成以上任务。

射手位作为C位，他只有一个目标：输出输出输出！你就可以想象成MAIN线程，也只有一个目标：更新更新更新！而辅助位，你就可以认为是一个协程，就是：脏活累活背黑锅。

代码可以如下：

```
fun 射手发育() {
	补兵
	补兵
	辅助插眼()
	补兵
	...
}
```

这有什么好处呢？

有两点妙处：

1. **性价比拉满。**

   众所周知，在游戏中，射手位（MAIN）操作是拉满的，是very busy的，但是辅助（协程）并不是，很多时候傻愣着没事干，这时候他就应该去做一些必要但是耗时的事情，这些事情让MAIN做是一种浪费，但是让协程去做就属于充分利用资源。

2. **异步变同步。**

   举个栗子，倘若没有辅助位，射手想插眼，就要打字：打野，给我插个眼。过了一会，打野给你回复：嗯好了。你看了一眼，确实好了，这就要保证打野一定给你回复（回调）。但是有了辅助（协程），就方便多了，你看着它晃悠悠的去插眼，然后晃悠悠的回来。你俩相视一笑，懂了，事情已经办成了。

协程的存在让多任务变得简单，那它和线程有啥关系？

我们可以认为协程就是**一个轻量线程（lightweight thread）, 轻量线程是不会映射到native thread（内核线程）的，因此系统不会频繁的调度，切换上下文，所以效率大大的好。**

PS：补充下关于内核线程的小知识

> 内核线程就是直接由操作系统内核（Kernel）支持的线程，这种线程由内核来完成线程切换，内核通过操纵调度器（Scheduler）对线程进行调度，并负责将线程的任务映射到各个处理器上。每个内核线程可以视为内核的一个分身，这样操作系统就有能力同时处理多件事情，支持多线程的内核就叫做多线程内核（Multi-Threads Kernel）。
>
> 我们常说的线程就是轻量级进程，这玩意不能直接被CPU调用，而是由内核线程支持，也就是说每个线程都要一一对应一个内核线程，然后再被CPU调度。



**Reference**

[Mastering Kotlin Coroutines In Android - Step By Step Guide](https://blog.mindorks.com/mastering-kotlin-coroutines-in-android-step-by-step-guide)

[JAVA线程映射为linux内核线程](https://blog.csdn.net/u012017344/article/details/81987701)



## 什么时候用协程？

现在，我要举个例子，假如UI线程是《亮剑》中的李云龙，你现在要攻打县城。

> 二营长，把你的意大利炮拿出来

这时候你应该怎么写代码？

首先你应该先找二营长。

```
fun fetch二营长(): 营长 {
    // make call
    // return 二营长
}
```

然后让二营长把意大利炮拿出来。

```
fun 开炮(二营长: 营长) {
    // Fire
}
```

所以这时候你写的方法应该就是：

```
fun 二营长开炮() {
    val 二营长 = fetch营长()
    开炮(二营长)
}
```

如果你这么写，那么就会出现**NetworkOnMainThreadException**错误，为啥呢？

因为你堂堂团长找个营长，还得亲自跑过去，等你跑过去找二营长再跑回来，鬼子援兵早就到了。直接发个微信不香吗？没有微信，那就找个传令兵，总之MAIN是不能参与耗时操作的。

这时候咋办，一般用三种方式：

1. **CallBack**

   ```
   fun 二营长开炮() {
       fetch营长{营长->
        开炮(二营长)
       }
   }
   ```

2. **RXJAVA**

   我不会。

3. **Coroutines**

   ```
   suspend fun 二营长开炮() {
       val 二营长 = fetch营长()
       开炮(二营长)
   }
   ```

这时候fetch二营长()也要变成suspend函数

```
suspend fun fetch二营长(): 营长 {
	return GlobalScope.async(Dispatchers.IO) {
        // make call
    	// return 二营长
    }.await()
}
```

至于开炮方法就不用了suspend 了，毕竟这是不耗时的，拉根绳就开了。

等等！suspend是什么东西？Dispatchers是什么？GlobalScope又是什么东西？

我们下回再说。



## 什么是Suspend？

**一句话：告诉别人，我这里有耗时任务，等我一会儿。**

既然是个耗时任务，自然也可以started, paused, 以及resume了。

而且Suspend还有一个特点，就是严格按照顺序执行。

还是拿李云龙举例子。

```
suspend fun 二营长开炮() {
    val 二营长 = fetch营长()
    开炮(二营长)
}
```

在这里，fetch营长是一个耗时间的操作，但是找二营长和开炮有严格的先后顺序，也就是一定要先找到二营长，然后才能开炮。正如之前所言，找二营长需要时间，这时候系统已经调用到开炮的话，那必然出错。

因此在suspend方法中，代码是有严格先后顺序的，也就是，只有上一句执行完，下一句才可以执行。

那么有小伙伴可能会问了，那会不会导致UI线程阻塞呀？

答案并不会，只是单纯的这个suspend中大的代码被阻塞了，UI线程还是该干嘛干嘛，只是当找到二营长后，UI才知道现在可以开炮了，然后开炮就行了。

并不是说UI线程就等着开炮，其他什么事都不做了，所以这里是不会阻塞的。

## 什么是dispatcher？

说到dispatcher，不知道还有没有人记得我之前说，李云龙找二营长，需要一个传令兵吗？

这个dispatcher 就是传令兵，它可以帮你决定，你要把任务放在什么线程上。主要有：**IO, Default, 还有Main**. 

IO主要就是处理网络，磁盘数据传输的任务，Default 就是处理CPU运算任务，MAIN就是处理UI刷新的，这就和线程池的概念比较相似了。其他不常用的还有Unconfined 或者什么都不写，或者自己写的。

我们做个实验：

```
 fun testDispatcher(){
        GlobalScope.launch {
            Log.i(TAG, "null: ${Thread.currentThread().name}")
        }
        GlobalScope.launch(Dispatchers.Main) {
            Log.i(TAG, "Main: ${Thread.currentThread().name}")
        }
        GlobalScope.launch(Dispatchers.IO) {
            Log.i(TAG, "IO: ${Thread.currentThread().name}")
        }
        GlobalScope.launch(Dispatchers.Unconfined) {
            Log.i(TAG, "Unconfined: ${Thread.currentThread().name}")
        }
        GlobalScope.launch(Dispatchers.Default) {
            Log.i(TAG, "Default: ${Thread.currentThread().name}")
        }
        GlobalScope.launch(newSingleThreadContext("MyOwnThread")) {
            Log.i(TAG, "MyOwnThread: ${Thread.currentThread().name}")
        }
    }
```

打印的结果是

```
null: DefaultDispatcher-worker-1
IO: DefaultDispatcher-worker-1
Unconfined: main
Default: DefaultDispatcher-worker-1
MyOwnThread: MyOwnThread
Main: main
```

Unconfined走的还是MAIN，但是我奇怪的是为啥MAIN是最后一个打印出来的，有点奇怪。

```
  GlobalScope.launch(Dispatchers.Main) {
            val startTime = System.currentTimeMillis()
            launch(Dispatchers.Main) {fire(startTime)}
            val yz =  launch(Dispatchers.IO) { fetch2YZ() }
            val bullet = launch(Dispatchers.IO) { fetchBullet() }
            //fire(startTime)
        }
```

即便写成这样，依然是最后打印MAIN里面的任务，很奇怪，目前不知道为什么。

这东西啥时候使用呢？那就是需要委托别人做事的时候才需要，比如说MAIN让IO去干一个事，这时候就需要传令兵dispatcher了。

我们看这段代码：

```
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    
    GlobalScope.launch(Dispatchers.Main) {
        val 二营长 = fetch营长()
    	开炮(二营长)
    }
    
}
```

这时候就有小伙伴说了，你这不对呀？本来就在onCreate就是MAIN啊，你为啥又要找个传令兵去到MAIN里呀？

是这样的，我们之前说了supend意味着耗时，只能用在supend方法里面，或者协程里面。

我在这解释下为啥，第一如果一个正常方法里面包含suspend方法，那一定意味着这个方法也是个耗时方法，所以它也必须有suspend前缀。

第二种情况就是suspend方法可以在协程中被调用，而协程就是为了处理耗时方法，所以无论哪个suspend方法最开始的地方一定是一个协程。

针对于以上的代码可能有人问，MAIN没有处理复杂操作呀，为啥要用一个协程？虽然没有耗时的操作，但是MAIN从调用IO然后获得结果最后开炮，这个时间是漫长的，但是MAIN不会在此阻塞，最后我们还得用MAIN去开炮，所以一开始的时候我们的确需要一个Dispatchers.Main来开启这整个耗时任务。

PS：尽量不要使用GlobalScope来开启协程，具体原因后续会说。

## **Launch vs Async** vs withContext

Launch 和 Async是启动协程的两种不同方式, 简单来说,launch没有返回值，而Async有返回值。

举个例子，还是二营长开炮。

这里有一个关键，就是我去找二营长，最后二营长一定要跟我来平安县，也就是说一定要有返回值的，要不然我去找二营长，二营长说一句我知道了没后续了就。

```
suspend fun fetch2YZ():YingZhang{
        delay(1000)
        Log.d(TAG, "fetch2YZ: ")
        return YingZhang("2")
    }
```

假如说这里还有一个找炮弹的过程。

```
suspend fun fetchBullet():Bullet{
        delay(2000)
        Log.d(TAG, "fetchBullet: ")
        return Bullet()
    }
```

开火为：

```
fun fire(yingZhang: YingZhang,bullet: Bullet,startTime:Long){
    Log.d(TAG, "Time: "+ (System.currentTimeMillis() - startTime))
}
```

那么开火这过程可以写成：

```
GlobalScope.launch(Dispatchers.Main) {
            val startTime = System.currentTimeMillis()
            val yz = async{ fetch2YZ() }
            val bullet = async{ fetchBullet() }
            fire(yz.await(), bullet.await(),startTime) // Time: 2017,说明并行了
}
```

这里注意下，获取营长和炮弹是并性的，但是开火的时候需要营长和炮弹都准备好。

也就是说，这里await方法是阻塞了这个协程，只有await获取到了返回值，才能继续走下去，于是这里只有营长.await(), 炮弹.await()都完事了，开炮才能执行。

那么问题来了。

如果我这么写，会发生什么。

```
GlobalScope.launch(Dispatchers.Main) {
            val startTime = System.currentTimeMillis()
            val yz = async(Dispatchers.IO) { fetch2YZ() }.await()
            val bullet = async(Dispatchers.IO) { fetchBullet() }.await()
            fire(yz, bullet,startTime) // Time: 3008,说明串行了
}
```

这样写的话，就变成了串行执行。找到营长后，才能找炮弹，找到炮弹后才能开炮，低效了。

那如果假设这里没有用async用的是launch.

```
 val yz = launch(Dispatchers.IO) { fetch2YZ() }
 val bullet = launch(Dispatchers.IO) { fetchBullet() }
 //Time: 7 也是并行
 fire(startTime)
```

这时候又变成并行了。

最后还有一个**withContext**，他和async最大的不同就是没有await，还有就是串行执行。

```
val yz = withContext(Dispatchers.IO) { fetch2YZ() }
val bullet = withContext(Dispatchers.IO) { fetchBullet() }
//Time: 3011 也是串行
fire(yz,bullet,startTime)
```

现在又变成了串行，相当于async的await省略了，但是还是有阻塞的。

最后还有一个runblocking也是串行的，不建议使用。

```
   fun testWithRunBlocking() {
        GlobalScope.launch(Dispatchers.Main) {
            val startTime = System.currentTimeMillis()
            val yz = runBlocking(Dispatchers.IO) { fetch2YZ() }
            val bullet = runBlocking(Dispatchers.IO) { fetchBullet() }
            fire(yz, bullet, startTime)
        }
    }
```

小伙伴们，要记住了，现在使用规律如下：

**第一步，判断有没有返回值，没有返回值，就用launch。**

**第二步，判断要不要并行，并行的话选async，不并行用async+await或者withcontext.**

## 什么是Scope？

Scope你可以简单理解为领域，举个栗子。

Scope有啥用呢？主要就是告诉你，我们阶段性的任务已经结束了，把那些还在执行中的任务都取消吧。举个例子，1949年解放战争都结束了，西北还有一些清朝老兵在守边疆，这就是scope搞错了呀。皇上都没有了，任务也该取消了，结果他们当时接收的信息是皇上生命周期有一万年，这就出了问题。

以平安县战役为例，从平安县开战到结束，这个时间范围就是一个scope。

还记得我之前说不要用GlobalScope，这玩意就像明明我打的是平安城，结果你告诉我战争结束时间是整个抗日结束时间一样离谱。

在这里举个例子。

我们的协程在activity 中启动，自然也希望在activity结束的时候结束，于是我们在onDestroy中标记：

```
override fun onDestroy() {
    super.onDestroy()
    Log.d(TAG, "平安战役结束: ")
}
```

我们让二营长连续开炮。

```
for(i in 1..10){
    val yz = async(Dispatchers.IO) { fetch2YZ() }.await()
    val bullet = async(Dispatchers.IO) { fetchBullet() }.await()
    fire(yz, bullet,startTime)
}
```

这时候我们启动，然后推出界面，此时打印：

```
fetch2YZ: 
fetchBullet: 
开炮，总Time: 6011
平安战役结束: 
fetch2YZ: 
fetchBullet: 
开炮，总Time: 9015
fetch2YZ: 
fetchBullet: 
开炮，总Time: 12030
...
...
```

这时候小伙伴可能会问一个经典问题：

**那咋整？**

这时候你应该让你的activity 实现CoroutineScope

然后重写coroutineContext的值。

```
class MainActivity : AppCompatActivity(),
    CoroutineScope  
{
    override val coroutineContext: CoroutineContext
        get() = Dispatchers.Main + job
    private lateinit var job: Job
```

在oncreat中

```
 job = Job()
 launch {
    for(i in 1..10){
    val yz = async(Dispatchers.IO) { fetch2YZ() }.await()
    val bullet = async(Dispatchers.IO) { fetchBullet() }.await()
    fire(yz, bullet,startTime)
}
```

在onDestroy中

```
job.cancel()
```

这时候再打印结果：

```
fetch2YZ: 
fetchBullet: 
开炮，总Time: 9036
平安战役结束: 
```

完美了。

等等，coroutineContext是什么，job是什么？他俩咋整成一个scope的？、

为啥直接一个launch｛｝就行了，之前不还要加什么dispatcher嘛？

下节再说。

## 什么是CoroutineScope

现提出一个概念structured concurrency（结构化并发）

main_func 里，创建了两个子任务：myfunc(), anotherfunc，这里的 func 是一个控制流结构，入口就是 func 调用开始，出口是 func 调用结束，派生出来的两个子任务需要在 main_func 调用结束之前先完成。当 main_func 结束，它涉及到的资源也都会被释放掉。外部调用者无法也无需感知 main_func 里面到底是串行的还是并行的，它只需要调用 main_func，然后等待它结束即可。这就是所谓的 Structured。

![在这里插入图片描述](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X2pwZy84WGt2Tm5UaWFwT05YN2NhMFE4TFJ1RlRiVFJCVEFvSHd4QUxBZTZJRmZrRGx6VmZLdmZuR1dpYzhGZ2Fyd0tCUVBnVlRSYjA2b1NpY1A0NHBRNjVKOU5XQS82NDA?x-oss-process=image/format,png)



## suspend与Continuation的终极奥义

suspend只能在suspend方法中使用，或者在协程中使用，为什么？

```
suspend fun coroutine(number: Int, delay: Long){
	println("Coroutine $number starts work")
	delay(delay)
	println("Coroutine $number has finished")
}
```

如以上的代码，如果我们编辑后，真实的代码应该如下：

```
fun coroutine(number: Int?, delay: Long?, continuation: Continuation<Any?>): Unit {
	  when(continuation.label){
        0 -> {
    		    println("Coroutine $number starts work.")
    		    delay(delay)
        }
	      1 -> {
	          println("Coroutine $number has finished")	  
            continuation.resume(Unit)
	      }
    }
}
```

最大的不同就是新加了一个参数continuation。

这continuation又是何方神圣？

从源码中可见：

```
public interface Continuation<in T> {
    /**
     * The context of the coroutine that corresponds to this continuation.
     */
    public val context: CoroutineContext

    /**
     * Resumes the execution of the corresponding coroutine passing a successful or failed [result] as the
     * return value of the last suspension point.
     */
    public fun resumeWith(result: Result<T>)
}
```

也就是说，要实现一个suspend方法就要创建一个Continuation，那么怎么创建？

我们看一下launch中是怎么处理的。

```
val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
 coroutine.start(start, coroutine, block)
```

StandaloneCoroutine继承了 AbstractCoroutine，而AbstractCoroutine实现了Continuation<T>

所以这个coroutine就是Continuation实现类对象。

我们再看coroutine.start(start, coroutine, block)，这个start方法。

```
 public fun <R> start(start: CoroutineStart, receiver: R, block: suspend R.() -> T) {
        start(block, receiver, this)
    }
```

这里调用了CoroutineStart的invoke方法,这个this就是上文的coroutine，也就是Continuation实现类对象。

```
    public operator fun <R, T> invoke(block: suspend R.() -> T, receiver: R, completion: Continuation<T>): Unit =
        when (this) {
            DEFAULT -> block.startCoroutineCancellable(receiver, completion)
            ATOMIC -> block.startCoroutine(receiver, completion)
            UNDISPATCHED -> block.startCoroutineUndispatched(receiver, completion)
            LAZY -> Unit // will start lazily
        }
```

startCoroutine中的completion就是上文的coroutine，然后就调用了Continuation中的startCoroutine方法。

```
public fun <T> (suspend () -> T).startCoroutine(
    completion: Continuation<T>
) {
    createCoroutineUnintercepted(completion).intercepted().resume(Unit)
}
```

接着看createCoroutineUnintercepted。

```
@SinceKotlin("1.3")
public expect fun <T> (suspend () -> T).createCoroutineUnintercepted(
    completion: Continuation<T>
): Continuation<Unit>
```

这是个expect方法，真正的实现在actual方法里。

```
public actual fun <T> (suspend () -> T).createCoroutineUnintercepted(
    completion: Continuation<T>
): Continuation<Unit> {
    val probeCompletion = probeCoroutineCreated(completion)
    return if (this is BaseContinuationImpl)
        create(probeCompletion)
    else
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function1<Continuation<T>, Any?>).invoke(it)
        }
}
```

这里的create方法我们看看。

```
public open fun create(completion: Continuation<*>): Continuation<Unit> {
    throw UnsupportedOperationException("create(Continuation) has not been overridden")
}
```

返回值是Continuation<Unit>

明白了，只要我们有一个Continuation还有一个suspend的lambda就可以启动一个协程了。

```
class TestContinuation() : Continuation<String> {

    override fun resumeWith(result: Result<String>) {
        println(Thread.currentThread().name)
        println("resumeWith result: $result")
    }

    override val context: CoroutineContext
        get() = kotlin.coroutines.EmptyCoroutineContext

}
suspend fun suspendFun(): String {
    delay(3000L)
    return "this is a suspendFun";
}

fun demo(){

    val mySuspendlambda: suspend() -> String = {
        println( "result: ")
        suspendFun()
    }

    val testCoroutine = TestContinuation()
    val coroutine = mySuspendlambda.startCoroutine(testCoroutine)
    //println(coroutine)
}
fun main()  {
    demo()
    Thread.sleep(3000)
}
```

以上的代码中，我们写了一个suspend lambda，还有一个TestCoroutine，也就是Continuation实现类对象。然后我们运行：

> result: 
> kotlinx.coroutines.DefaultExecutor
> resumeWith result: Success(this is a suspendFun)
>
> Process finished with exit code 0

如果我没有看错的话，我们应该是成功的在协程中运行了一个suspend方法。

现在还有一个问题：为什么`if (this is BaseContinuationImpl)`是成立的。

this就是我们写的suspend lambda，让我们反编译下。

就变成了如下代码：

```
final class com/example/coroutinedemo/CoroutineContextDemo/TestContinuationKt$demo$mySuspendLamda$1 extends kotlin/coroutines/jvm/internal/SuspendLambda implements kotlin/jvm/functions/Function1
```

变成了继承SuspendLambda的一个类。

看进去代码后，我们发现继承关系如下：

![image-20210721113029206](C:\Users\mxshang\AppData\Roaming\Typora\typora-user-images\image-20210721113029206.png)

所以我们就能知道为什么this is BaseContinuationImpl了。

然后create方法返回的就是一个continuation。这个continuation调用`suspend lambda`的`continuation`的`onresume`.

然后调用的是BaseContinuationImpl中的resumeWith，这个又调用了invokeSuspend就是我们之前写的suspendLambda的方法（反编译后可以看到），然后调用我们写的suspend方法得到返回值。(continuation.resume->BaseContinuationImpl.resumewith->invokeSuspend)

```
internal abstract class BaseContinuationImpl(
    // This is `public val` so that it is private on JVM and cannot be modified by untrusted code, yet
    // it has a public getter (since even untrusted code is allowed to inspect its call stack).
    public val completion: Continuation<Any?>?
) : Continuation<Any?>, CoroutineStackFrame, Serializable {
    // This implementation is final. This fact is used to unroll resumeWith recursion.
    public final override fun resumeWith(result: Result<Any?>) {
        // This loop unrolls recursion in current.resumeWith(param) to make saner and shorter stack traces on resume
        var current = this
        var param = result
        while (true) {
            // Invoke "resume" debug probe on every resumed continuation, so that a debugging library infrastructure
            // can precisely track what part of suspended callstack was already resumed
            probeCoroutineResumed(current)
            with(current) {
                val completion = completion!! // fail fast when trying to resume continuation without completion
                val outcome: Result<Any?> =
                    try {
                        val outcome = invokeSuspend(param)
                        if (outcome === COROUTINE_SUSPENDED) return
                        Result.success(outcome)
                    } catch (exception: Throwable) {
                        Result.failure(exception)
                    }
                releaseIntercepted() // this state machine instance is terminating
                if (completion is BaseContinuationImpl) {
                    // unrolling recursion via loop
                    current = completion
                    param = outcome
                } else {
                    // top-level completion reached -- invoke and return
                    completion.resumeWith(outcome)
                    return
                }
            }
        }
    }
```

然后判断返回值是不是”COROUTINE_SUSPENDED“，不是的话，就把result设置成成功。然后调用的是`completion.resumeWith(outcome)`此处的completion就是我自己写的continuation的实现类。

![image-20210721163306963](C:\Users\mxshang\AppData\Roaming\Typora\typora-user-images\image-20210721163306963.png)



## Coroutine状态机和COROUTINE_SUSPENDED

上文我们说到我们需要判断值是不是COROUTINE_SUSPENDED，我们之前的代码返回的值都不是这个，那么怎么才能让他返回一个COROUTINE_SUSPENDED呢？

想想看，我们需要返回COROUTINE_SUSPENDED表示我们的suspend方法正在挂起，但是当我们要完成任务的时候，还需要重新调用这个方法，返回正确的返回值表示任务完成。关键在于第二次调用这个方法。

我们从以下这个网站中发现了奥秘：

https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.intrinsics/suspend-coroutine-unintercepted-or-return.html

```
suspendCoroutineUninterceptedOrReturn()
```

这个方法可以让我们获取当前suspend的continuation,然后我们就可以使用`continuation.resume`来调用`BaseContinuationImpl.resumewith`方法了。

然后我们写个新的suspend方法。

```
suspend fun suspendReturnSuspend() = suspendCoroutineUninterceptedOrReturn<String> { continuation ->
    Thread {
        TimeUnit.SECONDS.sleep(3)
        println("Continuation->>>>>again")
        println(continuation)
        //调用又会重新调用BaseContinuationImpl的resumeWith函数
        continuation.resume("not end")
    }.start()

    println("suspendReturnSuspend return")
    "haha"
//    kotlin.coroutines.intrinsics.COROUTINE_SUSPENDED
}
```

我们把返回值写成"haha"而不是COROUTINE_SUSPENDED，这样的话会导致系统认为我们的continuation已经完成了，然后第二次调用

continuation.resume就会出错。

因为在`releaseIntercepted`这个方法会判断，当前的intercepted 是什么。

如果按照以上的写法，就会出现：

> This continuation is already complete

然后就会checkNotNull，抛出空指针异常。

那如果我返回一个COROUTINE_SUSPENDED呢？

自然就不会报错了，这里提一嘴，官方提供我们一个更好的方法就是用suspendCoroutine，这个会自动帮我们判断是不是要返回COROUTINE_SUSPENDED，具体内容不看了。

我们分析下以下代码的具体过程：

```
class TestContinuation() : Continuation<String> {

    override fun resumeWith(result: Result<String>) {
        println("Thread name: "+Thread.currentThread().name)
        println("resumeWith result: $result")
    }

    override val context: CoroutineContext
        get() = kotlin.coroutines.EmptyCoroutineContext

}
//不建议使用suspendCoroutineUninterceptedOrReturn
suspend fun suspendNoSafe() = suspendCoroutineUninterceptedOrReturn<String> { continuation ->
    Thread {
        TimeUnit.SECONDS.sleep(3)
        println("Continuation no safe ->>>>>again")
        continuation.resume("suspendNoSafe")
    }.start()

    println("suspendNoSafe return COROUTINE_SUSPENDED")
    kotlin.coroutines.intrinsics.COROUTINE_SUSPENDED
}
suspend fun suspendSafe() = suspendCoroutine<String> { continuation ->
    Thread {
        TimeUnit.SECONDS.sleep(3)
        println("Continuation safe ->>>>>again")
        continuation.resume("suspendSafe")
    }.start()

    println("suspendSafe return COROUTINE_SUSPENDED")
    "haha"
}

fun demo(){

    val mySuspendLamda: suspend () -> String = {
        println( "result: ")
        suspendNoSafe()
        suspendSafe()
    }

    val testCoroutine = TestContinuation()
    val coroutine = mySuspendLamda.startCoroutine(testCoroutine)
    println("suspend so I print this")
//    val createCoroutine = mySuspendLamda.createCoroutineUnintercepted(testCoroutine)
//    createCoroutine.resume(Unit)
//    println(coroutine)
}
fun main()  {
    demo()
}
```

方便我看:反编译下：

```
static final class mySuspendLamda extends SuspendLambda implements Function1 {
    int label;
    Object result;

    public final Object invokeSuspend(Object $result) {
        Object SUSPEND_FLAG = IntrinsicsKt.getCOROUTINE_SUSPENDED();
        int this.label;
        if(this.label== 0) {
            ResultKt.throwOnFailure($result);
            this.label = 1;
            //（一）此处返回SUSPEND_FLAG，等mySuspendOne调用传入Continuation对象
            //重新调用到invokeSuspend函数
            if(TestContinuation.suspendNoSafe(((Continuation)this)) == SUSPEND_FLAG) {
                //(二) 相等所以返回挂起标识给父类结束本次调用。
                return SUSPEND_FLAG;
            }

            label_21:
            this.label = 2;
            //(三) mySuspendOne调用传入Continuation的resume函数回调这
            result = TestContinuation.suspendSafe(((Continuation)this));
            //(四) 此处同样返回挂起标识（一个挂起函数没有立即返回结果那么必然要返回COROUTINE_SUSPENDED）
            if(result == COROUTINE_SUSPENDED) {
                //(五)
                return COROUTINE_SUSPENDED;
            }
        }
        else {
            if(this.label!= 1) {
                if(this.label== 2) {
                    //(六) 最后返回结果，SafeSuspend回调Continuation的resume的回到这
                    ResultKt.throwOnFailure($result);
                    return $result;
                }

                throw new IllegalStateException("call to \'resume\' before \'invoke\' with coroutine");
            }

            ResultKt.throwOnFailure($result);
            goto label_21;
        }
        return result;
    }
}
```

第一步：mySuspendLamda调用resume->BaseContinuationImpl的resumewith()

第二步： resumewith调用invokeSuspend()

第三步：label = 1，调用suspendNoSafe，启动线程，返回COROUTINE_SUSPENDED

第四步：mySuspendLamda返回COROUTINE_SUSPENDED给resumewith()，resumewith()函数中判断COROUTINE_SUSPENDED

第五步：resumewith直接return, 打印`suspend so I print this`

第六步，suspendNoSafe的线程再次调动resume.

第七步：很关键，前面步骤重复，但是label变成了1之后不return COROUTINE_SUSPENDED了，而是直接变成label =2然后调用suspendSafe。

第八步：suspendSafe 继续return COROUTINE_SUSPENDED，然后里面的线程再度调动resume。这时候直接返回result了。

## 什么是CoroutineContext？

源码中的解释：

> ```
> It is an indexed set of [Element] instances. 索引集
> ```

是一个元素实例的索引集，每一个索引集都有一个独特的key。

> 在数学中,集合 A 的元素有时可以凭借某个集合 J 来索引(index)或标定(label),这时便称集合 J 为*索引集*

![diagram](https://github.com/takahirom/kotlin-coroutines-class-diagram/raw/master/diagram.png)

首先它是一个context, 其次他是一个集合，集合里面有很多元素，就是CoroutineContext.Element，这些元素就有以下这些类型：

**Job**：一个可以cancel的task，用来规定生命周期的，这就是为啥我们之前用job来限定activity的生命周期的。

**ContinuationInterceptor**： 监听continuation的，并且可以拦截 resumption。

**CoroutineExceptionHandler**：处理错误机制的。

所以当你启动一个协程，你就可以规定你自己的context, 比如你可以整一个job，JOB是实现了CoroutineContext.Element的，这时候你就可以自己规定一个领域了。



## job

```
fun main() = runBlocking { // this: CoroutineScope
     GlobalScope.launch { // 启动一个新协程并保持对这个作业的引用
        delay(3000L)
        println("Hello!")
    }
    println("world")
}
```

> world
>
> Process finished with exit code 0

程序结束后，hello，没有打印出来

JVM的生命周期：当程序中的所有非守护线程都终止时，JVM才退出。

## Dispatchers的终极奥义

我们之前已经讲过Dispatchers的基本要义，此处便不再赘述。

我们在这仔细研究下Dispatchers，我们发现他的几种类型本质都是实现了CoroutineDispatcher。

我们看看CoroutineDispatcher是何方神圣。

`interceptContinuation`这个方法返回了DispatchedContinuation，其中包含两个参数，第一个`CoroutineDispatcher`对象，第二个被代理的对象。

观察`DispatchedContinuation`的`resumewith`方法。

```
override fun resumeWith(result: Result<T>) {
    val context = continuation.context
    val state = result.toState()
    if (dispatcher.isDispatchNeeded(context)) {
        _state = state
        resumeMode = MODE_ATOMIC
        dispatcher.dispatch(context, this)
    } else {
        executeUnconfined(state, MODE_ATOMIC) {
            withCoroutineContext(this.context, countOrElement) {
                continuation.resumeWith(result)
            }
        }
    }
}
```

重点又在于`dispatcher.dispatch(context, this)`

观察dispatch方法。

```
public abstract fun dispatch(context: CoroutineContext, block: Runnable)
```

说明this，也就是DispatchedContinuation继承了DispatchedTask实现了Runnable方法。

在DispatchedTask中实现了run方法。

```
public final override fun run() {
    assert { resumeMode != MODE_UNINITIALIZED } // should have been set before dispatching
    val taskContext = this.taskContext
    var fatalException: Throwable? = null
    try {
        val delegate = delegate as DispatchedContinuation<T>
        val continuation = delegate.continuation
        withContinuationContext(continuation, delegate.countOrElement) {
            val context = continuation.context
            val state = takeState() // NOTE: Must take state in any case, even if cancelled
            val exception = getExceptionalResult(state)
            /*
             * Check whether continuation was originally resumed with an exception.
             * If so, it dominates cancellation, otherwise the original exception
             * will be silently lost.
             */
            val job = if (exception == null && resumeMode.isCancellableMode) context[Job] else null
            if (job != null && !job.isActive) {
                val cause = job.getCancellationException()
                cancelCompletedResult(state, cause)
                continuation.resumeWithStackTrace(cause)
            } else {
                if (exception != null) {
                    continuation.resumeWithException(exception)
                } else {
                    continuation.resume(getSuccessfulResult(state))
                }
            }
        }
    } catch (e: Throwable) {
        // This instead of runCatching to have nicer stacktrace and debug experience
        fatalException = e
    } finally {
        val result = runCatching { taskContext.afterTask() }
        handleFatalException(fatalException, result.exceptionOrNull())
    }
}
```
