
# Channels: Access Control and Data Routing with the Sync Function

Channels are the intermediaries between documents and users. Every document belongs to a set of channels, and every user has a set of channels s/he is allowed to access. Additionally, a replication from Sync Gateway specifies what channels it wants to replicate; documents not in any of these channels will be ignored (even if the user has access to them.)

Thus, channels have three purposes:

1. Authorizing users to see documents;
2. Partitioning the data set;
3. Constraining the amount of data synced to (mobile) clients.

There is no need to register or preassign channels. Channels come into existence as documents are assigned to them. Channels with no documents assigned are considered empty.

Valid channel names consist of Unicode letter and digit characters, as well as "_", "-" and ".". The empty string is not allowed. The special meta-channel name "*" denotes all channels. Channel names are compared literally, so they are case- and diacritical-sensitive.

## Mapping documents to channels

There are currently two ways to assign documents to channels. Both of these operate implicitly: there's not a separate action that assigns a doc to a channel, rather the _contents_ of the document determine what channels its in.

### Explicit property

The default (simple and limited) way is to add a `channels` property to a document. Its value is an array of strings. The strings are the names of channels that this document will be available through. A document with no `channels` property will not appear in any channels.

### Sync function

The more flexible way is to define a **sync function**. This is a JavaScript function, similar to a CouchDB validation function, that takes a document body as input and can decide based on that what channels it should go into. Like a regular CouchDB function, it may not reference any external state and it must return the same results every time it's called on the same input.

The sync function is specified in the config file for your database. Each sync function applies to one database.

To add the current document to a channel, the function should call the special function `channel` which takes one or more channel names (or arrays of channel names) as arguments. For convenience, `channel` ignores `null` or `undefined` argument values.

Defining a sync function overrides the default channel mapping mechanism; that is, the document's `channels` property will be ignored. The default mechanism is equivalent to the following simple sync function:

    function (doc) { channel(doc.channels); }

## Replicating channels to Couchbase Lite, CouchDB or TouchDB

The basics are simple: When pulling from Sync Gateway using the CouchDB API, configure the replication to use a filter named `sync_gateway/bychannel`, and a filter parameter `channels` whose value is a comma-separated list of channels to fetch. The replication will now only pull documents tagged with those channels.

(Yes, this sounds just like CouchDB filtered replication. The difference is that the implementation is much, much more efficient because it uses a B-tree index to find the matching documents, rather than simply calling a JavaScript filter function on every single document.)

### Removal from channels

There is a tricky edge case of a document being "removed" from a channel without being deleted, i.e. when a new revision is not added to one or more channels that the previous revision was in. Subscribers (downstream databases pulling from this db) should know about this change, but it's not exactly the same as a deletion. CouchDB's existing filtered-replication mechanism does not address this, which has made things difficult for some clients.

Sync Gateway's `_changes` feed includes one more revision of a document after it stops matching a channel. It adds a `removed` property to the entry where this happens. (No client yet recognizes this property, though.) The value of `removed` is an array of strings, each string naming a channel this revision no longer appears in.

The effect on the client will be that after a replication it sees the next revision of the document, the one that causes it to no longer match the channel(s). It won't get any further revisions until the next one that makes the document match again.

This could seem weird ("why am I downloading documents I don't need?") but it ensures that any views running in the client will correctly no longer include the document, instead of including an obsolete revision. If the app code uses views to filter instead of just assuming that all docs in its local db must be relevant, it should be fine.

Note that in cases where the user's access to a channel is revoked, this will not remove documents from the user's device which are part of the revoked channels but have already been synced.




## Channels Authorization

The `all_channels` property of a user account determines what channels that user may access.

Any GET request to a document not assigned to one or more of the user's available channels will fail with a 403.

Accessing a `_changes` feed with any channel names that are not available to the user will fail with a 403.

Write protection is done by document validation functions, as in CouchDB: the function can look at the current user and determine whether that user should be able to make the change; if not, it throws an exception with a 403 Forbidden status code.

There is currently an edge case where after a user is granted access to a new channel, their client will not automatically sync with pre-existing documents in that channel (that they didn't have access to before.) The workaround is for the client to do a one-shot sync from only that new channel, which having no checkpoint will start over from the beginning and fetch all the old documents.

### Programmatic Authorization

It is possible for documents to grant users access to channels; this is done by writing a sync function that recognizes such documents and calls a special `access()` function to grant access.

`access() takes two parameters: the first is a user name or array of user names; the second is a channel name or array of channel names. For convenience, null values are ignored (treated as empty arrays.)

A typical example is a document representing a shared resource (like a chat room or photo gallery), which has a property like `members` that lists the users who should have access to that resource. If the documents belonging to that resource are all tagged with a specific channel, then a sync function can be used to detect the membership property and assign access to the users listed in it:

	function(doc) {
		if (doc.type == "chatroom") {
			access(doc.members, doc.channel_id)
		}
	}

In this example, a chat room is represented by a document with a "type" property "chatroom". The "channel_id" property names the associated channel (with which the actual chat messages will be tagged), and the "members" property lists the users who have access.

`access()` can also operate on roles: if a username string begins with `role:` then the remainder of the string is interpreted as a role name. (There's no ambiguity here, since ":" is an illegal character in a user or role name.)

## Validation: Authorizing Document Updates

As mentioned earlier, sync functions can also authorize document updates. Just like a CouchDB validation function, a sync function can reject the document by throwing an exception:

    throw({forbidden: "error message"})

A 403 Forbidden status and the given error string will be returned to the client.

To validate a document you often need to know which user is changing it, and sometimes you need to compare the old and new revisions. For those reasons the sync function actually takes up to three parameters, like a CouchDB validation function. To get access to the old revision and the user, declare it like this:

    function(doc, oldDoc, user) { ... }

`oldDoc` is the old revision of the document (or empty if this is a new document.) `user` is an object with properties `name` (the username), `roles` (an array of the names of the user's roles), and `channels` (an array of all channels the user has access to.)
