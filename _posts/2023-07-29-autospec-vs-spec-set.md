---
layout: post
title:  "autospec vs spec_set"
date: 2023-07-29
categories: python, unittest, mock
---

What is the difference with respect to patching a function? This whole post only applius to mocking a function, not a class or instance.

> The spec and spec_set keyword arguments are passed to the MagicMock if patch is creating one for you.
[patch docs](https://docs.python.org/3/library/unittest.mock.html#patch)

> spec_set: A stricter variant of spec. If used, attempting to set or get an attribute on the mock that isn’t on the object passed as spec_set will raise an AttributeError.
[Mock docs](https://docs.python.org/3/library/unittest.mock.html#unittest.mock.Mock)

> A more powerful form of spec is autospec. If you set autospec=True then the mock will be created with a spec from the object being replaced. All attributes of the mock will also have the spec of the corresponding attribute of the object being replaced. Methods and functions being mocked will have their arguments checked and will raise a TypeError if they are called with the wrong signature. For mocks replacing a class, their return value (the ‘instance’) will have the same spec as the class. See the create_autospec() function and Autospeccing.
[patch docs](https://docs.python.org/3/library/unittest.mock.html#patch)

Does that clear anything up? How about some examples?

## no spec_set, no autospec
{% highlight python %}
# define the function
In [2]: def func(first: str, last: str):
   ...:     print(first, last)
   ...:

# call the function
In [3]: func("foo", "bar")
foo bar

# No errors, do whatever you want.
In [37]: with patch("__main__.func") as mock_func:
    ...:     mock_func.a
    ...:

In [40]: with patch("__main__.func") as mock_func:
    ...:     mock_func.a = "b"
    ...:

In [38]: with patch("__main__.func") as mock_func:
    ...:     mock_func()
    ...:

In [39]: with patch("__main__.func") as mock_func:
    ...:     mock_func("test")
{% endhighlight %}

## autospec
{% highlight python %}
# with autospec cannot reference nonexistent attribute.
In [41]: with patch("__main__.func", autospec=True) as mock_func:
    ...:     mock_func.a
    ...:
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
Input In [41], in <cell line: 1>()
      1 with patch("__main__.func", autospec=True) as mock_func:
----> 2     mock_func.a

AttributeError: 'function' object has no attribute 'a'

# but assignment is fine
In [42]: with patch("__main__.func", autospec=True) as mock_func:
    ...:     mock_func.a = "b"
    ...:

# calling a function with incorrect signature, not OK
In [44]: with patch("__main__.func", autospec=True) as mock_func:
    ...:     mock_func()
    ...:
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
Input In [44], in <cell line: 1>()
      1 with patch("__main__.func", autospec=True) as mock_func:
----> 2     mock_func()

File <string>:2, in func(*args, **kwargs)

File ~/.pyenv/versions/3.10.3/lib/python3.10/unittest/mock.py:184, in _set_signature.<locals>.checksig(*args, **kwargs)
    183 def checksig(*args, **kwargs):
--> 184     sig.bind(*args, **kwargs)

File ~/.pyenv/versions/3.10.3/lib/python3.10/inspect.py:3179, in Signature.bind(self, *args, **kwargs)
   3174 def bind(self, /, *args, **kwargs):
   3175     """Get a BoundArguments object, that maps the passed `args`
   3176     and `kwargs` to the function's signature.  Raises `TypeError`
   3177     if the passed arguments can not be bound.
   3178     """
-> 3179     return self._bind(args, kwargs)

File ~/.pyenv/versions/3.10.3/lib/python3.10/inspect.py:3094, in Signature._bind(self, args, kwargs, partial)
   3092                 msg = 'missing a required argument: {arg!r}'
   3093                 msg = msg.format(arg=param.name)
-> 3094                 raise TypeError(msg) from None
   3095 else:
   3096     # We have a positional argument to process
   3097     try:

TypeError: missing a required argument: 'first'
{% endhighlight %}

## spec_set
{% highlight python %}
# with spec_set referencing non-existent attribute is not OK
In [45]: with patch("__main__.func", spec_set=True) as mock_func:
    ...:     mock_func.a
    ...:
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
Input In [45], in <cell line: 1>()
      1 with patch("__main__.func", spec_set=True) as mock_func:
----> 2     mock_func.a

File ~/.pyenv/versions/3.10.3/lib/python3.10/unittest/mock.py:634, in NonCallableMock.__getattr__(self, name)
    632 elif self._mock_methods is not None:
    633     if name not in self._mock_methods or name in _all_magics:
--> 634         raise AttributeError("Mock object has no attribute %r" % name)
    635 elif _is_magic(name):
    636     raise AttributeError(name)

AttributeError: Mock object has no attribute 'a'

# neither is assignment
In [46]: with patch("__main__.func", spec_set=True) as mock_func:
    ...:     mock_func.a = "b"
    ...:
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
Input In [46], in <cell line: 1>()
      1 with patch("__main__.func", spec_set=True) as mock_func:
----> 2     mock_func.a = "b"

File ~/.pyenv/versions/3.10.3/lib/python3.10/unittest/mock.py:749, in NonCallableMock.__setattr__(self, name, value)
    745     return object.__setattr__(self, name, value)
    746 elif (self._spec_set and self._mock_methods is not None and
    747     name not in self._mock_methods and
    748     name not in self.__dict__):
--> 749     raise AttributeError("Mock object has no attribute '%s'" % name)
    750 elif name in _unsupported_magics:
    751     msg = 'Attempting to set unsupported magic method %r.' % name

AttributeError: Mock object has no attribute 'a'

# but spec_set doesn't care about function signature
In [47]: with patch("__main__.func", spec_set=True) as mock_func:
    ...:     mock_func("test")
    ...:
{% endhighlight %}

## spec_set assertion typos
{% highlight python %}
# useful for assertion typos
In [50]: with patch("__main__.func", spec_set=True) as mock_func:
    ...:     mock_func.asret_called_once()
    ...:
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
Input In [50], in <cell line: 1>()
      1 with patch("__main__.func", spec_set=True) as mock_func:
----> 2     mock_func.asret_called_once()

File ~/.pyenv/versions/3.10.3/lib/python3.10/unittest/mock.py:634, in NonCallableMock.__getattr__(self, name)
    632 elif self._mock_methods is not None:
    633     if name not in self._mock_methods or name in _all_magics:
--> 634         raise AttributeError("Mock object has no attribute %r" % name)
    635 elif _is_magic(name):
    636     raise AttributeError(name)

AttributeError: Mock object has no attribute 'asret_called_once'
In [51]: with patch("__main__.func", spec_set=True) as mock_func:
    ...:     mock_func("test", "test")
    ...:     mock_func.assert_called_once()
    ...:
{% endhighlight %}

## autospec assertion typos
{% highlight python %}
# same for autospec
In [52]: with patch("__main__.func", autospec=True) as mock_func:
    ...:     mock_func.asret_called_once()
    ...:
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
Input In [52], in <cell line: 1>()
      1 with patch("__main__.func", autospec=True) as mock_func:
----> 2     mock_func.asret_called_once()

AttributeError: 'function' object has no attribute 'asret_called_once'

In [53]: with patch("__main__.func", autospec=True) as mock_func:
    ...:     mock_func("test", "test")
    ...:     mock_func.assert_called_once()
    ...:
{% endhighlight %}

# Conclusion

The take away,  I think, is that autospec creates the mock with all the attributes of the spec, functions having matching signatures, and attributes are also autospecced. spec_set really only cares about the attempt to assign an attribute that doesn't exist in the spec.

# Bonus Autospeccing Example

The block below explains that autospeccing can trigger code when inspecting the mocked object

> This isn’t without caveats and limitations however, which is why it is not the default behaviour. In order to know what attributes are available on the spec object, autospec has to introspect (access attributes) the spec. As you traverse attributes on the mock a corresponding traversal of the original object is happening under the hood. If any of your specced objects have properties or descriptors that can trigger code execution then you may not be able to use autospec. On the other hand it is much better to design your objects so that introspection is safe.
[autospeccing docs](https://docs.python.org/3/library/unittest.mock.html#autospeccing)

Assumed, based on the docs that code in a property would be triggered.

{% highlight python %}
In [1]: from unittest.mock import patch

In [2]: class Foo:
   ...:     @property
   ...:     def get_data(self):
   ...:         raise Exception
   ...:

# no exception raised
In [3]: with patch("__main__.Foo", autospec=True) as mock_foo:
   ...:     mock_foo is Foo
   ...:
{% endhighlight %}

So I dug, under what circumstances does autospeccing cause code to be executed? See create_autospec() snippet below taken from cpython.

{% highlight python %}
    for entry in dir(spec):
        if _is_magic(entry):
            # MagicMock already does the useful magic methods for us
            continue

        # XXXX do we need a better way of getting attributes without
        # triggering code execution (?) Probably not - we need the actual
        # object to mock it so we would rather trigger a property than mock
        # the property descriptor. Likewise we want to mock out dynamically
        # provided attributes.
        # XXXX what about attributes that raise exceptions other than
        # AttributeError on being fetched?
        # we could be resilient against it, or catch and propagate the
        # exception when the attribute is fetched from the mock
        try:
            original = getattr(spec, entry)
        except AttributeError:
            continue
{% endhighlight %}

Ok, so it appears `getattr()` is the concern. getattr is a builtin, I searched the web for "can getattr cause code execution?" instead of searching for the code. That led to a github [issue](https://github.com/python/cpython/issues/55342) raising a concern about getattr_static executing code in some conditions.

Followed the [code](https://github.com/python/cpython/blob/5113ed7a2b92e8beabebe5fe2f6e856c52fbe1a0/Lib/inspect.py#L1855) and found this comment

{% highlight python %}
def getattr_static(obj, attr, default=_sentinel):
    """Retrieve attributes without triggering dynamic lookup via the
       descriptor protocol,  __getattr__ or __getattribute__.

       Note: this function may not be able to retrieve all attributes
       that getattr can fetch (like dynamically created attributes)
       and may find attributes that getattr can't (like descriptors
       that raise AttributeError). It can also return descriptor objects
       instead of instance members in some cases. See the
       documentation for details.
    """
{% endhighlight %}

Ok, so using `getattr()` with an object that implements descriptor protocol. Never heard of that, but I landed [here](https://docs.python.org/3/howto/descriptor.html). So, lets try autospeccing an object that has a descriptor as an attibute

{% highlight python %}
from unittest.mock import patch

class Bar:
    def __get__(self, obj, ojbtype=None):
        raise Exception

class Foo:
    data = Bar()

if __name__ == "__main__":
    with patch("__main__.Foo", autospec=True) as mock_foo:
        mock_foo is Foo
{% endhighlight %}

Bingo!

{% highlight python %}
Traceback (most recent call last):
  File "/home/morgan/d/lab/python/mocks/main.py", line 11, in <module>
    with patch("__main__.Foo", autospec=True) as mock_foo:
  File "/home/morgan/.pyenv/versions/3.10.4/lib/python3.10/unittest/mock.py", line 1533, in __enter__
    new = create_autospec(autospec, spec_set=spec_set,
  File "/home/morgan/.pyenv/versions/3.10.4/lib/python3.10/unittest/mock.py", line 2685, in create_autospec
    mock = Klass(parent=_parent, _new_parent=_parent, _new_name=_new_name,
  File "/home/morgan/.pyenv/versions/3.10.4/lib/python3.10/unittest/mock.py", line 2083, in __init__
    _safe_super(MagicMixin, self).__init__(*args, **kw)
  File "/home/morgan/.pyenv/versions/3.10.4/lib/python3.10/unittest/mock.py", line 1086, in __init__
    _safe_super(CallableMixin, self).__init__(
  File "/home/morgan/.pyenv/versions/3.10.4/lib/python3.10/unittest/mock.py", line 441, in __init__
    self._mock_add_spec(spec, spec_set, _spec_as_instance, _eat_self)
  File "/home/morgan/.pyenv/versions/3.10.4/lib/python3.10/unittest/mock.py", line 496, in _mock_add_spec
    if iscoroutinefunction(getattr(spec, attr, None)):
  File "/home/morgan/d/lab/python/mocks/main.py", line 5, in __get__
    raise Exception
Exception
{% endhighlight %}

# TODO

Another post mocking a class instead of a function.