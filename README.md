TinySync
========

TinySync is a set of libraries and protocol convention to persist and synchronize data between mobile applications and a centralized server.


## Libraries

There are currently two libraries supporting TinySync functionality: a native Android library for client-side storage and a Ruby library for server-side storage.

[TinySync Android Client Library](https://github.com/TinyMission/tinysync-android)

[TinySync Ruby Server Library](https://github.com/TinyMission/tinysync-ruby)


## Architecture

### Sync Requests and Responses

The basic mechanism of synchronizing data between the client and server in TinySync can be summarized as:

1. The client sends a *sync request* to the server containing all entities that have changed on the client since the last time it synced. They separated by whether they are new, existing but updated, or deleted.
2. The server updates its database based on the updated entities in the *sync request*
3. The server returns a *sync response* that contains all entities that have changed on the server since the *last_synced* value in the *sync request*.
4. The client updates its database based on the updated entities in the *sync response*.

The *sync request* and *sync response* are JSON payloads with the same general form:

```javascript
    {
        "last_synced": "2014-02-14T09:12:43-0700",
        "entities": [
            {
                "name": "post",
                "scope": {"author_id": "52212589594cc44541000016"},
                "created": [
                    {
                        "id": "521fa720594cc48c1d000003",
                        "author_id": "52212589594cc44541000016",
                        "updated_at": "2014-02-17T17:23:54-0700",
                        "body": "Some text in a new post..."
                    },
                    ...
                ],
                "updated": [
                    {
                        "id": "521fa720594cc48c1d000016",
                        "author_id": "52212589594cc44541000016",
                        "updated_at": "2014-02-16T12:05:24-0700",
                        "body": "Some new text in an existing post..."
                    },
                    ...
                ],
                "deleted": [
                  "521fa720594cc48c1d000016", ...
                ]
            },
            ...
        ]
    }
```

The *sync response* format is identical to the *sync request* format except for the addition of two more fields in each entity object:

* saved: a list ids of objects that were sent in the *sync request* and successfully saved to the server's database
* errors: a list of errors that occurred while saving objects sent in the *sync request*
 
So, an example *sync response* will look something like this:

```javascript
    {
        "last_synced": "2014-02-14T09:12:43-0700",
        "entities": [
            {
                "name": "post",
                "scope": {"author_id": "52212589594cc44541000016"},
                "created": [
                    {
                        "id": "521fa720594cc48c1d000042",
                        "author_id": "52212589594cc44541000016",
                        "updated_at": "2014-03-02T18:24:54-0800",
                        "body": "A post that was saved on the server"
                    },
                    ...
                ],
                "updated": [
                    ...
                ],
                "deleted": [
                    ...
                ],
                "saved": [
                  "521fa720594cc48c1d000003", "521fa720594cc48c1d000016"
                ],
                "errors": [
                  {
                    "id": "521fa720594cc48c1d000013",
                    "field": "body",
                    "message": "Body must not be empty"
                  }
                ]
            },
            ...
        ]
    }
```

The client library will automatically mark the records in *saved* as successfully synced. 
It is up to the individual client implementations to deal with the *errors* returned from the server.


### Sync Scopes

When a client sends a *sync request* to the server, each root entity it wants to sync needs a corresponding *sync scope*.
The *sync scope* is a JSON object that defines a query used to limit the updated entities sent back in the *sync response*.

In the example above, the *sync scope* for the *post* entity is `{"author_id": "52212589594cc44541000016"}`.
In this case, the client in question is only interested in storing posts with an author_id of "52212589594cc44541000016".
Only posts matching that query will be returned in the *sync response*.

Clients can leave the *sync scope* null and the server will return all updated values of that entity.
However, it is assumed that for a non-trivial system, it will be impractical or impossible to sync the entire server dataset to each client.
Designing an appropriate *sync scope* is the key to having a system that syncs quickly with low client-side memory footprint.

NOTE: The *last_synced* time from the request will be automatically included into the scope to test against *updated_at* in the database.


### IDs

All records have a unique identifier that follows the [BSON ObjectId](http://api.mongodb.org/java/current/org/bson/types/ObjectId.html) format. 
The actual ID values may be persisted in different formats depending on the database being used, but they are always serialized as 24-character hex-encoded strings.

This format ensures that all IDs, whether generated on the client or server, will be globally unique.


### Dates/Times

Date and time values are serialized using the [ISO 8601](http://en.wikipedia.org/wiki/ISO_8601) standard string format.
