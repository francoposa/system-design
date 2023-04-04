# System Design

## Fundamental Components

### Databases

#### Relational Databases

Relational database models stem from the need to do transaction processing and batch processing.

Relational databases store data in tables, organized by rows and columns.
Tables usually have a **primary key**, a unique identifier which is indexed to provide fast lookup of a single given record in the table.
Records can link themselves to records in other tables via **foreign keys**, which reference an unique identifier in another table.

This allows deconstructing data so that there is not repetition of identical data, in a process called **normalization**.

##### Relational Database Normalization and Online Transaction Processing (OLTP)

Traditional relational databases enforce very strict transactional consistency rules known as ACID.
The entire database instance always moves from one valid state to another, in accordance with all intra-and inter-table constraints declared.

This stems from the original use cases for relational databases, called **OnLine Transaction Processing (OLTP)**.
OLTP databases use database constraints to strictly enforce business rules.

This use case has implications for how data is laid out on disk, as well as how concurrency is handled.

A standard example of OLTP use case is a user interacting with their bank.
The user largely only wants to see their own information - all of their accounts, all the information about an account etc.

In order to rapidly access all of a user's data, the data is laid out on disk in a row-oriented format.
All fields of a given row in a table are likely to be stored consecutively, so the database can jump to the correct row, then read the whole row at once.

As someone is using their banking, they could be transferring money and withdrawing money, all while their restaurant bill and monthly phone bill are being processed.

An OLTP database will process the transactions one at a time, in accordance with any rules such as a daily limit on withdrawals or overdrafts.
The transactions processed in order, and each time they will be checked for validity against the rules of the database.

A database with a single queue to enforce transactions would have very poor performance.
This would also be overly restrictive, as most operations would have nothing to do with one another.
John withdrawing money will never interfere with Jane withdrawing money, so we can process them both at the same time.

Instead, ACID transactional databases use locking to enforce consistency during concurrent updates.
The database engine does its best to only lock the rows needed in order to continue processing unrelated transactions.

However, eventually there will be concurrent operations which cannot be safely processed at the same time without potentially violating the database constraints and returning invalid data in violation of the ACID principles.

In this case, the relational database engine may have to lock an entire database table or large sections of the table while changes are processed.
If the changes take long enough, other operations waiting to access that table will have to wait, potentially long enough to time out completely.
In this way, ACID transactional databases prioritize *consistency* over *availability*.

One way to mitigate the need to lock entire database tables is to apply database organization techniques known as **normalization**, which can reduce the amount of space utilized by a database, as well as limit how much of a database gets locked up during transactions.

A simple example of a relational database without any normalization applied might have a data layout like this:

```
Orders
order_id,user_firstname,user_lastname,product_name
1,John,Doe,Big Bang Hot Sauce
2,John,Doe,Big Bang Hot Sauce
3,Jane,Doe,Big Bang Extra Hot Sauce
4,Jane,Doe,Big Bang Hot Sauce
```

This can introduce several problems.
We are wasting space duplicating data.
Further, if anything - like the product name - needs to change, you have to change every record in the table.
In a large enough table, this could be an extremely expensive operation spanning many minutes or hours which could slow down or completely lock up the database making it unable to process your orders.

Instead, with normalization we can store in a more decoupled layout:

```
Orders
order_id,user_id,product_id
1,1,1
1,1,1
1,2,2
1,2,1

Users
user_id,user_firstname,user_lastname
1,John,Doe
2,Jane,Doe

Products
product_id,product_name
1,Big Bang Hot Sauce
2,Big Bang Extra Hot Sauce
```

Now if any aspect of a product or user changes, only a single record in one table needs to change.
The orders table does not have to change when Big Bang Hot Sauce rebrands their products - the Orders table just holds a foreign key `product_id`, which is used to go look up the current product name.

Because only one record needs to be locked up while the changes are made, a change to a product that could affect a massive amount of orders is accomplished in milliseconds without any operations or locking on the `Orders` table.



##### Relational Database Denormalization and Online Analytics Processing

* Column oriented to speed up aggregation queries
* Denormalized - fewer tables (star and snowflake schemas)


### Scaling Databases

#### Vertical and Horizontal Scaling
Scaling can be vertical (increasing the size of the machine running the database) or horizontal (increase the number of machines ).

Vertical scaling is simplest, particularly in virtual-machine-based environments which support live resizes of nodes.
Vertical scaling generally does not require any architecture changes or distributed system consistency concerns.
Relational databases traditionally have relied on vertical scaling as their consistency guarantees are difficult to manage across multiple machines.
However, there are physical limits to the size of server machines, so vertical scaling has a ceiling.
In addition, scaling down if you eventually do not need the larger machine may be tricky or impossible to accomplish without significant downtime, depending on database architecture.
Vertical scaling also does not offer any of the high-availability properties which can be provided by horizontal scaling.

Horizontal scaling has gained popularity for its "elastic" or "infinite" scalability.
There is no limit to how many machines can be added to a database cluster.
Modern "NOSQL" databases which gained popularity throughout the 2000s were designed with horizontal scalability in mind.
NOSQL database traditionally have looser consistency and transactional guarantees, in order to performing expensive locking operations across unreliable network links between nodes.

Both relational and NOSQL databases can take advantage of horizontal and vertical scaling, even though they each may be better designed for one or the other.

Since the very popular NOSQL databases appeared, relational databases engines have been developing ways to be better compatible with horizontal scaling, and NOSQL databases have worked on ways to provide better consistency and transactional guarantees despite the difficulties posed by their distributed nature.

#### Vertical and Horizontal Partitioning

Vertical and horizontal partitioning are not necessarily analogous to vertical and horizontal scaling.
While vertical and horizontal scaling refers to how larger or more machines are used to handle database size and load, vertical and horizontal partitioning refers to how database tables are split up.

**Vertical partitioning** involves splitting database tables by its columns - more tables with fewer columns per table.
Vertical partitioning changes the structure of the tables.
Normalization is a form of vertical partitioning, but fully-normalized databases can be further vertically partitioned.
For instance, a fully normalized table could be split into one table for mostly-static data, and another for the data that changes often.
Operations which only read from the static table will be faster, as they won't have to wait on rows which are locked during write operations.

Columnar databases may be designed as an extreme case of vertical partitioning, where each column functions as its own table.

In vertical partitioning, some database tables may live on a completely different machine.
This is horizontal scaling with vertical partitioning.

**Horizontal partitioning** involves splitting database tables by its rows - more tables with fewer rows per table.
Horizontal partitioning does not changes the table structure.
Horizontal partitioning involves some sort of hardcoded or hash-function based process to determine which of the partitioned tables to search in - for example, you may split a table of financial transactions by year, month, or day.
This reduces the index size for each table, reducing lookup time when searching within a date range.
This also implies a need to balance the partitions - if you split into two tables, and one partition grows much faster, you may end up in a situation where you still have one very large partition, with the same performance issues as the original single table.

A more complete form of horizontal partitioning is **sharding**, in which you duplicate the same schema across multiple database instances which do not interact with one another.
You may store users in different databases by their country or region, in order to reduce each database's size and provide a database located closer to the users for latency or data sovereignty compliance.
