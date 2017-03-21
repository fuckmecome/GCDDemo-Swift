## GCD 内容相关Demo, Swift版
Swift3.0相关代码已在github上更新。之前关于iOS开发多线程的内容发布过一篇博客，其中介绍了NSThread、操作队列以及GCD，介绍的不够深入。今天就以GCD为主题来全面的总结一下GCD的使用方式。GCD的历史以及好处在此就不做过多的赘述了。本篇博客会通过一系列的实例来好好的总结一下GCD。GCD在iOS开发中还是比较重要的，使用场景也是非常多的，处理一些比较耗时的任务时基本上都会使用到GCD, 在使用是我们也要主要一些线程安全也死锁的东西。

本篇博客中对iOS中的GCD技术进行了较为全面的总结，下方模拟器的截图就是我们今天要介绍的内容，都是关于GCD的。下方视图控制器中每点击一个Button都会使用GCD的相关技术来执行不同的内容。本篇博客会对使用到的每个技术点进行详细的讲解。在讲解时，为了易于理解，我们还会给出原理图，这些原理图都是根据本篇博客中的实例进行创作的，在其他地方可见不着。

![](http://images2015.cnblogs.com/blog/545446/201605/545446-20160515214503648-186877899.png)

上面每个按钮都对应着一坨坨的代码，上面这个截图算是我们本篇博客的一个，下面我们将会对每坨代码进行详细的介绍。通过这些介绍，你应该对GCD有了更全面而且更详细的了解。建议参考着下方的介绍，然后自己动手去便实现代码，这样效果是灰常的好的。本篇博客中的所有代码都会在github上进行分享，本篇博客的后方会给出github分享地址。其实本篇博客可以作为你的GCD参考手册，虽然本篇博客没有囊括所有GCD的东西，但是平时经常使用的部分还是有的。废话少说，进入今天博客的主题，接下来我们将一个Button接着一个Button的介绍。

## 一、常用GCD方法的封装

为了便于实例的实现，我们首先对一些常用的GCD方法进行封装和提取。该部分算是为下方的具体实例做准备的，本部分封装了一些下面示例所公用的方法。接下来我们将逐步的对每个提取的函数进行介绍，为下方示例的实现做准备。在封装方法之前，要说明一点的是在GCD中我们的任务是放在队列中在不同的线程中执行的，要明白一点就是我们的任务是放在队列中的Block中，然后Block再在相应的线程中完成我们的任务。

如下图所示，在下方队列中存入了三个Block，每个Block对应着一个任务，这些任务会根据队列的特性已经执行方式被放到相应的线程中来执行。队列可分为并行队列（Concurrent Qeueu）和串行队列（Serial Queue），队列可以进行同步执行(Synchronize)以及异步执行（Asynchronize）, 稍后会进行详细的分析与介绍。我们要知道队列第GCD的基础。

![](http://images2015.cnblogs.com/blog/545446/201605/545446-20160512103228905-1463046040.png)

#### 1.获取当前线程与当前线程休眠

首先我们将获取当前线程的方法进行封装，因为有时候我们会经常查看我们的任务是在那些线程中执行的。在此我们使用了NSThread的currentThread()方法来获取当前线程。下方的getCurrentThread()方法就是我们提取的获取当前线程的方法。方法内容比较简单，在此就不做过多赘述了。
```Swift
/**
 获取当前线程
 
 - returns: NSThread
 */
func getCurrentThread() -> Thread {
    let currentThread = Thread.current
    return currentThread
}
```
　　

上述代码段是获取当前线程的方法，接着我们要实现一个让当前线程休眠的方法。因为我们在示例时，常常会让当前线程来休眠一段时间来模拟那些耗时的操作。下方代码段中的currentThreadSleep()函数就是我们提取的当前线程休眠的函数。该函数有一个NSTimeInterval类型的参数，该参数就是要休眠的时间。NSTimeInterval其实就是Double类型的别名，所以我们在调用currentThreadSleep()方法时需要传入一个Double类型的休眠时间。当然你也可以调用sleep()方法来对当前线程进行休眠，但是需要注意的是sleep()的参数是UInt32位的整型。下方就是我们休眠当前线程的函数。
```Swift
/**
 当前线程休眠
 
 - parameter timer: 休眠时间/单位：s
 */
func currentThreadSleep(_ timer: TimeInterval) -> Void {
    Thread.sleep(forTimeInterval: timer)
    
    //或者使用
    //sleep(UInt32(timer))
}
```

#### 2.获取主队列与全局队列

下方封装的getMainQueue()函数就是获取主队列的函数，因为有时候我们在其他线程中处理完耗时的任务（比如网络请求）后，需要在主队列中对UI进行更新。因为我们知道在iOS中有个RunLoop的概念，在iOS系统中触摸事件、屏幕刷新等都是在RunLoop中做的。因为本篇博客的主题是GCD, 在此就对RunLoop做过多的赘述了，如果你对RunLoop不太了解，那么你就先简单将RunLoop理解成1/60执行一次的循环即可，当然真正的RunLoop要比一个单纯的循环复杂的多，以后有机会的话在以RunLoop为主题更新一篇博客吧。言归正传，下方就是获取我们主队列的方法，简单一点的说因为我们要更新UI，所以要获取主队列。
```Swift
/**
 获取主队列
 
 - returns: dispatch_queue_t
 */
func getMainQueue() -> DispatchQueue {
    return DispatchQueue.main
}
```
　　

接下来我们要封装一个获取全局队列（Global Queue）的函数，在封装函数之前我们先来聊聊什么是全局队列。全局队列是系统提供的一个队列，该队列拿过来就能用，按执行方式来说，全局队列应该称得上是并行队列，关于串并行队列的具体概念下方会给出介绍。我们在获取全局队列的时候要知道其队列的优先级，优先级越高的队列就越先执行，当然该处的优先级不是绝对的。队列真正的执行顺序还需要根据CUP当前的状态来定，大部分是按照你指定的队列优先级来执行的，不过也有例外。下方实例会给出详细的介绍。下方就是我们获取全局队列的函数，在获取全局队列为为全局队列指定一个优先级，默认为DISPATCH_QUEUE_PRIORITY_DEFAULT。
```Swift
/**
 获取全局队列
*/

func getGlobalQueue() -> DispatchQueue {
    return DispatchQueue.global()
    
}

```
　　

#### 3.创建串行队列与并行队列

因为我们在实现实例时会创建一些并行队列和串行队列，所以我们要对并行队列的创建于串行队列的创建进行提取。GCD中是调用dispatch_queue_create()函数来创建我们想要的线程的。dispatch_queue_create()函数有两个参数，第一个参数是队列的标示，用来标示你创建的队列对象，一般是域名的倒写如“cn.zeluli”这种形式。第二个参数是所创建队列的类型，DISPATCH_QUEUE_CONCURRENT就说明创建的是并行队列，DISPATCH_QUEUE_SERIAL表明你创建的是串行队列。至于两者的区别，还是那句话，下方实例中会给出详细的介绍。
```Swift
/**
 创建并行队列
 
 - parameter label: 并行队列的标记
 
 - returns: 并行队列
 */
func getConcurrentQueue(_ label: String) -> DispatchQueue {
    return DispatchQueue(label: label, attributes: DispatchQueue.Attributes.concurrent)
}


/**
 创建串行队列
 
 - parameter label: 串行队列的标签
 
 - returns: 串行队列
 */
func getSerialQueue(_ label: String) -> DispatchQueue {
    return DispatchQueue(label: label)
}
```

### 二、同步执行与异步执行

同步执行可分为串行队列的同步执行和并行队列的同步执行，而异步执行又可分为串行队列的异步执行和并行队列的异步执行。也许听起来有些拗口，不通过下方的图解你会很好的理解这一概念。上一部分算是我们的准备工作，接下来才是我们真正的主题。在第一部分我们实现了获取当前线程，对当前线程休眠，获取主队列和全局队列以及创建并行队列和串行队列。在本部分将要利用上述函数来进一步讨论串行队列与并行队列的同步执行，以及串行队列与并行队列的异步执行。并且会给出同步执行与异步执行的区别。

在聊同步执行与异步执行之前我们先聊聊串行队列（Serial Queue）与并行队列（Concurrent Queue）的区别。无论是Serial Queue还是Concurrent Queue，都是队列，只要是队列都遵循FIFO（First In First Out -- 先入先出）的规则，排队嘛，当然是谁先来的谁先走了。不过在Serial Queue中要等到前面的任务出队列并执行完后，下一个任务才能出队列进行执行。而Concurrent Queue则不然，只要是队列前面的任务出队列了，并且还有有空余线程，不管前面的任务是否执行完了，下一任务都可以进行出队列。

关于串行队列和并行队列的问题，我们可以拿银行办业务排队来类比一下。比如你现在在串行队列中排的是1号窗口，你必须等前面一个人在1号窗口办完业务你才可以去1号窗口中去办你的业务，就算其他窗口空着你也不能去，因为你选择的是串行队列。但是如果你是在并行队列中的话，只要你前面的人去窗口中办业务了，此时你无需关系你前面的人的业务是否办完了，只要有其他窗口空着你就可以去办理你的业务。总结一下：串行队列就是认准一个线程，一条道走到黑，比较专注；并行队列就是能利用其他线程就利用，比较灵活，不钻牛角尖。接下来我们要看一下两个队列的不同执行方法。

#### 1.同步执行

首先我们先来介绍同步执行，关于同步执行的主要实例对应着“同步执行串行队列”和“同步执行并行队列”这两个按钮。Serial Queue可以同步执行，Concurrent Queue亦可以同步执行。我们先抛开队列，看一下同步执行的代码如何。下方的函数就是对同步执行的任务进行封装。同步执行就是使用dispatch_sync()方法进行执行。在下方函数中通过for-in循环以同步执行的方式往queue（队列）中添加了3个Block执行块。函数的参数是队列类型（dispatch_queue_t）,可以给该函数传入串行队列和并行队列。
```Swift
/**
队列的同步执行
 
 - parameter queue: 队列
 */
func performQueuesUseSynchronization(_ queue: DispatchQueue) -> Void {
    
    for i in 0..<3 {
        queue.sync {
            currentThreadSleep(1)
            print("当前执行线程：\(getCurrentThread())")
            print("执行\(i)")
        }
        print("\(i)执行完毕")
    }
    print("所有队列使用同步方式执行完毕")
}
```

也就是说要同步执行串行队列就给函数传入串行队列的对象，如果要同步执行并行队列就传入并行队列对象。此时我们就用到了之前封装的创建串行队列和并行队列的方法（参见第一部分）。下方代码段就是点击“同步执行串行队列”和“同步执行并行队列”这两个按钮所做的事情。点击“同步执行串行队列”按钮时就创建一个串行队列的对象传给上面同步执行的函数（performQueuesUseSynchronization()），点击“同步执行并行队列”按钮时就创建一个并行队列的对象给上面的函数。
```Swift
    //同步执行串行队列
    @IBAction func tapButton1(_ sender: AnyObject) {
        print("\n同步执行串行队列")
        performQueuesUseSynchronization(getSerialQueue("syn.serial.queue"))
    }
    
    //同步执行并行队列
    @IBAction func tapButton2(_ sender: AnyObject) {
        print("\n同步执行并行队列")
        performQueuesUseSynchronization(getConcurrentQueue("syn.concurrent.queue"))
    }

```
　

下方截图是点击两个按钮所运行的结果。红框中是同步执行串行队列的结果，可以看出来是在当前线程（主线程）下按着FIFO的顺序来执行的。而绿框中的是同步执行并行队列的运行结果，从结果中部门不难看出，与红框中的结果一致，也是在当前线程中按着FIFO的顺序来执行的。

![](http://images2015.cnblogs.com/blog/545446/201605/545446-20160512125713093-1246190699.png)
　　

通过上面两种不同队列的同步执行方式我们给出了下面的分析图。Serial Queue与Concurrent Queue中都有4个Block（编号为1--4），然后使用dispatch_sync()来同步执行。由上述示例我们可以得出，同步执行方式，也就是使用dispatch_sync()来执行队列不会开辟新的线程，会在当前线程中执行任务。如果当前线程是主线程的话，那么就会阻塞主线程，因为主线程被阻塞了，就会会造成UI卡死的现象。因为同步执行是在当前线程中来执行的任务，也就是说现在可以供队列使用的线程只有一个，所以串行队列与并行队列使用同步执行的结果是一样的，都必须等到上一个任务出队列并执行完毕后才可以去执行下一个任务。我们可以使用同步执行的这个特点来为一些代码块加同步锁。下方就是上面代码以及执行结果的描述图。

![](http://images2015.cnblogs.com/blog/545446/201603/545446-20160330103502082-703612739.png)
   

 

#### 2、异步执行

接下来我们看异步执行，同样异步执行也分为串行队列的异步执行和并行队列的异步执行。在GCD中使用dispatch_async()函数来进行异步执行，dispatch_async()函数的参数与dispatch_sync()函数的参数一致。只不过是dispatch_async()异步执行不会在当前线程中执行，它会开辟新的线程，所以异步执行不会阻塞当前线程。下方代码段就是我们封装的异步执行的函数，其中主要是对dispatch_async()函数的使用。下方为了让队列中的Block的三个输出语句顺序输出，我们将其放在了一个同步队列中来执行，从而这三个输出语句可以顺序执行。
```Swift
/**
 队列的异步执行
 
 - parameter queue: 队列
 */
func performQueuesUseAsynchronization(_ queue: DispatchQueue) -> Void {
    
    //一个串行队列，用于同步执行
    let serialQueue = getSerialQueue("serialQueue")
    
    for i in 0..<3 {
        queue.async {
            currentThreadSleep(Double(arc4random()%3))
            let currentThread = getCurrentThread()
            
            serialQueue.sync(execute: {              //同步锁
                print("Sleep的线程\(currentThread)")
                print("当前输出内容的线程\(getCurrentThread())")
                print("执行\(i):\(queue)\n")
            })
        }
        
        print("\(i)添加完毕\n")
    }
    print("使用异步方式添加队列")
}

```
　　 

##### (1)、串行队列的异步执行

有了上面的函数后，我们就可以给上面的函数传入Serial Queue队列的对象，从而观察串行队列异步执行结果。对应这我们第一张截图中的“异步执行串行队列”的按钮，下方是点击该按钮执行的方法。在该按钮点击的方法中我们调用了performQueuesUseAsynchronization()方法，并且传入了一个串行队列。也就是串行队列的异步执行。
```Swift
    @IBAction func tapButton3(_ sender: AnyObject) {
        print("\n异步执行串行队列")
        performQueuesUseAsynchronization(getSerialQueue("asyn.serial.queue"))
    }
```
　　

点击按钮就会执行上述方法，下方是点击按钮后，也就是“异步执行串行队列”时在控制台中输出的结果。从输出结果中我们不难看出，异步执行并没有阻塞当前线程。使用dispatch_saync()开辟了新的线程（线程的number = 3）来执行Block中的内容。而Block内容外的东西依然在之前的线程（在该示例中是main_thread）中进行执行。从下方的结果中来分析，就是for循环执行完毕后主线程的任务就结束了，至于Block中的内容就交给新开辟的线程3来执行了。

![](http://images2015.cnblogs.com/blog/545446/201605/545446-20160512143000562-264723884.png)
　　

 

根据上面的输出结果，我们可以画出下方异步执行串行队列的分析图。在线程1中的一个串行队列如果使用异步执行的话，会开辟一个新的线程2来执行队列中的Block任务。在新开辟的线程中依然是FIFO, 并且执行顺序是等待上一个任务执行完毕后才开始执行下一个任务。如下所示。

![](http://images2015.cnblogs.com/blog/545446/201603/545446-20160330103516660-460259528.png)
 　　

#### (2)、并行队列的异步执行

接下来来讨论一下并行队列的异步执行方式。其实并行队列与异步执行方式相结合才能大大的提供效率，因为使用异步执行并行队列时会开辟多个线程来同时执行并行队列中的任务。比如现在开辟了10个线程，那么异步队列会根据FIFO的顺序出来10个任务，这10个任务会进入到不同的线程中来执行，至于每个任务执行完的先后顺序由每个任务的复杂度而定。异步队列的特点是只要有可用的线程，任务就会出队列进行执行，而不关心之前出队列的任务（Block）是否执行完毕。下方的方法就是点击“异步执行并行队列”按钮所调用的方法。该方法会调用performQueuesUseAsynchronization()函数，并传入一个并行队列的对象。
```Swift
    @IBAction func tapButton4(_ sender: AnyObject) {
        print("\n异步执行并行队列")
        performQueuesUseAsynchronization(getConcurrentQueue("asyn.concurrent.queue"))
        
    }
```
　　

点击按钮就会执行上述方法，并行队列就会异步执行。下方结果就是并行队列异步执行后输出的结果，解析来让我们来分析一下输出结果。下方第一个红框中是并行队列中任务的顺序，由前到后为0、1、2，紧接着是每个任务执行后所输出的结果。从任务执行完打印结果我们可以看出，执行完成的顺序是2、1、0，每个任务都会在一个新的线程中来执行的。如果你在点击一下按钮，执行完成的顺序有可能是2、0、1等其他的顺序，所以并行队列异步执行中每个任务结束时间有主要由任务本身的复杂度而定的。

![](http://images2015.cnblogs.com/blog/545446/201605/545446-20160512145425140-1349786405.png)

根据上面的执行结果，我们画出了下方的解说图。当并行队列异步执行时会开辟多个新的线程来执行队列中的任务，队列中的任务出队列的顺序仍然是FIFO，只不过是不需要等到前面的任务执行完而已，只要是有空余线程可以使用就可以按FIFO的顺序出队列进行执行。 

![](http://images2015.cnblogs.com/blog/545446/201603/545446-20160330103525363-1740347484.png)



### 三、延迟执行

在GCD中我们使用dispatch_after()函数来延迟执行队列中的任务, dispatch_after()是异步执行队列中的任务的，也就是说使用dispatch_after()来执行队列中的任务不会阻塞当前任务。等到延迟时间到了以后就会开辟一个新的线程然后执行队列中的任务。要注意一点是延迟时间到了后再开辟新的线程然后立即执行队列中的任务。下方是dispatch_after()函数的使用方式。

在下方代码中使用了两种方式来创建延迟时间，一个是使用dispatch_time()来创建延迟时间，另一个是使用dispatch_walltime()来创建时间。前者是取的是当前设备的时间，后者去的是挂钟的时间，也就是绝对时间，如果设备休眠了那么前者也就休眠了，而后者是是根据挂钟时间不会有当前设备的状态而左右的。下面在创建dispatch_time_t对象的时候，有一个参数是NSEC_PER_SEC，从命名只能怪我们就可以知道NSEC_PER_SEC表示什么意思，就是每秒包含多少纳秒。你可以将该值进行打印，发现NSEC_PER_SEC = 1_000_000_000。也就是一秒等于10亿纳秒。如果下方的time不乘于NSEC_PER_SEC那么就代表1纳秒的时间，也就是说此处的时间是按纳秒（nanosecond）来计算的。下方就是延迟执行的的代码，因为改代码输出结果比较简单，在此就不做过多的赘述了。需要注意的是延迟执行会在新开辟的队列中进行执行，不会阻塞新的线程。
```Swift
/**
 延迟执行
 
 - parameter time: 延迟执行的时间
 */
func deferPerform(_ time: Double) -> Void {
    
    //dispatch_time用于计算相对时间,当设备睡眠时，dispatch_time也就跟着睡眠了
    let delayTime: DispatchTime = DispatchTime.now() + Double(Int64(time * Double(NSEC_PER_SEC))) / Double(NSEC_PER_SEC)
    getGlobalQueue().asyncAfter(deadline: delayTime) {
        print("执行线程：\(getCurrentThread())\ndispatch_time: 延迟\(time)秒执行\n")
    }
    
    //dispatch_walltime用于计算绝对时间,而dispatch_walltime是根据挂钟来计算的时间，即使设备睡眠了，他也不会睡眠。
    let nowInterval = Date().timeIntervalSince1970
    let nowStruct = timespec(tv_sec: Int(nowInterval), tv_nsec: 0)
    let delayWalltime = DispatchWallTime(timespec: nowStruct)
    getGlobalQueue().asyncAfter(wallDeadline: delayWalltime) {
        print("执行线程：\(getCurrentThread())\ndispatch_walltime: 延迟\(time)秒执行\n")
    }
    
    print(NSEC_PER_SEC) //一秒有多少纳秒
}

```

### 四、队列的优先级

队列也是有优先级的，但其优先级不是绝对的大部分情况因为XUN内核用于GCD不是实时性的，优先级只是大致的来判断队列的执行优先级。队列分为四个优先级，由高到底分别是High > Default > Low > Background。上面在获取全局队列时我们可以为获取的队列指定优先级，并且可以使用dispatch_set_target_queue()函数将一个队列的优先级赋值给另一个队列。下方我们先给全局队列指定优先级，然后在将其赋值给其他队列。

#### 1.为全局队列指定优先级

本部分对应着“设置全局队列的优先级”这个button，点击该button就会获取4个不同优先级的全局队列，然后异步进行全局队列的执行，最后观察执行的结果。下方就是点击该按钮所要执行的函数。我先获取了四种不同优先级的全局队列，然后进行异步执行，并打印执行结果。

![](http://images2015.cnblogs.com/blog/545446/201605/545446-20160512165620937-1464678076.png)

上述代码的运行结果如下，虽然在上述代码中优先级高的代码放在了最后来进行异步执行，可是却先被打印了。打印的顺序是Hight->Default->Low->Background，这个打印顺序就是其执行顺序，从打印顺序中我们不难看出优先级高的先被执行。当然这不是绝对的。

![](http://images2015.cnblogs.com/blog/545446/201605/545446-20160512165807843-1528734355.png)
　　 

#### 2.为自创建的队列指定优先级

在GCD中你可以使用dispatch_set_target_queue()函数为你自己创建的队列指定优先级，这个过程还需借助我们的全局队列。下方的代码段中我们先创建了一个串行队列，然后通过dispatch_set_target_queue()函数将全局队列中的高优先级赋值给我们刚创建的这个串行队列，如下所示。
```Swift
/**
 给串行队列或者并行队列设置优先级
 */
func setCustomeQueuePriority() {
    //优先级的执行顺序也不是绝对的
    // DispatchQoS.userInteractive
    //Work is virtually instantaneous.
    
    //DispatchQoS.userInitiated
    //Work is nearly instantaneous, such as a few seconds or less.
    
    //DispatchQoS.utility
    //Work takes a few seconds to a few minutes.
    
    //DispatchQoS.background
    //Work takes significant time, such as minutes or hours.
    
    print("userInteractive & userInitiated")
    let queue1 = DispatchQueue(label:"zeluli.queue1", qos: DispatchQoS.userInteractive)
    let queue2 = DispatchQueue(label:"zeluli.queue2", qos: DispatchQoS.userInitiated)
    queue1.async {
        for i in 100..<110{
            print("😄", i, getCurrentThread())
        }
    }
    
    queue2.async {
        for i in 200..<210{
            print("😭", i, getCurrentThread())
        }
    }
    
    
    sleep(1)
    
    print("\n\n=========第二批==========\n")
    print("userInitiated & utility")
    let queue3 = DispatchQueue(label:"zeluli.queue3", qos: DispatchQoS.userInitiated)
    let queue4 = DispatchQueue(label:"zeluli.queue4", qos: DispatchQoS.utility)
    
    queue3.async {
        for i in 300..<310{
            print("😄", i, getCurrentThread())
        }
    }
    
    queue4.async {
        for i in 400..<410{
            print("😭", i, getCurrentThread())
        }
    }
    
    
    sleep(1)
    print("\n\n=========第三批==========\n")
    print("utility & background")
    let queue5 = DispatchQueue(label:"zeluli.queue5", qos: DispatchQoS.utility)
    let queue6 = DispatchQueue(label:"zeluli.queue6", qos: DispatchQoS.background)
    
    queue5.async {
        for i in 500..<510{
            print("😄", i, getCurrentThread())
        }
    }
    
    queue6.async {
        for i in 600..<610{
            print("😭", i, getCurrentThread())
        }
    }
}

```
　　

 

### 五、任务组dispatch_group

GCD的任务组在开发中是经常被使用到，当你一组任务结束后再执行一些操作时，使用任务组在合适不过了。dispatch_group的职责就是当队列中的所有任务都执行完毕后在去做一些操作，也就是说在任务组中执行的队列，当队列中的所有任务都执行完毕后就会发出一个通知来告诉用户任务组中所执行的队列中的任务执行完毕了。关于将队列放到任务组中执行有两种方式，一种是使用dispatch_group_async()函数，将队列与任务组进行关联并自动执行队列中的任务。另一种方式是手动的将队列与组进行关联然后使用异步将队列进行执行，也就是dispatch_group_enter()与dispatch_group_leave()方法的使用。下方就给出详细的介绍。

#### 1.队列与组自动关联并执行

首先我们来介绍dispatch_group_async()函数的使用方式，该函数会将队列与相应的任务组进行关联，并且自动执行。当与任务组关联的队列中的任务都执行完毕后，会通过dispatch_group_notify()函数发出通知告诉用户任务组中的所有任务都执行完毕了。使用通知的方式是不会阻塞当前线程的，如果你使用dispatch_group_wait()函数，那么就会阻塞当前线程，直到任务组中的所有任务都执行完毕。

下方封装的函数就是使用dispatch_group_async()函数将队列与任务组进行关联并执行。首先我们创建了一个concurrentQueue并行队列，然后又创建了一个类型为dispatch_group_t的任务组group。使用dispatch_group_async()函数将两者进行关联并执行。使用dispatch_group_notify()函数进行监听group中队列的执行结果，如果执行完毕后，我们就在主线程中对结果进行处理。dispatch_group_notify()函数有两个参数一个是发送通知的group，另一个是处理返回结果的队列。
```Swift
/**
 一组队列执行完毕后在执行需要执行的东西，可以使用dispatch_group来执行队列
 */
func performGroupQueue() {
    print("\n任务组自动管理：")
    
    let concurrentQueue: DispatchQueue = getConcurrentQueue("cn.zeluli")
    let group: DispatchGroup = DispatchGroup()
    
    //将group与queue进行管理，并且自动执行
    for i in 1...3 {
        concurrentQueue.async(group: group) {
            currentThreadSleep(1)
            print("任务\(i)执行完毕\n")
        }
    }
    
    //队列组的都执行完毕后会进行通知
    group.notify(queue: getMainQueue()) {
        print("所有的任务组执行完毕！\n")
    }

    print("异步执行测试，不会阻塞当前线程")
}
```
　　
调用上述函数的输出结果如下。从输出结果中我们不难看出，队列中任务的执行以及通知结果的处理都是异步执行的，不会阻塞当前的线程。在任务组中所有任务都处理完毕后，就会在主线程中执行dispatch_group_notify()中的闭包块。

![](http://images2015.cnblogs.com/blog/545446/201605/545446-20160513101051827-1882544247.png)
　　

#### 2.手动关联队列与任务组

接下来我们将手动的管理任务组与队列中的关系，也就是不使用dispatch_group_async()函数。我们使用dispatch_group_enter()与dispatch_group_leave()函数将队列中的每次任务加入到到任务组中。首先我们使用dispatch_group_enter()函数进入到任务组中，然后异步执行队列中的任务，最后使用dispatch_group_leave()函数离开任务组即可。下面的函数中我们使用了dispatch_group_wait()函数，该函数的职责就是阻塞当前线程，来等待任务组中的任务执行完毕。该函数的第一个参数是所要等待的group，第二个参数是等待超时时间，此处我们设置的是DISPATCH_TIME_FOREVER，就说明等待任务组的执行永不超时，直到任务组中所有任务执行完毕。
```Swift

/**
 * 使用enter与leave手动管理group与queue
 */
func performGroupUseEnterAndleave() {
    print("\n任务组手动管理：")
    let concurrentQueue: DispatchQueue = getConcurrentQueue("cn.zeluli")
    let group: DispatchGroup = DispatchGroup()
    
    //将group与queue进行手动关联和管理，并且自动执行
    for i in 1...3 {
        group.enter()                     //进入队列组
        
        concurrentQueue.async(execute: {
            currentThreadSleep(1)
            print("任务\(i)执行完毕\n")
            
            group.leave()                 //离开队列组
        })
    }
    group.wait()//阻塞当前线程，直到所有任务执行完毕
    print("任务组执行完毕")
    
    group.notify(queue: concurrentQueue) {
        print("手动管理的队列执行OK")
    }
}
```
　　

下方是上述函数执行后的输出结果，dispatch_group_wait()函数下方的print()函数在所有任务执行完毕之前是不会被调用的，因为dispatch_group_wait()会将当前线程进行阻塞。当然虽然是手动的将队列与任务组进行关联的，display_group_notify()函数还是好用的。运行结果如下所示。

![](http://images2015.cnblogs.com/blog/545446/201605/545446-20160513102949046-1178719390.png)
　　

 

### 六、信号量(semaphore)同步锁

有时候多个线程对一个数据进行操作的时候，为了数据的一致性，我们只允许一次只有一个线程来操作这个数据。为了保证一次只有一个线程来修改我们的资源数据，我们就得用到信号量同步锁了。也就是说一个存放资源房间的门后边又把锁，当有一个线程进到这个房间后就将这把锁锁上。当这个线程修改完该资源后，就将锁给打开，锁打开后其他的线程就可以持有资源了。如果你上了锁不打卡，而其他线程等待使用该资源时，就会产生死锁。所以当你不使用的时候，就不要持有资源呢。

上述这个过程，在GCD中我们可以使用信号量机制来完成。在GCD中有一个叫dispatch_semaphore_t的东西，这个就是我们的信号量。我们可以对信号量进行操作，如果信号量为0那么就是上锁的状态，其他线程想使用资源就得等待了。如果信号量不为零，那么就是开锁状态，开锁状态下资源就可以访问。下方代码就是信号量的具体使用代码。

下方第一个红框中就是通过dispatch_semaphore_create()来创建信号量，该函数需要一个参数，该参数所指定的就是信号量的值，我们为信号指定的值为1。第二个红框中是“上锁的过程”，通过dispatch_semaphore_wait()函数对信号量操作，该函数中的第一个参数是所操作的信号量，第二个参数是等待时间。dispatch_semaphore_wait()函数是对信号量减一，信号量为零的话就对当前线程所操作的资源加锁。其他线程等待当前线程操作资源的时间为DISPATCH_TIME_FOREVER，也就是说其他线程要一直等下去，等待当前线程操作资源完毕。当当前线程对资源操作完毕后调用dispatch_semaphore_signal()将信号量加1，将资源进行解锁，以便于其他等待的线程进行资源的访问。当解锁后，其他线程等待的时间结束，就可以进行资源的访问了。
```Swift

//信号量同步锁
func useSemaphoreLock() {
    
    let concurrentQueue = getConcurrentQueue("cn.zeluli")
    
    //创建信号量
    let semaphoreLock: DispatchSemaphore = DispatchSemaphore(value: 1)
    
    var testNumber = 0
    
    for index in 1...10 {
        concurrentQueue.async(execute: {
            semaphoreLock.wait()//上锁
            
            testNumber += 1
            currentThreadSleep(Double(1))
            print(getCurrentThread())
            print("第\(index)次执行: testNumber = \(testNumber)\n")
            
            semaphoreLock.signal()                      //开锁
            
        })
    }
    print("异步执行测试\n")
}
```
　　

 

### 七、队列的循环、挂起、恢复

在本篇博客的第七部分，我们要聊一下队列的循环执行以及队列的挂起与恢复。该部分比较简单，但是也是比较常用的。在重复执行队列中的任务时，我们通常使用dispatch_apply()函数，该函数循环执行队列中的任务，但是dispatch_apply()函数本身会阻塞当前线程。如果你使用dispatch_apply()函数来执行并行队列，虽然会开启多个线程来循环执行并行队列中的任务，但是仍然会阻塞当前线程。如果你使用dispatch_apply()函数来执行串行队列的话，那么就不会开辟新的线程，当然就会将当前线程进行阻塞。说到队列的挂起与恢复你可以使用dispatch_suspend()来挂起队列，使用dispatch_resum()来恢复队列。请看下方实例。

#### 1、dispatch_apply()函数

dispatch_apply()函数是用来循环来执行队列中的任务的，使用方式为：dispatch_apply(循环次数, 任务所在的队列) { 要循环执行的任务 }。使用该函数循环执行并行队列中的任务时，会开辟新的线程，不过有可能会在当前线程中执行一些任务。而使用dispatch_apply()执行串行队列中的任务时，会在当前线程中执行。无论是使用并行队列还是串行队列，dispatch_apply()都会阻塞当前线程。下方代码段就是dispatch_apply()的使用示例：
```Swift
/**
 循环执行_类似于dispatch_apply
 */
func useDispatchApply() {
    
    print("循环多次执行并行队列")
    DispatchQueue.concurrentPerform(iterations: 10) { (index) in
        currentThreadSleep(Double(index))
        print("第\(index)次执行，\n\(getCurrentThread())\n")
    }
}
```
　　

下方则是上述函数的运行结果。在结果中我们将每次执行任务所使用的线程进行了打印。

![](http://images2015.cnblogs.com/blog/545446/201605/545446-20160513133453202-605153159.png)
　　

 

#### 2.队列的挂起与唤醒

队列的挂起与唤醒相对较为简单，如果你想对一个队列中的任务的执行进行挂起，那么你就使用dispatch_suspend()函数即可。如果你要唤醒某个挂起的队列，那么你就可以使用dispatch_resum()函数。这两个函数所需的参数都是你要挂起或者唤醒的队列，鉴于知识点的简单性就不做过多的赘述了。下方是对异步执行的并行队列进行挂起，在当前线程休眠2秒后唤醒被挂起的线程。具体代码如下：
```Swift
//暂停和重启队列
func queueSuspendAndResume() {
    let concurrentQueue = getConcurrentQueue("cn.zeluli")
    
    concurrentQueue.suspend()   //将队列进行挂起
    concurrentQueue.async { 
        print("任务执行")
    }
    
    currentThreadSleep(2)
    concurrentQueue.resume()    //将挂起的队列进行唤醒
}

```
　　

 

### 八、任务栅栏dispatch_barrier_async()

顾名思义，任务栅栏就是将队列中的任务进行隔离的，是任务能分拨的进行异步执行。我想用下方的图来介绍一下barrier的作用。我们假设下方是并行队列，然后并行队列中有1.1、1.2、2.1、2.2四个任务，前两个任务与后两个任务本中间的栅栏给隔开了。如果没有中间的栅栏的话，四个任务会在异步的情况下同时执行。但是有栅栏的拦着的话，会先执行栅栏前面的任务。等前面的任务都执行完毕了，会执行栅栏自带的Block ，最后异步执行栅栏后方的任务。这么一说有点与前面的dispatch_group类似，当执行完一些列的任务后，我们想做一些事情的话，我们也可通过dispatch_barrier_async()来实现。
![](http://images2015.cnblogs.com/blog/545446/201605/545446-20160513135414905-908843240.png)
　　

下方代码段就是我们dispatch_barrier_async(), 具体的使用方式。上面的红色框中的代码是异步执行的第一批任务，中间是我们给任务队列添加的任务栅栏，dispatch_barrier_asyn()的一个参数就是栅栏所在的队列，而后边的尾随闭包就是在栅栏前面的所有任务都执行完毕后就会执行该尾随闭包中的内容。而最下方黄色框中的部分就是第二批次执行的任务，该批任务会在dispatch_barrier_asyn()栅栏的尾随闭包执行后会继续执行。
```Swift
/**
 使用给队列加栅栏
 */
func useBarrierAsync() {
    let concurrentQueue: DispatchQueue = getConcurrentQueue("cn.zeluli")
    for i in 0...3 {
        concurrentQueue.async {
            currentThreadSleep(Double(i))
            print("第一批：\(i)\(getCurrentThread())")
        }
    }
    
    
    concurrentQueue.async(flags: .barrier, execute: {
        print("\n第一批执行完毕后才会执行第二批\n\(getCurrentThread())\n")
    }) 
    
    
    for i in 0...3 {
        concurrentQueue.async {
            currentThreadSleep(Double(i))
            print("第二批：\(i)\(getCurrentThread())")
        }
    }
    
    print("异步执行测试\n")
}

```
　　

接下来我们来看一下上述代码的运行结果，点击我们第一部分截图的“使用任务隔离栅栏”按钮就会执行上述方法。下方就是上述代码片段的运行结果。从下面的输出结果中不难看出，dispatch_barrier_asyn之前的任务会先异步执行，也就是下方的第一批任务。第一批任务完成后，会在第一批任务中的最后完成任务的线程中来执行栅栏中的任务块。当栅栏中的任务执行完毕后，队列中的第二批任务中的第一个会进入执行栅栏任务的线程中来执行，其他的会开辟新的线程。如下所示。
![](http://images2015.cnblogs.com/blog/545446/201605/545446-20160514155735359-1027486544.png)
　　

我们可以用一个图来结合上述示例来解释栅栏的工作方式。下图画的就是栅栏工作的方式，需要注意的是队列中的第一批任务中的最后一个任务与栅栏中的任务已经第二批第一个任务是用一个线程来执行的。这就是为什么栅栏能进行任务隔离的根本了。从下方的图中我们不难发现，任务1.3、栅栏任务、任务2.1在线程5中是同步执行的。具体请看下图。
![](http://images2015.cnblogs.com/blog/545446/201605/545446-20160514161629609-1711874945.png)
　　

九、dispatch_source

dispatch_source在GCD中是一个比较灵活的东西，功能也是非常强大的。简单的说，dispatch_source的主要功能就是对某些类型事件的对象进行监听，当事件发生时将要处理的事件放到关联的队列中进行执行。dispatch源支持事件的取消，我们也可以对取消事件的进行处理。下方是dispatch源的不同类型，因为篇幅有限在此就不做过多的赘述了，关于这些类型的资料网上一抓一大把。今天就以DATA_ADD, DATA_OR, TIMER为例，看一下source的使用方式。
![](http://images2015.cnblogs.com/blog/545446/201605/545446-20160515161625289-1795649922.png)
　
#### 1.DATA_ADD 与DATA_OR

DISPATCH_SOURCE_TYPE_DATA_ADD和DISPATCH_SOURCE_TYPE_DATA_OR用法差不多一个是将数据源进行相加，一个是进行或操作。我们就以相加的为例，或操作的代码在博客中就不给出了，不过我们github上分享的代码会有完整的示例。下方函数是DISPATCH_SOURCE_TYPE_DATA_ADD类型的dispatch源的使用。

首先我们获取了一个全局队列queue，然后创建了一个dispatch源，命名为dispatchSource。在创建dispatch源时，我们为dispatch源指定了类型，并且为其关联的一个queue队列。关联这个队列的作用是用来处理dispatch源中的事件的。然后我们使用dispatch_source_set_event_handler()为我们的source指定处理事件。该事件会在一定的条件下回触发，至于触发的条件有dispatch源的类型锁定。因为此处我们dispatch源的类型是DISPATCH_SOURCE_TYPE_DATA_ADD，所以使用dispatch_source_merge_data()就可以触发上面我们指定的事件。因为dispatch源创建后是处于挂起的状态，所以我们需要使用dispatch_resume()对我们创建的事件源进行恢复。恢复后的dispatch源才可以监听条件从而触发事件。

下方代码段在for循环中调用dispatch_source_merge_data()方法。在执行过程中我们还可以调用dispatch_source_cancel()对dispatch源进行取消。当dispatch source被取消后，就会执行我们所设置取消dispatch_source要处理的事件。我们通过dispatch_source_set_candel_handel()来指定取消dispatch source要执行的事件。关于dispatch_source的取消，我们会在下面倒计时的时候给出。

我们此处创建的dispatch_source的类型是Data Add类型，也就是说当我们指定的源事件未处理完时，那么下一个Data就要进行等待。而等待的数据会通过dispatch_source_merge_data()方法进行合并。如果你创建的是DISPATCH_SOURCE_TYPE_DATA_ADD类型的dispatch_source，那么就会按照加法进行合并。如果你是创建的DISPATCH_SOURCE_TYPE_DATA_OR类型的dispatch_source, 那么就会通过或运算进行合并。合并在一起的数据会一同触发我们设定的事件。
```Swift
/**
 以加法运算的方式合并数据
 */
func useDispatchSourceAdd() {
    var sum = 0     //手动计数的sum, 来模拟记录merge的数据
    
    let queue = getGlobalQueue()
    
    //创建source
    let dispatchSource:DispatchSource = DispatchSource.makeUserDataAddSource(queue: queue) as! DispatchSource
    
    dispatchSource.setEventHandler {
//        print("source中所有的数相加的和等于\(dispatchSource.Date)")
        print("sum = \(sum)\n")
        sum = 0
       currentThreadSleep(0.3)
    }

    dispatchSource.resume()
    
    for i in 1...10 {
        sum += i
        print(i)
//        dispatchSource.mergeData(value: UInt(i))
        currentThreadSleep(0.1)
    }
}

```
　　

上述代码段就是对DATA_ADD类的的dispatch源进行的测试。我们定义了一个变量sum来模拟数据的合并，然后观察每次合并的数据与我们自定的sum中计算的数据是否相同。合并后每次执行一次事件我们都将sum进行归零，然后进行下一轮的合并。下方就是上述代码输出的结果。从下方的结果中我们可以看出，在上述的10次循环中执行了四次我们指定的source事件，而且每次执行事件所merge的Data与我们手动记录的sum一致。这就是DATA_ADD的工作方式，运行效果如下所示。关于Data_Or的运行方式在此就不做过多的赘述了。
![](http://images2015.cnblogs.com/blog/545446/201605/545446-20160515212302148-661453011.png)
　　

 

#### 2.定时器

在GCD的dispatch源中还有定时器类型，我们可以创建定时器类型的dispatch源，然后通过dispatch_source_set_event_handler()来设定源事件。然后通过dispatch_source_set_timer()函数来为定时器类型的dispatch_source指定时间间隔，该函数第一个参数就是dispatch source，第二个参数就是触发事件的时间间隔，第三个参数就是允许误差的时间。当我们设定的倒计时的次数到是，我们就调用dispatch_source_cancel()来进行dispatch_source的取消，当取消后就会执行dispatch_source_set_cancel_handel()方法中的尾随闭包。

下方示例是使用DISPATCH_SOURCE_TYPE_TIMER类型的dispatch source进行的10秒到计时，等我们设置的事件执行10次后我们就取消dispatch_source。对于下方的示例来说，当dispatch source通过dispatch_resume()函数进行唤醒后，会开始倒计时。会在倒计时10秒后结束计时。
```Swift
/**
 使用dispatch_source创建定时器
 */
func useDispatchSourceTimer() {
    let queue: DispatchQueue = getGlobalQueue()
    let source = DispatchSource.makeTimerSource(flags: DispatchSource.TimerFlags(rawValue: 0), queue: queue)
   
    //设置间隔时间，从当前时间开始，允许偏差0纳秒
    let timer = UInt64(1) * NSEC_PER_SEC
    source.scheduleRepeating(deadline: DispatchTime.init(uptimeNanoseconds: UInt64(timer)), interval: DispatchTimeInterval.seconds(Int(1)), leeway: DispatchTimeInterval.seconds(0))
    
    var timeout = 10    //倒计时时间
    
    //设置要处理的事件, 在我们上面创建的queue队列中进行执行
    source.setEventHandler {
        print(getCurrentThread())
        if(timeout <= 0) {
            source.cancel()
        } else {
            print("\(timeout)s")
            timeout -= 1
        }
    }
    
    //倒计时结束的事件
    source.setCancelHandler { 
        print("倒计时结束")
    }
    source.resume()
}

```
　　

 

下方就是上述倒计时代码所执行后的结果。从运行结果中我们不难看出，当倒计时开始时，会新开辟一些新的线程来顺序执行倒计时任务。尽管你使用的是并行队列，虽然每次开辟的线程可能会不同，但是都会顺序的执行倒计时任务.

![](http://images2015.cnblogs.com/blog/545446/201605/545446-20160515213807055-720183647.png)
　　
