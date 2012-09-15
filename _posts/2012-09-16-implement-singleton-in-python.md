---
layout: post
title: "implement Singleton in python"
description: "use metaclass"
category: python
tags: [python]
---
{% include JB/setup %}
# Singleton#
I've found many ways in stackoverflow for implementing Singletons. To my surperise, there isn't an elegant way to implement Singleton in python. [Singletons in python][1], the best answer give a work around. He said, there is no way of creating private classes or private constructors in python, so we cann't protect against multiple instantiation. The only way that we can do is using convention APIs. We could put methods in our module and consider the module as Singleton.


This method is so ugly....

# use metaclass implement Singleton#
I thought metaclass is another way to archive our Singleton goals.(PS: someone said this isn't a Singleton). 

just show the code:
	
	class Singleton(type):
		def __init__(cls, name, bases, dict):
			print "Singleton __init__"
			super(Singleton, cls).__init__(name, bases, dict)
			cls.instance = None
    
		def __call__(cls, *args, **kw):
			print "Singleton __call__"
			if cls.instance is None:
				cls.instance = super(Singleton, cls).__call__(*args, **kw)
			return cls.instance

	class Myclass(object):
		__metaclass__ = Singleton
		def __init__(self):
			print "Myclass __init__"
			
		def __new__(cls):
			print "Myclass __new__"
			return super(Myclass, cls).__new__(cls)
			
		def __call__(self, *args, **kw):
			print "Myclass __call__"
    
    if __name__ == '__main__':
		a = Myclass()
		b = Myclass()
		print id(a)
		print id(b)


the result of above codes is: 

	Singleton __init__
	Singleton __call__
	Myclass __new__
	Myclass __init__
	Singleton __call__
	140203664067280
	140203664067280

metaprogramming is really magical.

[1]: http://stackoverflow.com/questions/31875/is-there-a-simple-elegant-way-to-define-singletons-in-python# 
