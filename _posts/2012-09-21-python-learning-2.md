---
layout: post
title: "python Learning 2"
description: "misc"
category: python
tags: [python, learning]
---
{% include JB/setup %}

# lambda#

{% highlight python %}
lambda vars: expr
{% endhighlight %}

{% highlight python %}
>>> def make_inc(n):
	    return lambda x: x + n
>>> f = make_inc(12) #assign n = 12, return the lambda expression
>>> f = f(0)
12
>>> f = f(1)
13
{% endhighlight %}

# metaclass#
implement delegationg by using Meta-class

{% highlight python %}
def dlgt(cls, mthd):
    def ret (self, *args, **kwargs):
        print 'ret__'
        mthd(self.__tgt__, *args, **kwargs)
    return ret
    
class mydelegate(type):
    def __init__(cls, name, bases, dict):
        print 'mydelegate __init__'
        super(mydelegate, cls).__init__(name, bases, dict)
        src = getattr(cls, '__tgtclass__', None)
        print 'dir',dir(src)
        for attr in dir(src):
            val = getattr(src, attr, None)
            print attr
            print 'attr',val
            if (callable(val)):
                setattr(cls, attr, dlgt(cls, val))
            
def clsname(self):
    return self.__class__.__name__

class A:
    def bar(self):
        print clsname(self), 'bar'
        
    def baz(self):
        print clsname(self), 'baz'
        
class B:
    __metaclass__ = mydelegate
    __tgtclass__ = A
    def __init__(self, tgt):
        self.__tgt__ = tgt

    def boo(self):
        print clsname(self), 'boo'
    
if __name__ == '__main__':
    b = B(A())
    b.bar()
    b.baz()
    b.boo()
    print b.__dict__

{% endhighlight %}
