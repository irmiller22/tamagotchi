# Databases - Considerations and Insights

## Intro

Link: https://medium.com/@rakyll/things-i-wished-more-developers-knew-about-databases-2d0178464f78

A large number of computer systems are likely to maintain some sort of state, and in turn depend on a storage system. Databases tend to be at the core of system design goals.

## High-level Insights

- You're lucky 99.9999% of the time if the network is not a problem.
- ACID has many meanings.
- Each database has different consistency and isolation capabilities.
- Optimistic locking is an option when you can't hold a lock.
- There are anomalies other than dirty reads and data loss.
- DB doesn't always necessarily agree on ordering.
- App-level sharding can live outside the application.
- Autoincrement can be harmful.
- Clock skews happen between any clock sources.
- Evaluate performance requirements per transaction.
- Transactions shouldn't maintain application state.
- Query planners tell a lot about databases.
- Online migrations are complex but possible.
- Significant DB growth introduces unpredictability.

## ACID

ACID stands for atomicity, consistency, isolation, and durability. They are the database properties that transactions need to guarantee to their users for validity in the event of a failure. Most relational databases attempt to be ACID compliant, but there are other database types (like NoSQL) that implemented database paradigms without ACID because it's too expensive to implement.

Of the 4 ACID properties, consistency and isolation are the most difficult to implement, and tend to be expensive capabilities. They require coordination and increasing contention in order to keep data consistent. As the database scales up across multiple geographical regions and data centers, it becomes even harder to keep that data consistent due to decreasing availability and increasing network partitions. The CAP Theorem, which states that its impossible for a database to simultaneously provide 2 out of the 3 guarantees (consistency, availability, partition tolerance), describes this phenomenon. For example, in the presence of a network partition, one would have to choose between consistency and availability.

Databases tend to provide different consistency and isolation capabilities. They also provide a wide breadth of isolation layers so the application developers can pick the most cost effective isolation layer based on their trade-offs. Stronger isolation will eliminate some potential data race cases, but will be slower and might introduce contention that will slow down the database to a point where it may cause outages.

There are four isolation levels to be mindful of (in order from most expensive to least expensive):

- serializable
- repeatable reads
- read committed
- read uncommitted

A serializable execution produces the same effect as some serial execution of those transactions. A serial execution is one in which each transaction executes to completion before the next transaction begins. One note about Serializable level is that it is often implemented as “snapshot isolation” (e.g. Oracle) due to differences in interpretation and “snapshot isolation” is not represented in the SQL standard.

A repeatable reads execution allows for uncommitted reads in the current transaction to be visible to the current transaction but changes made by other transactions (such as newly inserted rows) won’t be visible.

A read committed execution allows for uncommitted reads to not be visible to the transactions. Only committed writes are visible but the phantom reads may happen. If another transaction inserts and commits new rows, the current transaction can see them when querying.

For a read uncommitted execution, dirty reads are allowed and transactions can see not-yet-committed changes made by other transactions. In practice, this level could be useful to return approximate aggregates, such as COUNT(\*) queries on a table.

## Optimistic Locking

Locks can be extremely expensive not only because they introduce more contention in your database but they might require consistent connections from your application servers to the database. Exclusive locks can be effected by network partitions more significantly and cause deadlocks that are hard to identify and resolve. In cases where being able to hold exclusive locks is not easy, optimistic locking is an option.

Optimistic locking is a method when you read a row, you take note of a version number, last modified timestamps or its checksum. Then you can check the version hasn’t changed atomically before you mutate the record.

```sql
UPDATE products
SET name = 'Telegraph receiver', version = 2
WHERE id = 1 AND version = 1
```

Update to products table is going to affect 0 rows if another update has changed this row earlier. If no earlier updates have been done, it will affect 1 row and we can tell our update has succeeded.

## DB Anomalies

We tend to pay attention primarily to race conditions that lead to dirty reads and data loss. There are other types of anomalies as well. An example of this type of anomalies is write skews. Write skews are harder to identify because we are not actively looking for them. Write skews are caused not when dirty reads happen on writes or lost but logical constraints on data is compromised.

Serializable isolation, schema design or database constraints can be helpful to eliminate write skews. Developers need to be able to identify such anomalies during development to avoid data anomalies in production. Having said that, identifying write skews in code bases can be extremely hard. Especially in large systems, if different teams are responsible for building features based on same tables without talking to each other and examining how they access the data.

## DB Ordering

There are a few reasons why generating primary keys via auto-incrementing may not be not ideal:

- In distributed database systems, auto-incrementing is a hard problem. A global lock would be needed to be able to generate an ID. If you can generate a UUID instead, it would not require any collaboration between database nodes. Auto-incrementing with locks may introduce contention and may significantly downgrade the performance for insertions in distributed situations. Some databases like MySQL may require specific configuration and more attention to get things right in master-master replication. The configuration is easy to mess up and can lead to write outages.

- Some databases have partitioning algorithms based on primary keys. Sequential IDs may cause unpredictable hotspots and may overwhelm some partitions while others stay idle.

- The fastest way to access to a row in a database is by its primary key. If you have better ways to identify records, sequential IDs may make the most significant column in tables a meaningless value. Please pick a globally unique natural primary key (e.g. a username) where possible.

## Clock Skew

A hidden secret in computing is that all time APIs lie. Machines generally do not have an accurate representation of what the current time is. All computers contain a quartz crystal that produces a signal to tick time. However, quartz crystals don't accurately tick and drift in time, and tend to either be faster or slower than the actual clock. Drift can be anywhere from a few milliseconds to 20 seconds per day. As a result, computers need to be synchronized by the actual time every now and then to maintain accuracy.

NTP servers are responsible for this synchronization. However, this can be delayed due to network latency. Atomic and GPS clocks are more optimal sources of determining the current time, but they tend to be expensive and complicated to set up. Data centers tend to rely on a hybrid approach where they leverage atomic / GPS clocks, and then use servers to broadcast the time to the rest of the machines in the data center. However this does mean that every machine is susceptible to clock drift due to latency.

## Performance Requirements per Transaction

Sometimes databases advertise their performance characteristics and limitations in terms of write and read throughput and latency. Although this may give a high level overview of the major blockers, when evaluating a new database for performance, a more comprehensive approach is to evaluate critical operations (per query and/or per transaction) separately.

Examples:

- Write throughput and latency when inserting a new row in to table X (with 50M rows) with given constraints and populating rows in related tables.
- Latency when querying the friends of friends of a user when average number of friends is 500.
- Latency of retrieving the top 100 records for the user timeline when user is subscribed to 500 accounts which has X entries per hour.

Evaluation and experimentation might contain such critical cases until you are confident that a database will be able to serve your performance requirements. A similar thumb of rule is also considering this breakdown when collecting latency metrics and setting SLOs.

Be careful about high cardinality when collecting metrics per operation. Use logs, even collection or distributed tracing if you need high cardinality debugging data. See [Want to Debug Latency?](https://medium.com/@rakyll/want-to-debug-latency-7aa48ecbe8f7) for an overview on latency debugging methodologies.

## Nested Transactions

It's very easy for nested transactions to cause surprising programming errors that are not always easy to identify until it becomes clear that you are seeing anomalies. Encapsulating transactions in different layers can contribute to surprising nested transaction cases and from a readability point-of-view, it might be hard to understand the intent.

Imagine a data-layer with several operations (e.g. newAccount) already is implemented in their own transactions. What happens when you run them in higher level business logic that runs in it own transaction? What would be the isolation and consistency characteristics would be?

Instead of dealing with such open-ended questions, avoid nested transactions. Your data layer can still implement high level operations without creating their own transactions. Then, business logic can start transactions, run the operations on the transaction, commit or abort.

## Query Planners

Query planners determine how your query is going to be executed in the database. They also analyze the queries and optimize them before running. Planners can only provide some possible estimations based on signals it has.

There are two ways to retrieve the results:

- Full table scan: We can go through every entry on the table and return the articles where author name is matching, then order.
- Index scan: We can use an index to find the matching IDs, retrieve those rows and then order.

The query planner’s role is to determine which strategy is the best option. Query planners have limited signals about what they can predict and might result in poor decisions. DBAs or developers can use them to diagnose and fine tune poorly performing queries. New releases of databases can tweak query planners and self-diagnosing them can help you when upgrading your database if new version introduces performance problems. Reports such as the slow query logs, latency problems, or stats on execution times could be useful to determine the queries to optimize.

Some metrics the query planner provides could be noisy, especially when it estimates latency or CPU time. As a supplement to query planners, tracing and execution path tools can be more useful to diagnose these issues even though not every database provides such tools.

## Database growth introduces unpredictability

Database growth makes you experience unpredictable scale issues. The more we know about the internals of our databases, the less we might predict how they might scale but there are things we can’t predict.

With growth, previous assumptions or expectations on data size and network capacity requirements can become obsolete. This is when large scheme rewrites, large-scale operational improvements, capacity issues, deployment reconsiderations or migrating to other databases happen to avoid outage.

Don’t assume knowing a lot about the internals of your current database is the only thing you need, scale will introduce new unknowns. Unpredictable hotspots, uneven distribution of data, unexpected capacity and hardware problems, ever growing traffic and new network partitions will make you reconsider your database, your data model, your deployment model and the size of your deployment.
