---
layout: post
title:  "Problem of Eric Elliott"
date:   2015-12-28 01:14:29
categories: elliott inheritance
---
Eric Elliott is well known JavaScript programmer and blogger. He is advocate of 

> favor object composition over class inheritance.

mantra in JavaScript - famous and correct sentance from "Gang of Four" book. He is also author of such words:

> If constructor behavior is the frying pan, classical inheritance isn’t the fire; it’s the fire from Dante’s seventh circle of hell.

He proposed a solution: 

> I settled on an interesting idea: a factory function that helps you produce factory functions that you can inherit from and compose together. I called the composable factories “stamps,” and the library, “Stampit.” (...) or simply cloning objects by copying properties from a source object to a new object (e.g. `Object.assign()`, `$.extend()`, `_.extend()`, etc…). 

Many JS developers refer to his ideas to implement such approach in their own programming journeys. 

However, there is one small problem: 

**Eric Elliott completely misunderstands what object composition is and he is using misrepresented and twisted terms in his posts.**

How it could be? I had not realized this too until I stumbled upon [this][reddit-post] post on Reddit (author's [blog][author-blog]). Author shows that what Elliott really means by *classical inheritance* is actually *not multiple inheritance*, and what really Elliott thinks he solves by using StampIt or `_.extend()` is actually form of *multiple inheritance* not a form of *object composition*.

**So Elliott thinks he is using object composition when he really is using multiple inheritance.**

Definition of *object composition* from "Gang of Four" (to which Elliot is referring): 

> Object composition is defined dynamically at run-time through objects acquiring references to other objects.

For example (object contains has-a relationships):

{% highlight javascript %}
{
  author: createAuthor(),
  posts: createPosts()
}
{% endhighlight %}

Definition of *inheritance* (from Wikipedia, but is industry standard): 

> Inheritance is when an object or class is based on another object (prototypal inheritance) or class (class-based inheritance), using the same implementation (inheriting from an object or class) specifying implementation to maintain the same behavior (realizing an interface; inheriting behavior).

Hence using *_.extend()* or *Stampit* is a form of *inheritance* (multiple), because we are inheriting behaviors and components interfaces become your interface. It does not matter that they do it by shallow copying.

Moreover, *multiple inheritance* suffers from some problem, called "diamond problem" (from Wikipedia):

> "diamond problem" is an ambiguity that arises when two classes B and C inherit from A, and class D inherits from both B and C. If there is a method in A that B and C have overridden, and D does not override it, then which version of the method does D inherit: that of B, or that of C?

*_.extend()* or *Stampit* "solve" this problem by just overwriting methods of the same name by last object in the call. 

Author of Reddit's post shows how *Stampit* gives completely unexpected results due to this problem:

{% highlight javascript %}
var storable = stampit().methods({
    read: function () {
        console.log('storable::read');
    },

    write: function () {
        console.log('storable::write');
    }
});

var transmitter = stampit().compose(storable).methods({
    write: function () {
        console.log('transmitter::write');
    }
});

var receiver = stampit().compose(storable).methods({
    read: function () {
        console.log('receiver::read');
    }
})

var radio = stampit().compose(transmitter, receiver);

radio().read(); // receiver::read -- ok
radio().write(); // storable::write -- wait, what? tramsmitter::write lost due to diamond problem
{% endhighlight %}

Thanks [JeffM][author-blog] for pointing this out!

[reddit-post]: https://www.reddit.com/r/javascript/comments/3oy9c3/composition_vs_eric_elliott/

[author-blog]: https://medium.com/@jeffm712/

