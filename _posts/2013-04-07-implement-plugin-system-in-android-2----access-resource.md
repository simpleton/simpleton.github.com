---
layout: post
title: "Implement Plugin System in Android (2) Access Resource"
description: ""
category: android
tags: [android, plugin]
---
{% include JB/setup %}
关于资源的替换首先就要提到AssetManager，基本上所有的资源都是通过这个单例进行访问的。[AssetManager][1].(PS: 可以参照代码，所有基于R.res.xxx的访问最终也会调用AssetManager的方法，只不过是不可见的我而已。)所以如果我们在构建插件的时候，在不构建插件的Context得前提下（这里由于插件没有安装，想要构建出其context十分困难）。我们就需要曲线救国了。

在ContextThemeWrapper中我们可以看到如下代码：

{% highlight java %}
	
	@Override
    public Resources getResources() {
        if (mResources != null) {
            return mResources;
        }
        if (mOverrideConfiguration == null) {
            mResources = super.getResources();
    n        return mResources;
        } else {
            Context resc = createConfigurationContext(mOverrideConfiguration);
            mResources = resc.getResources();
            return mResources;
        }
    }
	
{% endhighlight %}

以及在ContextWrapper中的：

{% highlight java %}

	/**
     * @return the base context as set by the constructor or setBaseContext
     */
    public Context getBaseContext() {
        return mBase;
    }

    @Override
    public AssetManager getAssets() {
        return mBase.getAssets();
    }

    @Override
    public Resources getResources()
    {
        return mBase.getResources();
    }
	
{% endhighlight %}

自然而然，我们可以Override这些函数，让系统使用我们构造出的resource。

例如我们要在framework使用plugin的资源，我们就可以使用如下做法：

{% highlight java %}

	Class<?> testclass ;
    Object drinstance;
        try {
            testclass = mCurrentLoader.loadClass("com.example.messagelist.plugin.DailyReportActivity");
            for (Method mt : testclass.getDeclaredMethods()) {
                Debug.i(TAG, mt.getName());
            }
            String resPath = mContext.getDir("dex", Context.MODE_PRIVATE)
                    .getAbsolutePath() + "/" + apkName;
            try {
                AssetManager am = (AssetManager) AssetManager.class
                        .newInstance();
                am.getClass().getMethod("addAssetPath", String.class)
                        .invoke(am, resPath);

                Resources superRes = mContext.getResources();
                
                Resources res = new Resources(am, superRes.getDisplayMetrics(),
                        superRes.getConfiguration());
                
                Theme thm = res.newTheme();
                
                drinstance = testclass.newInstance();
                new Smith<Object>(drinstance, "res").set(res);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
            Debug.i(TAG,(String)testclass.getDeclaredMethod("getHello",new Class[]{}).invoke(drinstance, new Object[]{}));
            //testclass.getDeclaredMethod(name, parameterTypes);
        } catch (Exception e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }

{% endhighlight %}

另外，我们plugin的activity就需要做出一些小小的改动，方便我们替换这些资源（PS:当然，不替换也是可以的，不过就需要反射系统mBase context里面的响应的域，并替换之）。这里我们让所有activity都继承我们自己写的BaseActivity，这样做会比较简单，可控：

{% highlight java %}

	private AssetManager asm;
	private Resources res;
	private Theme thm;
	private static final String TAG = "BasePluginActivity";
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		String apkName = getIntent().getStringExtra("apk_name");

		if (apkName != null && apkName.length() != 0) {
			String resPath = getDir("dex", Context.MODE_PRIVATE)
					.getAbsolutePath() + "/" + apkName;
			try {
				AssetManager am = (AssetManager) AssetManager.class
						.newInstance();
				am.getClass().getMethod("addAssetPath", String.class)
						.invoke(am, resPath);

				Resources superRes = super.getResources();

				res = new Resources(am, superRes.getDisplayMetrics(),
						superRes.getConfiguration());

				asm = am;
				thm = res.newTheme();
				thm.setTo(super.getTheme());
			} catch (Exception e) {
				throw new RuntimeException(e);
			}
		}

		super.onCreate(savedInstanceState);
	}

	@Override
	public AssetManager getAssets() {
	    if (asm == null) {
	        Log.d(TAG, "asm is null");
	    }
		return asm == null ? super.getAssets() : asm;
	}

	@Override
	public Resources getResources() {
	    if (res == null)
	        Log.d(TAG, "res is null");
		return res == null ? super.getResources() : res;
	}

	@Override
	public Theme getTheme() {
	    if (thm == null) {
	        Log.d(TAG, "thm is null");
	    }
		return thm == null ? super.getTheme() : thm;
	}
	
{% endhighlight %}

这样就可以做到，让plugin拥有自有的资源，并且可以让framework app进行调用。上述代码都是一些简单的demo，如果有什么更好的建议，feel free to contact me.

[1]: http://developer.android.com/reference/android/content/res/AssetManager.html
