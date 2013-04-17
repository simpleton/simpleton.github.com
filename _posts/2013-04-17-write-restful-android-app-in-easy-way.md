---
layout: post
title: "write restful android app in easy way"
description: ""
category: android
tags: [android]
---
{% include JB/setup %}
## JSON
[GSON][1]. 不多说了，google出品，是神器，传说就是速度慢一些，不过就实际使用的情况来讲没有感觉到任何障碍。

## Retrofit
[Restrofit][2]
This module is release by square. This is based on GSON and support Type safe interface for access restful API.

{% highlight java %}

	public class OrgClient {
	private static final String TAG = "OrgClient";
	private static final String API_URL = "https://id.b.qq.com";

	class Content {
		int code;
	    long qquin;
	    organization_node data;
	}

	class member_node {
		String d;
		int g;
		String j;
		String m;
		String n;
		String p;
		String e;
		String q;
		String u;
	}
	
	class organization_node {
		long i;
		ArrayList<member_node> m;
		String n;
		ArrayList<organization_node> o;
	}
	
	interface Organization {
		@GET("/hrtx/clientcall/mobileApi/getTree")
		Content contributors(
		);
	}
	
	static class TwitterDemoTask extends AsyncTask<String, Void, Void> {

		@Override
		protected Void doInBackground(String... params) {
			String uin = params[0];
			String skey = params[1];
			initClient(uin, skey);

			return null;
		}

	}
	
	public static void initClient(final String uin, final String skey) {
		Headers headers = new Headers() {			
			@Override
			public List<Header> get() {
				List<Header> headers = new ArrayList<Header>();
				StringBuilder Cookie = new StringBuilder().append("uin=o").append(uin).append(";")
						.append("skey=").append(skey).append(";").append("ptisp=ctc");
				headers.add(new Header("Cookie", Cookie.toString()));
				//headers.add(new Header("skey", skey));
				
				return headers;
			}
		};
		
		RestAdapter restAdapter = new RestAdapter.Builder()
			.setServer(API_URL)
			.setDebug(true)
			.setHeaders(headers)
			.build();

		// Create an instance API interface.
		Organization organization = restAdapter.create(Organization.class);

		// Fetch and print a list of the contributors to this library.
		Content content = organization.contributors();
		Log.d(TAG, "code" + content.code + "qquin" + content.qquin + "data" + content.data.i + content.data.n );
	}  
	
	public static void run(final String uin, final String skey) {
		new TwitterDemoTask().execute(uin, skey);
	}
}
{% endhighlight%}

[1]: https://github.com/square/retrofit
[2]: https://github.com/square/retrofit
