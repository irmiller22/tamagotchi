# Data Models

Applications need to have permanent storage for user or applications specific
data. In memory data structures like linked list, arrays are optimised for
access by CPU via pointers. Permanent storage is optimised for read/write
access by clients/processes connecting to database server. A very important
aspect of permanent persistence is data modelling. I will devote this post on
how to choose a good data model for your application.

## Relational Database

The most famous and prevalent data modelling technique is using relational
tables. In relational tables, data is organised into records of a table. Tables
are related to one another using primary key foreign key. There are a lot of
reasons why your should choose relational table.

1. You are just building out v1 of your app and data access patterns are not
   quite clear yet. Relational schemas are always a good first default choice.
   v1 of all apps are generally bad, and no one will fault you for starting
   with relational tables, instead of fancier NoSQL or Graph DB.
1. You need to enforce strict schema on write constraints.
1. You want to maintain zero data redundancy. Normalisation of schema in
   relational models has the effect of shredding information into many tables.
1. Your data model has many to one and many to many relationships. In other
   words you know which joins will be performed before hand. Joining and
   querying relational databases using a declarative language like SQL is one
   of the greatest secret sauce of relational databases. A lot of research and
   effort has gone into making relational queries super fast. An application
   developer just has to specify the expected data pattern of the query. The
   query engine will convert the SQL query into an optimised code to fetch/write
   data.

## NoSQL Database

Application development is done using object oriented programming. However when
data storage is done using tables, there is a translation required from objects
to “shredded” relational tables. ORM frameworks provide boilerplate code to
reduce the effort required for this translation, but still there is work to
be done.

NoSQL solves this problem by representing a record as a self contained JSON
document. For example, let’s say we need to store patient demographic
information along with his current conditions. One way to represent this
record in a NoSQL database can be:

```
{
    “first_name”: “John”,
    “last_name”: “Doe”,
    “conditions”: [
        {
            “name” : “T2DM”,
            “onset”: “12–12–1990”
        }
    ]
}
```

If you need read profile data for a patient in your app, you need not issue
multiple joins as all data is inside one NoSQL document. Typically, if
your data model exhibits a tree like, one to many relationships, using a
NoSQL database might make more sense.

In the patient example above, what if we wanted to store ICD10 standard
codes for conditions instead of name of condition. This is a little troublesome
in document based NoSQL databases as they have little support for joins. You
can still do a join at the application layer but this will be always suboptimal
compared to the joins done at a typical relational database layer. NoSQL
databases become less desirable in this case.

Lastly in case your data model does not have a fixed schema, going the NoSQL
route might make more sense. Consider, the patient example above and say we
need to also store patient’s date of birth. In NoSQL case, we can add a new
field, ‘dob’ to new documents. At the application level we can also add
code to handle reading old documents without dob field. In a relational
database, the solution to handle dob would be to alter schema and make
data migrations. Data migrations are slow and require downtime and
consequently generally avoided.

## Graph Database

Graph database makes a lot of sense when your application’s data model needs
to support many ‘many-many’ relationships. The relational model can handle a
few many to many relationships, but beyond a point all the relational joins
become messy and slow. Using graph databases also provides an added advantage
of easily extending relationships between heterogeneous objects.

A graph consists of two kinds of objects — nodes and edges. Nodes contain
description of objects or entities. Edges contain description of relationships
between nodes. For example, say a person suffers from an allergic reaction because
of exposure to an substance. You can model person, allergic reaction and substance
as nodes. You can also model relationships between nodes in a graph database.
For example a relationship between person and allergy can be a unidirectional
relationship “person-has-allergy” from person node to allergy node. Relationship
between allergy and and substance can be a unidirectional relationship
“triggered-by-exposure-to” from allergy to substance.

Why not do this in relational tables ? Well you can. You can create three
tables — person, allergy and substance and set up the appropriate Pk-FK constraints.
Like I said earlier, using graph databases makes sense when we have a lot of
many-many relationships. For example, let’s say you introduce a location
object in the overall scheme of things so that you can capture location
of the person where the allergy reaction occurred. Location can be
neighbourhood, city, state, country, continent or hemisphere. Basically,
location information can be available at various levels of granularity.
Using SQL to create a declarative query will be messy. SQL needs to know
in advance which joins will be part of the query. In graph database, on the
other hand, you can traverse many nodes and edges before arriving at the
target node. You can express the fact of traversing a graph once or many time
quite concisely using a graph database declarative query language like Cypher,
for Neo4j graph database.
