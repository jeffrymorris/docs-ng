
# Authentication & Authorization

Sync Gateway supports user accounts that are allowed to access only a subset of channels (and hence documents).

## Accounts

Accounts are managed through a parallel REST interface that runs on port 4985 (by default, but this can be customized via the `authaddr` command-line argument). This interface is privileged and for internal use only; instead, we assume you have some other server-side mechanism for users to manage accounts, which will call through to this API.

The URL for a user account is simply "/_database_/user/_name_" where _name_ is the username and _database_ is the configured name of the database. The typical GET, PUT and DELETE methods apply. The contents of the resource are a JSON object with the properties:

* "name": The user name (same as in the URL path). Names must consist of alphanumeric ASCII characters or underscores.
* "admin_channels": An array of strings -- the channels that the user is granted access to by the administrator. The name "*" means "all channels". An empty array or missing property denies access to all channels.
* "all_channels": Like "admin_channels" but also includes channels the user is given access to by other documents via a sync function. (This is a derived property and changes to it will be ignored.)
* "roles": An optional array of strings -- the roles (q.v.) the user belongs to.
* "password": In a PUT or POST request, put the user's password here. It will not be returned by a GET.
* "passwordhash": Securely hashed version of the password. This will be returned from a GET. If you want to update a user without changing the password, leave this alone when sending the modified JSON object back through PUT.
* "disabled": Normally missing; if set to `true`, disables access for that account.
* "email": The user's email address. Optional, but Persona login (q.v.) needs it.

You can create a new user either with a PUT to its URL, or by POST to `/$DB/user/`.

There is a special account named `GUEST` that applies to unauthenticated requests. Any request to the public API that does not have an `Authorization` header is treated as the `GUEST` user. The default `admin_channels` property of the guest user is `["*"]`, which gives access to all channels. In other words, it's the equivalent of CouchDB's "admin party". If you want any channels to be read-protected, you'll need to change this first.

To disable all guest access, set the guest user's `disabled` property:

    curl -X PUT localhost:4985/$DB/user/GUEST --data '{"disabled":true, "channels":[]}'

## Roles

A user account can be assigned to zero or more _roles_. Roles are simply named collections of channels; a user inherits the channel access of all roles it belongs to. This is very much like CouchDB; or like Unix groups, except that roles do not form a hierarchy.

Roles are accessed through the admin REST API much like users are, through URLs of the form "/_database_/role/_name_". Role resources have a subset of the properties that users do: `name`, `admin_channels`, `all_channels`.

Roles have a separate namespace from users, so it's legal to have a user and a role with the same name.

## Authentication

[Like CouchDB](http://wiki.apache.org/couchdb/Session_API), Sync Gateway allows clients to authenticate using either HTTP Basic Auth or cookie-based sessions. The only difference is that we've moved the session URL from `/_session` to `/dbname/_session`, because in the Sync Gateway user accounts are per-database, not global to the server.

### Persona

Sync Gateway also supports [Mozilla's Persona (aka BrowserID) protocol](https://developer.mozilla.org/en-US/docs/persona) that allows users to log in using their email addresses.

To enable it, you need to add a top-level `persona` property to your server config file. Its value should be an object with an `origin` property containing your server's canonical root URL, like so:

    {
        "persona": { "origin": "http://example.com/" },
        ...

Alternatively, you can use a `-personaOrigin` command-line flag whose value is the origin URL.

(This is necessary because the Persona protocol requires both client and server to agree on the server's ID, and there's no reliable way to derive the URL on the server, especially if it's behind a proxy.)

Once that's set up, you need to set the `email` property of user accounts, so that the server can determine the account based on the email address.

Clients log in by POSTing to `/dbname/_persona`, with a JSON body containing an `assertion` property whose value is the signed assertion received from the identity provider. Just as with a `_session` login, the response will set a session cookie.

### Indirect Authentication

An app server can also create a session for a user by POSTing to `/dbname/_session` on the privileged port-4985 REST interface. The request body should be a JSON document with two properties: `name` (the user name) and `ttl` time-to-live measured in seconds. The response will be a JSON document with properties `cookie_name` (the name of the session cookie the client should send), `session_id` (the value of the session cookie) and `expires` (the time the session expires).

This allows the app server to optionally do its own authentication: the client sends credentials to it, and then it uses this API to generate a login session and send the cookie data back to the client, which can then send it in requests to Sync Gateway.


