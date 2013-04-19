---
layout: post
title: "Node learning (1)"
description: ""
category: server
tags: [NodeJs]
---
{% include JB/setup %}

## CoffeeScript##

invoke setInterval method:


## socekt io
<Node Application with MongoDB and Backbone> example make me such confused. The example emit a same event in client and server code. That is differet from Socekt.io offical example. But actully, the example works very well. I don't understand the reason...here is the example code:

** Client:**
{% highlight javascript %}

	extends layout

	block scripts
		script(type='text/javascript', src='/socket.io/socket.io.js')
		script(type='text/javascript')
		var socket = io.connect('http://localhost:3000');
		socket.on('chat', function(data) {

	document.getElementById('chat').innerHTML =
        '<p><b>' + data.title + '</b>: ' + data.contents + '</p>';
    });
    var submitChat = function(form) {

      socket.emit('chat1', {text: form.chat.value});
      return false;
    };

	block content
	div#chat
  
	  form(onsubmit='return submitChat(this);')
    input#chat(name='chat', type='text')
    input(type='submit', value='Send Chat')

{% endhighlight %}


** Server**

{% highlight javascript %}

	io.sockets.on('connection',
        (socket) ->
                console.log 'socket on Connection'
                sendChat = (title, text) ->
                        socket.emit 'chat'
                        , {
                        title: title,
                        contents: text
                        }
                setInterval((socket) ->
                                randomIndex = Math.floor Math.random()*catchPhrases.length
                                sendChat 'Stooge', catchPhrases[randomIndex]
                        , 50000)

                sendChat('Welcome to chat room', 'Here is online status')
                socket.on 'chat1', (data) ->
                        sendChat 'You', data.text
                        console.log 'chat test'
        )
		
{% endhighlight %}

You can find the offical code in [socket.io][1]
I think emit different event make more clear for reading code.

## CoffeeScript##
use setInterval function:

** in Javascript format**

{% highlight javascript %}

	setInterval(function() {
		var randomIndex = Math.floor(Math.random()*catchPhrases.length)
		sendChat('Stooge', catchPhrases[randomIndex]);
	}, 5000);

{% endhighlight %}

**in CoffeeScript format:**

{% highlight javascript %}
	
	 setInterval((socket) ->
		 randomIndex = Math.floor Math.random()*catchPhrases.length
		 sendChat 'Stooge', catchPhrases[randomIndex]
	 , 50000)

{% endhighlight %}

[1]: http://socket.io/
