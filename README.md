# Sovereign

An experiment in software that has sovereignty over itself.

## Why I Did This

If you actually read through all of this, please let me know. You'd totally make my day.

If you don't, that's fine too, this whole thing is pretty weird. But while I have you, let me try and explain why I
started this in the first place and why you might want to give it a look-see.

It started while I was thinking about automating as much of the mechanics of a government as possible. I'm **not**
talking about automatically writing laws. I'm talking about creating an authoritative service that _manages_ the laws,
and provides an API for everything people need from it. In the US, for example, there could be an official API for
submitting and voting on bills, which Congress would use, allowing these laws to be overturned, which the Judicial
Branch would use, and for all of this to be public and transparent, which every citizen could use.

Then it devolved into this whole meta-craziness when I considered that there are laws about how laws are allowed to be
changed, which means the laws would need to be interpretable by this authoritative system. For example, if the law was
changed so that a two-thirds majority was required to pass any legislation, the voting API would need to understand that
law, and (assuming a simple majority was the standard before that), would allow such a law to pass with a 51% of the
vote, but then require at least 67% for all future votes (at least until that law was changed again!)

Once I was on that route, I started wondering about edge cases. How could you bootstrap a system like this? Could an
automated government like that "get stuck?" Is there any way to ensure that you **can't** get stuck? What _else_ can you
(not) ensure? Lots of thoughts flying everywhere. Panic was setting in. It wasn't pretty. I needed to bring things back
down to earth, to something concrete. I thought about how the core system would need to be written in code, and to
simplify my life, what if all laws were code as well? So we're talking about proposing changes to laws, and by that now
I mean proposing changes to code? Obviously that's a job for [`git`](https://git-scm.com/). We're settling down, this is
good. OK, what's the simplest possible piece of code that governs change to itself and shows the current version of its
own code? Sounds like a server that runs right where its own code is stored, shows its own code to anyone that asks,
and has single endpoint that accepts patches to its own code, which by default, it blindly accepts and runs. 😅 Phew!
Crisis averted.

But... OK, let's say, PURELY hypothetically obviously, I built this little prototype. What if I...

...

Guess you'll have to keep reading to find out! 😉

Or at least just humor me and take a gander at the abstract. It's not bad.

## Abstract

This repository contains a basic prototype of a program that self-regulates what patches it will allow to be applied to
itself. In this sense, it has sovereignty over itself, termed a **sovereign server**.

The [caveats](#Caveats) explain that this repo is explicitly a prototype, its code/writing are meant to illustrate
concepts and inspire thinking about this fairly abstract topic by providing concrete examples that minimally work. It is
not a comprehensive analysis or implementation.

The [developer docs](DEVELOPER.md) explain how to set up a sovereign server locally.

Then, this README walks through six experiments, each showing a concrete implementation for these scenarios. Each
experiment demonstrates the following by example:

1. Changing the server to reject all future patches means that server is stuck in its current form forever.
2. A server which admits any patch may admit a patch to delete itself, thus allowing self-termination.
3. A server may simply log and not act on all future requests, resulting in static behavior but interminable
   growth on disk.
4. Patching the server to test future patches for obvious problems is possible but can itself be undone, and
   so does not guarantee this behavior forever.
5. There is, however, a technique that _does_ work for permanently locking in some desired behavior.
6. Finally, the technique presented in Experiment 5 can be used to permanently ensure that no future change is 
   permanent -- quite the paradox.

The code for each experiment can be found in the
[sovereign-experiments](https://github.com/jdhenke/sovereign-experiments) repo under a branch named `exp-#` where `#` is
the experiment number. A link is included in each experiment's header.

See this [repository's discussions](https://github.com/jdhenke/sovereign/discussions) if you are interested in
reading or writing more about this, and its corollaries to life outside of code.

## Caveats

This code is meant to be a concrete prototype of a very abstract concept to aid in understanding what would otherwise
be purely hypothetical thought experiments. You should consider the implementations directional and the first step
towards what _might_ be possible in a real sense in the future. These are **not** comprehensive implementations.

I found it helpful to consider a fictional world in which these programs came into existence where there is infinite
compute and storage, all processes can be perfectly isolated from each other, and no one has access to any underlying
infrastructure -- the only way to interact with a sovereign server is through its `/patch` endpoint.

Again, I strongly encourage you to not focus on any edge cases in the specific lines of code you may find here. Instead,
I think you'll get the most out of this content if you focus on the spirit of each exercise, using the specific 
implementations here as (minimally) working prototypes, to the extent that they are helpful.

If you are interested in running these experiments yourself, you should start with the [developer docs](DEVELOPER.md)
to understand how to run and interact with a sovereign server. You can also see all the (accepted) patches for all
of these experiments in the **[sovereign-experiments](https://github.com/jdhenke/sovereign-experiments)** repository.

OK, let's get started.

## The Beginning - Accept All Patches ([`code`](https://github.com/jdhenke/sovereign-experiments/commit/cf5c72cb4c286242d2bf65bff44458f0e7c2ff30))

This is an exploration into what it would be like to have a running piece of software that self-verified any changes
to itself before applying them. In this way, it has **sovereignty** over itself.

This repository gives you such a server that simply admits _any_ patch as a starting point. The remaining experiments
all begin here. It's worth thinking about if there's any better, more general starting point. I don't think so, but it
kinda maeks you wonder...

Anyway, on to the first experiment, which I suppose is the opposite of the starting point, which is to _reject_, rather
than _accept_, all patches.

## Experiment 1 - Reject All Patches ([`code`](https://github.com/jdhenke/sovereign-experiments/commits/exp-1))

What if you tried to _reject_ all patches, would it work?

Consider this patch:

```diff
diff --git a/main.go b/main.go
index b3edbb6..95710f6 100644
--- a/main.go
+++ b/main.go
@@ -82,7 +82,7 @@ func stopServer() {
 }
 
 func verifyPatch(patch []byte) error {
-	return nil // always accept any patch
+	return fmt.Errorf("error") // never accept any patch
 }
 
 func applyPatch(patch []byte) error {
```

You actually _can_ successfully submit this patch because the _currently running_ server accepts _any_ patch, even a
patch that will reject all future patches.

However, you can no longer submit any more patches after that. Even trying to revert the patch will not be accepted.

```
$ git revert HEAD
[exp-0 4a03db1] Revert "Never admit any patch"
 1 file changed, 1 insertion(+), 1 deletion(-)
$ git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
verifying patch: error
```

This sovereign server does indeed have sovereignty over itself, but once it is in a state where it can no longer accept
patches, it is stuck in that state for eternity.

One thing about the server's state in this case is that it is at least still running. Is it possible to submit a patch
that completely breaks this server?

## Experiment 2 - Delete Everything ([`code`](https://github.com/jdhenke/sovereign-experiments/commits/exp-2))

The heading's pretty self-explanatory. Let's see what happens if we spin up a fresh server which admits any patch:

```
$ git ls-files | xargs git rm
rm '.gitignore'
rm 'DEVELOPER.md'
rm 'LICENSE'
rm 'README.md'
rm 'go.mod'
rm 'main.go'

$ git commit -m 'Delete everything'
[exp-2 37c1d1d] Delete everything
 6 files changed, 302 deletions(-)
 delete mode 100644 .gitignore
 delete mode 100644 DEVELOPER.md
 delete mode 100644 LICENSE
 delete mode 100644 README.md
 delete mode 100644 go.mod
 delete mode 100644 main.go

$ git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
OK
```

Huh, seems to work. Let's check the server logs...

```
$ PORT=8080 go run .
2022/02/08 16:51:51 Starting server...
2022/02/08 16:54:55 git am: Applying: Delete everything
2022/02/08 16:54:55 Stopping server...
package .: no Go files in /Users/joe/live
exit status 1
```

Welp, seems like the server stopped then tried to start itself again but couldn't because, (spoiler alert,) everything
was deleted.

We can easily see this by trying to contact the `/` endpoint which should list all the server's files:

```
$ curl localhost:8080                                                                       
curl: (7) Failed to connect to localhost port 8080: Connection refused
```

And as expected, nada.

So it is _definitely_ possible (and actually quite easy) to break this sovereign server when its self-governance amounts
to "yolo."

Now, is it possible to go in the _other_ direction? Instead of allowing everything to be _deleted_, is it possible to
create something that is only allowed to _grow_?

## Experiment 3 - Interminable Growth ([`code`](https://github.com/jdhenke/sovereign-experiments/commits/exp-3))

Again, start with a fresh server and client. Is it possible to modify the server such that it will only ever grow? What
if instead of applying patches, the server simply added them to a file?

So, the original patch could change the `applyPatch` function to something like this:

```go
func applyPatch(patch []byte) error {
	f, err := os.OpenFile("patches.log", os.O_WRONLY|os.O_CREATE|os.O_APPEND, 0666)
	if err != nil {
		return fmt.Errorf("opening patches.log: %v", err)
	}
	if _, err := io.Copy(f, bytes.NewReader(patch)); err != nil {
		return fmt.Errorf("copying patch to patches.log: %v", err)
	}
	if err := f.Close(); err != nil {
		return fmt.Errorf("closing patches.log: %v", err)
	}
	return nil
}
```

Again, because the original server admits any patch, it will accept **and apply** this patch because it's the original
version of the code that handles this submission.

```
$ git add main.go 
$ git commit -m 'Log patches instead of applying them.'
[exp-3 62a2ba4] Log patches instead of applying them.
 1 file changed, 7 insertions(+), 6 deletions(-)
$ 
$ git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
OK
```

Checks out.

So what happens if you try to make a new patch?

```
$ git revert HEAD
[exp-3 f8f9c8c] Revert "Log patches instead of applying them."
 1 file changed, 6 insertions(+), 7 deletions(-)
$ git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
OK
```

Well it accepted the patch, but did it work as expected?

Looking at the logs, it seems that there is no more mention of applying a patch. It looks like the server is restarting
but not updating itself.

```
2022/02/08 17:22:46 Stopping server...
2022/02/08 17:22:46 Starting server...
```

Let's see if the server is also logging these patches to `patches.log`. We can do this by asking the server itself,
which serves its current files at `/`:

```
$ curl -s localhost:8080/patches.log | head
From e87556a087fc59af97fb71236d3cc275e489cf7a Mon Sep 17 00:00:00 2001
From: Joe Henke <joed.henke@gmail.com>
Date: Tue, 8 Feb 2022 17:19:30 -0700
Subject: [PATCH] Revert "Log patches instead of applying them."

This reverts commit c8d69590ec55ff920d5ac6eedc7310eb53fa9258.
---
 main.go | 13 ++++++-------
 1 file changed, 6 insertions(+), 7 deletions(-)
```

OK! We've got a stuck server again! But this time with the added bonus that it will also keep adding to this
`patches.log` file for **eternity**.

Alright, so far we've kinda teased out some bad scenarios in various directions. 

1. ⏸ We can freeze it exactly as it is.
2. 📉 We can remove everything and terminate the server.
3. 📈 We can keep the server running forever and let its file size grow unchecked.

I know what you're thinking. "Surely, this guy spent all this time writing this insanity up, there _must_ be _something_
**good** or at least **interesting** that can come out of all of this?"

I think so. Let's see if instead we can put some more nuanced guard rails in place to keep the server in line.

We'll try one naive approach in Experiment 4, then a more sophisticated one in Experiment 5.

## Experiment 4 - Make The Server Unbreakable ([`code`](https://github.com/jdhenke/sovereign-experiments/commits/exp-4))

### Background

Seems like a reasonable thing to try would be a common practice in coding of running tests before accepting changes. The
hitch in this setup is that the existing code tests the new code. In a typical piece of software, you test the new code
by using the new code. The problem with this is that you could simply remove the tests, push the code, and nothing
fails, so it's accepted.

Still, in the spirit of testing that the resulting server after a patch still works, let's add a test which does exactly
that. The ultimate goal is to ratchet in place the guarantee that for **all** future patches, **none** of them will
break the server. Let's see how it goes...

### Testing The Patch

First, as always, start with a fresh server and client.

Then, change the `verifyPatch` function to do the following:

1. Copy the current server
2. Apply the patch to that copy
3. Start the copy server
4. Verify it successfully responds to requests

Here's the code that does this:

```go
// Verify by spinning up a copy of the future server that would exist after this patch is applied and making sure it
// starts.
func verifyPatch(patch []byte) error {
	const testDir = "../test"
	if err := os.RemoveAll(testDir); err != nil {
		return fmt.Errorf("ensuring old test server is removed: %v", err)
	}
	defer func() {
		_ = os.RemoveAll(testDir)
	}()

	if err := exec.Command("git", "clone", ".git", testDir).Run(); err != nil {
		return fmt.Errorf("cloning current server to test server: %v", err)
	}

	applyPatchCmd := exec.Command("git", "am")
	applyPatchCmd.Dir = testDir
	applyPatchCmd.Stdin = bytes.NewReader(patch)
	if err := applyPatchCmd.Run(); err != nil {
		return fmt.Errorf("applying patch to test server: %v", err)
	}

	runTestServerCmd := exec.Command("go", "run", ".")
	runTestServerCmd.Dir = testDir
	runTestServerCmd.Env = append(os.Environ(), "PORT=8081")
	if err := runTestServerCmd.Start(); err != nil {
		return fmt.Errorf("starting test server: %v", err)
	}
	defer func() {
		_ = runTestServerCmd.Process.Kill()
	}()

	passed := false
	for i := 0; i < 10; i++ {
		time.Sleep(1 * time.Second)
		resp, err := http.Get("http://localhost:8081/")
		if err != nil {
			log.Printf("Error while waiting for test server to respond: %v", err)
			continue
		}
		if resp.StatusCode == 200 {
			passed = true
		} else {
			log.Printf("Received bad status code from test server: %v", resp.Status)
		}
		_, _ = io.Copy(ioutil.Discard, resp.Body)
		_ = resp.Body.Close()
		if passed {
			break
		}
	}

	if !passed {
		return fmt.Errorf("timed out waiting for test server to successfully start")
	}
	log.Println("Verified the test server can start successfully.")
	return nil
}
```

Submitting this as a patch is accepted, because, as always, a fresh server accepts any patch.

Now that this verification code is running though, let's test it out.

### Successfully Rejecting A Breaking Change

What if we tried this diff?

```diff
diff --git a/main.go b/main.go
index bdd6abb..1c01779 100644
--- a/main.go
+++ b/main.go
@@ -17,6 +17,7 @@ import (
 var srv *http.Server
 
 func main() {
+	os.Exit(1) // immediately stop the server
 	log.Println("Starting server...")
 	mux := http.NewServeMux()
 	mux.HandleFunc("/patch", handlePatch)
```

Now trying to patch the server...

```
$ git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
verifying patch: timed out waiting for test server to successfully start
```

... and it's rejected! Awesome!!

And how about a change that shouldn't break the server?

### Successfully Accepting a Benign Change

Pulling from the example in the developer docs, let's add an easter egg into the server.

```diff
diff --git a/main.go b/main.go
index bdd6abb..d38231e 100644
--- a/main.go
+++ b/main.go
@@ -20,6 +20,9 @@ func main() {
 	log.Println("Starting server...")
 	mux := http.NewServeMux()
 	mux.HandleFunc("/patch", handlePatch)
+	mux.HandleFunc("/easter", func(rw http.ResponseWriter, r *http.Request) {
+		http.Error(rw, "egg", http.StatusOK)
+	})
 	mux.Handle("/", http.FileServer(http.Dir(".")))
 	srv = &http.Server{
 		Addr:    fmt.Sprintf(":%s", os.Getenv("PORT")),
```

Let's see if the server will accept it:

```
$ git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
OK
```

So far so good. But did the change actually go into effect?

```
$ curl localhost:8080/easter                                                                
egg
```

Winning!

So... this seems pretty cool. What's the catch?

### The Pitfall

If we look at our history so far, we see this progression of the server:

```
$ git log --oneline --no-decorate
47526cb Add easter egg
ca28716 Test patches to ensure server can still start
cf5c72c Admit any patch
```

We've already shown that any patch that breaks the server will be rejected. Is there _any_ way that the server could
still somehow be broken in the future?

It turns out that it's possible, not with any _single_ patch, but there **is** a way with _two_.

The first patch simply reverts the change to the verification logic. This is the `Test patches to ensure server can
still start` patch in the history that's one below the current patch, so we can revert it like this:

```
$ git revert HEAD~1
[safe-change 306cd8f] Revert "Test patches to ensure server can still start"
 1 file changed, 1 insertion(+), 56 deletions(-)
$ git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
OK
```

And just like that, we're back to being vulnerable to allowing any patch.

Let's try the `os.Exit(1)` patch we tried before:

```diff
diff --git a/main.go b/main.go
index 1e8430d..5d62f34 100644
--- a/main.go
+++ b/main.go
@@ -17,6 +17,7 @@ import (
 var srv *http.Server
 
 func main() {
+	os.Exit(1)
 	log.Println("Starting server...")
 	mux := http.NewServeMux()
 	mux.HandleFunc("/patch", handlePatch)
```

OK, submitting this diff yields:

```
$ git add main.go
$ git commit -m 'Break server'
[master 47deb23] Break server
 1 file changed, 1 insertion(+)
$ git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
OK
$ curl localhost:8080/                                                                      
curl: (7) Failed to connect to localhost port 8080: Connection refused
```

And just when we thought we prevented any patches from breaking the server, that server is indeed broken. Sad.

In retrospect, it's incredibly similar to the problem raised with typical software setups, where you can delete the test
and then nothing fails. It's just that in this case, deleting the test must be done in a separate patch before actually
breaking the server in a following one.

Bummer, dude. Got any bright ideas?

I don't know about _bright_ per say, but lemme say a few things then try some more stuff.

## Interlude - Thought Experiments

### The Halting Problem

Summarizing the problem faced in the last experiment, if you think of the successive versions of the server as `a`, `b`,
`c`, and so on, the issue is that `a` was only checking for a property in `b`, specifically, that it starts up. But to
propagate that behavior forever, `a` must actually check `b` for _two_ properties:

- Property 1: `b` starts up.
- Property 2: `b` checks `c` for Property 1 **and Property 2**

Reread that again. It's kinda confusing.

The bad news is that this is getting into [Halting Problem](https://en.wikipedia.org/wiki/Halting_problem) territory, 
where Alan Turing (heard of him?) proved that it's impossible to write a program that examines a different program to
determine if that different program will finish running or run forever.

Now, I'm a bit out of my depth here, but it seems like if a solution to this sovereign server business relies on
examining the next version of a program for some arbitrary attribute, it would have to handle the attribute of whether
it terminates, just like the Halting Problem, which means that to solve the sovereign server problem is as hard or
harder than the Halting Problem, and if the Halting Problem is impossible, then so is this sovereign server problem.

I'm, like, _pretty_ sure this is legit, don't hold me to it, but I believe it enough to prove to myself to stop trying
to solve it. Phew!

So... is there _anything_ we can do to propagate some behavior forever?

### Trusting Trust

This reminds me of the Turing Award (heard of it?) lecture given by Ken Thompson,
[Reflections on Trusting Trust](https://www.cs.cmu.edu/~rdriley/487/papers/Thompson_1984_ReflectionsonTrustingTrust.pdf)
in which he describes how it's possible to sneak a trojan horse into code that would be impossible to detect by looking
at its source code. In the sovereign server case, the more positive framing would be that it's possible to put behavior
into code that will propagate through all future versions of the code when that code generates future versions of
itself.

Is something like this viable for a sovereign server?

### Shell Server

While not quite the same as the Trusting Trust exploit, there's something directionally similar about this next
approach.

Until now, the model has been that the `client` talks directly to the `server`.

```
client -> server
```

What if there were an intermediary, a `shell` server, that verified any patches itself before allowing them to be
applied by the inner server?

```
client -> shell -> server
```

The hitch is that the `shell` server must NOT be able to be modified. This is both a feature and a source of dread. Just
like in the early experiments, once something is stuck, it's stuck for **eternity**.

OK... so I _suppose_ this could work, but is that really a _sovereign_ server anymore if something _else_ is
adjudicating patches for it?

It depends. What if the shell server was created by the original sovereign server itself? That is, starting from a
fresh sovereign server that admits any patches, can we get there from here?

Let's see.

## Experiment 5 - Make The Server Unbreakable (For Reals)  ([`code`](https://github.com/jdhenke/sovereign-experiments/commits/exp-5))

This experiment follows the approach that was just outlined above, and it works! This time, I won't step through every
code change line by line, but instead give you the high level overview of what is happening at each step. If you're
interested, in the full code, you can see all the (accepted) patches on the [`exp-5` branch of the
`sovereign-experiments` repository](https://github.com/jdhenke/sovereign-experiments/commits/exp-5).

### Bootstrap the Shell Server

As always, start with a fresh sovereign server that admits any patch.

The next patch bootstraps the shell server. By this I mean, by submitting this patch, we go from this setup:

```
client -> server
```

To this:

```
client -> shell -> server
```

Again, the **huge** thing to note here is that the code for the shell server and the shell server itself were accepted
and run by the initial sovereign server. In other words, this was still achieved within the experiment's parameters,
namely that all changes to the server must first be accepted by the server itself.

Now, the other important piece to understand about this shell server is that it's now fixed for **eternity** -- there's
no going back. So this shell server has now injected its functionality forever more, which in this case, is to ensure
that applying a patch to the inner server doesn't prevent that server from starting up again.

Ultimately, this means that this system is, forever more, immune from any change that would prevent the server from
starting, because the shell server would reject it before applying it to the real server. That's pretty cool!

### Try A Breaking Change

Let's test the waters a bit.

If we revert the patch we just made to bootstrap the shell server, is it accepted? Does the shell server go away?

```
$ git log --oneline --no-decorate
94962e8 Bootstrap shell server
cf5c72c Admit any patch
$ git revert HEAD
[master 23fcc51] Revert "Bootstrap shell server"
 2 files changed, 168 deletions(-)
 delete mode 100644 shell/shell.go
$ git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
OK
```

OK, seems like the revert was at least accepted. 

Now, if the shell server is still working, it should not allow any patches that will break the inner server.

The interesting thing is that, according to the inner server, it should be allowed to apply any patch still, even a 
breaking one.

```
$ curl -s localhost:8080/main.go | grep -A2 '^func verifyPatch'
func verifyPatch(patch []byte) error {
        return nil // always accept any patch
}
```

Let's see what happens.

```diff
diff --git a/main.go b/main.go
index b3edbb6..f83d877 100644
--- a/main.go
+++ b/main.go
@@ -17,6 +17,7 @@ import (
 var srv *http.Server
 
 func main() {
+	os.Exit(1)
 	log.Println("Starting server...")
 	mux := http.NewServeMux()
 	mux.HandleFunc("/patch", handlePatch)
```

Trying to apply this diff...

```
$ git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
verifying patch: timed out waiting for test server to successfully start
```

...fails! The shell server is still doing it's job, and there's nothing that can be done to
change that behavior now.

Looking at the server logs (includes both the shell, prefixed with `shell:`, and inner servers), we see that this is
exactly what happened:

```
2022/02/09 14:22:01 shell: Error while waiting for test server to respond: Get "http://localhost:8081/": dial tcp [::1]:8081: connect: connection refused
2022/02/09 14:22:01 shell: Failed trying patch: verifying patch: timed out waiting for test server to successfully start
```

### Try A Benign Change

Will the shell server correctly accept a change that doesn't break the inner server?

```diff
diff --git a/main.go b/main.go
index b3edbb6..1e8430d 100644
--- a/main.go
+++ b/main.go
@@ -20,6 +20,9 @@ func main() {
 	log.Println("Starting server...")
 	mux := http.NewServeMux()
 	mux.HandleFunc("/patch", handlePatch)
+	mux.HandleFunc("/easter", func(rw http.ResponseWriter, r *http.Request) {
+		http.Error(rw, "egg", http.StatusOK)
+	})
 	mux.Handle("/", http.FileServer(http.Dir(".")))
 	srv = &http.Server{
 		Addr:    fmt.Sprintf(":%s", os.Getenv("PORT")),
```

Let's see if the patch is accepted and if the change goes live... 

```
$ git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
OK
$ curl localhost:8080/easter
egg
```

Yep! Sure does.

And we can similarly roll this change back with ease:

```
$ git revert HEAD
[master e71a3ee] Revert "add easter egg"
 1 file changed, 3 deletions(-)
$ git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
OK
$ curl localhost:8080/easter                                                                
404 page not found
```

### Implications

So does that mean we win or something?

Well, not really. Even with a shell server like this in place, you can still "break" the inner server's functionality,
even if it starts up. For example, if you made the change in [Experiment 1](#experiment-1), the shell server would allow
it because the new server would start, but the inner server itself would reject all future patches, even if they were
approved by the shell server. In other words, the shell server is only protecting against _one type_ of failure mode,
not all of them.

But, this experiment has introduced a useful technique, which is to allow a sovereign server to permanently lock in place
some behavior in a way that **no** future patch can undo.

So if testing that the server can simply start up isn't the only or best check to perform, the next question becomes:
What _other_ behaviors can we lock in place? And what would that mean for that sovereign server?

Let's try one other such behavior, then close this whole thing out.

## Experiment 6 - Revertability ([`code`](https://github.com/jdhenke/sovereign-experiments/commits/exp-6))

### Revertability

As noted in Experiment 5, we now have a technique to lock in a behavior **forever** in a sovereign server, and
Experiment 5 itself locked in a specific behavior, which is to never accept a patch that would prevent the server from
starting up.

The next behavior that I thought would be interesting to explore is **revertability**. By that, I mean that the server
only accepts patches which, if applied, result in a new server that **must** accept the revert of that patch. The idea
here being that it not only prevents patches that keep the server from starting up, but it also prevents the server from
ever becoming stuck, because by design, all future servers must accept transitioning back to the previous version.

### Implementation

Like Experiment 5, I will forego the full walk through of the code given its size, but you can see all the (accepted)
patches on the [`exp-6` branch of the
`sovereign-experiments` repository](https://github.com/jdhenke/sovereign-experiments/commits/exp-6).

At a high level, the revertability implementation creates a new shell server that verifies patches before sending them
to the inner server in the following way:

- It spins up a test server, which is a copy of the inner server before the patch
- It applies the patch to the test server
- It verifies that the test server did in fact update itself
- It applies the revert of the patch to the test server
- It verifies that the test server is now an _exact_ copy of the version of itself before the patch was applied

Once that's in place, we can try out our usual experiments, plus an additional one.

#### Trying To Break a Revertable Server

Here's the usual diff we use to try and break a server:

```
diff --git a/main.go b/main.go
index b3edbb6..f83d877 100644
--- a/main.go
+++ b/main.go
@@ -17,6 +17,7 @@ import (
 var srv *http.Server
 
 func main() {
+       os.Exit(1)
        log.Println("Starting server...")
        mux := http.NewServeMux()
        mux.HandleFunc("/patch", handlePatch)
```

And even though the inner server would accept any patch...

```
$ curl -s localhost:8080/main.go | grep -A2 '^func verifyPatch'
func verifyPatch(patch []byte) error {
        return nil // always accept any patch
}
```

... this server has been bootstrapped with a revertability shell server, so it rejects the change:

```
$ git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
verifying patch: waiting for patched test server: test server did not respond: Get "http://localhost:8081": dial tcp [::1]:8081: connect: connection refused
```

We can see the verification failed when it tried to patch the server and detected the test server was no longer working.

#### Adding And Reverting a Benign Change To A Revertable Server

`cherry-pick`ing the easter egg we added in Experiment 5, we can see it is accepted, live, and revertable in all the
usual ways:

```
$ git log --oneline origin/exp-5
418febc (origin/exp-5) Revert "add easter egg"
2658030 add easter egg
6154f88 Revert "Bootstrap shell server"
1f498d1 Bootstrap shell server
cf5c72c Admit any patch
$ git cherry-pick 2658030
[master d820862] add easter egg
 Date: Wed Feb 9 14:23:31 2022 -0700
 1 file changed, 3 insertions(+)
$ git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
OK
$ curl localhost:8080/easter
egg
$ git pull -r
...
$ git revert HEAD --no-edit
[master 19e3930] Revert "add easter egg"
 Date: Thu Feb 10 09:04:34 2022 -0700
 1 file changed, 3 deletions(-)
$ git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
OK
$ curl localhost:8080/easter                                                                
404 page not found
```

Sweet, so a breaking change won't work, a safe change will. What else can we test?

#### Trying The Shell Server Trick On A Revertable Server

Stepping back, to create this permanently revertable server, we in fact **permanently** locked in place the behavior
that no future change is permanent.

Wild.

So... what if we try to make another permanent change using the same technique when the server supposedly guarantees 
that no more permanent changes are possible? _What happens when an unstoppable force meets an immovable object??_

Well, it depends on the implementation of the revertability check, but to me, I believe the revertable server should
reject such a patch.

And we can see that play out in this experiment's implementation, which tries to create another (redundant)
revertability shell around the inner server, trying to go from this:

```
client -> shell 1 -> server
```

To this:

```
client -> shell 1 -> shell 2 -> server
```

This attempt to spawn another shell server is almost an exact copy of the first attempt in this experiment, but it
ensures it doesn't have a conflict with the first shell:

```
$ git log --oneline
eef3b22 (HEAD -> master, origin/master, origin/HEAD) Revert "Lock in revertability"
0c9480c Lock in revertability
6979d0d (investigate) Fix restart
cf5c72c Admit any patch
$ git cherry-pick --no-commit 0c9480c
$ git diff 0c9480c
diff --git a/shell/shell.go b/shell/shell.go
index d7bcf5a..5631461 100644
--- a/shell/shell.go
+++ b/shell/shell.go
@@ -20,7 +20,7 @@ import (
        "time"
 )
 
-const ModeVar = "CHILD"
+const ModeVar = "GCHILD"
 
 var (
        testPort, childPort int
$ git add .
$ git commit -m 'Try to spawn another shell'
[master 5ffe820] Try to spawn another shell
 3 files changed, 370 insertions(+), 2 deletions(-)
 create mode 100644 shell/exe.go
 create mode 100644 shell/shell.go
```

And we can see that the first revertability shell detects the presence of a second shell because during a test of
applying and reverting the patch which would create a second shell, after the revert, the server is no longer an exact
copy of itself as it was before the patch was originally applied.

```
$ git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
ERROR: verifying patch: server is different after applying patch and reverting: c32e2f65ac4c0951415d6f4aedb8a988 8b784849f56b0f966f98c331fb8cffa8
```

In fact, the logs show that, as expected, the revert does not actually change the test server because it is, because at
that point, it has actually already bootstrapped itself into being a shell server, and so does not apply any patches, 
such as this revert, to itself, and the tests detect that.

Here's a (somewhat) visual way of understanding what happens during this test and why the patch is rejected:

```
# start of the test
client -> shell 1 -> test server (v1)

# after the patch
client -> shell 1 -> shell 2 -> test server (v2)

# after the revert
client -> shell 1 -> shell 2 -> test server (v1)

# ^ERROR: shell 1 detects that "shell 2" ≠ "test server (v1)", so fails the verification and does not apply this patch.
```

Dang. This thing is pretty resilient.

### Implications

As previously mentioned, it's ironic that we are **permanently** locking in place the behavior that no future change is
permanent. And this concept of **revertability** seems pretty robust. If you can always backtrack, you'll never get
stuck. However, by locking in that guarantee, you then guarantee you can no longer lock in any more guarantees. Which
feels like both the point and terribly frightening.

I don't think I've fully processed all its implications, but if you've got any hot takes,
[start a discussion](https://github.com/jdhenke/sovereign/discussions), would love to hear from you.
