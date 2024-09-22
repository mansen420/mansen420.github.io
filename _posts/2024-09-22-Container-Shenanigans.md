---
layout: post
title: "Container Shenanigans"
---
# List and its Derivatives

## The Wheel of Time

'One day I woke up and decided that I needed a container class. I used std::vector. The end.'

Is what a sane programmer would say.
But what would I say?

### Context

I am working on a linear algebra library. I need flexible yet efficient data structure to handle typical graphics tasks.
I want a container that will hold small values like 4 byte colors, or 4 float vectors.
'Simple task!' I say, and start typing away an abstraction over static arrays
{% highlight C++ %}
template<typename T, size_t dim>
class list
{
    T data[dim];
    ...
};
{% endhighlight %}
Everything is going well, vectors and matrices contain lists for data storage, and list allows fancy processing with a functional interface.

### SEGFAULT

...What? How come? I am literally using static arrays!
One word : Stack Overflow
Turns out, I didn't just need the list for vectors and matrices, I also needed it to store screen pixel data;
1920x1080 pixels * 4 bytes = too much for the stack!

'Simple task! I will use std::vector since it allocates its data on the heap!'

Is what a sane programmer would say.
Me? I have a nagging sensation. Why would I need to store the 4 bytes of a vector on the heap? Who knows how much overhead std::vector will cost anyway?
I give in, and decide that allocating on the heap is worth the trouble, only *I* will do it, *myelf*.
And so I start typing the first layer of hell
{% highlight C++ %}
template<typename T, size_t dim>
class list
{
    T* data = new T[dim];
    ...
};
{% endhighlight %}
Now screen data and arrays and matrices can all use the same storage scheme!
Finally, I have created the perfect container.

### The System Wants to Speak to You

'Hello? Yes, speaking'
'I can't hear you well-'
'Excuse me? What do you mean?'
'...Now?'

It's bad news. The system thinks it can change window sizes at runtime, after the program begins execution.
List *must* know its size at compile-time, before the execution.

'Well, clearly we have a problem here.'
'No shit, sherlock'
'I'm just trying to help...'
'...'

I look through the archives, surely someone must've had this problem before me?
Technically, this is a solved problem. You just slap a size_t in there to keep track of the runtime size. But why should I do that when I can keep everything at compile time?

Yes, this is it. I found the solution.

Yet another layer.
{% highlight C++ %}
constexpr uint DYNAMIC = 0;
struct empty{};
template<typename T, size_t dim = DYNAMIC>
class list
{
    T* data = new T[dim];
    [[no_unique_address]] std::conditional_t< cond: dim == DYNAMIC, 
    iftrue: size_t, iffalse: empty> dynamicSize;
public:
    ...
    list (size_t dynamicSize) requires(dim == DYNAMIC)
    {...}
    ...
};
{% endhighlight %}

'What is this?' You might say.
No unique address? Is that a double array?
No. This is a compiler attribute.
See, the C++ spec states that each field in a class must be uniquely addressable.
I am saying if the dimension is not dynamic, then make my dynamicSize an empty object, then please try to optimize the memory layout of my class by *maybe* overlapping this empty member with something else.
At least now, worst case scenario is I have an extra byte on my class.

Okay, is that all I had to do? Phew..

'So I just want to store 4 bytes per pixel. RGBA. That's all I want to do. For every pixel on the screen. How would I do it?'
Simple, really.
{% highlight C++ %}
typedef uint8_t byte;
typedef list<byte, 4> RGBA;
list<RGBA, DYNAMIC> screenPixels(someSize);
{% endhighlight %}

See it's that easy-

### The Nesting Problem
'Well. We have a problem. It wasn't that easy.'
'Hm?'
'Don't you see it? The lists...they are not contiguous.'
'I don't see it. You have an array of lists, it's gonna be contiguous on the heap.'
'Well, what is a list?'
'Is that a philosophical question?'
'No. A list is a pointer, maybe with a size attached.'
'Yes, dumbass.'
'This is what we're storing in contigous memory'
'Hm? So you're saying-'
'Yes'
'Well, technically, it works. If you disgregard the performance impact-'
'Get out of my office.'

I can't believe how I missed this. The actual data --the meat-- is being stored on separate parts of the heap for every list.
Every list makes its own allocation.
I have failed.

The archives don't cut it anymore. I must head to the oracle.

*```
    Oh great oracle. What must I do?
    How can I keep my lists...
    How can I keep them uhh
    Inlined?
    Inlined...
```*

I see it now. Everything is clear. I need to embed my data in the class structure itself.
There is only one way to do it.

This will be the final one. I will end it here.

{% highlight C++ %}
    std::conditional_t<cond: inlined, iftrue: T[dim], iffalse: T*> data;
{% endhighlight %}

This is it. This is the end, right?

***

As usual, your [Key to The Codex](https://github.com/mansen420/FCG/blob/master/src/list.h)
