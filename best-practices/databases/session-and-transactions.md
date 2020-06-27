> Database Sessions - Best Practices

The purpose of this document is threefold:

- provide an overview of database sessions and database transactions
- provide specifics around how sessions and transactions are managed via SQLAlchemy
- introduce best practices for managing database sessions and transactions

The itention is to come away from this document with knowledge on how to manage database sessions and transactions in the backend codebase.

> What is a database session?

A database session is the code construct that establishes and maintains all communication between your application and the database(s). It is intermediary in which all application logic and database abstractions (SQLAlchemy model, for example) are loaded. Most importantly, it manages database transactions initiated in the form of queries.

A database session lifecycle looks like the following:

1. Database session is created.
1. Database session retrieves query requests, and the query results are associated with the database session.
1. Database model objects are added to the database session.
1. Database session starts to manage the changes requested against the database model objects.
1. Database session commits the changes (if commit fails, then a rollback occurs).
1. Database session is closed.

In short, a database session is the code-level agent responsible for orchestrating communication with the database.

> What is a database transaction?

A database transaction is an instruction that the database session sends to the database. We understand these instructions in the form of SQL queries. A rule of thumb to remember when exactly you're in a transaction is to keep track of `BEGIN` and `COMMIT`/`ROLLBACK` SQL queries. These define the boundaries of a transaction. When we send the `BEGIN` SQL query to the database, we've entered a transaction. When we send the `COMMIT` or `ROLLBACK` SQL query to the database, we've exited a transaction.

Transactions exist to provide two things: atomicy and isolation. Atomicy simply means that we want all changes to occur during a successful commit (`COMMIT`), or that we don't want anything to occur during a failed commit (`ROLLBACK`). Isolation means that we don't want database changes occuring within a transaction to be available to other database sessions until those changes have been committed. The atomicy and isolation concepts make up 2 of the 4 concepts for an ACID-compliant database. Typically ORMs like SQLAlchemy by default will wrap single SQL queries in transactions to ensure atomicity and isolation.

There are 3 states associated with transactions: `active`, `idle in transaction`, and `idle`. Most are familiar with the `active` state - this is simply a database connection that is doing work.

The other two states, `idle` and `idle in transaction` are important to keep in mind for debugging purposes. The `idle` state means that the database connection is not doing anything, and is waiting. This is a good point to disconnect from the database since this won't result in data loss. The `idle in transaction` state means that the connection started a transaction at some point, and is waiting for something. This can be anything like data being processed on the application side, the database loading some data before returning it to the application, or something along those lines. Whenever you have connections that are in the `idle in transaction` state, this usually means that your database connection setup warrants more investigation, because the connections are not being used efficiently.

> Explain it to me like I'm 5.

Imagine that you're going to a bank. You're planning to deposit 3 checks, and also withdraw some cash.

You walk into the bank, and you wait in line until a teller window opens up. You fill out the deposit slips, and then deposit the checks one by one. After each deposit, the teller gives you confirmation that the check has been deposited, as well as the amount that has changed in your bank account. Finally, you request the cash withdrawal. The teller hands you the cash.

Once you're finished, you step out of the line, and walk out of the bank.

Imagine that walking into the bank is like creating a session, and walking out of the bank is like closing a session. Waiting in the teller line for an available teller is analogous to waiting for an available connection via the connection pool. Handing over each check to be deposited and retrieving confirmations are database transactions. Finally, asking for and retrieving the cash is a separate database transactions.

> SQLAlchemy

> How does SQLAlchemy leverage sessions and transactions?

Let's step through an example of SQLAlchemy interaction with a PostgreSQL database.

```python
import sqlalchemy
from sqlalchemy import orm

# Configure a connection to the database at the URL specified by the
# DATABASE_URL environment variable.
# Remember that we're using `echo=True` so we can see all generated SQL.
engine = sqlalchemy.create_engine(os.environ['DATABASE_URL'], echo=True)

# Create a session factory. Calling `Session()` will create new SQLAlchemy
# ORM sessions.
Session = orm.sessionmaker(bind=engine)

# Create a new session which we'll use for the following investigation.
session = Session()
```

We've initiated a session to the database specified via the `DATABASE_URL`. Now we'll start querying the database for `Payment` records.

```python
q = session.query(Payment).filter(Payment.amount >= 500)
payment = q.first()
```

This will retrieve a list of payments where the amount is 500 and over, and we'll select the first result. Since we're using PostgreSQL for the database, let's leverage `echo=True` in the `create_engine` invocation to log out what's happening at the database level.

```
select version()
{}
select current_schema()
{}
SELECT CAST('test plain returns' AS VARCHAR(60)) AS anon_1
{}
SELECT CAST('test unicode returns' AS VARCHAR(60)) AS anon_1
{}
show standard_conforming_strings
{}
BEGIN (implicit)
SELECT payment.id AS payment_id, payment.amount AS payment_amount, payment.user_id as payment_user_id
FROM payment
WHERE payment.amount >= %(amount_1)s
 LIMIT %(param_1)s
{'amount_1': 500, 'param_1': 1}
```

So what's happening here as part of the SQLAlchemy query?

The first five statements are compatibility checks that the database performs prior to each query. The sixth query is the start of our database transaction, which is signified by a `BEGIN` statement. The seventh and last query is our actual query that loads the payment record that we asked for.

Next, let's check `pg_stat_activity` and see what's happening there via the following query:

```sql
SELECT
    backend_start,
    xact_start,
    query_start,
    state_change,
    state,
    substring(query for 50) AS query
FROM pg_stat_activity
WHERE
    backend_type = 'client backend' AND
    pid != pg_backend_pid();
```

We get the below output back:

```
-[ RECORD 1 ]-+---------------------------------------------------
backend_start | 2020-06-22 15:21:32.10754+01
xact_start    | 2020-06-22 15:21:32.122549+01
query_start   | 2020-06-22 15:21:32.122586+01
state_change  | 2020-06-22 15:21:32.123075+01
state         | idle in transaction
query         | SELECT payment.id AS payment_id, payment.amount AS paym
```

To summarize:

- `backend_start` - when session was opened
- `xact_start` - when transaction was opened
- `query_start` - when the query was initiated via the transaction
- `state_change` - when the connection changed from `active` to `idle_in_transaction` - we also know that the query took 0.5 ms to run (`state_change` - `query_start`)

At this point, the transaction is still open and is waiting for the database connection to do something.

What happens when we do a `COMMIT` or a `ROLLBACK`? Let's see what happens:

```python
session.query(Payment).first()
session.commit()
```

Here's the transaction log:

```
BEGIN (implicit)
SELECT payment.id AS payment_id, payment.amount AS payment_amount, payment.user_id AS payment_user_id
FROM payment
 LIMIT %(param_1)s
{'param_1': 1}
COMMIT
```

And here's the `pg_stats_activity` output:

```
-[ RECORD 1 ]-+------------------------------
backend_start | 2020-06-22 15:39:32.10754+01
xact_start    |
query_start   | 2020-06-22 15:39:58.10754+01
state_change  | 2020-06-22 15:39:59.1075431+01
state         | idle
query         | COMMIT
```

On the `COMMIT`, the connection status has changed from `idle_in_transaction` to `idle`. This outcome the same for `ROLLBACK` as well. We can safely close out the session without any concern for data loss.

> Best Practices to keep in mind

1. When using SQLAlchemy model properties for read-only purposes, `expunge` the session objects.

```python
payment = session.query(Payment).first()

# what happens here?
print(f"Payment amount: {payment.amount}")
```

We actually initiate another SQL transaction as a result of the `print` statement above. Let's look at the SQLAlchemy logs and `pg_stat_activity`:

```bash
# sqlalchemy logs
BEGIN (implicit)
SELECT payment.id AS payment_id, payment.amount AS payment_amount, payment.user_id as payment_user_id
FROM payment
 LIMIT %(param_1)s
{'param_1': 1}
Payment amount: 500
```

```bash
# pg_stat_activity
-[ RECORD 1 ]-+---------------------------------------------------
backend_start | 2020-06-22 13:02:08.225582+01
xact_start    | 2020-06-22 13:02:08.335367+01
query_start   | 2020-06-22 13:02:08.335425+01
state_change  | 2020-06-22 13:02:08.335628+01
state         | idle in transaction
query         | SELECT payment.id AS payment_id, payment.amount AS paym
```

SQLAlchemy's default behavior during property access post-transaction is to reload the object from the database in a new transaction. This is the same for all cases outside of `COMMIT` or `ROLLBACK`.

In order to make properties read-only outside of the session context, do the following:

```python
payment = session.query(Payment).first()
session.expunge(payment)

# no BEGIN statement will be emitted to the database here
print(f"Payment amount: {payment.amount}")
```

2. Close database sessions to enforce no follow-up transactions.

By closing a database session, you're enforcing that no database transactions will be opened up whenever a SQLAlchemy object's property is accessed. If you close a database session without expunging the object, then try to read the object's `amount` property, you will get a `DetachedInstanceError` exception.

3. Keep the database session lifecycle as short as possible.

If you expect a database request to take more than 5 seconds, then the underlying query needs to be optimized. If you expect a database request to be long-running, such as a migration, then this should be executed in a separate context from the REST API layer.
