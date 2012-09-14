---
layout: post
title: "Learning Python 1"
description: "metaclass"
category: python
tags: [python, learning]
---
{% include JB/setup %}

  Recently, iam working on the two APP's server side. Life is so brief, so we choose python for implement the server. I have learned python for a short while,i only know the basic work flow such as ,for while if and so on.I just write python program in C sytle...shame on me. So i want to learning how to write python in python sytle.I will record every thing i have learned.

# First thing, metaclass #
  That is cost me whole day to understand metaprogram. Summarizing MetaProgram is just program make the program.That sounds amazing, just like science film. But actually python has whole mechanism for supporting MetaProgram.

The beginning of the story is that i want to refactor my model module in starfish project to Singleton. But i found some many way to implement Singleton in python on stackoverflow.com. I feel so confused.


[singleton_in_python][1]
[1]: http://stackoverflow.com/questions/31875/is-there-a-simple-elegant-way-to-define-singletons-in-python "singleton"

  I found one way use metaclass to archieve the Singleton, i feel it's interesting.So i spend whole noon search the knowledge about that on internet.

[metaclass][2]
[2]: http://stackoverflow.com/questions/100003/what-is-a-metaclass-in-python "metaclass"

[metaclass in chinese][3]
[3]: http://jianpx.iteye.com/blog/908121 "metaclass in chinese"

I didnot understand what they are talking about, so i will write the what i learned in furture.

# '__call__' #
override this method, u can use the object of the Class as a function, as same as override the () operator.

	class Factorial:
		def __init__( self ):
			self.cache = {}
		def __call__( self, n ):
			if n not in self.cache:
				self.cache[n] = n*self.__call__( n-1 )
			return self.cache[n]
	
	fact = Factorial()
	
now,u have a `fact` object which is callable,just like a function.



