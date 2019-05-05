---
layout: post
title: "Use Promise Pattern in Android"
date: 04-22-2014 00:00:00
tags:
 - android

---

I've read deep in NodeJS today. Although I haven't written nodeJS for a long time, but I learn a lot for the book.
Recommanded for you.

![deep in NodeJS](http://img5.douban.com/lpic/s27134708.jpg)

# Promise/Defered #

Promise pattern is wilderly used in JQuery framework, it solved Pyramid of Doom issue. And it quit useful in Android development.
In our company project, we always wrapped a asynchronously framework for network request. some complex requirement will be implemented by request server several times, and the code will be ugly.

Promise pattern will help us to solve this proplem.I found a [java framework to implement promise pattern](https://github.com/jdeferred/jdeferred).


```java
	Deferred deferred = new DeferredObject();
	Promise promise = deferred.promise();
	promise.done(new DoneCallback() {
		public void onDone(Object result) {
		...
		}
	}).fail(new FailCallback() {
	public void onFail(Object rejection) {
		...
		}
	}).progress(new ProgressCallback() {
	public void onProgress(Object progress) {
		...
		}
	}).always(new AlwaysCallback() {
	public void onAlways(State state, Object result, Object rejection) {
		...
		}
	});
```


```java
	DeferredManager dm = new DefaultDeferredManager();
	Promise p1, p2, p3;
	// initialize p1, p2, p3
	dm.when(p1, p2, p3)
	  .done(…)
      .fail(…)
```

It makes you source code more readable.
