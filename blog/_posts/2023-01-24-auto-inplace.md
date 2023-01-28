---
layout: post
title: 'Building automatic inplace in python'
---

When using data related python libraries (pandas, numpy, etc.) a lot of the functions accept an inplace parameter. By setting it to `True`, we change the source object without creating an additional copy, in some cases this can have a significant memory impact.

In this article I'll show that we can automatically detect some patterns where using `inplace=True` has no downsides:

{% highlight python %}

# There are no other references to df
df = my_fct(df) # should set inplace=True

{% endhighlight %}

To do this we need to solve two main issues:

First, we need to be sure that we have no other references on the source object, for example:

This is a valid use case.

{% highlight python %}

df = pd.DataFrame(...)

df = my_fct(df)

{% endhighlight %}

This is not. As we'll end up also changing `other_df`, which may not be intended by the user.

{% highlight pyhon %}

df = pd.DataFrame(...)

other_df = df

df = my_fct(df)

{% endhighlight %}

Second, we need to make sure that the same variable we get as a parameter is used as a target, example:

This is valid.

{% highlight python %}
df = my_fct(df)
{% endhighlight %}

This is not, as the changes to the source df are not expected.

{% highlight python %}
other_df = my_fct(df)
{% endhighlight %}

In the following, we'll use the following function as an example:

{% highlight python %}
def f(l: list[float], inplace: bool = False) -> list[float]:
	if inplace:
		res = l
	else:
		res = copy.deepcopy(l)

	for i in range(len(res)):
		res[i] *= 3

	return l

{% endhighlight %}


References counting
-------------------

To make sure that we don't accidentally change another object by automatically setting `inplace=True`, we need to use reference counting.

Basically, we should only have 2 references to our source object:

{% highlight python %}

def f(l, inplace=True): # <----- one here
	...

l = [2] * 1_000

l = f(l) # <----- one here

{% endhighlight %}

To count the references we can use : `sys.getrefcount(l)`.

If you try this, you may be surprised that the value is `4` not `2`.

This is due to `sys.getrefcount` also creating a reference to its argument.
The second additional ref count is explained here: [https://stackoverflow.com/a/46146772](https://stackoverflow.com/a/46146772), which looks like an implementation detail in the new way _CPython_ handles arguments passing starting _3.6_.

Our code now is:

{% highlight python %}
def f(l, inplace=True):
	if sys.getrefcount(l) == 4:
		# Check the second condition
		...

	# the rest of f
	...
{% endhighlight %}

Confirming the source and target objects are the same
-----------------------------------------------------

To solve the second issue, we'll:

* Retrieve information on the outer frame, since we're already in the function the outer frame will refer to the one from which we call the function.
* Retrieve the code portion referring to the function call.
* Using the code to build a small ast, to check that the first argument of the call has the same name as the target.

To retrieve the outer frame we can use the `inspect` module:

{% highlight python %}
frame = inspect.getouterframes(inspect.currentframe())[1]
{% endhighlight %}

We can get the code using `frame.code_context`.

Once we have the code, we can use the `ast` module to check if we're on the valid use case:

{% highlight python %}

# We parse the code to an ast (abstract syntax tree)
exprs = ast.parse(code_context[0].strip()).body

# We check that it's an assignement
if len(exprs) == 1 and isinstance(exprs[0], ast.Assign):
# We check that user is affecting the return value somewhere
# We also check that the second operand is a function call
if len(exprs[0].targets) == 1 and isinstance(exprs[0].value, ast.Call):
    lname = exprs[0].targets[0].id
    rname = exprs[0].value.args[0].id

    # If the target name and source name are the same we set inplace to True
    if lname == rname:
	inplace = True

{% endhighlight %}


Wrapping up
------------

We can offer an easy way to add automatic inplace to existing functions using decorators:

{% highlight python %}
def auto_inplace(func):
    def wrapper(*args, inplace=False, **kwargs):
        print("Something is happening before the function is called.")
        if sys.getrefcount(args[0]) == 4:
            frame = inspect.getouterframes(inspect.currentframe())[1]
            code_context = frame.code_context

            exprs = ast.parse(code_context[0].strip()).body

            if len(exprs) == 1 and isinstance(exprs[0], ast.Assign):
                if len(exprs[0].targets) == 1 and isinstance(exprs[0].value, ast.Call):
                    lname = exprs[0].targets[0].id
                    rname = exprs[0].value.args[0].id

                    if lname == rname:
                        inplace = True
        print(inplace)
        return func(*args, inplace=inplace, **kwargs)
    return wrapper
{% endhighlight %}

This way we can just decorate our functions that offer an inplace parameter:

{% highlight python %}
@auto_inplace
def f(l: list[float], inplace: bool = True):
	...
{% endhighlight %}

Limitations
-----------

There are some limitations with the presented approach:

Our decorator expects the source object to be the first argument, and expect the function to return exactly one value. This is the most common, but we can easily extend our decorator to more usages by adding parameters.

The second limitation is that the `frame.code_context` will only return one line of the code, this may not sound like an issue but it will make it hard to detect use cases like this one:

{% highlight python %}
df = f(
	df
)
{% endhighlight %}

This is a little trickier to solve and may cause some missed auto inplace opportunities.



