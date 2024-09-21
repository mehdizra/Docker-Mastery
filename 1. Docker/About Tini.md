---
tags:
  - docker
  - linux
up: "[[1. Docker Concepts]]"
date: 2024-03-12
---
[What is advantage of Tini?](https://github.com/krallin/tini/issues/8)

text of above link:

First, let's talk a little bit about Docker. When you run a Docker container, Docker proceeds to isolate it from the rest of the system. That isolation happens at different levels (e.g. network, filesystem, processes).

Tini isn't really concerned with the network or the filesystem, so let's focus on what matters in the context of Tini: processes.

Each Docker container is a PID namespace, which means that the processes in your container are isolated from other processes on your host. A PID namespace is a tree, which starts at PID 1, which is commonly called `init`.

Note: when you run a Docker container, PID 1 is whatever you set as your `ENTRYPOINT` (or if you don't have one, then it's either your shell or another program, depending on the format of your `CMD`).

Now, unlike other processes, PID 1 has a unique responsibility, which is to reap zombie processes.

Zombie processes are processes that:

- Have exited.
- Were not `wait`ed on by their parent process (`wait` is the syscall parent processes use to retrieve the exit code of their children).
- Have lost their parent (i.e. their parent exited as well), which means they'll never be `wait`ed on by their parent.

When a zombie is created (i.e. which happens when its parent exits, and therefore all chances of it ever being `wait`ed by it are gone), it is reparent to `init`, which is expected to reap it (which means calling `wait` on it).

In other words, _someone_ has to clean up after "irresponsible" parents that leave their children un-`wait`'ed, and that's PID 1's job.

That's what Tini does, and is something the JVM (which is what runs when you do `exec java ...`) does **not** do, which his why you don't want to run Jenkins as PID 1.

Note that _creating_ zombies is usually frowned upon in the first place (i.e. ideally you should be fixing your code so it doesn't create zombies), but for something like Jenkins, they're unavoidable: since Jenkins usually runs code that isn't written by the Jenkins maintainers (i.e. your build scripts), they can't "fix the code".

This is why Jenkins uses Tini: to clean up after build scripts that create zombies.

---

Now, Bash actually does the same thing (reaping zombies), so you're probably wondering: why not use Bash as PID 1?

One problem is, if you run Bash as PID 1, then all signals you send to your Docker container (e.g. using `docker stop` or `docker kill`) end up sent to Bash, which does **not** forward them anywhere (unless you code it yourself). In other words, if you use Bash to run Jenkins, and then run `docker stop`, then Jenkins will never see the stop command!

Tini fixes by "forwarding signals": if you send a signal to Tini, then it sends that same signal to your child process (Jenkins in your case).

A second problem is that once your process has exited, Bash will proceed to exit as well. If you're not being careful, Bash might exit with exit code 0, whereas your process actually crashed (0 means "all fine"; this would cause Docker restart policies to not do what you expect). What you _actually_ want is for Bash to return the same exit code your process had.

Note that you _can_ address this by creating signal handlers in Bash to actually do the forwarding, and returning a proper exit code. On the other hand that's more work, whereas adding Tini is a few lines in your Dockerfile.

---

Now, there would be another solution, which would be to add e.g. another thread in Jenkins to reap zombies, and run Jenkins as PID 1.

This isn't ideal either, for two reasons:

First, if Jenkins runs as PID 1, then it's difficult to differentiate between process that were re-parented to Jenkins (which should be reaped), and processes that were spawned by Jenkins (which shouldn't, because there's other code that's already expecting to `wait` them). I'm sure you could solve that in code, but again: why write it when you can just drop Tini in?

Second, if Jenkins runs as PID 1, then it may not receive the signals you send it!

That's a subtlety in PID 1. Unlike other unlike processes, PID 1 does not have default signal handlers, which means that if Jenkins hasn't explicitly installed a signal handler for `SIGTERM`, then that signal is going to be discarded when it's sent (whereas the default behavior would have been to terminate the process).

Tini does install explicit signal handlers (to forward them, incidentally), so those signals no longer get dropped. Instead, they're sent to Jenkins, which is _not_ running as PID 1 (Tini is), and therefore has default signal handlers (note: this is _not_ the reason why Jenkins uses Tini, they use it for signal reaping, but it _was_ used in the RabbitMQ image for that reason).

---

Note that there are also a few extras in Tini, which would be harder to reproduce in Bash or Java (e.g. Tini can register as a subreaper so it doesn't actually _need_ to run as PID 1 to do its zombie-reaping job), but those are mostly useful for specialist use cases.

Hope this helps!

Here are some references you might be interested in to learn more about that topic:

- More about zombies: [https://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/](https://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/)
- A more succinct explanation [https://github.com/docker-library/official-images#init](https://github.com/docker-library/official-images#init)

Finally, do note that there _are_ alternatives to Tini (like Phusion's base image).

Tini differentiates with:

- Doing everything PID 1 **needs** to do and **nothing else**. Things like reading environment files, changing users, process supervision are out of scope for Tini (there are other, better tools for those)
- It requires zero configuration to do its job properly (Tini >= 0.6 will also warn you if you're not running it properly).
- It's got a lot of tests.
