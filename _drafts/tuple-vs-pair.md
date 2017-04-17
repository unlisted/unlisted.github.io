---
layout: post
title:  "std::tuple vs std::pair"
date: 2017-04-09 
categories: c++
---

c++ is complex language. Compliant compilers support code going back 30 - 40 years. Complexity grows as new features are added to the standard. 

What's the difference between std::tuple vs std::pair?

Simple answer is pair makes more sense syntactically when working with pairs of values. 
{% highlight c++ %}
bool func()
{
    std::make_pair(1, 2);
}
{% endhighlight %}
