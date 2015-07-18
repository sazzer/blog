title: Optimistic Locking in MongoDB
date: 2015-07-18 10:31:31
tags:
- mongodb
category: programming
---
MongoDB has a lot of good things about it, and equally a lot of bad things about it. This is true of most database engines though, and you need to know what you're looking at to be able to make a good call over which system to use for what you're doing.

Optimistic locking is traditionally quite difficult to achieve. You need to ensure that the version in the database matches the version in the update request, and fail if that's not the case. There are a few ways of achieving this, but often with risks and race conditions involved.

<!-- more -->

MongoDB has the ability to execute a number of different atomic updates in a single statement, direct in the database. This can be used to our advantage here. Specifically here, we will be making use of the [$set](http://docs.mongodb.org/manual/reference/operator/update/set/), [$setOnInsert](http://docs.mongodb.org/manual/reference/operator/update/setOnInsert) and [$inc](http://docs.mongodb.org/manual/reference/operator/update/inc) operators for this.

### Check the version matches first
The first thing would be to ensure that the database contains the same version as the update request. This could be done by querying the database first,  as follows:

```
> db.test.find({"_id": "abcdef"})
{ "_id" : "abcdef", "name" : "Test", "description" : "Test Document", "version" : 1 }
> db.test.save({"_id": "abcdef", "version": 2, "name": "Test", "description": "Updated"});
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

If this returns anything then we can check that the database version matches. This works well enough, except for the risk of somebody else introducing a change to the data between us doing this check and us doing the update. We'll cover that in a bit

### Perform atomic updates instead of a complete rewrite
The above replaced the entire document with a new version of it. This works, but requries that the entire document is provided every time. Also, and more importantly here, it requires that all of the new fields are computed by the client code and not the database. We can do better than that, using the atomic updates that are supported. Instead of the above, we can do the following:
```
> db.test.update({"_id": "abcdef"}, {"$set": {"name": "Test", "description": "Updated", "version": 3}});
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

The two above statements do exactly the same. However, the update statement only sets the three fields specified - name, description and version - and doesn't touch any other fields at all. 

### Update the database only if the version matches
You'll notice that the above update statement starts with a query. We can use this to only update the database record if the version is the same. This means that we can avoid the check and race condition that goes along with it. Instead we ask the database to atomically update the record only if the version matches. This is simply done as follows:
```
> db.test.update({"_id": "abcdef", "version": 3}, {"$set": {"name": "Test", "description": "Updated", "version": 4}});
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

If, however, we do this and the version doesn't match then nothing gets updated:
```
> db.test.update({"_id": "abcdef", "version": 3}, {"$set": {"name": "Test", "description": "Updated", "version": 4}});
WriteResult({ "nMatched" : 0, "nUpserted" : 0, "nModified" : 0 })
```

### Update the version number in the database
Even with the above, we still need to know what the version number should be updated to before we can save it. This is not too bad in the case of the version number, but still if we can do it in the database then there are other benefits that we'll see later.
```
> db.test.update({"_id": "abcdef", "version": 4}, {"$set": {"name": "Test", "description": "Updated"}, "$inc": {"version": 1}});
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

In this case, we're incrementing the version number by 1. This can be whatever rate we want to use, but importantly as long as we use the same for every update then that part of the document never changes. We dynamically write the version number in the query but not in the update. This makes our client code that little bit simpler.

### But what if we're saving a new document?
Now, all this is well and good, but we can go better still. MongoDB supports what's called Upserting. This means that if the document to update doesn't exist then we can instruct it to create a new one in its place. This is where the incrementing of the version number really wins.

```
> db.test.update({"_id": "fedcba"}, {"$set": {"name": "Test", "description": "Updated"}, "$inc": {"version": 1}}, {"upsert": true});
WriteResult({ "nMatched" : 0, "nUpserted" : 1, "nModified" : 0, "_id" : "fedcba" })
```

You'll notice that there are only two differences between this command and the previous one. We added a third parameter telling MongoDB to perform an Upsert, and we didn't include a Version number in the query. Unfortunately, if you do include a version number - even with a value of null - then the "$inc" operator will fail on creating a new record. However, missing off that field altogether isn't exactly hard. But what has it actually done?
```
> db.test.find({"_id": "fedcba"});
{ "_id" : "fedcba", "name" : "Test", "description" : "Updated", "version" : 1 }
```

It's created a brand new document, with the ID that we queried for and a version number of 1. That's pretty much perfect. You will notice that it used the ID that we queried for. This means that the client code must be responsible for ID generation. That's not a huge problem though - just generate a UUID and you're done. If you miss the ID out of the query altogether then you'll instead update every single document in the database. 

If you want to continue using the traditional Save command to save a new document and have MongoDB generate an ID for you then that's also easy, but it does mean that Save and Create have different code paths. The above means that we can have a single code path for both. That's a huge win.

### How does this affect optimistic locking?

Now, given that Upserting means that we create a new document if the query doesn't find an existing one, how does this affect Optimistic Locking? If we provide an ID and Version that don't exist in the database, won't that just create a new document instead and cause problems? No. MongoDB handles this by the fact that the ID must be unique. Again, this only works if you are generating IDs yourself.

```
> db.test.update({"_id": "fedcba", "version": 1}, {"$set": {"name": "Test", "description": "Updated"}, "$inc": {"version": 1}}, {"upsert": true});
WriteResult({
        "nMatched" : 0,
        "nUpserted" : 0,
        "nModified" : 0,
        "writeError" : {
                "code" : 11000,
                "errmsg" : "E11000 duplicate key error index: test.test.$_id_ dup key: { : \"fedcba\" }"
        }
})
```

Now, this time the error was different. It did indeed try to create a new record, but it failed because the key was already in use. 

There is a problem with this, but it's a very minor one. It does highlight how important the key generation is though. If you try to save a record with no version number - because it's a new record - but with an ID that matches an existing one then you will update that existing one instead of creating a new one. This is potentially disasterous, but if your ID generation is unique enough - UUID has such a minimal chance of collisions that it's not worth worrying about - then you can manage. If that's not good enough then you may need to do something stricter - I can think of a few ID generation techniques that would work - or else use the traditional MongoDB mechanisms that will generate an ID for you on saving a new record.

### What about a Creation Date field?

Sometimes you want to store on your document some details that are set when the document is created but are not updated. For example, a Creation Date or an Owner ID. The above would seem to make this problematic, since creation and editing of the document are the same command now. Once again, Mongo makes this easy for us to solve, using the "$setOnInsert" command. This is identical to the "$set" command, but only if the document was being created anew. If it already existed then it gets ignored. As such, we can do the following:

```
> db.test.update({"_id": "123456"}, {"$set": {"name": "Test", "description": "Updated", "modified": new Date()}, "$setOnInsert": {"created": new Date()}, "$inc": {"version": 1}}, {"upsert": true});
WriteResult({ "nMatched" : 0, "nUpserted" : 1, "nModified" : 0, "_id" : "123456" })
> db.test.find({"_id": "123456"});
{ "_id" : "123456", "name" : "Test", "description" : "Updated", "modified" : ISODate("2015-07-18T11:16:55.818Z"), "created" : ISODate("2015-07-18T11:16:55.818Z"), "version" : 1 }
```

Which will create a brand new document, setting the modified and created dates to the current date and time. If we then do the following:

```
> db.test.update({"_id": "123456", "version": 1}, {"$set": {"name": "Test", "description": "Updated", "modified": new Date()}, "$setOnInsert": {"created": new Date()}, "$inc": {"version": 1}}, {"upsert": true});
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
> db.test.find({"_id": "123456"});
{ "_id" : "123456", "name" : "Test", "description" : "Updated", "modified" : ISODate("2015-07-18T11:17:15.732Z"), "created" : ISODate("2015-07-18T11:16:55.818Z"), "version" : 2 }
```

You'll notice that the modiified date has changed but that the created date hasn't. Again, one command for Create and Update and it just does The Right Thing.

### Conclusion

All in all, MongoDB makes it possible to harmonise Create and Edit of records in a single statement, with virtually no risk of it doing the wrong thing. It may not be the right way to handle every data set, but I suspect it will work well enough for most.