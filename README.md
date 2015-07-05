# CouchDB in a Hurry: for devs without time to relax

This guide is about breezing through the high-level questions a developer, possibly familiar with how other database systems are traditionally used in web development, would need answered before being able to understand the average guide describing how to do something using CouchDB (and consequently with PouchDB). It puts special focus on clearing up the bits that would appear to be contradictory, especially preconceptions based on the way things work in other databases.

I wrote it because other CouchDB guides tend to miss the middle ground between "recapping things you already know in excruciatingly boring detail" and "assuming you are already a CouchDB contributor" (especially when it comes to resolving apparent contradictions), which is especially frustrating when you care more about learning to use PouchDB than reading a page and a half explaining how `curl` does HTTP requests.

This guide does cover things that only matter to CouchDB, but mostly only in a sense so you can understand how PouchDB relates to CouchDB in production (specifically why using PouchDB works like your user is the only one in the world). I specifically note where things don't apply when working with PouchDB, so you can skip over them when you're just trying to first grok the CouchDB paradigm, if you don't care how this stuff is modeled outside of PouchDB.

I'm writing this as I learn CouchDB, so if something isn't clear in this document (like what the conventions are for naming databases), that's likely because I haven't learned it yet.

## A couple observations up front

CouchDB almost works more like a remote app than it does a traditional database. CouchDB is very much a model one designs an app around, rather than a system you use to model your app: if the CouchDB model doesn't match the way your app thinks of data, you should use a different system to express that model.

CouchDB is a natural model for apps where users have their own documents, which should be shared between locations (ie. across their own machines), with reasonably quick (~10 seconds) synchronization. This matches a lot of common app patterns (ie. most of what would traditionally be done with a "native" app on desktop or mobile), but there are many apps on the Internet that do not cleanly fit this model (especially entailing real-time interaction). For those apps, you're better off building your own API (possibly taking a few cues from CouchDB), backed by a datastore whose feature set would better allow you to express the app's model.

## CouchDB's data model

Like MongoDB or RethinkDB, CouchDB holds collections of JSON documents. CouchDB uses a few special fields on documents, with underscore-prefixed fixed names, for its own metadata, the values of some of which (`_id`) are more malleable/relevant for apps' use than others (`_rev`).

Documents are kept in *databases*, which are basically buckets. Databases in CouchDB are cheap (growing to keep hundreds of thousands of them in an app is not uncommon), and less resemble "databases" (in the sense that other systems usually use the word) than they do collections, tables, instances, or shards.

The way CouchDB models access/ownership, each user or group of users usually have their own database or databases (which is why you can create and replicate from a local PouchDB database to a remote CouchDB database without it getting weird).

Database names can be any string that matches the expression `^[a-z][a-z0-9_$()+/-]*$`, though using a database name with `/` in it will entail percent-escaping that slash in URLs for CouchDB's HTTP API (to distinguish it from the slash separating the database name from a document name).

Document IDs may be any arbitrary string, though the normal convention is to use unhyphenated hexadecimal random UUIDs (this is what PouchDB and CouchDB generate by default). Not using random IDs can lead the B-Tree which holds the documents of a database to grow unbalanced.

## CouchDB's interaction model

CouchDB works as a node you do create-read-update-delete operations with, that can then do those operations in turn when replicating data to other nodes. It does a bit of limited revision tracking to handle conflicts when receiving operations or replicating, which is what lets CouchDB be distributed and fault-tolerant (when two nodes disagree on something, CouchDB marks the conflict by setting a value in the document's `_conflict` field and stores both of the disagreeing histories until you resolve that conflict).

Data interactions match traditional REST semantics as closely as they can; replication is triggered via an RPC-like POST with a JSON body of named `source` and `target` arguments.

### Replicating

Both `source` and `target` may refer to either local or remote databases, meaning a CouchDB server can push its data to another server, pull data from another server, push data from one local database to another, or coordinate a replication between two remote databases (on one remote server or two).

Multiple databases on one CouchDB server have basically the same indepedent interaction paradigm as databases on remote CouchDB servers: the same stuff that can happen when replicating to, from, or between remote databases can happen when replicating to, from, or between local databases.

## CouchDB's administration / auth / user model

(This only applies in PouchDB when you're working with a remote CouchDB server.)

User accounts in CouchDB actually match up to end users in an app (unlike most other database systems you'd use in web development, in which first-class user accounts usually entail a single user for the app's server use for all operations).

Administrator accounts (which only get used when initializing your app's model) get saved in the server's `local.ini` file rather than the database; they can still be manipulated with REST requests, though after creating the first admin account, all further requests that require admin permissions (namely, further editing admin accounts / permissions) will require admin credentials (with no admin accounts specified, the server treats all requests as admin requests, in what is called "Admin Party" mode).

## Working with databases and documents in CouchDB's HTTP API

(The PouchDB JS API partially resembles the verbs of the CouchDB REST API, especially around document manipulation, which is why it helps to be familiar with the CouchDB API, although you can skip this and just read the next section if you only care about PouchDB.)

(I'm assuming you're already familiar with HTTP, how the GET, POST, and PUT verbs are supposed to be used, and how headers like `Accept` and `Content-Type` work.)

Creating and updating documents in CouchDB is done with PUT operations to the document's name (under the database name, ie. after the database name plus a slash) with JSON bodies (the name you pick will implicitly be the `_id` of the doc). When you create a document with PUT, you have to come up with a name first; this is why it's recommended (but not forced) that you use non-colliding random UUIDs to generate your document IDs.

(You can also POST the document to the *database's* route, in which case CouchDB will generate a UUID for you; however, since pre-generating your UUID means you're safe from inadvertently creating the document twice, it's recommeded that code pre-calculate UUIDs and use PUT for creation.)

You create databases in a similar fashion to the way you create documents, with a PUT to the database name (no body in the request, a status object for the response).

CouchDB's API returns JSON responses, though they only get returned as `application/json` if you make your request with an Accept header that matches it. (Otherwise it comes back as `text/plain`, mostly so, when debugging, making a request in a browser tab will display the response as text.)

## Working with databases and documents in PouchDB's JS API

(I'm assuming you're already familiar with the notion of asynchronous functions that either take a Node-style error-first callback or return a Promise, as seen in pretty much every async JS API designed since 2014.)

You create databases in PouchDB with `var db = new PouchDB('name-of-the-database')`. You create and update documents in that database with `db.put(doc)`, and get them with `db.get(docId)`.

## Attachments

CouchDB provides a mechanism for "attachments", keeping files alongside a document: they work essentially like the document is a directory in the database, and you push files inside that document-directory.

In general, though, [you're better off keeping your files somewhere else](http://pouchdb.com/2014/06/17/12-pro-tips-for-better-code-with-pouchdb.html).
