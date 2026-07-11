---
layout: post
title: "Yapping About Code Abstractions"
categories: programming c yapping
tags: programming c yapping
---

### Explicit abstractions over implicit abstractions, no need to hide implementations

Just like every thing in life, some abstractions are good, some are bad. 

One common mistake that I see people do ALL THE TIME is trying to over simplify, over abstract or 
over generalize things away from the end user. 

For example, imagine you are tasked to design a bespoke storage service for storing/retrieving a 
custom binary data block format that has a mixture of arbitrary text metadata and video binary data 
in it where it is normally quite large. Therefore, it is fine to read/write one data block at a time.

Some people would over complicate things and think, "We are using X as a backend right now, but what
if we change that in the future and store it to the cloud? What if we use the filesystem directly? 
What if...". Some other people might think, "The user should only concern about what they want to 
store or get, they shouldn't care what's going on under the hood because we are handling everything 
for them.". Either way, 

This would likely be what many people will come up with:

```c
struct DataBlock;
typedef struct DataBlock DataBlock;
struct Storage; //Called `Storage` because it _might_ not be a database
typedef struct Storage Storage;

bool SaveDataBlock(Storage* storage, DataBlock* dataBlock);
DataBlock* GetDataBlock(Storage* storage, const char* query);   //Returns NULL if not found
```

Seems good enough right? After all, this does everything the user needed.

<sub>_I am no database expert and here's what it could look like. But you don't have to read it._</sub>

```c
bool SaveDataBlock(Storage* storage, DataBlock* dataBlock)
{
    if(!storage || !dataBlock)
        return false;
    
    //A video should be at least 1 MB, otherwise something wrong
    if(GetVideoByteSize(dataBlock) < 1024 * 1024)
        return false;
    
    const char* binaryPath = WriteBinaryDataToDisk(dataBlock);  //Write to an unique path
    if(!binaryPath)
        return false;
    
    //Transaction is just a list of actions to be performed atomicly
    const char* transactionPath = StartTransaction();
    {
        if( !WriteMetadataToTransaction(dataBlock, binaryPath, transactionPath) ||
            //We need the database to not change before we do indexing and apply it
            !LockDatabase(storage, false /*is read?*/) ||
            !UpdateExistingIndexToTransaction(storage, query, transactionPath))
        {
            goto failed;
        }
        
        EndTransaction(transactionPath);
        if(!ApplyTransaction(storage, transactionPath))    //Atomic operation
            goto failed;
    }
    UnlockDatabase(storage);
    return true;
    
    failed:;
    UnlockDatabase(storage);
    DiscardTransaction(transactionPath);
    return false;
}
```

```c
DataBlock* GetDataBlock(Storage* storage, const char* query)
{
    if(!storage || !query)
        return NULL;
    
    if(!LockDatabase(storage, true /*is read?*/))
        return NULL;
    
    DataBlock* retDataBlock = NULL;
    
    //Update index if needed
    AttributesList attributesThatNeedIndexing = {0};
    if(!GetAttributesRequireIndexing(storage, query, &attributesThatNeedIndexing))
        goto cleanup;
    
    if(attributesThatNeedIndexing.Length > 0)
    {
        //We need to add new index to database
        UnlockDatabase(storage);
        
        //Relock it again with write access
        if(!LockDatabase(storage, false /*is read?*/))
            return NULL;
        
        //Refresh the list again since something could have changed between unlock and lock
        if(!GetAttributesRequireIndexing(storage, query, &attributesThatNeedIndexing))
            goto cleanup;
        
        const char* transactionPath = StartTransaction();
        if(!AddIndexToTransaction(storage, &attributesThatNeedIndexing, transactionPath))
        {
            DiscardTransaction(transactionPath);
            goto cleanup;
        }
        EndTransaction(transactionPath);
        if(!ApplyTransaction(storage, transactionPath))    //Atomic operation
        {
            DiscardTransaction(transactionPath);
            goto cleanup;
        }
        
        UnlockDatabase(storage);
        //Relock it again with read access
        if(!LockDatabase(storage, true /*is read?*/))
            return NULL;
    }
    
    Node entry = QueryNextEntry(storage, q, NULL); //Find the first entry
    if(entry.Valid)
        retDataBlock = getDataBlockFromNode(&entry);
    
    cleanup:;
    UnlockDatabase(storage);
    return retDataBlock;
}
```

Just like what they were designed to, most of the inner workings such as handling multi-read, 
single write locking, validating video size (> 1MB), adding/updating indexes, atomic udpates, etc...
are all hidden away from the user.

Additionally, it is very common for all of the struct pointers to be just opaque pointers and the 
database functions not exposed at all as well. Therefore the user doesn't even know if it is a 
database.

This interface works fine for simple use case and a few users that perform read most of the time.

... Except **it isn't**. Because the users have no idea how it works under the hood, they are likely 
to query with a **generous amount** of attributes filters, resulting the need of creating many 
indexes in the database

This means the size and processing time explodes as the number of data blocks increases.

Also, if you think about the requirement for a bit longer, you might have noticed something odd. 

> [...] design a bespoke storage service for **storing/retrieving** a custom binary data block format [...]

Unsurprisingly, the requirement **forgot** to mention the ability to **remove** a stored data. 
And they need it quick because some users started to create data blocks by mistake and other users 
are getting these wrong data blocks.

They _could_ get around this by creating a dummy data block with a custom metadata attribute that 
flags the stale data blocks, like this:

```yaml
Metadata:
    UUID: 123
    ...
    Staled_UUID: 345
Binary: 
    <video data ...>
```

And check if the data block we got is staled or not by doing something like
```c
DataBlock* queriedDataBlock = GetDataBlock(storage, "UUID == 345");
if(queriedDataBlock)
{
    DataBlock* staled = GetDataBlock(storage, "Staled_UUID == 345");
    if(staled)
        return ...;
}
```

While this works (although not ideal), it will be slow everytime we mark/check a data block is 
staled because...

> [...] most of the inner workings such as [...] **validating video size (> 1MB)** [...] are all 
> hidden away from the user.

Meaning when we create a dummy data block, we will need to satisfy the video size to make it "valid".

So each time we want to mark something as staled, we are storing 1 MB of dummy video data and we are 
reading & copying 1 MB of dummy video data each time we want to check a given data block is staled or 
not.

You might say these all seem a bit artificial and cherry picked, but this happens all the time when 
the stated requirement deviate from the expectation for various reasons.

There's more: 

1. Now what if there's an user who wants to store/get many data blocks with a short video? The 
locking and updating of the indexes each time will slow the rest of the users to a halt.
    - **Let's create a batch specify version of the API functions which operates on multiple data 
    blocks at a time.**
2. Okay. Now turns out all of our users so far are storing MP4 (compressed) videos, and a new user 
wants to store an AVI video (less compressed) instead. The moment we are trying to store/retrieve 
this video, it blocks the rest of the user because it is so much larger than a normal video.
    - **Let's create a multi-part version of the API functions and spread these multi-part functions 
    so that it doesn't block the rest of the users.**
3. Now what if there's a write heavy/frequent user? Each time there's a write, it needs to lock the 
whole database. This means the no reads can be done when there's a write each time and therefore the 
read requests will pile up.
    - **Let's adjust the internal implementation to group reads and writes at a time for non critical 
    requests.**

All of this leads to my point of, requirements change all the time, niche cases happen all the time. 
Each time any of these happen, the API needs to be changed, the implementation needs to be changed.

> #### The user is powerless because everything is abstracted, hidden away.

So, how should it be fixed?

To approach this question, first we need to understand that people **RARELY** get their public API 
right the first time. There's always something missing and there's always some niche use cases that 
we will never think of.

So instead of hiding everything away from the user, we should be exposing the internal functions as 
much as possible alongside the main, intended public API, as long as the internal ones  are clearly 
labeled so.

They don't **have** to be API/ABI stable because they are intended as internal functions, but 
reserving some fields in a struct and a handful of parameters goes a long way, even for main public 
API functions, not just the internal ones.

I will actually go one step further and say the user should always know and have access to internal 
functions as much as possible. 

What you **REALLY** want is to treat the "original" public API functions as "helper functions" or 
"default options" to the "lower level" functions. Like this:

```c
typedef struct DataBlock
{
    //Internal Fields...
} DataBlock;

typedef struct Database
{
    //Internal Fields...
} Database;

typedef struct Options
{
    bool LockDatabase;
    bool UpdateIndexes;
    ...
    uint8_t Reserves[32];
} Options;

//Public API Function
Options CreateDefaultOptions(void);     //Returns a default options object that fit the main use case
bool SaveDataBlock(Database* db, DataBlock* dataBlock, Options options);
DataBlock* GetDataBlock(Database* db, const char* query, Options options);   //Returns NULL if not found
    
//Internal "Low Level" Functions
bool LockDatabase(Database* db, bool readOnly);
void UnlockDatabse(Database* db)
//...
//Transaction functions...
//Database functions...
//...
```

Basically, for the public API functions, the default options should be enough for the majority of 
the use cases. 

The `Options` struct along with the "low level" functions should give the users pretty good idea on 
what the public API functions are doing under the hood, should they care.

If the default options do not satisfy the user's use case, they have the ability to tinker with 
various options to see if that solves their problem.

If even that is not enough, the user has access to "low level" functions that they can use for their 
very niche use case, in which case they should be aware that internal functions are subject to change 
in the future.
