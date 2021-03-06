---
layout: post
title:  "ES6 generators and Go style in JavaScript with David Nolen"
date:   2015-12-05 23:14:29 -0800
categories: nolen es6 go generators
---
Today I have stumbled upon David Nolen's [post][nolen-post] titled `ES6 Generators Deliver Go Style Concurrency`. As title suggests it is post about trying to accomplish "Go style parallel programming model" in JavaScript, by using one of the most powerful ES6 features called Generators. However, I found this implementation quite "reduced" because it forces "Go channels" (implemented as arrays) to have only one element inside. Moreover, putting message to channel blocks execution of "Go routine" until someone will take this message from channel - as Go works. However, I would like to have non-blocking "putting mechanism". I was wondering for a while how this solution could be changed and I found quite simple solution I want to present here.   

For those of you who don't know `Go`: it is a language whose one of the core features is possibility to spawn "Go routines". They are lightweight threads, which can share memory. Communication between them can be done using Channels. Channels are basicly queues. When one thread wants to take message from a channel and channel is empty then it has to wait. Putting and getting data from channels are atomic operations. By using this style of programming we can write synchronous code in asynchronous environment. What is quite neat. 

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

It blocks operation of adding new value to channel until it is empty - *"park"* stands for blocking. This way our channel can't contain more than one value. 

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
  var goIter_ = yield;

  while(!step.done) {
    var arr   = step.value(),
        state = arr[0],
        value = arr[1];

    var goIterNext = goIter_.next.bind(goIter_);

    switch (state) {
      case "park":
        yield setImmediate(goIterNext);
        break;
      case "continue":
        yield setImmediate(goIterNext);
        step = machine.next(value);
        break;
    }
  }
}
{% endhighlight %}

By yielding asynchronous callback *goIterNext* by *setImmediate* we are pausing and resuming "local event loop" of our "Go routine". This way we let our "Go routines" work concurrently - of course JavaScript doesn't let true parallelism without Web Workers.

Last piece of code is:

{% highlight javascript %}
function go(machine) {
  var gen = machine();
  var goIter_ = go_(gen, gen.next());
  goIter_.next();
  goIter_.next(goIter_);
}
{% endhighlight %}

Now we can do something what would have not really worked without changes:

{% highlight javascript %}
function timeout(t) {
  var canContinue = false
  setTimeout(() => canContinue = true, t);

  return function() {
    if(canContinue === true) {
      return ["continue", null];
    } else {
      return ["park", null];
    }
  };
}

go(function* () {
  for(var i = 0; i < 10; i++) {
    yield put(c, i);
    console.log("process one put", i);

    yield timeout(Math.random() * 1000);
  }
  yield put(c, null);
});


go(function* () {
  while(true) {
    var val = yield take(c);
    if(val == null) {
      break;
    } else {
      console.log("process two took", val);
    }

    yield timeout(Math.random() * 1000);
  }
});
{% endhighlight %}

This illustrates example of putting and taking more than one element to/from channel in non-blocking manner, operations which are being executed concurrently. We are using also synchronous timeout, what is quite nice. But we can create much more advanced scenario.

[nolen-post]: http://swannodette.github.io/2013/08/24/es6-generators-and-csp/
