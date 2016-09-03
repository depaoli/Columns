---
title: DynamoDB Overview
author: Matteo De Paoli
published: 2015-08-24
updated: 2015-08-24
tag: [amazon, amzn, aws, cloud, database, dbms, dynamo, dynamodb, hash, hash and range, index, innodb, no-sql, nosql, primary key, secondary key, ssd]
headline: Overview of one of the most important NoSQL Cloud Based Database, analysing distinctive traits, pros and cons
image: https://goo.gl/IRTsmL
lead: This is just a small overview for the users who want to start using DynamoDB and who need to figure out the overall picture. Particularly, we'll look at it under the hood, in order to understand how the provisioned throughput (ie. "Capacity Units") must be evaluated, and the consequent performance of the different requests that can be performed against DynamoDB. We'll also give an overview of the features that can be expected by this NoSQL, key-value storage system.

---

##Abstract

Until few years ago, "database" word was considered synonymous of [SQL](https://en.wikipedia.org/?title=SQL "With a Title"), with just a few people who were considering their filesystem or their corporate file server a  database as well (ie. [Hierarchical Database](https://en.wikipedia.org/wiki/Hierarchical_database_model)).
The word "database" was associated with the black box containing all the structured data of the company, and excluding the directory services like [LDAP](https://en.wikipedia.org/wiki/Lightweight_Directory_Access_Protocol) (again, a Hierarchical Model Database), the market offered Relational Database solutions mostly, in form of a [DataBase Management System (DBMS)](https://en.wikipedia.org/wiki/Database) software that replies to the queries written in the [Structured Query Language (SQL)](https://en.wikipedia.org/wiki/SQL). 

SQL became the standard, and even if every vendor added its own extensions and advanced features, you can expect standard basic features and a pretty common language among the different SQL Dialects produced by different vendors (eg. Sun/Oracle, Microsoft, PostgreSQL, etc.).

These databases use the [Relational Model](https://en.wikipedia.org/wiki/Relational_database) (at minimum, because some of them use the [concept of object also](https://en.wikipedia.org/wiki/Object-relational_database)), that through **SQL language allows very rich and powerful queries, along with great reliability and security for the data** stored in them (thanks also to the [ACID transactions](https://en.wikipedia.org/wiki/ACID) that are usually implemented by [RDBMS](https://en.wikipedia.org/wiki/Relational_database_management_system)).
The design of these databases (eg. the tables and the columns defined) is mostly driven by the concepts of ["entity"](https://en.wikipedia.org/wiki/Entity) (every "table" represent an "entity" as rule of thumb) and "foreign key", [foreign keys](https://en.wikipedia.org/wiki/Foreign_key) that are used to link rows in the same or in a different table with each others, making the relations and creating more complex "entities", described by a new logical table/view created by combining/merging the related items. These concepts are the foundations for the rich queries, with their [union](https://en.wikipedia.org/wiki/Set_operations_(SQL)#UNION_operator), [join](https://en.wikipedia.org/wiki/Join_(SQL)), etc.

***[Amazon DynamoDB](https://en.wikipedia.org/wiki/Amazon_DynamoDB) is a Cloud Based, [NoSQL](https://en.wikipedia.org/wiki/NoSQL) database, designed to provide highly-available [key-value](https://en.wikipedia.org/wiki/Key-value_database) storage system.***

Forget about the the foreign keys, about the complex entities they create, and about operations that merge different data through the join, just to name a few; in other words, please forget any kind of advanced and super-rich query language.
At the very essence, **if you want to retrive the data/value from DynamoDB you only have to ask for the related key.**
For these reasons, the way used to design NoSQL tables must be different compared to the [Relational Database](https://en.wikipedia.org/wiki/Relational_database); also you shouldn't expect the same reliability and security features.

Furthermore, please **don't expect any standard!**
With SQL you can write the same query against every [RDBMS](https://en.wikipedia.org/wiki/Relational_database_management_system) expecting a valid result (unless the use of advance features and extensions from the specific SQL Dialect).
With NoSQL the story is different. Every platform uses its own standard, often an API that provides an interface for gathering [JSON](https://en.wikipedia.org/wiki/JSON) or [XML](https://en.wikipedia.org/wiki/XML) data from a remote server through [HTTP/s](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol). Indeed, NoSQL is mostly oriented to software developers who can then create the [front end](https://en.wikipedia.org/wiki/Front_and_back_ends) software that will be used by the end users in order to interact with the database (on edge cases, even when end users are analysts who would interact through regular SQL).

In general, every NoSQL implementation can decide to implement its own IP protocol, interface, data structure, etc.

On a side note, Amazon DynamoDB is based on the [Dynamo storage techniques and principles](https://en.wikipedia.org/wiki/Dynamo_(storage_system)) (a great paper about **internal Amazon Dynamo implementation from which DynamoDB was born is [here](http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)**); anyway don't think that these techniques mean that there is a standard.

About "cloud based" propriety, it's because, even if there is the possibility to run its reduced version locally for testing purposes, it is totally managed by Amazon; it's a typical Software as a Service.
It's a **fully managed database service**, and you just need to design your tables, define the traffic you expect, and set up a connection between it and your client/s.

>**NOTE**: On a side note, Amazon provides also [RDS](https://en.wikipedia.org/wiki/Amazon_Relational_Database_Service), that provides the possibility to run the most common relation databases in its cloud. Anyway, with DynamoDB, Amazon goes further by removing any kind of management need, not just the hardware of the server or the [OS](https://en.wikipedia.org/wiki/Operating_system), but also the [RDBMS](https://en.wikipedia.org/wiki/Relational_database_management_system) and its configurations and optimisations.

So, why NoSQL? Is SQL bad? Why do we need NoSQL after many years of great results with SQL?

![SQL vs. NoSQL](http://2.bp.blogspot.com/-0M-UnzLH40k/VjisOe7HdII/AAAAAAAALjo/XMNTbvq5Xug/s320/sqlvsno.jpg){: .center-block style="width: 50%;"}

In a brief: because **NoSQL DB are lighter and simpler, meaning great scalability, fast performance and lower cost.**

In DynamoDB, there is a tradeoff of consistency (the "C" of ACID) for availability, indeed **the DB is "[eventually consistent](https://en.wikipedia.org/wiki/Eventual_consistency)" but this way it should never deny a request that is within the configured throughput** (anyway, contrary to [the internal Dynamo implementation](http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) mentioned above, customer should not face the manual conflict resolution and other administrative tasks like [Quorum](https://en.wikipedia.org/wiki/Quorum_(distributed_computing)) management).
**With RDBMS there are some scalability and availability problems**, and it's pretty well accepted that **strong consistency and high data availability cannot be achieved simultaneously** (Ref. [CAP Theorem](https://en.wikipedia.org/wiki/CAP_theorem)).
For instance, because RDBMS enforce strict consistency, a request could be delayed or even denied if  for any reasons it's not replicated to all the nodes within a specific time, reducing the availability of the system.
**In situation where the complex queries, relations and features of RDBMS are not needed** (like when storing the state information of a [stateful service](https://en.wikipedia.org/wiki/Service_(systems_architecture)#Service_implementation)) **a simple mechanism for retrieving data from a primary key is just more than enough.**

Overall, it's a common belief that **NoSQL won't replace SQL databases, rather, NoSQL are now an option to consider when choosing a database model to use**, with its pros and its cons.


##DynamoDB Key Points

Amazon DynamoDB is a fully managed NoSQL database accessible through a web service that uses HTTPS for transport and JSON for message serialization.
It works with **Key-Value tables** and being NoSQL, it's **schema-less** and it doesn't support transactions and JOIN (more on the "DynamoDB Tables and Keys" section below).

It provides **fast and predictable performance, great scalability and availability**; it also permits capacity addition without impact on the performance and without downtime.

**DynamoDB born from the Amazon experience with the internal Dynamo system** (used for the complex management of the shopping cart and for other core services) **and from the understanding of the customer need for a system that is easy to manage, like SimpleDB** (more [here](http://www.allthingsdistributed.com/2012/01/amazon-dynamodb.html)); it's goal is to allow applications to have:

- A system that just works: no administrative tasks or decisions.
- Flexibility and elasticity:
    - no need to update any schema when the system evolves, and simple interface/queries for application's building blocks based on simple tables.
    - write and read throughput (ie. Capacity Units) can be easily added or removed without impacting the performance and without any change on the code of the application, in a fast and easy manner. Change can be performed through the WebInterface or even automatically through an API and CloudWatch.
- Infinitely scale capacity in throughput and in table size ([mostly](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Limits.html)).
- Predictable performance: the numbers of read and writes per minute is guaranteed (more about the proper calculation below on this columns).

These characteristics are perfect for applications like games, for instance, that could require more capacity suddenly (eg. a viral app that fast increases the throughput requirements).

Finally, worth to note that DynamoDB is **priced on the amount of queries**, and not on the amount of data stored.


##DynamoDB System Architecture and Performance


###The fleet

DynamoDB has a 3 way [synchronous replication](http://searchdisasterrecovery.techtarget.com/definition/synchronous-replication) across different [AWS Availability Zones (AZ)](https://en.wikipedia.org/wiki/Amazon_Web_Services#Architecture), with write confirmation when data are replicated on 2 AZ at least (the 3rd one will reach the consistency eventually).
DynamoDB is based on a [Three-tier architecture](https://en.wikipedia.org/wiki/Multitier_architecture#Three-tier_architecture), with a LoadBalancer, a FrontEnd and a BackEnd (Storage Node).
Right now, storage engine is [InnoDB](https://en.wikipedia.org/wiki/InnoDB) like, and data are saved on [SSD](https://en.wikipedia.org/wiki/Solid-state_drive) drives (permitting very predictable performance).


###The characteristics

It's a **key-value storage system** (a structure called dictionary, hash, associative array, etc. in different languages and environment); so, **almost all the operations you can perform on DynamoDB data (ie. items) require that you provide the primary key** (eg. you can't select all the elements  in the table with attribute "color == red" as easily and efficiently as with RDBMS).

Also, as stated before, DynamoDB does **not support cross-table joins, as it doesn't support operations that span multiple data items related between each others**. 

The rows of a DynamoDB table are called "**items**", and the "items" contain "**attributes**", and, except for the **mandatory Primary Key**, Dynamo DB tables are **schema-less** (ie. each item can have an arbitrary set of attributes and you can add new attributes to an existing table by simply storing them along with a key). Also, it is possible to create **Secondary Keys** (somewhat comparable to [Alternate Key](https://en.wikipedia.org/wiki/Alternate_key) in SQL), but we'll look deeper into this later on on this column.

In other words, **tables of a DynamoDB are independent** (eg. there are no [foreign keys](https://en.wikipedia.org/wiki/Foreign_key)) and **formed by items** that are **indexed by the Primary Key** and that are **composed of any number of attributes** (with a [limit of 400 KB on the item size](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Limits.html)).

You have **only one Primary Key**, and **you can access an item only by its Primary Key** (or, more recently, by a Secondary Key if created).


###The implementation

**Index is sorted and partitioned in multiple chunks, based on ranges of the primary key**.

![DynamoDB Table](http://www.allthingsdistributed.com/images/gsi1.png){: .center-block style="width: 80%;"}
***(Image courtesy of http://www.allthingsdistributed.com/2013/12/dynamodb-global-secondary-indexes.html)***

This way, when asking for a key, it's possible to know in which partition the key will be (if it exists).
This means that, if the queries against your table are well spread between partitions, the reads are parallel by design.
**Partitions are totally independent by each others**, and can be stored on different machines.
This allows to scale the size of the tables infinitely, and this means also that, **the more partitions you have, the more throughput you get** (we'll look deeper into this later on).
Indeed, thanks also to SSD and to extensive caching mechanisms, **every machine has a very constant throughput**, and the machine can be logically divided in slices, where every slice allows its portion of the total throughput (ie. slice performance = total machine performance / number of slices).


###The behavior

**If a user configures a DynamoDB table with more throughput than the amount available for a single slice, the table will be partitioned and spread in a sufficient number of slices in order to obtain the desired total throughput**.
DynamoDB tables are automatically partitioned by the system, there is no user control or knowledge of this.

In order to **avoid the [noisy neighbor behaviour](http://searchcloudcomputing.techtarget.com/definition/noisy-neighbor-cloud-computing-performance)**, DynamoDB uses a proactive approach rather than a reactive one, that is based on the **strict throughput reservation** that the customer configures.
This is one of the reason because in the first implementation of DynamoDB, **bursts of capacity above the reserved throughput were not allowed**; even if the customer used less throughput in the past time, exceeding traffic would be rejected usually, otherwise there would be no guarantee that the slices of the neighbors will be able to obtain the 100% of their expected and provisioned throughput.
More **recently, [DynamoDB may allow limited burst but only for short periods](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GuidelinesForTables.html#GuidelinesForTables.Bursting)**.


###The performance

As we seen, the **provisioning throughput model is very tied with the partition and allocation policy**.

The independent NoSQL table model that allows independent partitions, the SSD drives, the caching mechanisms, the rejection of almost all exceeding traffic, and making no use of overbooking/overprovisioning allow very **solid guarantee of predictable performance** for the users.

Anyway, because of its architecture, Amazon DynamoDB cannot always guarantee that the configured throughput levels (ie. Capacity Units) are achieved, because the configured value is an average.
**In order to reach the declared throughput, the workload must be well spread in space and time**.
Well spread in time because, exceeding traffic will always be rejected; low usage followed by **burst of traffic will result in lower throughput** consequently (similarly to the [policing of network traffic](http://get.matteo.tips/2014/08/quality-of-service.html)).
Well spread in space because, **if Hot Key/s exist, and the requests against the keys inside a single slice exceed the throughput of that slice, these requests will be dropped** (even if other slices used by the same table are unused).

Also, because DynamoDB targets high performances and so it makes use of extensive caching, when request rate is too low it's possible to observe lower performance in the first queries (till the caches are populated).

More tips on [this guideline](https://aws.amazon.com/blogs/aws/optimizing-provisioned-throughput-in-amazon-dynamodb/).


##DynamoDB Tables and Keys

In DynamoDB **there is no hierarchy**; no concepts of multiple databases inside the same instance like in MySQL, or folders or anything like that.
From your AWS account you can create just tables, organised by AWS Regions (ie. you can have multiple tables named 'Foo' in different Regions and they're totally independent, but only one 'Foo' table for each Region would be allowed), but there is no way to set up a folder/database for a specific customer in order to put all its tables in it; anyway, with the powerful API of DynamoDB you can easily implement an automated way to create multiple tables for every customer, for instance:

    :::python
    from boto.dynamodb2 import connect_to_region
    from boto.dynamodb2.table import Table

    users_table = Table(user+'_'+table_name,  # <---tables can be grouped by customer this way
                            connection = connect_to_region(db_region,
                            aws_access_key_id=id,
                            aws_secret_access_key=key))

Now, let's expand "The characteristics" section above, starting by listing the mentioned key points:

- Key-Value tables
    - Mandatory Primary Key
    - Optional Secondary Key
    - Schema-less
    - Rows are called "items"
        - "items" contain "attributes" (columns values), and can be accessed only by Primary or Secondary Keys
- No support for cross-table joins

This is an example of a DynamoDB table named "test":

![DynamoDB Table Example](http://1.bp.blogspot.com/-tAtyyX1iE2k/VjiefNzeL4I/AAAAAAAALjY/Q0Ao9ActhRg/s640/Table%2Bexample.png){: .center-block style="width: 100%;"}

In this test table we can see that items have independent attributes; for instance, the item '03' has no attributes 'name' and  'surname'.

Looking at the items, the JSON representation for the 2nd element is:

    :::json
    {
      "city": {
        "S": "Cordenons"
      },
      "id": {
        "N": "2"
      },
      "name": {
        "S": "Matteo"
      },
      "surname": {
        "S": "De Paoli"
      }
    }

>**NOTE**: the primary key attributes must be of type String, Number, or Binary.


###Operations

DynamoDB provides control operations to [create](http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_CreateTable.html), [update](http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_UpdateTable.html) and [delete](http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_DeleteTable.html) tables.
Also, the "Item Operations" enable you to add, update and delete items from a table.
You can also perform **[conditional updates](http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_UpdateItem.html#DDB-UpdateItem-request-ConditionExpression)** (eg. update a specific attribute only if the current value of the attribute is the same that the application retrieved previously) and have concurrency control through the "**atomic counter**" feature (where you can send a request to add or subtract to/from an existing attribute without interfering with other simultaneous write requests).
The 3rd and last operation category is the "Query and Scan": because **"Item Operations" can work only on single items**, "Query and Scan" are needed in order to retrieve "Items" in other ways (more info later on this column).

A [comprehensive list of DynamoDB operations is available here](http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Operations.html).


####Eventually Consistent and Strongly Consistent Read

When you receive an "operation successful" response to your write request, DynamoDB ensures that the write is durable on multiple servers (2 AZ at least, as we said before). However, it takes time for the update to propagate to all copies.

Because of this, and because **NoSQL misses the ACID transactions** of RDBMS as highlighted above, **a read request performed immediately after a write operation might not show the latest change** (consistency across all copies of the data is usually reached within a second). This characteristic is called "Eventual Consistency" and it's what happens in DynamoDB with the "[Eventually Consistent](https://en.wikipedia.org/wiki/Eventual_consistency)" read, that is the default option for GetItem, BatchGetItem, Query and Scan.

Anyway, DynamoDB offers you the option to **request the most up-to-date version of the data, through the use of the "Strongly Consistent" read** (not available for "Scan" operations).

>**NOTE**: "Strongly Consistent" reads cost twice as an "Eventually Consistent" (ie. [a "Read Capacity Unit" configured in DynamoDB represent 1 strongly consistent read or 2 eventually consistent reads](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Limits.html)).

In order to implement "Strongly Consistent" reads, please look at [parameters of each specific operation](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/operationlist.html).


###Primary Keys

The most important concept (imho) when you operate on DynamoDB tables, and even the first one you're asked for when creating a table, is the Primary Key.
There are 2 types of Primary Keys:

- **Hash Type Primary Key (Simple Key)**: it is for key-value lookups.
Again, it's like an [Hash/Dictionary structure (ie. Associative array)](https://en.wikipedia.org/wiki/Associative_array), where you have to ask for a specific key in order to retrieve the attributes of the item.
- **Hash and Range Type Primary Key (Composite Key)**: it is for Query, indeed in order to query the table, the primary key must be of the hash and range type.
Anyway, keep in mind that because DynamoDB is a key-value storage engine as said previously in this column, the query is restricted to a range of a specific primary/secondary key (again, you can't query for "color == red" in the whole table, but you could ask for all the items "color == red" of user "De Paoli").

In other words, it's very important **figuring out the way you will "ask" for your data** to DynamoDB, in addition to figure out the proper way to define uniqueness of every row (item) of your table (as is for almost every database anyway).

2 of the main operations for **data retrieval in DynamoDB depend by the type of the Primary Key** used:

- [GetItem](http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_GetItem.html): given the value of the Primary Key of one item (or multiple values of Primary Keys in the [BatchGetItem](http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_BatchGetItem.html) variant of this operation), it returns the attributes of that item. It's very similar and can be greatly tied to the [Hash/Dictionary structure](https://en.wikipedia.org/wiki/Associative_array) used in many programming languages.
- [Query](http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Query.html): it queries a table using the hash attribute and an optional range filter, and can return a range of results. You can query only tables whose primary key is of **hash-and-range type** (because a query to an Hash/Dictionary is just a Get of that single item).

It's worth to mention the "Scan" operation, that can read every item in the table but, contrary to Query, it always scans the entire table or the secondary index, and only after that full scan it filters out values to provide the desired result, with an obvious impact on the performance.
For large tables and secondary indexes, a **Scan can consume a large amount of resources**; it's very important to design the applications in order to use the Query operation mostly, because it is the most efficient way to retrieve items.

>**Cit.** [The larger the table or index being scanned, the more time the Scan will take to complete. In addition, **a sequential Scan might not always be able to fully utilize the provisioned read throughput capacity**: Even though DynamoDB distributes a large table's data across multiple physical partitions, a Scan operation can only read one partition at a time. For this reason, **the throughput of a Scan is constrained by the maximum throughput of a single partition**](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/QueryAndScan.html#QueryAndScanParallelScan).


###Secondary Indexes

A [secondary index](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SecondaryIndexes.html) lets you query the data in the table using an alternate key (similar to AK in SQL), with the main purpose of avoid the use of the inefficient "scan" operation while allowing flexible queries.
Basically, with secondary index **multiple copies of the same data are created**, indexed by different hash / hash-and-range key.

>**NOTE**: Secondary Indexes consume additional "Capacity Units" depending by different factors (eg. if, for instance, a write operation costs 1 Write Capacity Unit to be completed in 1 second, adding a Secondary Index might require 2 Write Capacity Units in order to perform the write in the same amount of time).

When Secondary Index born, the main table has been beginning to be called also "Primary Index". 


####Local Secondary Indexes

Local secondary index is an index that has the **same hash key as the table**, but a different range key. It's the first Secondary Index created.

When defining a Local Secondary Index (LSI), you define an alternate Range Key for your table, that can be used for additional and **more flexible query patterns** (ie. queries against an attribute that is not part of the primary key).
It's called "Local" because it is local to the Hash Key, meaning that the Primary Index of the table still be the same.
With LSI you can define also some "**projected attributes**", that are attributes of the main table (Primary Index) that will be cached with the LSI.

DynamoDB will use **additional write capacity units** to update the relevant indexes.
In other words, there can be two write operations when updating a table with an LSI, one for the primary index and another one for the secondary index, and both **operations will consume write capacity units from the main table allocation**.

**For reading operations, two different cases exist** depending by the item you retrive and by the projected attributes you set:

- **If querying from an LSI for its projected attributes**, the read capacity units allocated for the table are used and there could be also a lower cost if projected attributes are less than the table attributes.
- **If querying from an LSI for attributes stored in the primary index** (ie. not projected in the LSI), both the LSI and the primary index table must be read, consuming additional CUs.

>**NOTE**: Local Secondary Indexes can be created only when you create the table.

More on the [official guidelines](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GuidelinesForLSI.html).


####Global Secondary Indexes

Global Secondary Index (GSI) is an index with an **hash and range key that can be different** from the Primary Key.

Compared to the "Local Secondary Index", it gives **even more flexible Query patterns**.
**Global Secondary Indexes have their own provisioned throughput**, allowing more flexibility and also more efficiency, by tuning their capacity independently.

Even GSI permits the definition  of "projected attributes" as with LSI, and by default the primary key of the primary index is always projected.

More on the [official guidelines](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GuidelinesForGSI.html).


###Capacity Units and Performance Tenets (recap)

When you create a table in DynamoDB, you provision read ad write capacity units for the expected workload of the table, and [this is the parameter you're billed for](http://aws.amazon.com/dynamodb/pricing/).
Some things to remember:

- Capacity Units are an average measure, **ensure that the workload is well spread in space and time** (ensure your app monitor for hot items would be good).
- When request rate is too low it's possible to observe lower performance initially.
- A single **write CU represents one write per second for items as large as 1 KB** (items larger than 1 KB will require more than one write operation).
    - You cannot group multiple items in a single write operation, even if the items together are 1 KB or smaller (ie. if you want to write 100 items of 100 bytes each in 1 second, you need 100 write CUs).
- A single **read CU represents one "strongly consistent" read per second (or two "eventually consistent" reads per second) for items as large as 4 KB** (items larger than 4 KB will require more than one read operation).
    - You cannot group multiple items in a single read operation (ie. with "get", 1 read CU means 1 row/item of the table each second at most, unless queries).
    - You can use Batch operations to work multiple items (by requiring their specific keys) in parallel to minimize latency.
    - You can use the **Query and Scan operations in DynamoDB to retrieve multiple consecutive items** from a table in a single request. With these operations, DynamoDB uses the cumulative size of the processed items to calculate provisioned throughput.
- For tables with **both Local and Global secondary indexes, DynamoDB consumes additional CU**.

Lastly, [Best Practices for DynamoDB](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/BestPractices.html) provides great insight to avoid potential and well known problems.
