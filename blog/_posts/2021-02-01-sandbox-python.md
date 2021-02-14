---
layout: post
title: 'Sandboxing untrusted Python code using Chroot Jail'
---

In this article, I'll show you how we can run untrusted python code safely. We'll visit different python sanitization heuristics and show their limitations and why building a chroot jail is probably the safest way to do it.

I assume that the reader have some basic knowledge of Linux, but I'll do my best to explain every used concept.

## What is _sandboxing_

_Sandboxing_ is not really python specific, it basically means running an untrusted program , in a specific environement where you limit what the code can do (writing on the disk for example).

I recently started working on a web platform to make testing new candidates coding skills, using competitive problems (fow now i mainly use UVA judge data for testing), possible.

One of the problems I had is how can I safely run a code that i can make no assumptions about ?

I mean sure, the candidate may be ia very nice person and only try to solve the coding problem, but what if he doesn't ? What if he instead write a script that delete all your files ? Or maybe download a malicious program and run it on your server ?

This is where sandboxing comes to the rescue. A good sandbox make it possible to safely run the python script (or actually any program) without caring about what the script does or doesnt.

Before showing how we can build such a sandbox, we'll first try to solve this problem using sanitisation techniques and show why for dynamic programming languages such as python this is bound to fail.

## Sanitisation and limitations

Sanitisation is the process of removing or editing portions of code that are flagged as potentially dangerous. For example you can search for patterns like : `subprocess.Popen()` and delete them.

Let's try and build such a process.

### Version 0 : I trust the user.

First version, no protections you fully trust your users.

{% highlight python %}
import subprocess

# Not a typo but a protection in case a reader "try" the code on his machine
subxprocess.run(['rm', '-Rf', '/'])

{% endhighlight %}

### Version 1 : Static analysis of the source code

You now try to reject every code that has the string `subprocess`.

{% highlight python %}

eval(f"import {''.join(['sub', 'process'])} as sp; sp.run([...])")

{% endhighlight %}

### Version 2 : Adding eval to our blacklist

What about importlib ?

### Version 3 : Adding importlib ?

Whats about `os` module ?

### Version 4 : You give up

Now you start to see the limitations of sanitisation method. By using a sanitisation method you make the assumption that you already now all the possible attack vectors that a malicious actor may use. Which is, let's be honest, unrealistic, the second you get outmasrted by an attacker you loose control on your server.

This a common problem in computer security and also why it's recommended to use whitelist based approches instead of blacklists ones.

A possible whitelist based strategy can be to create a "subsystem" which only has python (and it s dependencies) and nothing else.

Building such a system is possible using chroot jail.


## Chroot Jail

## Présentaion of Jailkit

## Jailkit instalation 

## Jail configuration + sudo

## Running cmmand as coding user

## Reading input

## Writing output
