---
layout: post
title:  "Retrieving and updating records without locks in MongoDB"
date:   2025-03-11
categories: post
---

This is how I process a bunch of records.
```
ObjectId dispatchId = ObjectId.GenerateNewId();
await collection.UpdateManyAsync(
    Builders<TDocument>.Filter.Eq(x => x.IsProcessed, false),
    Builders<TDocument>.Update
        .Set(x => x.IsProcessed, true)
        .Set(x => x.DispatchId, dispatchId));

List<TDocument> toProcess = await collection.Find(Builders<TDocument>.Filter.Eq(x => x.DispatchId, dispatchId))
    .ToListAsync();

try
{
    ... doing some processing
} catch (Exception ex)
{
    await collection.UpdateManyAsync(
        Builders<TDocument>.Filter.Eq(x => x.Id, dispatchId),
        Builders<TDocument>.Update
            .Set(x => x.DispatchId, null)
            .Set(x => x.IsProcessed, false));
    throw;
}
```

Why? Because I don't need to implement my own locks and I don't have to worry about concurrent operations affecting the output.
The dispatch ID is unique to the operation, meaning that the documents retrieved are unique to the operation.

*Also, MongoDB's ObjectIds are seeded from the current time, so they will always be unique, and the document now contains when the operation was performed.*

---

\
I could do something like this:

```
List<TDocument> toProcess = await collection.Find(Builders<TDocument>.Filter.Eq(x => x.IsProcessed, false))
    .ToListAsync();

... doing some processing

await collection.UpdateManyAsync(
    Builders<TDocument>.Filter.In(x => x.Id, toProcess.Select(x => x.Id)),
    Builders<TDocument>.Update.Set(x => x.IsProcessed, true));

```

The problem with this is that if another concurrent operation is processing records, the other operation could affect the records between the retrieval and update, which could cause a write failure or data loss.

---

\
What about using `findOneAndUpdate` ?

`findOneAndUpdate` updates a document and returns the document in one operation. Here is another solution to the problem:

```
TDocument? doc = null;
List<TDocument> toProcess = [];
do
{
    doc = await collection.FindOneAndUpdateAsync(
        Builders<TDocument>.Filter.Eq(x => x.IsProcessed, false),
        Builders<TDocument>.Update.Set(x => x.IsProcessed, true));
    if (doc != null)
    {
        toProcess.Add(doc);
    }
}
while (doc != null);

try
{
    ... doing some processing
} catch (Exception ex)
{
    await collection.UpdateManyAsync(
        Builders<TDocument>.Filter.In(x => x.Id, toProcess.Select(x => x.Id)),
        Builders<TDocument>.Update
            .Set(x => x.IsProcessed, false));
    throw;
}
```

The catch is this from [the MongoDB docs](https://www.mongodb.com/docs/manual/reference/method/db.collection.findOneAndUpdate/#mongodb-method-db.collection.findOneAndUpdate)

> Retryable writes require the findOneAndUpdate() method to copy the entire document into a special side collection for each node in a replica set before it performs the update. This can make findOneAndUpdate() an expensive operation when dealing with large documents or large replica sets.

**everything above uses MongoDB.Driver 3.2.1**