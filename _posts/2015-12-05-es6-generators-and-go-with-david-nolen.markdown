---
layout: post
title:  "ES6 generators and Go style in JavaScript with David Nolen"
date:   2015-12-05 23:14:29 -0800
categories: nolen es6 go generators
---
Today I have stumbled upon David Nolen's [post][nolen-post] titled `ES6 Generators Deliver Go Style Concurrency`. As title suggests it is post about trying to accomplish "Go style parallel programming model" in JavaScript, by using one of the most powerful ES6 features called Generators. However, I found this implementation quite "poor" because it forces "Go channels" (implemented as arrays) to have only one element inside. I was wondering for a while how this solution could be improved and I found quite simple solution I want to present here.

For those of you who don't know `Go`: it is a language whose one of the core features is possibility to spawn "Go routines". They are lightweight threads. Communication between them can be done using Channels. Channels are basicly queues. When one thread wants to read from a channel and channel is empty then it has to wait. Putting and getting data from channels are atomic operations. By using this style of programming we can write synchronous code in asynchronous environment. What is quite neat. 

Basic problem with David's code is here:  

{% highlight javascript %}
function put(chan, val) {
  return function() {
    if(chan.length == 0) {
      chan.unshift(val);
      return ["continue", null];
    } else {
      return ["park", null];
    }
  };
}
{% endhighlight %}

It blocks operation of adding new value to channel until it is empty - *"park"* stands for blocking. This way our channel can't contain more than one value - what is not really useful. 

Hence the better version is:

{% highlight javascript %}
function put(chan, val) {
  return function() {
    chan.unshift(val);
    return ["continue", null];
  };
}
{% endhighlight %}

But to make it working we have to change code in other places too. Primaraly, we have to change *go_* function and make it a generator itself:

{% highlight javascript %}
function* go_(machine, step) {
  var go_Gen = yield;

  while(!step.done) {
    var arr   = step.value(),
        state = arr[0],
        value = arr[1];

    function next() {
      go_Gen.next();
    }

    switch (state) {
      case "park":
        yield setImmediate(next);
        break;
      case "continue":
        yield setImmediate(next);
        step = machine.next(value);
        break;
    }
  }
}
{% endhighlight %}

By yielding asynchronous callback *next* by *setImmediate* we are pausing and resuming "local event loop" of our "Go routine". This way we let our "Go routines" work in "parallel" - of course JavaScript doesn't let true concurrency without Web Workers.

Last piece of code is:

{% highlight javascript %}
function go(machine) {
  var gen = machine();
  var go_Gen = go_(gen, gen.next());
  go_Gen.next();
  go_Gen.next(go_Gen);
}
{% endhighlight %}

Now we can change "putting thread" code and do something what would have not really worked without changes:

{% highlight javascript %}
setTimeout(
  go.bind(this, function* () {
    while(true) {
      var val = yield take(c);
      if(val == null) {
        break;
      } else {
        console.log("process two took", val);
      }
    }
  })
, 1000);
{% endhighlight %}

This only illustrates example of adding and reading more than one element to/from channel in non-blocking manner. But we can create much more advanced scenario.

[nolen-post]: http://swannodette.github.io/2013/08/24/es6-generators-and-csp/
