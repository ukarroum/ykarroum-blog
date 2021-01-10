---
layout: post
title: 'Python Internals Serie : Int (Long) Object'
---

In this article, we will read and discuss the implementation details of _int_s in _CPython_.

Altough using integers in python is fairly easy : 

{% highlight python %}

a = 1

b = a + 5

{% endhighlight %}

The implementation file contains more than 5000 code lines.

Since the file contains roughly 116 functions/macros, I'll probably skip some (most of ?) functions.
 
Without further ado, let's dive into the code !

### longobject.c

First we need to find a starting point, we know that all pythons objects are stored in `cpython/Objects`.

Unfortunetly, we don't seem to have an `intobject.c` file. That's because the underlying C object for python integers is `PyLongObject`.

The file we're interessted in is : `longobject.c`

From now one I will refer to python's integers implementation both using _integers_ or _longs_, which is not technically accurate since in _C_ those are different types.

For the rest of this article, we'll revisit some of the most used operations/functions related to integers.

#### MEDIUM_VALUE(x)

{% highlight c %}
/* convert a PyLong of size 1, 0 or -1 to an sdigit */
#define MEDIUM_VALUE(x) (assert(-1 <= Py_SIZE(x) && Py_SIZE(x) <= 1),   \
         Py_SIZE(x) < 0 ? -(sdigit)(x)->ob_digit[0] :   \
             (Py_SIZE(x) == 0 ? (sdigit)0 :                             \
              (sdigit)(x)->ob_digit[0]))
{% endhighlight %}

The comment is pretty explanatory, the macro convert a given _long_ to an _sdigit_. But what's an sdigit ? Well it depends : 

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

We have an assert to protect us from accidently casting a big integer, which is not small enough to fit in an sdigit, to an sdigit, which may result in a loss of information, therefore potentiely issues which can be hard to detect.

Curiously, the assert check that the _size_ of x is bigger than -1, but can be less than 0 ? My guess is that size is unsigned to represent both the size and the sign of the integer, for exemple : `PY_SIZE(-15) = -2`.
This seems to be confirmed with `Py_SIZE(x) < 0 ? -(sdigit)(x)->ob_digit[0]`.

`ob_digit` looks like an array containing our integer. which can be confirmed in the file `longintrepr.h` : 

{% highlight c %}
struct _longobject {
    PyObject_VAR_HEAD
    digit ob_digit[1];
};
{% endhighlight %}

One curious fact, that I can't explain, is the small size of the array (one element ?), maybe it's somehow changed at runtime.

#### IS_SMALL_INT(ival)

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

To undesrtand why those 2 magic numbers and where this macro is used, let's dive into the next function.
	
#### static PyObject * get_small_int(sdigit ival)
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

We also use `Py_INCREF` to increment the reference count of the returned integer, recall that reference counting is what is used by the CPython garbage collector to detect which objects to free (Well not just that, since reference counting alone doesn't resolve circular references, but we'll discuss the _gc_ in more details in future articles).

#### maybe_small_long

Straitforward function, we check a given integer is small enough to fit into an sdigit, if it's the case we downcast it to an sdigit using the `MEDIUM_VALUE`. After downcasting we check if the integer is in the flyweight range _[-5, 257]_, if it's the case we decrement the referece counting (since we don't need two PyLong objects) and we return a reference on the already allocated number.

### How are longs created

Longs are created using the function `PyLongObject * _PyLong_New(Py_ssize_t size)`, size here refer to the number of digit of the target long.

{% highlight c %}
if (size > (Py_ssize_t)MAX_LONG_DIGITS) {
	PyErr_SetString(PyExc_OverflowError,
					"too many digits in integer");
	return NULL;
}
{% endhighlight %}

Well, looks like we can't have an indefinely big integer, but how big can our integers be ? 

{% highlight c %}
#define MAX_LONG_DIGITS \
    ((PY_SSIZE_T_MAX - offsetof(PyLongObject, ob_digit))/sizeof(digit))
{% endhighlight %}

If you don't already know it, _offsetof_ is a _C_ function wich will basicaly return an offset of the member (ob_digit) from the structure (PyLongObject), if you recall correctly, since our struture only contains `PyObject_VAR_HEAD` and 	`digit ob_digit`. so the offset is the memorry taken with VAR_HEAD.

So a integer has a maximal size of roughly Py_SSIZE_T_MAx.

According to this answer : [https://stackoverflow.com/a/42777910/14517936](https://stackoverflow.com/a/42777910/14517936) `Py_SSIZE_T_MAX = sys.maxsize`, which is according to the official documentation ( [https://docs.python.org/3/library/sys.html#sys.maxsize](https://docs.python.org/3/library/sys.html#sys.maxsize) ) equal to `2**31 - 1` on 32 bits machine or `2**63 - 1`.

`2**63 - 1` is a huge number of digits.

You can check this limit yourself by doing : 

{% highlight python%}
import sys
10**sys.maxsize
{% endhighlight %}

If the size is fine, we allocate enough memory to store our integer : 

{% highlight c %}
result = PyObject_MALLOC(offsetof(PyLongObject, ob_digit) +
						 size*sizeof(digit));
if (!result) {
	PyErr_NoMemory();
	return NULL;
}
{% endhighlight %}

As a good practice you should ALWAYS check that an allocation had correctly been performed, which may not be the case if you don't have enough memory for example, your futur self will thank you.

### Adding two integers

{% highlight c %}

static PyObject *
long_add(PyLongObject *a, PyLongObject *b)
{
    PyLongObject *z;

    CHECK_BINOP(a, b);

    if (Py_ABS(Py_SIZE(a)) <= 1 && Py_ABS(Py_SIZE(b)) <= 1) {
        return PyLong_FromLong(MEDIUM_VALUE(a) + MEDIUM_VALUE(b));
    }
    if (Py_SIZE(a) < 0) {
        if (Py_SIZE(b) < 0) {
            z = x_add(a, b);
            if (z != NULL) {
                /* x_add received at least one multiple-digit int,
                   and thus z must be a multiple-digit int.
                   That also means z is not an element of
                   small_ints, so negating it in-place is safe. */
                assert(Py_REFCNT(z) == 1);
                Py_SET_SIZE(z, -(Py_SIZE(z)));
            }
        }
        else
            z = x_sub(b, a);
    }
    else {
        if (Py_SIZE(b) < 0)
            z = x_sub(a, b);
        else
            z = x_add(a, b);
    }
    return (PyObject *)z;
}
{% endhighlight %}

First we check that the two integers are valid.
{% highlight c %}

#define CHECK_BINOP(v,w)                                \
    do {                                                \
        if (!PyLong_Check(v) || !PyLong_Check(w))       \
            Py_RETURN_NOTIMPLEMENTED;                   \
    } while(0)
{% endhighlight %}

One curious thing you may have noticed is the : 

{% highlight c %}

do {
	INSTR;
} while(0)

{% endhighlight %}

Which seems exactly the same as simply : 

{% highlight c %}
INST;
{% endhighlight %}

The purpose of doing this is macros in C are not really smart, the _preprocessor_ will simply do a find/replace and hope for the best. The problem with this can be illustrated with the following macro : 

{% highlight c %}
#define fct() \
	INSTR1; \
	INSTR2; \
{% endhighlight %}

But what happens if you do : 

{% highlight c %}
if(condition)
	fct()
else
	INSTR3;
{% endhighlight %}

the preprocessor will replace this with : 

{% highlight c %}
if(condition)
	INSTR1;
INSTR2;
else
	INSTR;
{% endhighlight %}

which is not valid _C_, since in _C_ if you omit the braces, the if body will only consists of the first instruction which is not what you would expect.

The do while solve this by enclosing all our instructions in one scope.

But why not just use : 

{% highlight c %}

{
	INSTR1;
	INSTR2;
}

{% endhighlight %}

As far as i know this is simply a syntaxic choice, since writing `MACRO();` is more natural than `MACRO()` (not that in the second case we have no semicolon).

The rest is : 

- if both a and b are negative we return -\|a + b\|
- if a is negative but not b we compute b - a
- if b is negative and a is positive we compute a - b
- if both are positive we compute \|a + b\|

This looks fairly complicated for a simple a + b, so why all the burden ?

Well recall that in `Python` integers can be very large (2^63 digits on 64 bits machines), this is achieved by storing the integers in arrays (each element representing a digit). 

We dive in the `x_add` code we see : 

{% highlight c %}
for (i = 0; i < size_b; ++i) {
        carry += a->ob_digit[i] + b->ob_digit[i];
        z->ob_digit[i] = carry & PyLong_MASK;
        carry >>= PyLong_SHIFT;
    }
    for (; i < size_a; ++i) {
        carry += a->ob_digit[i];
        z->ob_digit[i] = carry & PyLong_MASK;
        carry >>= PyLong_SHIFT;
    }
    z->ob_digit[i] = carry;
{% endhighlight c %}

## Conclusion
And that's it, the `long_sub` is fairly similar, and you can always (i encourage you to do so) check multiplication and division code.

We rarely stop to think about basic operations like integer operations, the longobject file shows use how complex and optimised long/integers implementation is in python.
