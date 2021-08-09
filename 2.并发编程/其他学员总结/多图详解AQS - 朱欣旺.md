# 多图详解AQS

## 什么是AQS

AQS 的全称是 AbstractQueuedSynchronizer ，从字面理解它就是抽象的队列同步器。它是可重入锁、各种同步工具类（共享锁）和条件等待唤醒机制实现的基石。

AQS 有一个重要的属性 state，它的值直接关系着其他线程能否获取到锁。

如果我们看过可重入锁、各种同步工具类（共享锁）的源码，会发现这些锁的关注点都在于通过 AQS 的 state 值或者能否通过 CAS 修改 state 的值来判断当前线程能否获取到锁（这里判断是否能获取到锁都是靠 AQS 的子类 Sync 和 Sync的子类实现的，而这些子类的具体方法是锁自己去实现的——归根到底这部分就是锁来实现的）。如果获取锁成功，直接扣减 AQS 的 State 值，不会涉及到 AQS。但如果当前线程获取锁失败，那么剩下的包括阻塞唤醒线程、重新发起获取锁之类的操作全都都会扔给 AQS 。简单来说就是 AQS 包揽了同步机制的各种工作。这就是为什么理解了 AQS 再去理解各种锁就会非常容易，它的重要性也就不言而喻了。

下图就是线程获取锁的大致流程：

<img src="https://i.loli.net/2021/08/09/xgLhv4a6AZTDUEV.png" alt="xgLhv4a6AZTDUEV.png" style="zoom:50%;">

下面就是使用 AQS 实现的最简单的独占锁，从代码也可以看出 AQS 大大降低了开发锁的难度：

```java
class Mutex {
    
    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int arg) {
            return compareAndSetState(0, 1);
        }

        @Override
        protected boolean tryRelease(int arg) {
            setState(0);
            return true;
        }

        @Override
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }
    }

    private final Sync sync = new Sync();

    public void lock() {
        sync.tryAcquire(1);
    }

    public void unlock() {
        sync.tryRelease(1);
    }

    public boolean isLocked() {
        return sync.isHeldExclusively();
    }

}
```



## AQS 的在 ReentrantLock 中的应用

各种锁对于 AQS 的使用方式大致相同，这里以 ReentrantLock 为例来讲解。

ReentrantLock 的实现比上面的例子会复杂一点，但大体思路是相同的。

ReentrantLock 并不是直接继承自 AQS，它实现了 Lock 的接口并对外提供锁的各种功能。它通过内部的 AQS 子类 **Sync** 来使用 AQS 的功能，这样设计的好处在于锁的功能和同步器的功能划分更清晰，方便扩展和维护。

由于 ReentrantLock 支持公平锁和非公平锁，**Sync** 又有两个分别实现了公平锁和非公平锁功能的子类 **FairSync** 和 **NonfairSync** 。继承关系如下图所示：

<img src="https://i.loli.net/2021/08/09/7lTu1qIycXO2Qp4.png" style="zoom:50%">

这里着重讲一下 ReentrantLock 的 `lock()`和`unlock`方法。`lock()`方法内部就一句代码``sync.lock();`实际是调用的 **FairSync** 或 **NonfairSync** 的`lock()`方法，而`unlock`方法是直接通过 **Sync** 类来调用 AQS 的`release()`方法。

```java
public void lock() {
	sync.lock(); // 实际是调用的 FairSync 或 NonfairSync 的`lock()`方法
}

public void unlock() {
	sync.release(1); // AQS的方法
}
```

下面再看看**FairSync** 或 **NonfairSync** 的`lock()`方法的具体实现：

### **FairSync**

```java
final void lock() {
	acquire(1); // AQS的方法
}
```
### **NonfairSync**

```java
final void lock() {
	if (compareAndSetState(0, 1)) // 尝试直接获取锁
		setExclusiveOwnerThread(Thread.currentThread()); // 获取锁成功后设置当前线程为独占线程
	else
		acquire(1); // AQS的方法
}
```

从上面的代码可以看出，加锁解锁其实本质都是去调用 AQS 的`acquire()`和 `release()`方法。这两个方法在后面会详细讲解。

在 **Sync**和 **Sync** 的子类里还有两个重要的方法：`tryAcquire()`和`tryRelease()`，它们都是AQS 为独占锁提供的勾子方法，分别代表尝试获取锁和尝试释放锁。其中`tryRelease()`是由 **Sync** 来实现的，`tryAcquire()`是由 **Sync** 的子类来实现的。这点其实也比较好理解，ReentrantLock 支持公平锁和非公平锁，这两种锁的差异就体现的尝试获取锁这里，而释放锁的逻辑是一致的。由于 ReentrantLock 不是本文的重点，这两个方法就不详细说了。

### ConditionObject

条件等待和条件唤醒功能一般都是 ReentrantLock 与 AQS 的内部类 ConditionObject 配合实现的。一个 ReentrantLock 可以创建多个 ConditionObject 实例，每个实例对应一个条件队列，以保证每个实例都有自己的等待唤醒逻辑，不会相互影响。

条件队列里的线程对应的节点被唤醒时会被放到 ReentrantLock 的同步队列里，让同步队列去完成唤醒和重新尝试获取锁的工作。可以理解为条件队列是依赖同步队列的，它们协同才能完成条件等待和条件唤醒功能。

## AQS 的构成

讲完应用，下面讲讲AQS 的构成。

 AQS 的继承关系如下图所示：

<img src="https://i.loli.net/2021/08/09/g4zQC65wZVFnjB3.png" style="zoom:50%;">

从图中可以看出 AQS 继承了另外一个抽象类 AbstractOwnableSynchronizer，这个类的功能其实就是持有一个不能被序列化的属性 exclusiveOwnerThread ，它代表独占线程。在属性中记录持有独占锁的线程的目的就是为了实现可重入功能，当下一次获取这个锁的线程与当前持有锁的线程相同时，就可以获取到锁，同时 AQS 的 state 值会加1。

图中红线部分标识了 AQS 的两个内部类，一个是 Node， 一个是 ConditionObject。

### Node

Node就是 AQS 实现各种队列的基本组成单元。它有以下几个属性：

* **waitStatus：**代表节点状态：CANCELLED(1)、SIGNAL(-1)、CONDITION(-2)、PROPAGATE(-3)、0（初始状态）
* **prev：**代表同步队列的上一个节点
* **next：**代表同步队列的下一个节点
* **thread：**节点对应的线程
* **nextWaiter：**在同步队列里用来标识节点是独占锁节点还是共享锁节点，在条件队列里代表条件条件队列的下一个节点

#### 队列

AQS 总共有两种队列，一种是同步队列，代表的是正常获取锁释放锁的队列，一种是条件队列，代表的是每个 ConditionObject 对应的队列，这两种队列都是 **FIFO** 队列，也就是先进先出队列。

##### 同步队列

而同步队列的节点分为两种，一种是独占锁的节点，一种是共享锁的节点，它们唯一的区别就是 nextWaiter 这个指针的值。如果是独占锁的节点，nextWaiter 的值是 null，如果是共享锁的节点，nextWaiter 会指向一个静态变量 SHARED 节点。独占锁队列和共享锁队列如下图所示：

<img src="https://i.loli.net/2021/08/09/EaweKZPbOm8FALt.png" style="zoom:50%;">

##### 条件队列

条件队列是单链，它没有空的头节点，每个节点都有对应的线程。条件队列头节点和尾节点的指针分别是 firstWaiter 和 lastWaiter ，如下图所示：

<img src="https://i.loli.net/2021/08/09/NGZIuyFCjr5UYLw.png" style="zoom:50%;">

#### waiteStatus

讲完队列我们来着重看一下节点的这个属性，它代表着节点的等待状态。这个属性非常重要，如果不理解它，后面 AQS 源码部分也会很难理解。

首先看一下 waiteStatus 的取值：

* **CANCELLED** 取消状态，值为1
* **0** 初始状态
* **SIGNAL** 通知状态，值为 -1
* **CONDITION** 条件等待状态，值为 -2
* **PROPAGATE** 传播状态，值为 -3

这些状态我们一个一个看。

**CANCELLED** 代表着取消状态，它的值是 1，注意这些状态里只有它的值是大于 0 的，所以源码里判断是取消状态是直接通过 waiteStatus 值是否大于 0 来判断。

如果 waiterStatus 的值为 0，有两种情况：1、节点状态值没有被更新过（同步队列里最后一个节点的状态）；2、在唤醒线程之前头节点状态会被被修改为 0。

**SIGNAL** 代表着通知状态，这个状态下的节点如果被唤醒，就有义务去唤醒它的后继节点。这也就是为什么一个节点的线程阻塞之前必须保证前一个节点是 SIGNAL 状态。

**CONDITION** 代表条件等待状态，条件等待队列里每一个节点都是这个状态，它的节点被移到同步队列之后状态会修改为 0。

**PROPAGATE** 传播状态，在一些地方用于修复 bug 和提高性能，减少不必要的循环。

## park() 和 unpark()

讲 AQS 源码之前有一个重要的概念需要理解一下，那就是 Unsafe 这个类的 `park()` 和 `unpark()` 方法。

注意在 AQS 里并没有直接调用 Unsafe 的这两个方法，而是通过 LockSupport 间接调用的 Unsafe 的这两个方法，LockSupport 里面封装了一些参数来简化调用过程。

Unsafe 的这两个方法其实就是对许可的管理，`park()` 方法是让线程去获取一个许可，如果获取失败就阻塞当前线程，`unpark()` 方法是释放一个许可，如果当前线程是阻塞的，会唤醒当前线程。

这个许可简单理解就像一个人要去过一个城关，规定有令牌才能过，没有令牌就得等着。`park()`方法就是城关的守卫来检查你的令牌，你能拿出来就过去了，没拿出来就得原地等着。`unpark()`方法就是有人送来一块令牌，如果发现你已经有了，就不送了，如果发现你正好没有，就送给你。而如果这个时候你刚好等在城门口，那么你顺手就把刚得到的令牌给守卫，守卫就放你过去了。

## AQS 源码分析

AQS 源码分析分为三部分：独占锁部分、共享锁部分和条件等待条件通知部分。

### 独占锁部分

#### acquire()

首先是 `acquire()` 方法，下面是源码:

```java
public final void acquire(int arg) {
	if (!tryAcquire(arg) &&
			acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) // 这里 Node.EXCLUSIVE 的值是 null
			selfInterrupt();
}
```

这里 `tryAcquire(arg)`方法是由具体的锁来实现的，这个方法主要是尝试获取锁，获取成功就不会再执行其他代码了，这个方法结束。获取失败会进入下一步`acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`。这里有个方法嵌套，我们先看`addWaiter(Node.EXCLUSIVE)`方法：

```java
private Node addWaiter(Node mode) {
	Node node = new Node(Thread.currentThread(), mode);
	// Try the fast path of enq; backup to full enq on failure
	Node pred = tail;
	if (pred != null) { // 如果尾节点不为空，就把节点放在尾节点后面并设置为新的尾节点
		node.prev = pred;
		if (compareAndSetTail(pred, node)) { // 尝试把节点设置为新的尾节点
			pred.next = node;
			return node;
		}
	}
	enq(node);
	return node;
}
```

这个方法前半段比较好理解，先创建一个节点，如果有尾节点，就让这个节点指向当前的尾节点，并把它设置成新的尾节点，设置失败也没关系，后面会进入一个重要方法`enq()`。如果当前没有尾节点，会直接进入到`enq()`方法。下面是`enq()`的源码：

```java
private Node enq(final Node node) {
	for (;;) {
		Node t = tail;
		if (t == null) { // 如果尾节点为空，那么队列也为空，新建一个头节点，让 head 和 tail 都指向它
			if (compareAndSetHead(new Node()))
				tail = head;
		} else { // 如果有尾节点，把传入的节点放入队尾
			node.prev = t;
			if (compareAndSetTail(t, node)) {
				t.next = node;
				return t;
			}
		}
	}
}
```

其实这个方法就做两件事：

1、判断如果没有尾节点，那么队列肯定是空的，也不会有头节点，这个时候就要去新增一个空节点，通过 CAS 将这个空节点设置成头节点，然后 tail 指针也指向这个空节点。

2、如果有尾节点，就把当前节点放在尾节点后面，然后通过 CAS 尝试将 tail 指针指向这个节点，直到成功为止。

下图展示了初始队列为空时节点的变化，队列不为空的情况也类似于下图单节点到双节点的情况，都是在尾节点后续追加节点。

<img src="https://i.loli.net/2021/08/09/rnOF1qiIH8flaVZ.png" style="zoom:50%;">

看完了`addWaiter()`方法，接下来就是另一个非常重要的方法`acquireQueued()`，源代码如下：

```java
final boolean acquireQueued(final Node node, int arg) {
	boolean failed = true;
	try {
		boolean interrupted = false;
		for (;;) {
			final Node p = node.predecessor(); // 核验并获取前一个节点，如果前一个节点不存在，直接抛异常
			if (p == head && tryAcquire(arg)) { // 如果前一个节点就是头节点，让这个节点的线程尝试获取锁
				setHead(node); //获取锁成功后把当前节点设置为头节点
				p.next = null; // 将之前头节点的 next 指针置空，后面 GC 时会回收这个节点
				failed = false;
				return interrupted;	
      }
			if (shouldParkAfterFailedAcquire(p, node) && // 判断是否应该阻塞当前线程（核心是判断并修正前面节点的 waitStatus）
				parkAndCheckInterrupt()) // 阻塞当前线程、返回并清除中断标记
				interrupted = true;
		}
	} finally {
		if (failed)
			cancelAcquire(node);
	}
}
```

这个方法也分两种情况：

1、当前活跃的线程对应的节点就是同步队列的第二个节点，那么就让当前线程去尝试获取锁，直到成功为止，如果获取锁成功就把这个节点设置成头节点；

2、当前活跃线程是第三个或者更后面的节点，那么就会进入判断是否需要阻塞并进而阻塞的逻辑。

出现第一种情况有两种可能：

1、这个节点刚入队列而且这个队列只有头节点和这个节点；

2、本来这个节点是排在后面的，前面的节点一个个被唤醒之后它的位置也不断往前移，最终它作为第二个节点也被唤醒了。

整个方法的大致流程如下图所示：

<img src="https://i.loli.net/2021/08/09/lAL2CIwd9HnaZoD.png" style="zoom:50%;">



下面重点讲一下第二种情况，也就是涉及阻塞线程的情况。

先看看`shouldParkAfterFailedAcquire()`源码：

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
	int ws = pred.waitStatus;
	if (ws == Node.SIGNAL) // 判断前面节点状态为 SIGNAL ，返回 true
		return true;
  if (ws > 0) { // 如果前面的节点状态为取消（CANCEL值为1）,就一直向前查找，直到找到状态不为取消的节点，把它放在这个节点后面
  	do {
    	node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    	pred.next = node;
  } else {
  	compareAndSetWaitStatus(pred, ws, Node.SIGNAL); // 如果前面节点不是取消也不是 SIGNAL 状态，将其设置为 SIGNAL 状态
  }
  return false;
}
```

这个方法主要是判断前一个节点的状态：

1、如果是 SIGNAL 就返回 true 表示可以阻塞当前线程；

2、如果前面节点状态大于零，也就是取消状态，那么一直往前移直到它前面的节点不是取消状态；

3、如果不是前两种状态，那么把前一个节点状态设置成 SIGNAL 。

除了第一种状态，后面两种状态都会返回 false，后面经过循环再次进去这个方法。

<img src="https://i.loli.net/2021/08/09/XV7xBqUAHsNCguW.png" style="zoom:50%;">

当前面一个方法返回 true 时，就会进入下一个判断，也就是`parkAndCheckInterrupt()`方法，下面是方法源码：

```java
private final boolean parkAndCheckInterrupt() {
	LockSupport.park(this); // 阻塞当前线程
  return Thread.interrupted(); // 返回并清除当前线程中断状态
}
```

这个方法比较简单，就做两件事：阻塞当前线程和返回并清除中断状态。

上面就是`acquire()`部分的源码，接着讲与它对应的另一个方法`release()`。

#### release()

下面是 `release()` 的源码：

```java
public final boolean release(int arg) {
	if (tryRelease(arg)) { // 尝试释放锁，如果成功则唤醒后继节点的线程
		Node h = head;
		if (h != null && h.waitStatus != 0)
			unparkSuccessor(h); // 唤醒后面节点中第一个非取消状态节点的线程
		return true;
	}
	return false;
}
```

在方法的开始做了一个判断：

1、如果当前线程释放锁失败，就直接返回了；

2、如果释放锁成功，那么就会接着判断头节点是否为空和头节点 waitStatus 是否不为 0 。

这里判断头节点状态是一个比较重要的点。为什么头节点的状态一定不能为 0 呢？从后面要讲到源码可以知道，在唤醒头节点的后继之前会做一个将头节点状态置为 0 的操作（虽然这个操作不一定成功）。如果头节点的状态为 0 了，说明正在释放后继节点，这时候也就不再需要释放了，直接返回 true。

头节点状态判断之后，就会进入到释放后继节点这一步，也就是`unparkSuccessor()`方法：

```java
private void unparkSuccessor(Node node) {
	int ws = node.waitStatus; // 这里取的是头节点的状态
  if (ws < 0)
  	compareAndSetWaitStatus(node, ws, 0); // 尝试将头节点状态设置为 0，不保证能设置成功
  Node s = node.next;
  if (s == null || s.waitStatus > 0) { // 如果后面这个节点状态为取消，那么就找到一个位置最靠前的非取消状态的节点
    s = null;
    for (Node t = tail; t != null && t != node; t = t.prev)
    	if (t.waitStatus <= 0)
      	s = t;
  }
  if (s != null)
  	LockSupport.unpark(s.thread); // 唤醒符合条件的后继节点
}
```

这个方法的核心目标是唤醒头节点符合条件的后继节点。因此前面做了一个判断，如果后面这个节点不是取消状态也不为空，那么就直接唤醒它。如果后面的节点不符合要求，那么就开始从后往前遍历，找到一个最靠前的并且是非取消状态的非空节点，然后唤醒它对应的线程。

整个方法的流程如下图所示：

<img src="https://i.loli.net/2021/08/09/UgRt2nJE1VqyS4m.png" style="zoom:50%;">

### 共享锁部分

#### acquireShared()

了解完独占锁的加锁和解锁的逻辑，接着来讲讲共享锁的加锁和解锁逻辑。

下面是 `acquireShared()` 方法的源码：

```java
public final void acquireShared(int arg) {
	if (tryAcquireShared(arg) < 0)
		doAcquireShared(arg);
}
```

这个方法就两行：第一行判断尝试获取锁的返回值是否小于0，这里的返回值是指当前信号量减去传入的信号量的结果，小于0就代表当前信号量不足，获取锁失败，这时候就需要 AQS 接管了；第二行是执行阻塞和唤醒后获取锁的方法。

下面是`doAcquireShared()` 方法的源码：

```java
private void doAcquireShared(int arg) {
	final Node node = addWaiter(Node.SHARED); // 1、共享节点入队
	boolean failed = true;
	try {
		boolean interrupted = false;
		for (;;) {
			final Node p = node.predecessor();
			if (p == head) {
        int r = tryAcquireShared(arg); // 2、尝试获取共享锁（相当于尝试扣减信号量）
        if (r >= 0) {
          setHeadAndPropagate(node, r); // 3、设置头节点并且做一些判断，符合条件会唤醒下一个节点
          p.next = null; // help GC
          if (interrupted)
            selfInterrupt();
          failed = false;
          return;
        }
      }
      if (shouldParkAfterFailedAcquire(p, node)
        parkAndCheckInterrupt()) // 线程会阻塞在这个位置，被唤醒后再继续循环
        interrupted = true;
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
}
```

这个方法大部分代码与前面讲的 `acquireQueued()` 方法是相同的。这里着重讲不同的地方。

首先是标记的第 1 处 `final Node node = addWaiter(Node.SHARED);` 这里参数是传的静态常量 SHARED ，这个值会赋给新生成节点的 nextWaiter 。正如前面说的，通过 nextWaiter 的值我们就能判断这个节点是独占锁的节点还是共享锁的节点。

然后是标记为 2 的这行代码 `int r = tryAcquireShared(arg);` 这代表尝试获取锁之后的值，如果剩下的信号量不为负，那就代表获取锁成功了，就会进入到标识为 3 的这个方法。

下面我们来看看标记为 3 的 `setHeadAndPropagate(node, r)` 方法的源码：

```java
private void setHeadAndPropagate(Node node, int propagate) {
  Node h = head; // Record old head for check below
  setHead(node); // 1
  if (propagate > 0 || h == null || h.waitStatus < 0 ||  // 2
      (h = head) == null || h.waitStatus < 0) {
    Node s = node.next;
    if (s == null || s.isShared())
      doReleaseShared();
  }
}
```

这个方法代码也不多，主要是两块内容：第一个是 `setHead(node)` 方法，这个方法让第二个节点变成头节点，置空之前头节点的部分指针；第二块内容做了大量的判断，然后如果符合条件会执行 `doReleaseShared();`，这个方法也是后面重点要讲的唤醒共享锁同步队列线程的方法。

这里详细讲一下第二块内容做的这些判断：

* `propagate > 0` ：propagate 是传入的参数，代表获取锁成功之后剩余的信号量，如果为正，说明其他线程也可能获取到锁，就会执行后面的唤醒逻辑；
* `h == null`：之前的头节点是空，这里代表异常情况，也需要唤醒线程避免后面的线程都不会被唤醒的情况出现；
* `h.waitStatus < 0`：这里代表保存旧的头节点和设置新的头节点的间隙又有新的节点将会或已经被阴塞了，这个情况也需要执行唤醒让线程重新尝试获取锁；
* `(h = head) == null `：这里代表新的头节点异常，与旧头节点异常一样需要做唤醒操作；
* `h.waitStatus < 0`：这个代表设置新节点成功到做这个判断的间隙又有新节点将会或已经被阻塞了，同样需要唤醒；
* `s == null`：这个代表队列只有头节点或者发生异常，统一做唤醒操作，主要还是处理异常情况；
* `s.isShared()`：这个判断代表只要是共享节点并且满足唤醒条件都会执行唤醒。

这个方法里实现了链式唤醒：当一个线程被唤醒并获取到锁，如果满足条件就会去唤醒其他线程来尝试获取锁，这种唤醒能一直传递下去，使得共享锁获取锁的效率大大提升。

#### releaseShared()

接着讲另一个重要的方法`releaseShared()`，下面是源码：

```java
public final boolean releaseShared(int arg) {
  if (tryReleaseShared(arg)) {
    doReleaseShared();
    return true;
  }
  return false;
}
```

这个方法除了返回值，核心代码也只有两行：第一行代表尝试释放锁，释放失败就直接返回了，释放成功就会执行唤醒后继节点线程操作；第二行就是具体的唤醒线程的方法；

下面是 `doReleaseShared()` 方法的源码：

```java
private void doReleaseShared() {
  for (;;) {
    Node h = head;
    if (h != null && h != tail) {
      int ws = h.waitStatus;
      if (ws == Node.SIGNAL) {
        if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
          continue;            // loop to recheck cases
        unparkSuccessor(h);
      }
      else if (ws == 0 &&
               !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
        continue;                // loop on failed CAS
    }
    if (h == head)                   // loop if head changed
      break;
  }
}
```

在这个方法的循环里对头节点做了大量的判断，头节点的状态满足条件才会执行唤醒操作，我们挨个来看看这些判断的作用：

* `ws == Node.SIGNAL`：从前面的源码可以知道，一个节点阻塞前它前面的节点的 waiteStatus 必须为 SIGNAL ，如果在做唤醒操作这个值就会变，做这个判断主要是确保当前队列没有其他线程在做唤醒操作；
* `!compareAndSetWaitStatus(h, Node.SIGNAL, 0)`：尝试将头节点 waiteStatus 值设置为 0，代表这个 FIFO 队列正在做唤醒操作，注意与独占锁不一样，这里要确保这个值是设置成功的；
* `ws == 0 &&!compareAndSetWaitStatus(h, 0, Node.PROPAGATE)`：这个判断主要是为了确保前面获取到的头节点 waiteStatus 的值与实时获取的头节点 waiteStatus 值相同。什么样的情况下前面做一个判断的间隙这里头节点的状态就变了呢？那就是有新节点入队放在头节点后面并准备阻塞或者已经阻塞了，由于是否阻塞有不确定性，这里就会重新循环获取最新的状态，避免同时做阻塞和唤醒的动作。

而 `unparkSuccessor()` 方法前面已经讲过，这里就不重复讲了。 

### 条件等待和条件通知

条件等待和条件通知功能主要由 AQS 内部类 ConditionObject 的两个重要的方法： `await()` 和 `signal()` 来实现。

#### await()

`await()`  方法正如字面意思一样，就是等待。它与 Object 对象的 `wait()` 方法不同的是，Object 的 `wait()` 方法调用后，任何对象调用 `notify` 都能唤醒它，而 `await()` 方法调用后，只有调用 `await()` 方法的实例调用的 `notify()` 方法才能唤醒它，因此 `await()` 是一个条件等待方法。

方法的源码如下：

```java
public final void await() throws InterruptedException {
  if (Thread.interrupted())
    throw new InterruptedException();
  Node node = addConditionWaiter(); // 生成一个新的节点添加到条件队列
  int savedState = fullyRelease(node); // 调用同步队列释放锁的方法
  int interruptMode = 0;
  while (!isOnSyncQueue(node)) { // 节点是否已经转移到了同步队列中
    LockSupport.park(this); // 没有被转移就阻塞线程
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) // 根据线程中断状态设置 interruptMode 
      break;
  }
  if (acquireQueued(node, savedState) && interruptMode != THROW_IE) // 调用同步队列阻塞和尝试获取锁的方法
    interruptMode = REINTERRUPT;
  if (node.nextWaiter != null) // clean up if cancelled
    unlinkCancelledWaiters(); // 把节点等待状态（waitStatus）不为 CONDITION 的节点移除
  if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode); // 根据不同的中断模式决定是抛出中断异常还是重新标记中断
}
```

需要特别注意的是这个方法是支持中断的，而且方法中很多判断和方法都是与中断有关的，具体哪些地方什么情况会抛出中断异常这里不详细说，这个不是本文的重点。

首先讲讲 `addConditionWaiter()` 这个方法，源码如下：

```java
private Node addConditionWaiter() {
  Node t = lastWaiter;
  // If lastWaiter is cancelled, clean out.
  if (t != null && t.waitStatus != Node.CONDITION) {
    unlinkCancelledWaiters(); // 把所有节点等待状态不为 CONDITION 的节点移除
    t = lastWaiter;
  }
  Node node = new Node(Thread.currentThread(), Node.CONDITION); // 新建一个条件队列的节点
  if (t == null)
    firstWaiter = node;
  else
    t.nextWaiter = node;
  lastWaiter = node;
  return node;
}
```

这个方法做了两件事：

1、如果队列不为空而且最后一个节点等待状态异常，就做一个全队列扫描，去掉异常的节点；

2、把节点入队，这里要做一个判断：如果队列为空，把新节点作为头节点，如果队列非空，把新节点放在队尾。

接着是 `isOnSyncQueue()` 方法，源码如下：

```java
final boolean isOnSyncQueue(Node node) {
  if (node.waitStatus == Node.CONDITION || node.prev == null)
    return false;
  if (node.next != null) // If has successor, it must be on queue
    return true;
  return findNodeFromTail(node);
}
```

这个方法主要是用来判断当前线程的节点是否已经在同步队列了，这个方法涉及三个判断：

1、如果节点等待状态是 CONDITION 或者节点的 prev 指针为空（节点在同步队列这个指针才有值），那么一定不是在同步队列；

2、如果节点的 next 指针不为空，那么一定在同步队列；

3、遍历同步队列，看队列中有没有节点与这个节点相同。

#### signal()

`signal()` 方法是与 `await()`  方法对应的，一个负责通知，一个负责等待。

下面是 `signal` 方法的源码：

```java
public final void signal() {
  if (!isHeldExclusively())
    throw new IllegalMonitorStateException();
  Node first = firstWaiter;
  if (first != null)
    doSignal(first);
}
```

`isHeldExclusively()` 这个方法返回的是该线程是否正在独占资源，如果不是的话会抛出异常。

整个 `signal()` 方法的重点里面调用的 `doSignal()` 方法，传入的参数是头节点。

下面是 `doSignal()` 的源码：

```java
private void doSignal(Node first) {
  do {
    if ( (firstWaiter = first.nextWaiter) == null) // 判断头节点的下一个节点是否为 null
      lastWaiter = null; // 如果队列只有头节点，将 lastWaiter 指针置为 null
    first.nextWaiter = null; // 迁移节点前先将节点的 nextWaiter 置为 null
  } while (!transferForSignal(first) && // 把头节点迁移到同步队列中去
           (first = firstWaiter) != null); // 没有迁移成功就重新获取头节点判断不为空继续循环
}
```

这个方法也没有太复杂的内容，具体可以看看上面的注释，这里详细讲讲 `transferForSignal()` 。

方法源码如下：

```java
final boolean transferForSignal(Node node) {
  if (!compareAndSetWaitStatus(node, Node.CONDITION, 0)) // 将节点 waitStatus 通过 CAS 更新为 0
    return false; // 更新失败说明节点等待状态已经变了，返回 false 重新获取头节点然后重试
  Node p = enq(node); // 将节点放入同步队列中
  int ws = p.waitStatus;
  if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL)) // 判断前一个节点等待状态，如果状态是正常的赋值为 SIGNAL
    LockSupport.unpark(node.thread); // 前面节点状态为取消或就会唤醒当前节点，避免后面没办法被唤醒
  return true;
}
```

整个迁移队列变化如下图所示：

<img src="https://i.loli.net/2021/08/09/QKMvj8ezx4OYsT1.png" style="zoom:50%;">

## 总结

以上就是关于 AQS 的全部内容，看到这里大家应该就会有一个直观的感受：AQS 其实核心就是队列，队列又是为各种锁服务的。了解这些队列节点的阻塞唤醒时机对我们去了解各种锁非常有帮助。

