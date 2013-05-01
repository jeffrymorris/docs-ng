[COUCHBASE_LITE]: https://github.com/couchbase/couchbase-lite-ios
[TOUCHDB]: https://github.com/couchbaselabs/TouchDB-iOS
[COUCHDB]: http://couchdb.apache.org
[COUCHBASE_SERVER]: http://www.couchbase.com/couchbase-server/overview
[WALRUS]:https://github.com/couchbaselabs/walrus

# Couchbase Sync Gateway

Gluing [Couchbase Lite][COUCHBASE_LITE] (and [TouchDB][TOUCHDB] / [CouchDB][COUCHDB]) to [Couchbase Server][COUCHBASE_SERVER]


## About

This is an adapter that can allow [Couchbase Server 2][COUCHBASE_SERVER]* to act as a replication endpoint for [Couchbase Lite][COUCHBASE_LITE], [TouchDB][TOUCHDB], [CouchDB][COUCHDB], and other compatible libraries like PouchDB. It does this by running an HTTP listener process that speaks enough of CouchDB's REST API to serve as a passive endpoint of replication, and using a Couchbase bucket as the persistent storage of all the documents.

It also provides a mechanism called _channels_ that makes it feasible to share a database between a large number of users, with each user given access only to a subset of the database. This is a frequent use case for mobile apps, and one that doesn't work very well with CouchDB.

<small>\* _It can actually run without Couchbase Server, using a simple built-in database called [Walrus][WALRUS]. This is useful for testing or for very lightweight use. More details below._</small>

### Limitations

* Document IDs longer than about 180 characters will overflow Couchbase's key size limit and cause an HTTP error.
* Only a subset of the CouchDB REST API is supported: this is intentional. The gateway is _not_ a CouchDB replacement, rather a compatible sync endpoint.
* Explicit garbage collection is required to free up space, via a REST call to `/_vacuum`. This is not yet scheduled automatically, so you'll have to call it yourself.
* Performance is probably not that great. We'll be optimizing more later.

### License

Apache 2 license, like all Couchbase stuff.



