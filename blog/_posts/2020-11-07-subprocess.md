---
layout: post
title: 'Python Internals Serie : Subprocess.Popen'
---

The purpose of this serie is to review some parts of the _CPython_'s code.

Why ? 

Well there are multiple reasons : 

- Because we can. The project is opensource and the source code is freely available here : [https://github.com/python/cpython](https://github.com/python/cpython)
- I strongly believe that we, as developpers, can learn a lot by studying good and clean code. And I think we can safely assume that CPython's code, which is used practicaly everywhere, meet this crietria.
- I also think that studying python internals can make us better python programmers, for example ever wondered why changing _sys.stdout_ seems to have no effect on subprocesses ? Well at the end of this article You will know why.

I will try to regulary realease articles  on random functions/modules/data strucutres/algorithms.

Starting today I don't really have a plan or a timeline for future articles, I'll just start with _subprocess.Popen_ and go from there.

I'll try to link PEPs, articles and disscussions whenever I can.

DISCLAIMER : I am not a Python Core Developper, therefore a lot of claims that I'll make here may be wrong, especially if they're not backed by any evidence.

With that being said, let's read some code !


## Wait ... What the heck is CPython ? 

_CPython_ is the official python implementation. You need to understand that there is a difference between python specification and a python implementation.

A simple example can be the sorting function : 

The python specification specify that a valid implementation should offer a `sorted` function which can take a list and sort it.

But as you may recall from your painfull algorithms course, there are multiple sorting algorithms, so which one are being used ? Well, as a lot of things, it depends on the implementation.

In fact you can create your own implementation and use bubble sort if you want, not a very practical choice but you can !

_CPython_ is widely used nowadays, and a lot of its code is wrote using C, again this is an implementation detail not a specification one.

There are other python implementations like _PyPy_ (which use JIT compilation), or _Jython_ which is ran on a JVM machine.


## Getting the code

the first step is to get the cpython code. for that we gonna just clone the official repo on github : 

{% highlight bash %}
$ git clone https://github.com/python/cpython.git
{% endhighlight %}

## Subprocess.Popen : The class

Please note that we won't really analyze ALL the code of _Popen_, there are way too many options and use cases to fit in this article.

Instead we juste gonna try to understand what happens when you execute this line of code : 

{% highlight python %}
p = subprocess.Popen(['ls', '-l']) 
{% endhighlight %}

We need to find a starting point, we can simply do : 

{% highlight bash %}
$ find -name subprocess.py 

./Lib/asyncio/subprocess.py
./Lib/subprocess.py

{% endhighlight %}

we can safely assume that the module we're searching for is `Lib/subprocess`.

By searching for `Popen` in the file we find a class : 

{% highlight python %}
class Popen(object):
    """ Execute a child program in a new process.

    For a complete description of the arguments see the Python documentation.

    Arguments:
      args: A string, or a sequence of program arguments.

{% endhighlight %}


The first instruction is : 

{% highlight python %}
_child_created = False  # Set here since __del__ checks it
{% endhighlight %}

the `__del__` is basicaly a destructor method in python, this method is automatically called when the garbage collector try to free the object.

in CPython the garbage collector use what we call _reference counting_, this means that if you write something like this : 

{% highlight python %}
a = [1, 2, 3] 

a = [4, 5, 6] # we affect a new list to a
{% endhighlight %}


In the second line we no longer have a reference to the list `[1, 2, 3]`, which means that the garbage collector may delete it (or not) any time.

my guess, is that if you run a command like this : 

{% highlight python %}
subprocess.Popen(['my command'])
{% endhighlight %}

since you're not referencing the Popen object, it is a _garbage candidate_, and the gc may try to delete it before it run. Which is not the excepected behaviour since you except the command to run even if you don't care about what it'll return.

What surprises me tough is the fact that it's a class attribute, there is surely a reason behind it, but i'm not sure what.

### The `__init__` method

the first instruction in the init method is : 

{% highlight python %}
_cleanup()
{% endhighlight %}

from the comment it looks like the purpose of the function is to avoid zombie processes. one thing that i like about this function is how the autor of the except explictly explain why he passes when he catches a ValueError and why it's okay. I see a lot of developpers using the pattern `try: except (some exception or just Exception) pass` instead of treating the error proply. This can be very risky as it may hide a real error that you wish you saw before wasting 2 days trying to debug a random issue.

After that we can see some basic parameters validation : like _is bufsize an integer_ ?

This is also a very good habit to have, especialy in a dynamicaly typed language like Python. 

your code may still work with the wrong types and give you a wrong result, as an example : 

{% highlight python %}
def to_upper(list_str):
	return [s.upper() for s in list_str]

to_upper(['aaa', 'bbb']) # ['AAA', 'BBB']
to_upper('ab') # ['A', 'B']
{% endhighlight %}

As you can see altough it works, example 2 is clearly not what the user would expect.

You can also start to see that some code parts depends on the target plateforme (windows / POSIX), for the rest of this article, I'll on the focus part.

{% highlight python %}
(p2cread, p2cwrite,
c2pread, c2pwrite,
errread, errwrite) = self._get_handles(stdin, stdout, stderr)
{% endhighlight %}

next we get the _file decriptors_ related to the optional arguments stdin/stdout/stderr.

A file descriptor is an abstraction managed by the process, each file descriptor points to an entry in a system wide table called _Open file table_.

In python you can do : 

{% highlight python %}
open("my file").fileno()
{% endhighlight %}

This will return the file descriptor.

Based on the comments and common sense it looks like, we will not always need 6 file descriptors, but only if we choose to communicate with the child process (which is not our case for our example).

In our example it looks like : `c2pwrite` will be the fd of the output of our script (1 if stdout) and `p2cread` the fd of the input of our script (0 if stdin).

In the `_get_handles` function : 

looks like the default values (if no alternative stdout/in/err) is provided) : -1.

{% highlight python %}
p2cread, p2cwrite = -1, -1
c2pread, c2pwrite = -1, -1
errread, errwrite = -1, -1

 
if stdin is None:
	pass

if stdout is None:
	pass
{% endhighlight %}
 
the function seems to confirm our hypothesis : 

{% highlight python %}
else:
	# Assuming file-like object
	p2cread = stdin.fileno()
else:
	# Assuming file-like object
	c2pwrite = stdout.fileno()

{% endhighlight %}
it does bug me how similar the stdin / stdout / stderr code is. I know if I had to write this code I will desperatly try to somehow refactore it in one block, I'm not sure tough if it will be a good idea, refactoring is not always a good thing.

{% highlight python %}
 # How long to resume waiting on a child after the first ^C.
 # There is no right value for this.  The purpose is to be polite
 # yet remain good for interactive users trying to exit a tool.
 self._sigint_wait_secs = 0.25  # 1/xkcd221.getRandomNumber()

{% endhighlight %}
I found the comment funny, for refecence this the xkcd comic that it refers to : 

![xkcd comic](https://imgs.xkcd.com/comics/random_number.png)

next we call the function : `_execute_child` (POSIX version in our case) : 

### `_execute_child` function

Again some basic parameter validation.

{% highlight python %}
sys.audit("subprocess.Popen", executable, args, cwd, env) 
{% endhighlight %}

this will send a _signal_ and subsribed functions may do things accordenly (no idea what tough).

{% highlight python %}

if (_USE_POSIX_SPAWN
	and os.path.dirname(executable)
os_posix_spawnp_impl1712                     
	and preexec_fn is None
	and not close_fds
	and not pass_fds
	and cwd is None
	and (p2cread == -1 or p2cread > 2)
	and (c2pwrite == -1 or c2pwrite > 2)
	and (errwrite == -1 or errwrite > 2)
	and not start_new_session
	and gid is None
	and gids is None
	and uid is None
	and umask < 0):
		self._posix_spawn(args, executable, env, restore_signals,
				p2cread, p2cwrite,
				c2pread, c2pwrite,
				errread, errwrite)
		return
{% endhighlight %}
if the `_USE_POSIX_SPAWN` is true, python will try to use it to create the child process.

according to : [https://web.archive.org/web/20190922113430/https://www.oracle.com/technetwork/server-storage/solaris10/subprocess-136439.html ](https://web.archive.org/web/20190922113430/https://www.oracle.com/technetwork/server-storage/solaris10/subprocess-136439.html)

will not copy the data of the parent process, which can make the script start faster for larger processes.

{% highlight python %} 

 if sys.platform == 'linux' and libc == 'glibc' and version >= (2, 24):
 	# glibc 2.24 has a new Linux posix_spawn implementation using vfork
 	# which properly reports errors to the parent process.
 	return True

{% endhighlight %}
that if you are on a linux plateform and the version of _glibc_ is greater than _2.24_ you ll use `posix_spanw()`.

the 2.24 version of glibc has been released around 2016 so it's fair to assume that you have a recent linux distribution you ll also use `posix_spawn()`.

If you want to check 

{% highlight bash %}
$ ldd --version

ldd (Ubuntu GLIBC 2.31-0ubuntu9.1) 2.31

{% endhighlight %}

If you're interessted you can read the disccusion (on the python bug tracker) which lead to trying to use `posix_span whenever` : [https://bugs.python.org/issue35537](https://bugs.python.org/issue35537)


### `_posix_spawn` function 

{% highlight python %}
if env is None:
	env = os.environ
{% endhighlight %}

looks like our child process will inherit all our defined env variables

one interessting thing : 

{% highlight python %}
for fd, fd2 in (
 (p2cread, 0),
 (c2pwrite, 1),
 (errwrite, 2),
):
 if fd != -1:
     file_actions.append((os.POSIX_SPAWN_DUP2, fd, fd2))
{% endhighlight%}

we iterate over the fds (in/out/err) of our child process, which are -1 by default
as you can see if they're different than -1, based on [https://linux.die.net/man/2/dup2](https://linux.die.net/man/2/dup2), will close fd2 (standard input/output), and copy the value of fd (our new fd) place of it.

which mean in our case (-1) nothing happen ! so even if sys.stdout is different than 1, this won't affect children process, and this explain why _contextredirect_ will not affect children output/error

### `os.posix_spawn` function

we open `os.py` (same folder), we do a `/f posix_spawn` : nada. There is no function in this module with this name, my guess is that the function added to the exported functions somehow, and that it may be defined in the c code.

by doing a `grep -R 'posix_spawn'`, we seem to have a strong candidate here : 

{% highlight bash %}
Modules/clinic/posixmodule.c.h:    {"posix_spawnp", (PyCFunction)(void(*)(void))os_posix_spawnp, METH_FASTCALL|METH_KEYWORDS, os_posix_spawnp__doc__},
Modules/clinic/posixmodule.c.h:os_posix_spawnp_impl(PyObject *module, path_t *path, PyObject *argv,
Modules/clinic/posixmodule.c.h:os_posix_spawnp(PyObject *module, PyObject *const *args, Py_ssize_t nargs, PyObject *kwnames)

{% endhighlight %}
i guess that our function is `'os_posix_spawnp'` : 

Note : the `PyObject` is a C wrapper aroud python objects, so each variable you use in python, will be presented by a PyObject in C.

we first see some initializations.

and then some ... _gotos_ ? gonna admit that is kinda weird, since, well :  

![xkcd gogo comic](https://imgs.xkcd.com/comics/goto.png).

After some googling it appear that it's not a bad practice to use for this kind of situation : 

{% highlight c %}
void foo()
{
    if (!doA())
        goto exit;
    if (!doB())
        goto cleanupA;
    if (!doC())
        goto cleanupB;

    /* everything has succeeded */
    return;

cleanupB:
    undoB();
cleanupA:
    undoA();
exit:
    return;
}

{% endhighlight %}

source : [https://stackoverflow.com/a/245761](https://stackoverflow.com/a/245761)

goto are apprently also appropriate for jumping out of nested loops (fun fact : this is the only accepted form of goto in java, last time i checked)
after multiple conditions we go here : `os_posix_spawnp_impl`

### `os_posix_spawnp_impl` function
------------------------

`grep -R os_posix_spawnp_impl`, Modules/posixmodule.c looks like a strong candidate.

if you're intriged by this line :

{% highlight c %} 
/*[clinic end generated code: output=7b9aaefe3031238d input=c1911043a22028da]*/
{% endhighlight %}

_clinic_ is a boilerpate code generator for CPython, we may inspect it in futur articles.

### `py_posix_spawn`
-----------------
As above, some checks and gotos.

then we call `posix_spawnp `

by scrolling over the file we see this : 

{% highlight python %}
 os.posix_spawnp
     path: path_t
         Path of executable file.
     argv: object
         Tuple or list of strings.
     env: object
         Dictionary of strings mapping to strings.
     /
     *
     file_actions: object(c_default='NULL') = ()
         A sequence of file action tuples.
     setpgroup: object = NULL
         The pgroup to use with the POSIX_SPAWN_SETPGROUP flag.
     resetids: bool(accept={int}) = False
         If the value is `True` the POSIX_SPAWN_RESETIDS will be activated.
     setsid: bool(accept={int}) = False
         If the value is `True` the POSIX_SPAWN_SETSID or POSIX_SPAWN_SETSID_NP will be activated.
     setsigmask: object(c_default='NULL') = ()
         The sigmask to use with the POSIX_SPAWN_SETSIGMASK flag.
     setsigdef: object(c_default='NULL') = ()
         The sigmask to use with the POSIX_SPAWN_SETSIGDEF flag.
     scheduler: object = NULL
         A tuple with the scheduler policy (optional) and parameters.
 
Execute the program specified by path in a new process.
{% endhighlight %}

it looks like our function `os.posix_spawnp =  os_posix_spawnp_impl`

