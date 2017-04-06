---
layout: post
title:  "Mocking Subprocess"
date:   2017-01-22 
categories: jekyll update
---

I need to unit test a function wich calls subprocess.Popen() a few times with different arguments. The result of the first call will determine whether sebsequent calls to Popen are made. How do you write a unit test to exercise all paths?

Say you have a function called do_proc().   
{% highlight python %}
def do_proc():
    """execute some external commands."""
    call = subprocess.Popen(['ls'], stderr=subprocess.STDOUT, stdout=subprocess.PIPE)
    out, err = call.communicate()
            
    matched = re.search(r'foobar.txt', out, flags=re.MULTILINE)
    if matched is not None:
        try:
            call = subprocess.Popen(['cat', 'foobar.txt'], stderr=subprocess.STDOUT, stdout=subprocess.PIPE)
            out, err = call.communicate()
        except OSError as err:
            return False
                                                                                            
    return True
{% endhighlight %}

We don't want subprocess to call out to the OS in our unit test, so we patch the test case with a [decorator](http://www.voidspace.org.uk/python/mock/patch.html). We then use the [mock](http://www.voidspace.org.uk/python/mock/mock.html) to check that Popen calls were made.
{% highlight python %}
class TestDoProc(unittest.TestCase):
    @mock.patch('subprocess.Popen')
    def test_one(self, popen_mock):
        pmock = mock.Mock()
        pmock.communicate.return_value = ("foobar.txt", "")

        popen_mock.return_value = pmock

        calls = [ 
                mock.call(['ls'],  stderr=subprocess.STDOUT, stdout=subprocess.PIPE),
                mock.call(['cat', 'foobar.txt'], stderr=subprocess.STDOUT, stdout=subprocess.PIPE)
                ]

        do_proc()
        popen_mock.assert_has_calls(calls)
{% endhighlight %}
The output of the test will look like this
{% highlight plaintext %}
morgan@toaster:~/work/testing$ ./subfun.py TestDoProc.test_one
F
======================================================================
FAIL: test_one (__main__.TestDoProc)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/usr/local/lib/python2.7/dist-packages/mock/mock.py", line 1305, in patched
    return func(*args, **keywargs)
  File "./subfun.py", line 41, in test_one
    popen_mock.assert_has_calls(calls)
  File "/usr/local/lib/python2.7/dist-packages/mock/mock.py", line 969, in assert_has_calls
    ), cause)
  File "/usr/local/lib/python2.7/dist-packages/six.py", line 718, in raise_from
    raise value
AssertionError: Calls not found.
Expected: [call(['ls'], stderr=-2, stdout=-1),
 call(['cat', 'foobar.txt'], stderr=-2, stdout=-1)]
Actual: [call(['ls'], stderr=-2, stdout=-1),
 call().communicate(),
 call(['cat', 'foobar.txt'], stderr=-2, stdout=-1),
 call().communicate()]

----------------------------------------------------------------------
Ran 1 test in 0.001s

FAILED (failures=1)
{% endhighlight %}
We can see the test case failed bacause the second call to Popen never occured. To fix this we can assign a [side_effect](http://www.voidspace.org.uk/python/mock/mock.html?highlight=side_effect#mock.Mock.side_effect) function to examine the arguments passed to Popen for each call and react appropriately.
{% highlight python %}
    @mock.patch('subprocess.Popen')
    def test_two(self, popen_mock):
        def proc_side_effect(*args, **kargs):
            proc_cmd = args[0]

            pmock = mock.Mock()
            if proc_cmd[0] == 'ls':
                pmock.communicate.return_value = ("foobar.txt", "")
            elif proc_cmd[0] == 'cat':
                pmock.communicate.return_value = ("go away", "")
            else:
                raise OSError

            return pmock

        popen_mock.configure_mock(side_effect=proc_side_effect)
        calls = [ 
                mock.call(['ls'],  stderr=subprocess.STDOUT, stdout=subprocess.PIPE),
                mock.call(['cat', 'foobar.txt'], stderr=subprocess.STDOUT, stdout=subprocess.PIPE)
                ]

        do_proc()
        popen_mock.assert_has_calls(calls)
{% endhighlight %}
Now the test passes.
{% highlight python %}
morgan@toaster:~/work/testing$ ./subfun.py
.
----------------------------------------------------------------------
Ran 1 test in 0.001s
{% endhighlight %}
