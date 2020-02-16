什么是python全局解释器锁（GIL）？【译】
======

*本文由 eric \<zephor@qq.com>翻译自 Abhinav Ajitsaria[《What is the Python Global Interpreter Lock (GIL)?》](https://realpython.com/python-gil/#author)*

**内容概览**
- [GIL解决了python的什么问题？](#GIL解决了python的什么问题？)
- [为什么选GIL作为解决方案？](#为什么选GIL作为解决方案？)
- [对python多线程程序的影响](#对python多线程程序的影响)
- [为什么还没有将GIL移除？](#为什么还没有将GIL移除？)
- [为什么GIL没有在python3中被移除？](#为什么GIL没有在python3中被移除？)
- [怎么处理python的GIL问题](#怎么处理python的GIL问题)

Python全局解释器锁（GIL），简单来说，是一种互斥锁（或锁），它允许仅有一个线程拥有python解释器的控制权。

这意味着在任意时刻仅有一个线程能处在执行状态。GIL的影响对于那些运行单线程程序的开发者并不明显，但是它却能成为多线程的cpu密集型代码的瓶颈。

由于GIL哪怕在多核cpu的多线程架构中也是这幅德行，它便成了python的一个“臭名昭著”的特性。

**在本文中你将了解GIL是如何影响你python程序的性能和怎样才能减轻这些影响**


GIL解决了python的什么问题？
--------
python的内存管理使用了引用计数。这意味着python中创建的对象都有一个计数变量，用来追踪指向这个对象的引用次数。当这个计数归零，这个对象所占用的内存将被释放。

让我们看一段简短的代码示例，它将展示引用计数如何工作：

    >>> import sys
    >>> a = []
    >>> b = a
    >>> sys.getrefcount(a)
    3

在这个例子中，空列表对象`[]`的引用计数是3。该列表对象分别被`a`, `b` 和 传给 `sys.getrefcount()`的参数引用。

回到GIL：

这个引用计数在两个线程同时增加或减少的时候会出现问题，需要保证它不受这种竞争情况的影响。如果不能保证，那么将出现内存泄露甚至是错误地释放一个依然在使用的对象内存。这将导致你的python程序崩溃或者是其他莫名其妙的bug。

引用计数可以通过对所有跨线程的数据结构加锁来保证它们不会被修改成不一致的情形。

但是，对每个对象或对象组加锁意味着将会出现多重锁，而这可能导致另一个问题——死锁（死锁仅会在多于一个锁的时候发生）。而不断地获取和释放锁也将导致性能降低。

GIL是施加在解释器上的唯一的锁，任何python字节码必须在获取该锁的条件下执行。这样防止了死锁（因为只有一个锁），而且也不会对性能有太多影响。但是这样就让所有CPU密集型的python程序变成了单线程。

尽管GIL也被像Ruby等其他语音解释器所使用，但它并不是解决这个问题的唯一方法。有些编程语言使用了诸如垃圾收集而不是引用计数的方式避免依赖GIL来达到线程安全的内存管理。

从另一方面来说，这些编程语言一般不得不使用如JIT编译器之类的性能加速器来补偿缺失GIL带来的单线程性能优势。


为什么选GIL作为解决方案？
-----
所以，为什么在python中选用这么一个有着明显短板的方案？这是python的开发者的一步臭棋吗？

如[Larry Hastings所说](https://youtu.be/KVKufdTphKs?t=12m11s)，GIL的设计正是使python现今这么流行的原因之一。

python刚开始流行的时候操作系统还没有线程的概念。python被设计得尽可能简单易用来保证开发速度，于是越来越多的开发者开始使用它。  
许多python里需要的特性被直接将已有的C库以扩展的形式接入了进来。而这些C扩展依赖GIL提供的线程安全的内存管理，这是为了避免不一致的改动。

GIL易于实现并很轻松地被添加到了python之中。因为只需要管理一个锁，它为了单线程程序带来了性能提升。

非线程安全的C库由此变得更容易整合。这些C扩展也成了python如此轻易地被不同团体采纳的原因之一。

如你所见，GIL对早起的CPython开发者来说是解决这个难题的一个非常务实的解决方法。


对python多线程程序的影响
------
当你观察一个典型的python程序——或是任何与此有关的计算机程序——cpu密集型的和那些I/O密集型的表现都有不同。

cpu密集型程序是利用cpu的极限工作的。它包括数值计算程序比如矩阵乘法、搜索、图像处理等等。

I/O密集型程序是那些花时间等待来自用户、文件、数据库、网络等输入输出的。I/O密集型程序有时会花上非常长的等待时间，等待它们需要的资源。因为事实上资源提供者准备好资源可能需要自己的处理过程，比如，一个用户思考输入什么或者数据库进程自己查询需要提供的结果。

让我们看一个简单的倒计数的cpu密集型程序：

    # single_threaded.py
    import time
    from threading import Thread

    COUNT = 50000000

    def countdown(n):
        while n>0:
            n -= 1

    start = time.time()
    countdown(COUNT)
    end = time.time()

    print('Time taken in seconds -', end - start)

在我的4核系统上运行这段代码得到如下结果：
>$ python single_threaded.py  
>Time taken in seconds - 6.20024037361145

现在我将代码稍微修改一下，用两个线程并行地做同样的倒计数：

    # multi_threaded.py
    import time
    from threading import Thread

    COUNT = 50000000

    def countdown(n):
        while n>0:
            n -= 1

    t1 = Thread(target=countdown, args=(COUNT//2,))
    t2 = Thread(target=countdown, args=(COUNT//2,))

    start = time.time()
    t1.start()
    t2.start()
    t1.join()
    t2.join()
    end = time.time()

    print('Time taken in seconds -', end - start)

然后我再运行：
>$ python multi_threaded.py  
>Time taken in seconds - 6.924342632293701

你可以看到，两个版本几乎花了一样的时间。多线程版本中，GIL防止了cpu密集型程序的并发执行。

GIL在I/O密集型程序中等待I/O的时候会被共享出来，因此对I/O型多线程程序几乎没什么影响。

但是对于一个线程都是cpu密集型的程序来说，比如一个用多线程处理图像的程序，不仅会变成实际单线程执行而且相较于本来就用单线程写的程序来说还会增加执行时间。参看上例。

增加的这部分就是获取和释放GIL的时间。


为什么还没有将GIL移除？
------
python的开发者收到很多关于这点的吐槽，但是就python这么流行的语言来说无法引入诸如移除GIL这样的巨变而不造成任何兼容性的负面影响。

GIL显然是可以被移除的，而且开发者和研究者们在过去也已经多次成功地做到了这一点，但是所有这些成功的尝试无法兼容已有的C扩展，因为它们都严重依赖GIL。

当然，也有其他不影响兼容的方案，但是它们一些以损失单线程和多线程I/O密集型的性能为代价，一些实在是太难以实现。毕竟，你也不会希望你现有的python程序因为新版本地发布而运行地更慢，对吧？

python之父，“仁慈的独裁者” Guido van Rossum早已在2007年的一篇文章[《移除GIL并不容易》](https://www.artima.com/weblogs/viewpost.jsp?thread=214235)给了社区一个回答：
>我很乐意接受一系列的补丁放到python3里，只要不影响单线程程序（和多线程非I/O密集型程序）的性能

不过至今没有任何尝试满足过这一条件。


为什么GIL没有在python3中被移除？
------
python3当时是有机会从头开始，并在那个过程中，打乱了一些已有的C扩展的兼容性，然后为了让它们适应python3增加了一些更新和迁移的工作。这是python3的早期版本社区进展较慢的原因。

但是为什么GIL没有一并移除呢？

移除GIL将会使得python3相较于python2的单线程性能更差，由此你可以设想一下会出现什么结果。没有人可以驳斥GIL带来的单线程性能优势。所以python3依然保留了GIL。

不过python3却也为已有的GIL带来了很关键的提升——

我们讨论过GIL对于仅仅是纯cpu密集型或I/O密集型多线程程序，但是如果有的程序一些线程是I/O密集另一些是cpu密集型的呢？

据悉这种情况下，Python的GIL会“饿死”I/O线程，因为I/O线程没有丝毫机会从cpu型线程手里获取GIL。

这是由于python的一个内部机制导致的。python解释器在线程运行一个固定时间间隔后会强制线程释放GIL，此时如果没有其他线程获取GIL，那这个线程就又可以继续获取到GIL。

    >>> import sys
    >>> # The interval is set to 100 instructions:
    >>> sys.getcheckinterval()
    100

这个机制的问题在于大多数情况下cpu密集型线程都能在其他线程抢到之前之前重新获取到GIL。David Beazley研究并将结果可视化在了[这里](http://www.dabeaz.com/blog/2010/01/python-gil-visualized.html)。

这个问题在2009年由Antoine Pitrou在python3.2中修复了。他增加了一个[机制](https://mail.python.org/pipermail/python-dev/2009-October/093321.html)：用一个变量记录其他线程获取GIL失败的次数，在原线程重新获取GIL之前必须保证其他线程能有机会执行。


怎么处理python的GIL问题
------
如果GIL给你带来了问题，那么你可以尝试一下下面的方法：

**多进程vs多线程：** 最流行的方案便是用多进程替代多线程。每个python进程都有独立的python解释器和内存空间，所以GIL就不会影响。python有一个[multiprocessing](https://docs.python.org/2/library/multiprocessing.html)模块，它可以让我们像这样方便地创建进程：

    from multiprocessing import Pool
    import time

    COUNT = 50000000
    def countdown(n):
        while n>0:
            n -= 1

    if __name__ == '__main__':
        pool = Pool(processes=2)
        start = time.time()
        r1 = pool.apply_async(countdown, [COUNT//2])
        r2 = pool.apply_async(countdown, [COUNT//2])
        pool.close()
        pool.join()
        end = time.time()
        print('Time taken in seconds -', end - start)

在我的系统上执行结果如下：
>$ python multiprocess.py  
>Time taken in seconds - 4.060242414474487

相对于多线程版本有了一个明显的提升对吧？

相对于上面时间并没有减少一半，因为进程管理有它的固定开销。多进程比多线程开销要大，所以请注意，这可能成为扩展的瓶颈。

**不同的python解释器：** python有不少解释器实现。最流行的一些如：CPython、Jython、IronPython和PyPy，分别由C、Java、C#和Python实现。GIL仅存在于原始的python实现CPython中。如果你的程序和使用的库能够使用其他实现，你也可以一试。

**等着吧：** 现在很多python用户正享受着GIL带来的单线程性能好处。多线程程序员也不用沮丧，因为python社区中一些绝顶聪明的人正在想办法移除GIL。其中之一就是[Giletomy](https://github.com/larryhastings/gilectomy)。

python GIL经常被当做一个神秘而又高难度的话题。但是请记住作为一个python高手，你只会在编写C扩展或者是多线程的cpu密集型程序的时候经常被它影响。

总的来说，这篇文章应该已经已经完整地告诉了你什么是GIL和你应该在自己的项目中如何处理它。如果你想了解GIL的底层工作，我建议你看一下 David Beazley的[《理解python GIL》](https://youtu.be/Obt-vMVdM8s)分享。
