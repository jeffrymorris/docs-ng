

# Installation and First Launch

## Quick installation

The Sync Gateway is known to work on Mac OS X, Windows and Ubuntu Linux. It has no external dependencies other than the Go language.

1. Install [Go](http://golang.org). Make sure you have version 1.0.3 or later. You can download and run a [binary installer](http://code.google.com/p/go/downloads/list), or tell your favorite package manager like apt-get or brew or MacPorts to install "golang".
2. Go needs [Git](http://git-scm.com) and [Mercurial](http://mercurial.selenic.com/downloads/) for for downloading packages. You may already have these installed. Enter `git --version` and `hg --version` in a shell to check. If not, go get 'em.
3. We recommend setting a `GOPATH` so 3rd party packages don't get installed into your system Go installation. If you haven't done this yet, make an empty directory someplace (like maybe `~/lib/Go`), then edit your .bash_profile or other shell startup script to set and export a `$GOPATH` variable pointing to it. (While you're at it, you might want to add its `bin` subdirectory to your `$PATH`.) Don't forget to start a new shell so the variable takes effect.
4. Now you can install the gateway simply by running `go get -u github.com/couchbaselabs/sync_gateway`

## Quick startup

The Sync Gateway launcher tool is `bin/sync_gateway` in the directory pointed to by your GOPATH (see item 3 above.) If you didn't set a GOPATH, it'll be somewhere inside your system Go installation directory (like `/usr/local/go`, or `/usr/local/Cellar/go` if you used Homebrew.)

If you've already added this `bin` directory to your shell PATH, you can invoke it from a shell as `sync_gateway`, otherwise you'll need to give its full path. Either way, start the gateway from a shell using the following arguments:

    sync_gateway -url walrus:

That's it! You now have a sort of mock-CouchDB listening on port 4984 (and with an admin API on port 4985.) It has a single database called "sync_gateway". It definitely won't do everything CouchDB does, but you can tell another CouchDB-compatible database (including TouchDB) to replicate with it.

(To use a different database name, use the `-dbname` flag: e.g. `-dbname mydb`.)

The gateway is using a simple in-memory database called [Walrus][WALRUS] instead of Couchbase Server. This is very convenient for quick testing. Walrus is _so_ simple, in fact, that by default it doesn't persist its data to disk at all, so every time the sync_gateway process exits it loses all of its data! Great for unit testing, not great for any actual use. You can make your database persistent by changing the `-url` value to a path to an already-existing filesystem directory:

    mkdir /data
    sync_gateway -url /data

The gateway will now periodically save its state to a file `/data/sync_gateway.walrus`. This is by no means highly scalable, but it will work for casual use.

## Connecting with a real Couchbase server

Using a real Couchbase server, once you've got one set up, is as easy as changing the URL:

0. Install and start [Couchbase Server 2.0][COUCHBASE_SERVER] or later.
1. Create a bucket named `sync_gateway` in the default pool.
2. Start the sync gateway with an argument `-url` followed by the HTTP URL of the Couchbase server:

```
    sync_gateway -url http://localhost:8091
```

If you want to use a different name for the bucket, or listen on a different port, you can do that with command-line options. Use the `--help` flag to see a list of options.
