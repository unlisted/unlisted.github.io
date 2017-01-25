---
layout: post
title:  "Mocking Subprocess"
date:   2017-01-22 
categories: jekyll update
---

I need to unit test a function wich calls subprocess.Popen() at most three times, each with different arguments.
{% highlight python %}
def go():
    call = subprocess.Popen(['git', 'fetch', '--tags', '--verbose'], 
            stderr=subprocess.STDOUT, stdout=subprocess.PIPE)

    out, err = call.communicate()
{% endhighlight %}
