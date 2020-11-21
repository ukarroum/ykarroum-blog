---
layout: post
title: 'Python Internals Serie : Int (Long) Object'
---

In this article, we will read and discuss the implementation details of ints in _CPython_.

Altough using ints in python is fairly easy : 

{% highlight python %}

a = 1

b = a + 5

{% endhighlight %}

The file implementing them in python is roughly 5000 code lines.

Without further ado, let's dive in the code !

## longobject.c

First we need to find a starting point, we know that all pythons objects are stored in `cpython/Objects`.

Unfortunetly, we don't seem to have an `intobject.c` file. That's because the underlying C object for python integers is `PyLongObject`.

The file we're interessted in is : `longobject.c`

For the rest of this article, we'll revisit each function/macro of this file in the order in which they appear.

## MEDIUM_VALUE(x)

{% highlight c %}
/* convert a PyLong of size 1, 0 or -1 to an sdigit */
#define MEDIUM_VALUE(x) (assert(-1 <= Py_SIZE(x) && Py_SIZE(x) <= 1),   \
         Py_SIZE(x) < 0 ? -(sdigit)(x)->ob_digit[0] :   \
             (Py_SIZE(x) == 0 ? (sdigit)0 :                             \
              (sdigit)(x)->ob_digit[0]))
{% endhighlight %}

The comment is pretty straightforward, the macro convert a given _long_ to an _sdigit_. But what's an sdigit ? Well it depends : 

{% highlight c %}
#if PYLONG_BITS_IN_DIGIT == 30 /* signed variant of digit */
typedef int32_t sdigit
#elif PYLONG_BITS_IN_DIGIT == 15
typedef short sdigit; /* signed variant of digit */
#else
#error "PYLONG_BITS_IN_DIGIT should be 15 or 30
#endif

{% endhighlight %}

The `PYLONG_BITS_IN_DIGIT` is defined either at configure time or in `pyport.h`.

We have an assert to protect us from accidently casting a big integer to a sdigit, which may result in a loss of information, therefore potentiely issues which can be hard to fix.

Curiously, the assert check that the _size_ of x is bigger than -1, but can be less than 0 ? My guess is that size is unsigned to reflect the sign of the int, for exemple : `PY_SIZE(-15) = -2`.
This seems to be confirmed with `Py_SIZE(x) < 0 ? -(sdigit)(x)->ob_digit[0]`.

`ob_digit` looks like an array containing our integer. which can be confirmed in the file `longintrepr.h` : 

{% highlight c %}
struct _longobject {
    PyObject_VAR_HEAD
    digit ob_digit[1];
};
{% endhighlight %}

Im not sure why use an array of one element instead of just an integer `digit num`.


## IS_SMALL_INT(ival)

{% highlight c %}
#define IS_SMALL_INT(ival) (-NSMALLNEGINTS <= (ival) && (ival) < NSMALLPOSINTS)
{% endhighlight %}

Pretty straightforward function.

`NSMALLPOSINTS` and `NSMALLNEGINTS` is defined in the `pycore_interp.h` file : 

{% highlight c %}
/* interpreter state */

#define _PY_NSMALLPOSINTS           257
#define _PY_NSMALLNEGINTS           5
{% endhighlight %}

To undesrtand why those 2 magic numbers and and where the two macros are used, let's dive into the next function.
	
## static PyObject * get_small_int(sdigit ival)
{% highlight c %}
static PyObject *
get_small_int(sdigit ival)
{
    assert(IS_SMALL_INT(ival));
    PyInterpreterState *interp = _PyInterpreterState_GET();
    PyObject *v = (PyObject*)interp->small_ints[ival + NSMALLNEGINTS];
    Py_INCREF(v);
    return v;
}
{% endhighlight %}

This function implements what is commonly known as the _Flyweight pattern_, as explained here : [https://python-patterns.guide/gang-of-four/flyweight/](https://python-patterns.guide/gang-of-four/flyweight/), the flyweight pattern is used in python to create at the initialisation phase all the integers in the range [-5, 256]. At the runtime, whenever you ask for a number in this range, you'll always get the _same_ number.

Which can be easily checked : 

{% highlight python %}

20 + 30 is 50 # True
200 + 300 is 500 # False

{% endhighlight %}


Those integers objects are stored in the interpreter state.

According to [https://tenthousandmeters.com/blog/python-behind-the-scenes-1-how-the-cpython-vm-works/](https://tenthousandmeters.com/blog/python-behind-the-scenes-1-how-the-cpython-vm-works/) : 

> An interpreter state is a group of threads along with the data specific to this group. Threads share such things as loaded modules (sys.modules), builtins (builtins.__dict__) and the import system (importlib). 

Apparently flyweight integers are stored there.

We also use `Py_INCREF` to increment the reference count of the returned integer, recall that reference counting is what is used by the CPython garbage collector to detect which objects to free (Well not just that, since reference counting alone doesn't resolve circular references).


