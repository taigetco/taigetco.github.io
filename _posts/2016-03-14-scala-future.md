---
layout:     post
title:      Scala中的Promise和Future
date:       2016-03-14
summary:    Promise, Future使用与理解
categories: Scala concurrency
---
----------
>hate to copy-paste the source code, not found better way to explain it clearly up to now

----------


最近看Spark RPC的代码，在看`NettyRpcEnv.ask`对Promise和Future使用时，总觉得其使用方式不对，但思前想后有找不出问题出在哪．遂断断续续花了几天时间来理解Promise和Future.

### Promise和Future概念
Future类似一个占位符对象，仅仅是一个只读对象，为某个计算过程创建，其计算结果在将来某个时间点才会完成，为了使Future完全非阻塞，注册Callback到Future中，一旦future完成，Callback会被异步执行．

Promise是一个可写的，只能赋一次值的容器，Promise存储计算结果，从Promise中可以得到Future，由Promise来完成Future.　也可以从字面上来理解，Promise也就是一个承诺，就好比去买一杯咖啡，付账过后，服务员承诺会给你一杯咖啡，但需要过几分钟才能领取这杯咖啡．服务员制作咖啡的过程就是一个Future，而付账过后，就得到服务员的一个Promise．

### 具体实现

Future简单实例

~~~scala
def add(i: Int) = i + 1
val addOne = Future(add(1))
~~~

其具体实现参考[impl/Future](https://github.com/scala/scala/blob/2.11.x/src/library/scala/concurrent/impl/Future.scala), 创建一个DefaultPromise来计算body，`Success(body)`执行计算，最后promise.future返回Future. 

~~~scala
object Future {
  class PromiseCompletingRunnable[T](body: => T) extends Runnable {
    val promise = new Promise.DefaultPromise[T]()
    override def run() = {
      promise complete {
        try Success(body) catch { case NonFatal(e) => Failure(e) }
      }
    }
  }

  def apply[T](body: =>T)(implicit executor: ExecutionContext): scala.concurrent.Future[T] = {
    val runnable = new PromiseCompletingRunnable(body)
    executor.prepare.execute(runnable)
    runnable.promise.future
  }
}
~~~

Future实现比较简单，重点在于Callback和多种combinators，由于实现深度依赖Promise，在介绍他们之前，先讨论Promise的实现DefaultPromise．
#### DefaultPromise的继承关系

~~~scala
package scala.concurrent.impl
class DefaultPromise[T] extends AbstractPromise with Promise[T]
trait Promise[T] extends scala.concurrent.Promise[T] with scala.concurrent.Future[T] {
  def future: this.type = this
}
~~~

可以看到DefaultPromise继承Future, 而future方法返回的就是Promise. 在AbstractPromise中主要实现对Promise状态的原子访问．

#### DefaultPromise的状态变化

DefaultPromise在`AbstractPromise.ref`存储它的状态的三种状态：

*  Incomplete, ref == `List[CallbackRunnable]`, 在Promise 完成时执行．
*  Complete, ref == Try[T]
*  Linked, ref == `DefaultPromise[T]`, 用来解决`Future.flatMap`的memory leaks

**注册callbacks**

~~~scala
def onComplete[U](func: Try[T] => U)(implicit executor: ExecutionContext): Unit = {
    val preparedEC = executor.prepare
    //note: callback可以同步也可以异步，关键看executor
    val runnable = new CallbackRunnable[T](preparedEC, func)
    dispatchOrAddCallback(runnable)
}
@tailrec
private def dispatchOrAddCallback(runnable: CallbackRunnable[T]): Unit = {
  getState match {
    case r: Try[_]  => runnable.executeWithValue(r.asInstanceOf[Try[T]]) //结果已经生成，立刻执行callback
    case _: DefaultPromise[_] => compressedRoot().dispatchOrAddCallback(runnable) //
    case listeners: List[_] => if (updateState(listeners, runnable :: listeners)) () else dispatchOrAddCallback(runnable) //添加到ref
  }
}
~~~
**执行Callbacks**

~~~scala
//由tryComplete调用，其主要目的是执行callback，如果Promise已经完成，tryComplete会返回false
@tailrec
private def tryCompleteAndGetListeners(v: Try[T]): List[CallbackRunnable[T]] = {
  getState match {
    case raw: List[_] =>
      val cur = raw.asInstanceOf[List[CallbackRunnable[T]]]
      if (updateState(cur, v)) cur else tryCompleteAndGetListeners(v)
    case _: DefaultPromise[_] =>
      compressedRoot().tryCompleteAndGetListeners(v)
    case _ => null 
  }
}
~~~

**解决memory leaks**

旧`Future.flatMap`代码

~~~scala
def flatMap[S](f: T => Future[S]): Future[S] ={
  val p = Promise[S]()
  onComplete{
    case f: Failure[_] => p complete f.asInstanceOf[Failure[S]]
    case Success(v) => try { f(v) onComplete p.complete } catch{ case NonFatal(e) => p.complete(Failure(e)) }
  }
  p.future
}
~~~

产生[OOM](https://groups.google.com/d/msg/play-framework-dev/58VZD-YXdJw/0F-_nnrGhTgJ)的代码实例, Java heap space with -Xms32m -Xmx32m

~~~scala
val step = 100000
val upper = 1000000
def loop(future: Future[Int]): Future[Int] = {
  future.flatMap { i =>
    if (i % step == 0) println(i)
    if (i < upper) loop(Future(i + 1)) else Future(i) }
}
println(Await.result(loop(Future(0)), 100 seconds))
~~~

为什么会产生OOM, 主要是后创建的Promise依赖之前创建的Promise,  如下代码片段

![code snippet](/images/snippet.png)

这里有５个Future, 其中三个是显示创建，剩余两个在flatMap中创建，标记这些对象为f1, f2, f3, fm1和fm2.

这个有个chain使fm1依赖fm2不能释放，

* fm2在第二个flatMap执行时创建， 只有在f3.onComplete时，fm2才会执行complete．
* 在第一个flatMap执行创建fm1，只有在fm2.onComplete时，fm1才会执行complete

以次类推，在执行上述代码实例时，会有大量的Future也就是Promise不能释放，产生OOM.

在新的Future.flatMap中，把新创建的Promise和旧的Promise link起来，`fm2.ref = fm1`，使fm1不再依赖fm2．具体代码如下，

~~~scala
def flatMap[S](f: T => Future[S])(implicit executor: ExecutionContext): Future[S] = {
  import impl.Promise.DefaultPromise
  val p = new DefaultPromise[S]()
  onComplete {
    case f: Failure[_] => p complete f.asInstanceOf[Failure[S]]
    case Success(v) => try f(v) match {
      //使用linkRootOf
      case dp: DefaultPromise[_] => dp.asInstanceOf[DefaultPromise[S]].linkRootOf(p)
      case fut => fut.onComplete(p.complete)(internalExecutor)
    } catch { case NonFatal(t) => p failure t }
  }
  p.future
}
~~~

DefaultPromise中的linkRootOf和其相关的方法，

~~~scala
def linkRootOf(target: DefaultPromise[T]): Unit = link(target.compressedRoot())

def compressedRoot(): DefaultPromise[T] = {
  getState match {
    case linked: DefaultPromise[_] =>
      val target = linked.asInstanceOf[DefaultPromise[T]].root
      if (linked eq target) target else if (updateState(linked, target)) target else compressedRoot()
    case _ => this
  }
}

def root: DefaultPromise[T] = {
  getState match {
    case linked: DefaultPromise[_] => linked.asInstanceOf[DefaultPromise[T]].root
      case _ => this
  }
}

def link(target: DefaultPromise[T]): Unit = if (this ne target) {
  getState match {
    case r: Try[_] =>
      if (!target.tryComplete(r.asInstanceOf[Try[T]])) {
        // Currently linking is done from Future.flatMap, which should ensure only
        // one promise can be completed. Therefore this situation is unexpected.
        throw new IllegalStateException("Cannot link completed promises together")
      }
    case _: DefaultPromise[_] =>
      compressedRoot().link(target)
    case listeners: List[_] => if (updateState(listeners, target)) {
      if (!listeners.isEmpty) listeners.asInstanceOf[List[CallbackRunnable[T]]].foreach(target.dispatchOrAddCallback(_))
    } else link(target)
  }
}
~~~

下面详细说下对上面代码的理解，

~~~scala
Future(1).flatMap(a => Future(2).flatMap(b => Future(3).flatMap(c => Future(4).flatMap(d => Future(5).flatMap(e => Future(6))))))

//可以翻译成
val fm5 = Future(5).flatMap(e => Future(6))
val fm4 = Future(4).flatMap(d => fm5)
val fm3 = Future(3).flatMap(c => fm4)
val fm2 = Future(2).flatMap(b => fm3)
val fm1 = Future(1).flatMap(a => fm2)
~~~

在执行fm1时，得到fm2.ref=fm1, 执行fm2时，得到fm3.ref=fm2, 最终得到Future(6).ref = fm5, 可以看到Future(6)会引用这些创建的Promise对象，按此步骤这些创建的Promise还是没有办法被gc回收，此时DefaultPromise中的`compressedRoot`发挥作用, 在root和当前ref中的`DefaultPromise`不是同一个对象时，对ref中的`DefaultPromise`进行压缩，直接删掉中间的`DefaultPromise`对象．所以最终Future(6).ref=fm1，中间的fm[2-5]已经被删除．

### **readings**
[1]http://sealedabstract.com/code/broken-promises/<br/>
[2]https://groups.google.com/forum/#!topic/play-framework-dev/58VZD-YXdJw/overview
