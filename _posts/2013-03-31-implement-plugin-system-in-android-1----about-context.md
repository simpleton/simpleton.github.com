---
layout: post
title: "Implement Plugin System in Android (1) ———— About Context"
description: ""
category: android
tags: [android, plugin]
---
{% include JB/setup %}
最近一直在研究android的plugin系统，觉得最酷的一种就是使用dexClassloader这种方式，具体方法可以参见[AndroidDynamicLoader][1]. 

这段代码的精髓就在于替换了系统本来的classLoader，使得系统启动的activity被替换成我们实现的activity了。首先反射出来我们要的ClassLoader:

{% highlight java %}

	Context mBase = new Smith<Context>(this, "mBase").get();
	
	Object mPackageInfo = new Smith<Object>(mBase, "mPackageInfo").get();
	
	Smith<ClassLoader> sClassLoader = new Smith<ClassLoader>(mPackageInfo, "mClassLoader");
	
	ClassLoader mClassLoader = sClassLoader.get();
			
{% endhighlight %}

其中从context使用了一系列反射得到了，运行时的classLoader.本篇博文就主要讲一下context的实现，补充一下这一系列反射背后的故事:

## Context##
整体来讲Android的官方文档对Context的定于如下：

	Interface to global information about an application environment. This is an abstract class whose implementation is provided by the Android system.It allows access to application-specific resources and classes, as well as up-calls for application-level operations such as launching activityes, broadcasting and receiving intents, etc.
	
讲的十分抽象，不过可以有一个大题的理解，就是context是一个类似于windows编程中的handler得东西。可以访问资源，并且是一个抽象类。

### TODO:add a picture###

## ContextImpl##
其大体实现了context所定义的抽象方法，

{% highlight java %}

	class ContextImpl extends Context {
		/*package*/ LoadApk mPackageInfo;
		
		/*register many service*/
		.....
		@Override
		public AssetManager getAssets() {
			return mResources.getAssets();
		}
		
		@Override
		public Resources getResources() {
			return mResources;
		}
		
		@Override
		public void startActivity (Intent intent, Bundle options) {
			if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
				throw new AndroidRuntimeException("error");
			}
			mMainThread.getInstrumentation().execStartActivity(getOuterContext(), mMainThread.getApplicationThread(), null, (Activity)null, intent, -1, options);
		}
 	}
	
{% endhighlight %}

## ContextWrapper##
Context的包装类,他有一个域是mBase是我们十分感兴趣的，它决定了运行时的Context.

{% highlight java %}

	public class ContextWrapper extends Context {
		Context mBase; // point to ContextImpl object
		
		protected void attachBaseContext(Context base) {
			if (mBase != null) {
				throw new IllegalStateException("Base Context already set");
			}
			mBase = base;
		}
		
		@Override
		public void startActivity(Intent intent) {
			mBase.startActivity(intent);
		}
	}
	
{% endhighlight %}

上文均没有摘录service相关的code,其实对于Activity Service和Application都是Context的子类。这个是题外话了。 有了上面几段代码我们就不难理解那些反射了，通过Application的Context反射出mBase,而mBase实际上是ContextImpl的实例，通过他反射出mPackageInfo.而ContextImpl实际上大多都是mPackageInfo的功能，可见它才是真正的大哥。

## LoadedApk##
这个类大多是做一些资源、库和类的初始化和维持的(maintained)。其中有mClassLoader,就是我们想要的啦。这个反射出来的就是ClassLoader了，然后我们就可以替换或者添加我们自己的classLoader了。

由于classLoader的双亲委派模型，我们是无法替换父Loader的方法。更多ClassLoader知识还是[戳这里][2]

[1]: https://github.com/Rookery/AndroidDynamicLoader
[2]: http://docs.oracle.com/javase/6/docs/api/java/lang/ClassLoader.html
