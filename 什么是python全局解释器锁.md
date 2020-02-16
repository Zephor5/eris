什么是python全局解释器锁（GIL）？
=
    内容概览 
    · GIL解决了python的什么问题？
    · 为什么选GIL作为解决方案？
    · 对python多线程程序的影响
    · 为什么还没有将GIL移除？
    · 为什么它没有在python3中被移除？
    · 怎么面对python的GIL

Python全局解释器锁（GIL），简单来说，是一种互斥锁（或锁），它允许仅有一个线程拥有python解释器的控制权。  

这意味着在任意时刻仅有一个
