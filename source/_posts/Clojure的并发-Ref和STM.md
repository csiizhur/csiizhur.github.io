---
title: Clojure的并发-Ref和STM
date: 2021-03-06 18:43:16
tags:
---

clojure处理并发的思路与众不同，采用的是所谓的STM的模型（软事物内存）。你可以将STM想象成数据库，只不过是内存型的，它只支持事物的ACI,也就是原子性，一致性，隔离性，但是不包括持久性，因为状态的保存都在内存里。

C lojure的并发API分为四种模型：

+ 管理协作式，同步修改可变状态的Ref
+ 管理非协作式，同步修改可变状态的Atom
+ 管理异步修改可变状态的Agent
+ 管理Thread local变量的Var

## Ref和STM

> 通过ref函数创建一个可变的引用，指向一个不可变的对象：(ref x)

```clojure
(def song (ref #{}))
```

> deref和@ 取引用的内容，解引用使用deref函数，也可以用reader宏@

```clojure
user=> (deref song)
#{}
user=> @song
#{}
```

> ref-set和dosync

```clojure
改变引用指向的内容，使用ref-set函数:(ref-set ref new-value)

user=> (ref-set song #{"dangerous"})
Execution error (IllegalStateException) at user/eval1 (REPL:1).
No transaction running
这是因为引用是可变的，对状态的更新需要进行保护，传统语言的话可能采用锁，Clojure是采用事务，将更新包装到事务里，这是通过dosync实现的：
user=> (dosync (ref-set song #{"dangerous"}))
#{"dangerous"}

dosync的参数接受多个表达式，这些表达式将被包装在一个事物里，事物支持ACI
（1）Atomic，如果你在事务里更新多个Ref，那么这些更新对事务外部来说是一个独立的操作。
（2）Consistent，Ref的更新可以设置 validator，如果某个验证失败，整个事务将回滚。
（3）Isolated，运行中的事务无法看到其他事务部分完成的结果。
dosync更新多个ref
(def singer (ref #{}))
(dosync (ref-set song #{"Dangerous"})
               (ref-set singer #{"MJ"}) )
@song      =>  #{"Dangerous"}
@singer    =>  #{"MJ"}
```

```clojure
user=> (dosync (ref-set song (conj @song "hah")))
["dangerous" "hah"]
user=> (dosync (ref-set song (conj @song "hah")))
["dangerous" "hah" "hah"]
user=> (dosync (ref-set song (conj @song "hah")))
["dangerous" "hah" "hah" "hah"]
user=> @song
["dangerous" "hah" "hah" "hah"]
```

> alter

完全更新整个引用的值还是比较少见，更常见的更新是根据当前状态更新，例如我们向歌曲集合添加一个歌曲，步骤大概是先查询集合内容，然后往集合里添加歌曲，然后更新整个集合：

```
(dosync (ref-set song (conj @song "heal the world")))
(dosync (ref-set song (conj @song "heal the world2")))
(dosync (ref-set song (conj @song "heal the world3")))

(count @song)
```

查询并更新的操作可以合成一步，这是通过alter函数：

```
(alter ref update-fn & args)
```

alter接收一个更新的函数，函数将在更新的时候调用，传入当前状态值并返回新的状态值，因此上面的例子可以改写为：

```clojure
(dosync (alter song conj "heal the world"))
user=> (dosync (alter song conj "we"))
["dangerous" "hah" "hah" "hah" "we"]
```

> **commute**

commute函数是alter的变形，commute顾名思义就是要求update-function是可交换的，它的顺序是可以任意排序。commute的允许的并发程度比alter更高一些，因此性能会更好。但是由于commute要求update-function是可交换的，并且会自动重排序，因此如果你的更新要求顺序性，那么commute是不能接受的,commute仅可用在对顺序性没有要求或者要求很低的场景：例如更新聊天窗口的聊天信息，由于网络延迟的因素和个人介入的因素，聊天信息可以认为是天然排序，因此使用commute还可以接受，更新乱序的可能性很低。
 另一个例子就不能使用commute了，如实现一个计数器：

```
(def counter (ref 0))
```

实现一个next-counter函数获取计数器的下一个值，我们先使用commute实现：

```
(defn next-counter [] (dosync (commute counter inc)))
```

这个函数很简单，每次调用inc递增counter的值，接下来写个测试用例：启动50个线程并发去获取next counter:

```
(dotimes [_ 50] (.start (Thread. #(println (next-counter)))))
```

这段代码稍微解释下，dotimes是重复执行50次，每次启动new并启动一个Thread,这个Thread里干了两件事情：调用next-counter，打印调用结果,第一个版本的next-counter执行下，这是其中一次输出的截取：

```
23
23
23

23
23
23
23
23
23
23
23
23
28
23
21
23
23
23
23
25
28
```

可以看到有很多的重复数值，这是由于重排序导致事务结束后的值不同，但是你查看counter，确实是50:

```
@counter  => 50
```

证明更新是没有问题的，问题出在commute的返回值上。



如果将next-counter修改为alter实现：

```
(defn next-counter [] (dosync (alter counter inc)))
```

此时再执行测试用例，可以发现打印结果完全正确了：

```
……
39
41
42
45
27
46
47
44
48
43
49
40
50
```

查看counter，也是正确更新到50了：

@counter => 50

最佳实践：**通常情况下，你应该优先使用alter**，除非在遇到明显的性能瓶颈并且对顺序不是那么关心的时候，可以考虑用commute替换。

> **validator**

类似数据库，你也可以为Ref添加“约束”，在数据更新的时候需要通过validator函数的验证，如果验证不通过，整个事务将回滚。添加validator是通过ref函数传入metadata的map实现的，例如我们要求歌曲集合添加的歌曲名称不能为空：

(def validate-song
   (partial every? #(not (nil? %))))

(def song (ref #{} :validator validate-song))


validate-song是一个验证函数，partial返回某个函数的半函数（固定了部分参数，部分参数没固定），你可以将partial理解成currying，虽然还是不同的。validate-song调用every?来验证集合内的所有元素都不是nil，其中#(not (nil? %))是一个匿名函数，%指向匿名函数的第一个参数，也就是集合的每个元素。ref指定了validator为validate-song，那么在每次更新song集合的时候都会将新的状态传入validator函数里验证一下，如果返回false，整个事务将回滚：



(dosync (alter song conj nil))
java.lang.IllegalStateException: Invalid reference state (NO_SOURCE_FILE:0)


更新失败，非法的reference状态，查看song果然还是空的：

@song => #{}


更新正常的值就没有问题：

 (dosync (alter song conj "dangerous"))  => #{"dangerous"}

> **ensure**

ensure函数是为了保护Ref不会被其他事务所修改，它的主要目的是为了防止所谓的“**写偏序**”(**write skew**)问题。写偏序问题的产生跟STM的实现有关，clojure的STM实现是基于[MVCC(Multiversion Concurrency Control)](http://en.wikipedia.org/wiki/Multiversion_concurrency_control)——多版本并发控制，对一个Ref保存多个版本的状态值，在更新的时候取得当前状态值的一个隔离的snapshot，更新是基于snapshot进行的。那么我们来看下写偏序是怎么产生，以一个比喻来描述：
 想象有一个系统用于管理美国最神秘的军事禁区——51区的安全巡逻，你有3个营的士兵，每个营45个士兵，并且你**需要保证总体巡逻的士兵人数不能少于100个人**。假设有一天，有两个指挥官都登录了这个管理系统，他们都想从某个军营里抽走20个士兵，假设指挥官A想从1号军营抽走，指挥官B想要从2号军营抽走士兵，他们同时执行下列操作：

Admin 1: if ((G1 - 20) + G2 + G3) > 100 then dispatchPatrol

Admin 2: if (G1 + (G2 - 20) + G3) > 100 then dispatchPatrol


我们刚才提到，Clojure的更新是基于隔离的snapshot，一个事务的更改无法看到另一个事务更改了部分的结果，因此这两个操作都因为满足(45-20)+45+45=115的约束而得到执行，导致实际抽调走了40个士兵，只剩下95个士兵，低于设定的安全标准100人，这就是写偏序现象。
 写偏序的解决就很简单，在执行抽调前加入ensure即可保护ref不被其他事务所修改。ensure比(ref-set ref @ref)允许的并发程度更高一些。



## Atom和缓存

Ref适用的场景是系统中存在多个相互关联的状态，他们需要一起更新，因此需要通过dosync做事务包装。但是如果你有一个状态变量，不需要跟其他状态变量协作，这时候应该使用Atom了。可以将一个Atom和一个Ref一起在一个事务里更新吗？这没办法做到，如果你需要相互协作，你只能使用Ref。Atom适用的场景是状态是独立，没有依赖，它避免了与其他Ref交互的开销，因此性能会更好，特别是对于读来说。

#### 定义Atom,采用atom函数，赋予一个初始状态

```
mem的初始状态定义成一个map
(def mem (atom {}))
```



#### deref和@ 取atom的值

```
@mem         => {}
(deref mem)  => {}
```

#### reset! 重新设置atom的值，不关心当前值

```clojure
(reset! mem {:a 1})
```

#### swap! 如果你的更新需要依赖当前的状态值，或者只想更新状态的某个部分，那么就需要使用swap!（类似alter)

```
(swap! an-atom f &args)

swap! 将函数f作用于当前状态值和额外的参数args之上，形成新的状态值，例如我们给mem加上一个keyword:
user=> (swap! mem assoc :b 2)
{:b 2, :a 1}
```

#### compare and set

```
类似原子变量AtomicInteger之类，atom也可以做compare and set的操作
(compare-and-set! atom oldValue newValue)

当且仅当atom的当前状态值等于oldValue的时候，将状态值更新为newValue，并返回一个布尔值表示成功或者失败:
user=> (def c (atom 1))
#'user/c
user=> (compare-and-set! c 2 3)
false
user=> (compare-and-set! c 1 3)
true
user=> @c
3
```

#### 缓存和atom

+ atom非常适合实现缓存，缓存通常不会跟其他系统状态形成依赖，并且缓存对读的速度要求更高。上面例子中用到的mem其实就是个简单的缓存例子，我们来实现一个putm和getm函数

```
;创建缓存
(defn make-cache [] (atom {}))

;;放入缓存
(defn putm [cache key value] (swap! cache assoc key value))

;;取出
(defn getm [cache key] (key @cache))


   这里key要求是keyword，keyword是类似:a这样的字符序列，你熟悉ruby的话，可以暂时理解成symbol。使用这些API：
user=> (def cache (make-cache))
#'user/cache
user=> (putm cache :a 1)
{:a 1}
user=> (getm cache :a)
1
user=> (putm cache :b 2)
{:b 2, :a 1}
user=> (getm cache :b)
2
```

+ **memoize**函数作用于函数f，产生一个新函数，新函数内部保存了一个缓存，缓存从参数到结果的映射。第一次调用的时候，发现缓存没有，就会调用f去计算实际的结果，并放入内部的缓存；下次调用同样的参数的时候，就直接从缓存中取，而不用再次调用f，从而达到提升计算效率的目的。
  memoize的实现就是基于atom，查看源码：

```
(defn memoize
  [f]
  (let [mem (atom {})]
    (fn [& args]
      (if-let [e (find @mem args)]
        (val e)
        (let [ret (apply f args)]
          (swap! mem assoc args ret)
          ret)))))

内部的缓存名为mem，memoize返回的是一个匿名函数，它接收原有的f函数的参数，if-let判断绑定的变量e是否存在，变量e是通过find从缓存中查询args得到的项，如果存在的话，调用val得到真正的结果并返回；如果不存在，那么使用apply函数将f作用于参数列表之上，计算出结果，并利用swap!将结果加入mem缓存，返回计算结果。
```

#### 性能测试

使用atom实现一个计数器，和使用java.util.concurrent.AtomicInteger做计数器，做一个性能比较，各启动100个线程，每个线程执行100万次原子递增，计算各自的耗时，测试程序如下

```clojure
(ns atom-perf)
(import 'java.util.concurrent.atomic.AtomicInteger)
(import 'java.util.concurrent.CountDownLatch)

(def a (AtomicInteger. 0))
(def b (atom 0))

;;为了性能，给java加入type hint
(defn java-inc [#^AtomicInteger counter] (.incrementAndGet counter))
(defn countdown-latch [#^CountDownLatch latch] (.countDown latch))

;;单线程执行缓存次数
(def max_count 1000000)
;;线程数 
(def thread_count 100)

(defn benchmark [fun]
  (let [ latch (CountDownLatch. thread_count)  ;;关卡锁
         start (System/currentTimeMillis) ]     ;;启动时间
       (dotimes [_ thread_count] (.start (Thread. #(do (dotimes [_ max_count] (fun)) (countdown-latch latch))))) 
       (.await latch)
       (- (System/currentTimeMillis) start)))
         

(println "atom:" (benchmark #(swap! b inc)))
(println "AtomicInteger:" (benchmark #(java-inc a)))

(println (.get a))
(println @b)
```

默认clojure调用java都是通过反射，加入type hint之后编译的字节码就跟java编译器的一致，为了比较公平，定义了java-inc用于调用AtomicInteger.incrementAndGet方法，定义countdown-latch用于调用CountDownLatch.countDown方法，两者都为参数添加了type hint。如果不采用type hint，AtomicInteger反射调用的效率是非常低的。

测试下来，在我的ubuntu上，AtomicInteger还是占优，基本上比atom的实现快上一倍：



atom: 9002
AtomicInteger: 4185
100000000
100000000


按~~照我的理解，这是由于AtomicInteger调用的是native的方法，基于硬件原语做cas，而atom则是用户空间内的clojure自己做的CAS，两者的性能有差距不出意料之外。

~~看了源码，Atom是基于java.util.concurrent.atomic.AtomicReference实现的，调用的方法是

 public final boolean compareAndSet(V expect, V update) {
    return unsafe.compareAndSwapObject(this, valueOffset, expect, update);
  }


而AtomicInteger调用的方法是：



  public final boolean compareAndSet(int expect, int update) {
  return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
  }


两者的效率差距有这么大吗？暂时存疑。

## Agent和Actor

除了用于协调同步的Ref，独立同步的Ref，还有一类非常常见的需求：你可能希望状态的更新是异步，你通常不关心更新的结果，这时候你可以考虑下使用Agent。

#### 创建agent

> 通过agent函数你就可以创建一个agent，指向一个不可变的初始状态。

```
user=> (def counter (agent 0))
#'user/counter

user=> counter
#<Agent@9444d1: 0>
```

#### **取agent的值**，这跟Ref和Atom没啥两样，都是通过deref或者@宏

```
user=> @counter
0
user=> (deref counter)
0
```

#### **更新agent**，通过send或者send-off函数给agent发送任务去更新agent

```
user=> (send counter inc)
#<Agent@9444d1: 0>
```

send返回agent对象，内部的值仍然是0，而非inc递增之后的1，这是因为send是异步发送，更新是在另一个线程执行，两个线程(REPL主线程和更新任务的线程)的执行顺序没有同步，显示什么取决于两者谁更快。更新肯定是发生了，查看counter的值：

user=> @counter
1


  果然更新到了1了。send的方法签名：

(send a f & args)


  其中f是更新的函数，它的定义如下：

(f state-of-agent & args)

  也就是它会在第一个参数接收当前agent的状态，而args是send附带的参数。

  还有个方法，send-off，它的作用于send类似：

user=> (send-off counter inc)
\#<Agent@9444d1: 1>
user=> @counter
2


  send和send-off的区别在于，send是将任务交给一个**固定大小的线程池**执行

final public static ExecutorService pooledExecutor =
    Executors.newFixedThreadPool(2 + Runtime.getRuntime().availableProcessors());

  默认线程池大小是**CPU核数加上2**。因此**send执行的任务最好不要有阻塞的操作**。而send-off则使用没有大小限制（取决于内存）的线程池：



final public static ExecutorService soloExecutor = Executors.newCachedThreadPool();


  因此，**send-off比较适合任务有阻塞的操作**，如IO读写之类。请注意，**所有的agent是共用这些线程池**，这从这些线程池的定义看出来，都是静态变量。

#### **异步转同步**，刚才提到send和send-off都是异步将任务提交给线程池去处理，如果你希望同步等待结果返回，那么可以使用await函数：

```
 (do (send counter inc) (await counter) (println @counter))
```

send一个任务之后，调用await等待agent所有派发的更新任务结束，然后打印agent的值。await是阻塞当前线程，直到至今为止所有任务派发执行完毕才返回。await没有超时，会一直等待直到条件满足，await-for则可以接受等待的超时时间，如果超过指定时间没有返回，则返回nil，否则返回结果。

 (do (send counter inc) (await-for 100 counter) (println @counter))


await-for接受的单位是毫秒。

#### 错误处理

agent也可以跟Ref和Atom一样设置validator，用于约束验证。由于agent的更新是异步的，你不知道更新的时候agent是否发生异常，只有等到你去取值或者更新的时候才能发现：

user=> (def counter (agent 0 :validator number?))
\#'user/counter

user=> (send counter (fn[_] "foo"))
\#<clojure.lang.Agent@4de8ce62: 0>


  强制要求counter的值是数值类型，第二个表达式我们给counter发送了一个更新任务，想将状态更新为字符串"foo"，由于是异步更新，返回的结果可能没有显示异常，当你取值的时候，问题出现了：

user=> @counter
java.lang.Exception: Agent has errors (NO_SOURCE_FILE:0)


 告诉你agent处于不正常的状态，如果你想获取详细信息，可以通过agent-errors函数：

user=> (.printStackTrace (agent-errors counter))
java.lang.IllegalArgumentException: No matching field found: printStackTrace for class clojure.lang.PersistentList (NO_SOURCE_FILE:0)


  你可以恢复agent到前一个正常的状态，通过clear-agent-errors函数：
 

user=> (clear-agent-errors counter)
nil
user=> @counter
0

#### 加入事物

agent跟atom不一样，agent可以加入事务，在事务里调用send发送一个任务，当事务成功的时候该任务将只会被发送一次，最多最少都一次。利用这个特性，我们可以实现在事务操作的时候写文件，达到ACID中的D——持久性的目的:

(def backup-agent (agent "output/messages-backup.clj" ))
(def messages (ref []))
(use '[clojure.contrib.duck-streams :only (spit)])
(defn add-message-with-backup [msg]
    (dosync
      (let [snapshot (commute messages conj msg)]
        (send-off backup-agent (fn [filename]
                    (spit filename snapshot)
                    filename))
      snapshot)))


定义了一个backup-agent用于保存消息，add-message-with-backup函数首先将状态保存到messages，这是个普通的Ref，然后调用send-off给backup-agent一个任务：

 (fn [filename]
     (spit filename snapshot)
     filename)

这个任务是一个匿名函数，它利用spit打开文件，写入当前的快照，并且关闭文件，文件名来自backup-agent的状态值。注意到，我们是用send-off，send-off利用cache线程池，哪怕阻塞也没关系。

利用事务加上一个backup-agent可以实现类似数据库的ACID，但是还是不同的，主要区别在于**backup-agent的更新是异步，并不保证一定写入文件**，因此持久性也没办法得到保证。

#### 关闭线程池

前面提到agent的更新都是交给线程池去处理，在系统关闭的时候你需要关闭这两个线程吃，通过shutdown-agents方法，你再添加任务将被拒绝：

user=> (shutdown-agents)
nil
user=> (send counter inc)
java.util.concurrent.RejectedExecutionException (NO_SOURCE_FILE:0)
user=> (def counter (agent 0))
\#'user/counter
user=> (send counter inc)  
java.util.concurrent.RejectedExecutionException (NO_SOURCE_FILE:0)


哪怕我重新创建了counter，提交任务仍然被拒绝，进一步证明这些***\*线程池是全局共享\****的。

#### 原理浅析

前文其实已经将agent的实现原理大体都说了，agent本身只是个普通的java对象，它的内部维持一个状态和一个队列：

  volatile Object state;
  AtomicReference<IPersistentStack> q = new AtomicReference(PersistentQueue.EMPTY);



任务提交的时候，是封装成Action对象，添加到此队列


  public Object dispatch(IFn fn, ISeq args, boolean solo) {
    if (errors != null) {
      throw new RuntimeException("Agent has errors", (Exception) RT.first(errors));
    }
    //封装成action对象
    Action action = new Action(this, fn, args, solo);
    dispatchAction(action);

​    return this;
  }


  static void dispatchAction(Action action) {
    LockingTransaction trans = LockingTransaction.getRunning();
    // 有事务，加入事务
    if (trans != null)
      trans.enqueue(action);
    else if (nested.get() != null) {
      nested.set(nested.get().cons(action));
    }
    else {
      // 入队
      action.agent.enqueue(action);
    }
  }


send和send-off都是调用Agent的dispatch方法，只是两者的参数不一样，dispatch的第二个参数 solo决定了是使用哪个线程池处理action:

(defn send
 [#^clojure.lang.Agent a f & args]
  (. a (dispatch f args false)))

(defn send-off
 [#^clojure.lang.Agent a f & args]
  (. a (dispatch f args true)))


send-off将solo设置为true，当为true的时候使用cache线程池：


  final public static ExecutorService soloExecutor = Executors.newCachedThreadPool();

  final static ThreadLocal<IPersistentVector> nested = new ThreadLocal<IPersistentVector>();

​    void execute() {
​      if (solo)
​        soloExecutor.execute(this);
​      else
​        pooledExecutor.execute(this);
​    }


执行的时候调用更新函数并设置新的状态：



try {
          Object oldval = action.agent.state;
          Object newval = action.fn.applyTo(RT.cons(action.agent.state, action.args));
          action.agent.setState(newval);
          action.agent.notifyWatches(oldval, newval);
        }
        catch (Throwable e) {
          // todo report/callback
          action.agent.errors = RT.cons(e, action.agent.errors);
          hadError = true;
        }

#### 跟actor的比较

Agent跟Actor有一个显著的不同，agent的action来自于别人发送的任务附带的更新函数，而actor的action则是自身逻辑的一部分。因此，如果想用agent实现actor模型还是相当困难的，下面是我的一个尝试：


(ns actor)

(defn receive [& args]
  (apply hash-map args))
(defn self [] *agent*)

(defn spawn [recv-map]
  (agent recv-map))

(defn ! [actor msg]
  (send actor #(apply (get %1 %2) (vector %2)) msg))
;;启动一个actor
(def actor (spawn 
       (receive :hello #(println "receive "%))))
;;发送消息 hello
(! actor :hello)


  利用spawn启动一个actor，其实本质上是一个agent，而发送通过感叹号!，给agent发送一个更新任务，它从recv-map中查找消息对应的处理函数并将消息作为参数来执行。难点在于消息匹配，匹配这种简单类型的消息没有问题，但是如果匹配用到变量，暂时没有想到好的思路实现，例如实现两个actor的ping/pong。

http://www.blogjava.net/killme2008/archive/2010/07/archive/2010/07/archive/2010/07/archive/2010/07/14/326027.html