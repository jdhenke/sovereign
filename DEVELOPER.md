# Try Out Sovereign Yourself

## Run Sovereign Locally

First, start a sovereign server by copying the contents of this repository, initializing a new git history so you only
see the patches you propose to the sovereign server, and run the server locally:

```bash
git clone git@github.com:jdhenke/sovereign.git live
cd live
rm -rf .git
git init
git add .
git commit -m 'Admit any patch'
PORT=8080 go run .
```

It should say something like:

```
2022/02/08 09:37:26 Starting server...
```

Now, you can go to http://localhost:8080 to see the live code of the running sovereign server, even including
[this page](http://localhost:/8080/README.md).

## Propose A Change

In a **new** terminal, one directory up from the `live` directory you created above, run the following to create a clone
of the _running_ sovereign server's code in a `client` directory:

```bash
git clone live/.git client
cd client
```

This `client` directory is where you'll actually be making the proposed changes to the sovereign server. This tutorial
enables you to easily make all the changes you'll need from a terminal, but as you explore more yourself this is the
directory where you'll want to use your IDE.

For example, you could add a new route to the server with a change like this:

```diff
diff --git a/main.go b/main.go
index 1d58d82..b1aba9e 100644
--- a/main.go
+++ b/main.go
@@ -21,6 +21,10 @@ func main() {
 	mux := http.NewServeMux()
 	mux.HandleFunc("/patch", handlePatch)
 	mux.Handle("/", http.FileServer(http.Dir(".")))
+	mux.HandleFunc("/easter", func(rw http.ResponseWriter, r *http.Request) {
+		http.Error(rw, "egg", http.StatusOK)
+	})
+
 	srv = &http.Server{
 		Addr:    fmt.Sprintf(":%s", os.Getenv("PORT")),
 		Handler: mux,
```

Note: On a mac, you can copy the full text of this diff, then run `pbpaste | git apply` to make this change.

Now, commit this change in the usual way, something like:

```
git add main.go
git commit -m 'Add easter egg'
```

Now, you can attempt to apply this patch to the sovereign server with this command:

```bash
git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch
```

Because the initial logic of the server is to accept any patch, it should respond with `OK`.

If you look back at the server logs, it should say something like this, showing how the server applied the patch you
proposed, then restarted, thereby running the latest version of the code, which includes your patch:

```
2022/02/08 10:02:32 git am: Applying: Add easter egg
2022/02/08 10:02:32 Stopping server...
2022/02/08 10:02:34 Starting server...
```

You can easily verify this by using the new endpoint you just defined:

```
$ curl localhost:8080/easter
egg
```

Lastly, to update your client code to match the server's code, which should have the same contents, just different git 
hashes, run:

```bash
git pull -r
```

Now your change is live, your code is in sync with the server, and you are ready to make any more changes you want. 

## Revert Your Change

Now, instead of adding something, let's try removing something. In particular, let's undo the patch you just applied.

You can do so by performing the following actions:

```bash
# ensure the client is in sync with the server
git pull -r

# revert your latest commit
git revert HEAD

# propose these changes as a patch to the server
git format-patch --stdout origin/master| curl -XPOST --data-binary @- localhost:8080/patch

# update to match the server's code
git pull -r
```

Now, you can see the endpoint is no longer there:

```
curl localhost:8080/egg
404 page not found
```

## What's The Big Deal?

This tutorial was only meant to show you the mechanics of how to setup and interact with a sovereign server. The changes
that you made were not necessarily super interesting.

However, the _interesting_ bits come in when you consider making changes to the patch verification process itself.

For example, consider this change:

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

Would it be accepted? If so, what are the implications for this sovereign server moving forward?

See the [README](README.md) for a full explanation of this scenario, among others, explained with concrete patches made
against real sovereign servers. And for additional thoughts, see this repository's
[discussions](https://github.com/jdhenke/sovereign/discussions).
