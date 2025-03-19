# Fundamentals of Database Engineering
## ACID 
### Transaction
- Transaction is a set of sql queries treated as one unit of work.
    - Ex : Account Deposit(Select,Update,update)
- Lifespan
    ```
    TRANSACTION BEGIN //always starts with begin
    
    `various steps....`

    TRANSACTION COMMIT // ends with commit data gets persisted now

    TRANSACTION ROLLBACK // if want to undo trasaction at any step or any crash

    DB are designed to optimise one or the other steps in transaction like PostgreSQL does commit the fastest, sqlServer takes longest to rollback
    ```
- Nature : Usually done to change /modify data but perfectly normal to have a read only transaction

## Atomicity (`A`CID)
- All queries in a trasaction must succeed, an atom can't be split(we know now it can :) . If any query fails in any step all must rollback(if db crashed on restart rollback)

## Isolation(AC`I`D)
- There can be various connections and each connection execute trasaction so concurrency can occur, so Isolation comes into picture
- Should other transaction see changes made by other transaction, so we get `READ PHENOMENA` which are not desirable,Isolation levels to solve this issue
- Isolation-Read Phenomenra
    1. Dirty Read : Read something which other transaction wrote but not yet commited, so it could be rolled back either due to fail or db terminate/stop
    2. Non-Repetable Reads : Reading same values again and again i.e different queries but reading same row/values
    3. Phantom Reads : assume a query over range some values doesn't exist and added after getting the query result can cause issue if you want to change something based on the previous result 
    4. Lost Updates : Wrote something but before commiting some other transaction overrite this so you value is lost
- Isolation Levels(For fixing read PHENOMENA)
    - Read Uncommited : No isolation, any change is visible whether it's commited or not.Bad but fast
    - Read commited : Each query in a transaction only sees commited changes by other trasactions
    - Repetable Read : The transaction will make sure when a query reads a row , that row will remain unchanged until the transaction is running, fixes non-repeatable read, doesn't fix phantom read
    - Snapshot : Each query in transaction only sees changes that have been commited until the start of transaction ,Guaranteed to solve every read phenomena
    - Serialize : Slowest,Transactions run as if they are serialized one by one
    - Each DBMS implement isolation level differently
- Table for understading

| **Isolation Level**       | **Dirty Reads**          | **Lost Updates**         | **Non-Repeatable Reads**   | **Phantom Reads**         |
|---------------------------|--------------------------|--------------------------|----------------------------|---------------------------|
| **Read Uncommitted**      | <span style="color:red">May occur</span> | <span style="color:red">May occur</span> | <span style="color:red">May occur</span>  | <span style="color:red">May occur</span> |
| **Read Committed**        | <span style="color:green">Don't occur</span> | <span style="color:red">May occur</span> | <span style="color:red">May occur</span>  | <span style="color:red">May occur</span> |
| **Repeatable Read**       | <span style="color:green">Don't occur</span> | <span style="color:green">Don't occur</span> | <span style="color:green">Don't occur</span> | <span style="color:red">May occur</span> |
| **Serializable**          | <span style="color:green">Don't occur</span> | <span style="color:green">Don't occur</span> | <span style="color:green">Don't occur</span> | <span style="color:green">Don't occur</span> |
- Each DBMS implement Isolation level differently 
    - Pessimistic : Row Level Locks,table locks,page  locks to avoid lost updates
    - Optimistic : No locks, just tracks if things changes and fail if it does 
    - Repeatable Read :Locks the rows it reads but could be expensive if read a lot of rows.Postgres Implement RR lock as snapshot so phantom read doesn't occur in repeatable read
    - Serialiable usually implemented with optimistic concurrency control , will be very slow, can implement it pesimitically as well with "SELECT FOR UPDATE"

## Consistency (A`C`ID)
- This was the property sacrificed in NoSQL and Graph DB(CAP theorem)
- Consistency In Data(In a given instance, at that level)
    - Defined by User/DBA/Developer Team
    - Referential integrity(Foreign Key)
    - Atomiticity
    - Isolation
- Consistency In Read(Full System, various partitions/readers etc this one is of CAP theorem)
    - If a trasaction commited a change andy read will give that updated value?Affect the system as a whole
    - Relational and NoSQL db suffers from this
    - Eventual Consistency 

## Durability (ACI`D`)
- No matter what happens to system when changes are commited they should be shown and remain persisted
- Changes made by committed transactions must be persisted in a durable non-volatile storage
- Durability Techniques
    - WAL : Write Ahead Log, we write only delta to DB
        - Writing all the changes is a lot of data and lot of time
        - so need something compressed and quick , that's WAL to write
        - OS Cache : A write request goes to OS cache, OS does that and on sudden crash data will be lost, Fsync OS writes always goes to DB disk but will be slow
    - Asynchronous Snapshot 
    - AOF : Append only file

## Serializable vs Repeatable Read Isolation Level
- Serilizable will make sure trasanction concurrently are done one by one so results will be different from Repeatable Read

## Eventual Consistency
- consistency is part of acid, is always a part of SQL/Relational DBs 
- Eventual Consistency is born with introduction of NoSQL, but SQL sufferes from it as well
- Consistency
    - Consistency in Data : across tables, several tables etc
    - Consistency in Reads : Both relational and non realtional suffers from that due to many servers.So eventual consistency , sometimes may get not latest value.Once you data is in 2 places you are inconsistent .The moment you scale horizontally or introduce cache there will be eventual consistency

## How Tables and Indexes Stored On Disk(And how are they queried)
- Dbs have row Ids, most db has it same as primary key but some have different like Postgres have a system column row_id(tuple_id)
- Page 
    - row/colums(depeding on which storage type db it is) are stored in logical pages
    - each page has a usually fix size, 8kb in postgres, 16 kb in mysql
    - DB doesn't read a row, it read a page or more in an IO operation
- IO : It's a read request to the disk, try to mimize this as much as possible
    - can't read single row, select one/more pages along with all rows in them
    - could go to OS or OS cache, postgres goes to cache first
- Heap : Heap is the DS where table is stored page by page
    - Traversing heap is expensive for reading , that's why we need indexes
- Index : Another DS , that has pointers to that heap
    - It has part of data , used to search for something quickly.Can index on one column or more
    - By finding value of index you can find more info about it
    - Index tell you which page to search exactly
    - Index is also stored as pages and fetching also consumes IO
    - Smaller the index, the better it fit into memory fater the search
    - B-Tree is popular for implementing Index
    - Not magic !!!
    - Index gives on basis of column exact page and then we fetch that page and respective rows
    - Sometimes the heap table can be organized around a single index. This is called clustered indem or Index Organized Table(Oracle DB uses this term)
    - Primary Key is usually a clustered index
    - MYSQL InnoDB always have a primary key (cluster index) other index points to primary key "value"
    - Postgres only have secondary indexes and all indexes point directly to the row_id which lives in the heap

## Row Vs Column Oriented Databases
### Row Oriented DBs
- Tables are stored as row in disk
- A single block IO fetches page with multiple rows with all columns
- More IOs for finding a particular row but get all columns so no extra IOs for it may be good or bad according to your need
- aggreagrate like function on column need to do excessive read
- OLTP
### Column Oriented DBs
- Tables are stored as columns in disk
- Each column entry has value and attached rowId/key for searching other columns if want almost like index like saving
- A single block IO read to the table fetches multiple coulmns with all matching rows
- Less IOs are required to get more values of a given column .But more IOs required if working with more coulmns
- OLAP
- Deleting anything in between will be painful, as each column has row/id attached and have to delete each column if a row is deleted
- Select * bad for this
- In case of duplicate values of a coulmn it's much better as it clubs common values
### Comparision
| Row-based  | Column Based |
| -------- | ------- |
| Optimal for Read/Writes  | Writes are Slower    |
| OLTP | OLAP(Analysis warehouse not much data change)  |
| Compression isn't efficient  | Compress greatly    |
|Aggregation isn't Efficient| Amazing for aggreation|
|Efficient queries w/multi columns|Inefficient queries with multi columns|
|Postgres, Mysql Most are by default row based|From DB engine you can store some table as row based while other column based|
- Don't join/can't join row based table with coulmn based table

## Primary Keys Vs Secondary Keys
- Data is saved in heap with no ordering and Primary Key forces an order, increasing cost as acts like cluster , so heap is organized around it
- But with organized aound id i.e. cluster created now the query execution will be faster around index
- With addition of secondary index solves the jumble related to various columns . In postgres all index are secodnary index, no primary index

## DataBase Indexing
### Basics/Intro
- An index is a DS that you build and assign on an existing table, which analyses/summarizes data on the table so that it can create shortcuts. Like a phone directory/dictniory mentioning A , B,... N..Z , so want to search Zebra got to Z
- Index are built using B-Tress or LSM trees(Until now in course)
- Postgres(And Mnay other) Db has index by default with primary key
```
explain analyze < your query> // it shows query result, anaylsis and time, can look for index usage

create index index_name on table(columns);
```
- Basically avoid full table scan as much as possible and index helps you avoid that many time on most scanned columns , without indexing queries can return answer in seconds and with index it can return answer in <100 ms, (of course depends on DB and volume but you get the gist)
- P.s like %someChar% is worse worse query, have to go all table and then match as well, even index can't satisfy expression as not single value

### `Explain` in Postgres
- explain < query > is command in postgres which gives us what query plan/path will postgres take over the given query
- give various info like first ... last meaning time it took to return first row in answer and last row in answer , number of rows it scanned through, workers planned, query on full table or index,average width(number of bytes in result) of a row etc etc
- with analyse it show actual time taken
#### Types of Scan
- Index Scan : Searching is done using index
- Seq Scan : All row is scanned on the table itself
- BitMap Scan : It will create a bitmap with saving pageNumber, then jump on heap by fetching all pageNumber and then can take the required rows, and bitMap is created on bits and on several bitmap you can AND, OR etc to get required values

### Key Vs Non-Key Column DB Indexing
- When you create an index, it's done on a column/cloumns and will be used for searching purposes
- On creating index on column you can add another column entry to be contained most times it is index/primary key
- so create index on table(someColumn) include(someOtherColumnMaybePrimaryId)
- keep in mind if you increase index element it will grow and won't be able to be stored inMemory and move to hard disk so time increases

#### Index scan is you have to scan the index and needed to go back to heap, Index Only scan is you only scanned the index and returned the query(scanned on index and other thing you need is found in index only so `better`)

### Combining DB indexes for better Performance
- Querying on index values results in querying through index, if there are several index columns involve in condition it will accordingly either scan both, scan create BitMap or if very less result directly fetch from the heap(i.e table itself)
- if index is created on let's say 2 or more columns index(a,b) any query on b will result in parallel seq scan i.e heap scan, so can use a&b , only b, a or b is full table scan most times
- if build index on b , it can help in querying b and a or b

### How DB optimizers Decide to use Indexes/Other IMP Points
- Basically if there is going to be very few rows in result or been asked only few rows better to scan the heap than index else in rest cases if index is available it will be used
- DB keep some type of cache/approx number of rows present with some values for getting an approximation
- If there are too many values returned from index DB will decide to go to heap
- Basically too little or too many rows directly go to heap else use index, directly row in postgres also used workers parallel query execution etc(like multithreading but not threads) so it will go for full scan if guess is result will have too many tables

| Database       | Model                         | Query Parallelism     | Notes                                           |
|-------------- |-----------------------------|----------------------|------------------------------------------------|
| MySQL         | **Thread-based**              | Limited (MySQL 8+)   | Each connection gets a thread                 |
| PostgreSQL    | **Process-based**             | Yes (Worker Processes) | Parallel queries with worker processes        |
| SQL Server    | **Hybrid (Threads + Pooling)** | Yes                  | Uses thread pools and intra-query parallelism |
| Oracle DB     | **Process or Thread-based**   | Yes                  | Can switch between multiprocess and multithreaded modes |
| IBM Db2       | **Multithreaded**             | Yes                  | Supports intra-query parallelism              |
| SQLite        | **Single-threaded (Configurable)** | No               | Designed for lightweight, embedded use        |
| Amazon Aurora | **Thread-based (Optimized MySQL/PostgreSQL)** | Yes | Cloud-native optimizations |
- Each approach has trade-offs:
    - Thread-based systems (MySQL, SQL Server) have lower overhead per connection but may require thread-pooling for efficiency.
    - Process-based systems (PostgreSQL, Oracle in multiprocess mode) are more robust against crashes but have slightly higher overhead.
    - Hybrid models (SQL Server, Oracle) try to balance the benefits of both.
- `your table is new and you inserted few row like 2-3, now you suddenly insert million rows and also start querying, db will think you only have 2-3 values so it will do an heap scan and you can get literally FUCKED UP if there are multiple queries getting triggered, so after inserting values do vaccum etc like command in Postgres to update, can always check analyse/explain before triggering a query`
- select ......someQUery ; //+index[ someIndex ] in comment write, DB will use index, `USE WITH CAUTION`, basically saying to optimiser I user/application is more knowlegeble about the DB
- OR,Join etc different queries can behave differently but you get the point
- If you create an index over a column, but most column have same values, doesn't result in great help during query execution

#### Create Index Concurrently
- `creating index temporarily stops writes over the table(depends on DB) , so take special care in PROD DB, reads works fine`
    - in postgres they created a workaround `create index concurrently on ......` so can read and write, basiclaly postgres does various scans and stops for any transaction to finish before pausing so will take way way more time, but no blocking, more cpu usage , and may fail as , so have to drop and recreate the index
    - most time you don't care about the time to create index and must want read , write to go through so concurrently would be better
    
| Database    | Default Index Creation Blocks Writes? | Concurrent Indexing Available? | Workaround for Concurrent Writes |
|------------|--------------------------------------|------------------------------|----------------------------------|
| MySQL      | Yes (Default)                        | Yes (MySQL 8 `ONLINE = ON`)  | Use `ALTER TABLE ... ADD INDEX, ONLINE=ON` |
| PostgreSQL | Yes (Default)                        | Yes (`CREATE INDEX CONCURRENTLY`) | Use `CREATE INDEX CONCURRENTLY` |
| SQLite     | Yes (Full DB Lock)                   | No                           | No workaround |
| SQL Server | No (If `ONLINE = ON` is used)        | Yes                          | Use `WITH (ONLINE = ON)` |
| Oracle     | No (If `ONLINE` is used)             | Yes                          | Use `CREATE INDEX ... ONLINE` |
| IBM Db2    | No (If `ALLOW WRITE ACCESS` is used) | Yes                          | Use `ALLOW WRITE ACCESS` | 
- Key Takeaways
    - PostgreSQL and MySQL require explicit options (CONCURRENTLY, ONLINE=ON) to allow concurrent writes.
    - SQLite does not support concurrent writes during indexing.
    - SQL Server, Oracle, and IBM Db2 provide better built-in support for online index creation.

### Bloom filters
- You have user in DB table, lot of query will involve around finding an User etc and each time going to DB will make it slower
- REDIS(Cache) get's introduced will help in reduce latency as DB hit will reduce and cache will hit, but some came up with other effcient solution namely `Bloom Filters`
- Basically create a hash and array/map of all possible values of hash result and mark true if present hash value at POST request time, ofcourse there will be collissions
    - On marked true/1 value might be present now hit DB, save some IOs, query hit to DB
    - Not perfect but good, Cassandra USES this consistent hashing[and many others]
    - we keep false positive probability/collision to from 0.1% to 1%
    - Of course actual implemetation of bloom filter is differnt, they use fancy thing and three hash function etc
### My OWN KEEDA(System Design Analysis for Bloom Filter)
**Bloom Filter Disk Size Estimation**

### **Key Factors Affecting Size**
1. **Number of elements (`n`)** → e.g., 1,000 to 1 billion users.
2. **False positive probability (`p`)** → e.g., 1% (0.01) or lower.
3. **Bit array size (`m`)** calculated using:
   \[
   m = -(n * ln * p)/((ln 2)^)2
   \]
4. **Optimal number of hash functions (`k`)**:
   \[
   k = (m)/(n)* 2
   \]

### **Size Calculation for Different Scales (`p = 0.01`)**
| Users (`n`)   | Bit Size (`m`) (bits) | Size in Bytes | Size in MB |
|--------------|----------------------|--------------|-----------|
| 1,000       | 9,585                 | ~1.2 KB      | ~0.001 MB |
| 1 Million   | 9.58 Million          | ~1.2 MB      | ~0.0012 GB |
| 1 Billion   | 9.58 Billion          | ~1.2 GB      | ~1.2 GB |

### **Observations**
- **Bloom filter size grows linearly with `n`**.
- **Lower false positive rate (`p`) increases memory needs**.
- **For 1 billion users at 1% false positives, ~1.2 GB is required**.
- **Use disk storage (e.g., memory-mapped files) for large-scale filters**.

### **Optimizations**
- **Reduce false positives (`p`)** → Increase `m`, but requires more space.
- **Parallelize hash functions (`k`)** for faster insertions.
- **Use partitioned or scalable Bloom filters** to adapt dynamically.

- When to use bloom filter vs Cache
    - Use a Bloom Filter when:
    You only need to check existence, not retrieve data.
    You want to reduce expensive DB queries by filtering non-existent lookups.
    Memory is very limited, and some false positives are acceptable.
    - Use a Cache (e.g., Redis) when:
    You need actual values (not just existence).
    You want fast lookups for frequently accessed data.You can afford the higher memory cost of storing full data.
- Can use hybrid approach as well i.e. use both bloom filter and cache
### Working with Billion-Row Table
- Basically on designing/using an DB , need to antcipate approx how much the data and rows will grow into in 1,2,5 years....
- On table, you can break table and search concurrently
- Can I not search whole table and look over only the subset? yeah creating indexing and query over it is the answer. So from billion rows we got into let's say million with index
- Can i reduce it even more? So do partitioning, a table is partitioned based on some column into various parts, so which part to search? Need indexing, so it reduces it more
- Can you distrubte to multiple hosts ? Yeah sharding but create problem of trasanction joins etc
- So shard -> partition -> index -> to row/s
- But what if we can avoid this billion row table?Design different if possible(First point) 

## B-Tress vs B+ Trees(In PROD DB System)
- Most DBs if not all are using B+Trees and not B-Trees
### Full Table Scans
- Reading full table, reading all pages page by Page
- Many I/O, large table require lot of time
- Need way to reduce time and I/O
### B-Tree
- Balanced DB for faster traversal, has nodes
- B-Tree has degree let's says 'm' meaning a node can have max child m
- Node has upto (m-1) elements
    - Each element has key and value, value is usually data pointer to row
    - data pointer can point to primary key or tuple
    - Root Node, internal node and leaf nodes
    - A node = disk page
#### How B-Tree Helps in Performance
- Key is basically index element, and it is sorted based on that and also has value which is rowId/page number where to get info from so it will be quicker, B-Tree is similar to Binary Search Tree, but there will be more m childs etc
- So searching will be faster and quicker , and you get the exact page you get full row data from
#### Limitations
- Storing key,value pair in each node, increases space used, so limitation, and ultimately slower
- Range query slower because of random access
- B+ Tree solve both these problems

### B+ Tree
- Root and Internal nodes save only keys and not values so less size, leaf keys save values
- Leaf nodes are linked to each another so you find one can prev and next one, great for range queries
#### B+ Tree & DBMS Consideration
- Extra cost minimum for all key value saved in leaf nodes, 1 node fits a DBMS page
- Can fit internal node easily in memory for fast traversal, leaf node live in data files in heap, can also fit in inMemory
- Most DBMS use B+ trees
#### B+ Tree Storage Cost In PostgreSQL vs MySQL
- Postgres(Tuples) Mysql(Primary Key)
- if primary key data is large can be expensive, as all secondary will point to primary key for MySQL like DB
- Leaf nodes in MYSQL(innoDB) contains the full row since it's an IOT/cluster index , so both advantages and disadvantages

## DataBase Partitioning
- Splitting a big table into various parts and deciding which part of table to hit I/O on , basis of a condition where
- Let's say customer table with 1 million rows, partitioned into 5 parts with 200k each records , ofcourse it will have same schema
### Vertical vs Horizontal Partitioning
- Horizontal Partitioning splits rows into partitions by ranges,by list etc
- Vertical partitioning splits column partitioning, like split a column of blob etc, and put it differently in slow access drive and other can be fast access(consider partioning means horizontal only)

### Partitioning Types
- By Range : dates , id etc
- By List : distinct Values like some state name, zip code etc
- By Hash : consistent hashing like say on ip

### Horizontal Partitioning vs Sharding
- HP splits big table into multiple tables in same DB , client is agnostic(client doesn't care, and doesn't know)
- Sharding splits big table but into multiple tables across different servers(distributed system)
- HP table name changes(or schema?!)
- Sharding everything is same but server changes

### Creating Partition
```
create table tableName (id serial not null, clm not null) parition by range(clm) // the column you are doing partition on shouldn't be null

// then create table which are partitions, I made 4 partitions of 25 values each
create table clm0025 (like tableName including indexes) //1 
create table clm2550 (like tableName including indexes) //2
create table clm5075 (like tableName including indexes) //3
create table clm75100 (like tableName including indexes)//4

//after creating all tables all partitions ones and original one, now attach one by one

alter table tableName attach parition clm0025 for values from (0) to (25);
alter table tableName attach parition clm2550 for values from (25) to (50);
alter table tableName attach parition clm5075 for values from (50) to (75);
alter table tableName attach parition clm75100 for values from (75) to (100);
// to is not included so 75 to 100 will not have 100 , if need 100 create from 75 to 101, we are assuming there won't be 100
//now if you insert into tableName it will save it and also depending upon the value it will get stored to respective one partition table

// since postgres 12 or something if you create an index on main table i.e. tableName here all partioned table will have index created as well
create index some_index_name on tableName(clm);
```
- benefit of partioning and index start when the original index(that is all values) are so big that can't fit into ram, but after partioning the index to search is smaller can be fast
- enable_partitioning_pruning this should be true for using index in intended way , should be on but mentioned just in case

### Pros Of Partitioning
- Improves query perforamnce when accessing a single(or few) partition vs a single huge table
- Sequential Scan Vs Scattered Index Scan is easier for decide for DB
- easy bulk loading attach partition
- archive old data that are barely accessed into cheap storage(in mariaDb/mysql can store in csv as well)
### Cons Of Partitioning
- Updates that move rows from one partition to another is slower or may fail
- inefficient queries can result in scanning all partitions
- Schema changes are challengin(DBMS can manage it though)

`you can write script to automate partitioning and create tables etc`

## Sharding
- On a single server as data and queries increases , processing becomes slower and slower 
- so people started dividing db into more servers/systems based on certain field, on some fields it is easy on some other it is troublesome, and consistent hashing is used
- consistent hashing returns server on which a data should be stored to , so on querying we can query that particular db server
- ***Horizontal Partitioning is done on same server data divided into multiple sub tables but Sharding data is divided into several server with all properties of table being equal, sharding everything is same but server changes***
- basically using hash and different servers stores values for sharding and then fetching from those server only
### Pros Of Sharding
- Scalibility : Both data,memory
- Security : users can access certains shards and not some other certain shards
- Optimal as size increases and smaller index
### Cons Of Sharding
- Complex Code(Client aware of shard)
- Transactions across the shards(how will handle atomiticity etc)
- RollBacks
- Schema changes are hard
- Join issue
- Key Has to be something you know in the query each time else won't know where the data resides
### When should you consider to shard your database
- It should be last thing you want to do
- Most time you can do something else like horizontal partitioning
- of course you can add caching  redis etc but other than that there are also other ways
- You can do replication as well, so backup as well as reading from various servers, and write in only one and push changes to other servers
- you can have several master according to geo location or some other criteria and most of the time they are pretty much separated but still if it is complex/not working go do sharding
- Sharding affects transaction etc working so be careful
- YouTube ran into similar problem(before google acquired them) , they had a single MySQL db, they did everything before finally write became so difficult they had to shard, application was aware of shard, this created coupling bad, so they created `Vittle` which is middleware for sharding , it decides which shard to hit
- Basically in one line before sharding always ask `Do You Really need it that badly`

## Concurrency Control
### Exclusive Lock vs Shared Lock
- Exclusive Lock(updating value)
    - Want to make sure when I work on some value/updating it that time nobody else is reading that value for security/consistency issue, and I am the only one udpating it
- Shared Lock(reading value)
    - when I Read some value from DB nobody is changing it, so many can acquire it
- can have both shared and exclusive lock together on some common value
- both are introduced for maintaning consistency specially needed in like a banking system
### DeadLock (DB specific)
- deadlock happens when two or more process are waiting for other process to release a lock while themselfs creating a lock on resource which is required by another, so infinite lock/wait
- Many DB auto catches these deadlocks and fail respective trasactions
```
T1                                          |        T2
begin trasaction                            | begin trasaction  

insert value into xyz values (20,...);      | insert value into xyz values (21,...); 
insert value into xyz values (21,...);      | insert value into xyz values (20,...);
//assuming this was first                   | //assuming this was last to enter deadlock 

T2 will fail with deadlock detected, T1 will stay active
```
### Two Phase Locking
- Basically working with idea of in first phase only acquire locks and second phase only release locks, once release can't aquire again
- An Example : Double booking , Cinema Hall, two users trying to book same seat same time
- so having an exclusive lock will avoid that `adding for update after you query obtains exclusive lock`
- `read more video 73`

### Offset SQL query is Bad
- select * from table offset x limit 10, DB select x+10 tables then remove x rows from it, so can be very slow
- instead ask user what last id it seen and get next 10 by order some desc, asc so it be will be faster and it has to hit only nearby 10 and not `offset+limit` rows

### Database Connection Pooling
- Database pooling is creation of a fixed number of connection established and using it when a query/thread executes
- Pooling and querying is faster as we don't have to wait for aquiring a connection

## Database Replication
- Repliaction between databases involve sharing info as to insure constintency between redundant dbs, Improve reliability, fault tolerence, accessibility in db

### Master/Standy(BackUp Replication)
- One master/leader node that accepts writes/ddls. One or more backup/standby nodes that recieve those writes from master
- Simple to implement, especially if can handle eventual consistency, or need consistent data have ways to solve that or read from master at that time
### Multi Master Replication
- Bit complex from single master one, more than one master and other slave node
- Need to resolve conflicts etc
### Synchronous vs Asynchronous Replication
- Sync Replication : a write trasaction to the master will be blocked until it is written to the backup/standby nodes
    - First 2,First 1, Any (don't wait for all)
    - so slower but full consistency not eventual consistency
- Async Replication : a write trasaction is consisdered successful if it's written to master db ,client got success, then asynchronous writes to backup nodes
- can manually change config file to attach one postgres db to another creating master and standy nodes.You can also mention in properties for making it synchronous or async
### Pros and Cons
- Pros
    - Horizontal Scaling
    - Region based queries , DB per region
- Cons
    - Eventual consistency
    - slow writes(synchronous)
    - complex implementation(multi - master)

## DB System Design
- CouchDB connects over Http/Https, So interact via RestAPIs
- Most databases (PostgreSQL, MySQL, MongoDB, etc.) use efficient binary protocols for better performance.

| **Database**   | **Protocol Used**             | **Type**     |
|---------------|------------------------------|-------------|
| **CouchDB**   | HTTP(S) (REST API)            | **Text-based** |
| **PostgreSQL** | PostgreSQL Binary Protocol   | **Binary** |
| **MySQL**      | MySQL Protocol (over TCP)    | **Binary** |
| **MongoDB**    | Mongo Wire Protocol          | **Binary** |
| **Redis**      | RESP (Redis Serialization Protocol) | **Binary** |
| **Cassandra**  | CQL Native Protocol          | **Binary** |

### TWITTER DB/System Design

| **Feature**            | **HTTP/1.1**         | **HTTP/2**         | **HTTP/3**         |
|-----------------------|--------------------|-------------------|-------------------|
| **Transport Protocol** | TCP               | TCP              | **QUIC (UDP-based)** |
| **Multiplexing**       | ❌ No (One request per connection) | ✅ Yes | ✅ Yes, improved (No head-of-line blocking) |
| **Header Compression** | ❌ No              | ✅ Yes (HPACK)  | ✅ Yes (QPACK, optimized for QUIC) |
| **Request Prioritization** | ❌ No | ✅ Yes | ✅ Yes |
| **Server Push**        | ❌ No | ✅ Yes | ✅ Yes |
| **Head-of-Line Blocking** | ❌ Yes (TCP delays all requests) | ❌ Yes (TCP-level) | ✅ No (QUIC prevents it) |
| **Encryption (TLS)**   | 🔹 Optional | 🔒 Required (TLS 1.2/1.3) | 🔒 Always encrypted (TLS 1.3) |
| **Adoption**           | 🌍 Still used | 🚀 Widely used | ⚡ Emerging (Google, Cloudflare, CDNs) |

`Twitter and URL shortner Watched video was good can't make notes`

## Database Engines
- These are software libraries which database management s/w uses to store low level data on disk or do CRUD
### What is A DB ENGINE?
- Library that take care of the on disk storage and CRUD
    - can be as simple as key-value store
    - Or as RICH and complex as full support ACID and trasactions with foreign keys
- DBMS can use the DB engine(Also called storage engine) and build features on top(server replication, isolation,stored procedures, etc..)
- Want to create new DB? don't start from scratch use DB engine
- Some DB gives felxibility to swtich engines like MySQL & MariaDB, other DBMS comes with built-in Engine and can't change like PostgreSQL

### MyISAM
- Stand for Index Sequetial Access method
- B-Tree indexes point to rows directly
- No transaction support
- Open Source & owned by oracle now(ealrier different).Inserts are fast, updates/deletes are problematic(fragments,offsets changes)
- DB crashes corrupts table(manually repair)
- Table level locking. Mysql,MariDB, Percona(mysql forks) support MyISAM, previously was default engine of mysql
    - ARIA engine was created as fork of mysql, mariadb , named after daughters, created after oracle acquired them,created specifically for MARIADB,  made for crash safety

### InnoDB
- Trasactional DB storage Engine, B+ trees with indexes points to PK and PK points to the row, must have PK, if not it will create one
- Default for MySQL & MariaDB, owned by oracle
- ACID compliant trasactional support
- Foreign keys, tablespaces, row level locking, spaital operations
    - Fork XtraDB engine was created and later went ineffective

### SQLite
- Lightweight , embeeded data used engine (like H2 DB), local data
- FULL ACID & table locking, B-TREE (LSM extension)
- Consurrent read& writes, Web SQL uses it
- Windows, maybe linux have it by default

### BerkeleyDB
- Now owned by Oracle
- Key-Value embedded DB
- Supports ACID, locks,replication etc
- Used to be used in bitcoin core(switched to levelDB)
- used MemcacheDB(not cache one it's different)
- Thrashed by levelDB and RocksDB

### LevelDB
- By Google 2011, LSM tree, great for high insert and SSD, B-TREE were slower as every insert may retrigger rebalance, LSM is O(1) insert, B-Tree with SSD bad, never UPDATE, No deletions maybe
- No trasactions, inspired by Google BigTable
- Levels of File
    - MemTable
    - Level 0 (young level)
    - Level 1 - 6
- WAL is supported, so in memeory but stored in disk 
- As files grow big levels are merged
- Used in bitcoin core blockchain,AutoCAD,Minecraft

### RocksDB
- FB looked LevelDB and created RockSB according to their usage(for from levelDB)
- Transactional, High Performance, Multi-Threaded compaction
- Allow read, write
- MyRocks for MySQL, MariabDB, percona
- MongoRocks for MongoDB , many many more started using it

### B/B+ Tree vs. LSM Tree in Databases

| Feature            | **B/B+ Tree** | **LSM Tree** |
|--------------------|-------------|-------------|
| **Data Structure** | Balanced Tree (Sorted Index) | Log-based Merge Tree (Append-Only) |
| **Read Performance** | ✅ **Fast** (O(log N)) | ❌ **Slower** (Multiple SSTables) |
| **Write Performance** | 🔹 **Moderate** (Random I/O) | ✅ **Very Fast** (Sequential Writes) |
| **Range Queries** | ✅ **Efficient** (Sequential Leaf Node Traversal) | ❌ **Slower** (Data Spread Across Levels) |
| **Disk Usage** | 🔹 **More Disk Writes** | ✅ **Efficient with Compaction** |
| **Concurrency** | 🔹 **Higher Read Performance, Lower Write Throughput** | ✅ **Optimized for High-Throughput Writes** |
| **Best For** | **Read-Heavy Workloads** (OLTP, Relational DBs) | **Write-Heavy Workloads** (NoSQL, Analytics, Logging) |
| **Example Databases** | ✅ MySQL (InnoDB), PostgreSQL, Oracle, SQL Server | ✅ Cassandra, RocksDB, LevelDB, Bigtable, ScyllaDB |
- single key LSM/NoSQL is faster range is slower

### MySQL with different Engines
- You can switch engine in mysql , not allowed in postgresql
- Can create a table with one engine and another with anotehr engine
```
show engines; //command for listing all engines currently availble and if can be used or not and other info
create table ..... engine = engineName // for creating table with that engineName
<!-- differnt engine will behave different way like transactions etc MyISAM doesn't support transaction, InnoDB does -->
```

## DB Cursors
- A database cursor is a pointer used to iterate over query results row by row. It allows controlled access to result sets, especially for large datasets, without loading everything into memory at once
    - Common in: SQL databases (MySQL, PostgreSQL, SQL Server).
    - Used for: Fetching, updating, and deleting records sequentially.
    - Saves lot of memory at client side especially, to work slowly
- In Paging(not easy),Streaming as well, can use the cursor, to pull data one by one and cancel query in between
### Cons
- It is stateful, so can't be shared from other transaction, other server. In devops can make sure to go to same server, same trasaction could be troublesome but good use of cursor
- Without trasnactions can't have cursor, so Long Trasaction is what we are looking into troublesome, other work is stopped as locked for this cursor's transaction
### Server Side vs Client Side(Java,Python etc services) Cursor
- `not sure what but client side your cursor fetch everything and server side(db handle cursor) keep it on server and send required one fast`
- Client-Side Cursors → Faster for small datasets, but use more memory.
- Server-Side Cursors → Better for large datasets, but slower due to multiple fetches.

## SQL vs NoSQL architecture
### MongoDB & It'sArchitecture
- MongoDB is document((Binary JSON)(BSON)) based NoSQL DB.Become popular becuse of it's way of storing data in schemaless way
- `can write and see more`
## DB Security
### By Enabling SSL/TLS
- Need to encrypt connection for data security over the connection
- can change conf file of postgres for enabling ssl but in order to do that need ssl-cert-file and ssl-cert-key(private key)(both pem)
- can enforce ssl/tls or even if we gave certs it will allow to connect without ssl/tls as well
- `can see more`
> Free version of postgres 14 MB worth of query crashes the DB(millions of or condition was added in statement condition)
### Best Practices Working with REST & Databases & Permissions
- Is it a good idea to check if a table is present in DB before application starts up and if not create it?
    - yeah it's fine but now your application has DDL permission , instead of DML or DQL which can be used to harm DB using SQL injection and drop table
    - So can create schema with completely different script and user and different user for application
- Don't store pass in text, store in some valut and make it/generate it most random and long

## HomoMorphic Encryption
### Encryption
- Using a symmetric key over a data to change it's form so as to be not readable, if using same key can be decrypted is called symmetric encryption and if using different keys called asymmetric encryption
- We encrypt as much as possible for data security, but can't always encrypt it
- Like want to do operation over data can't be done on encrypted one, so can't keep it encrypted.Indexing, Ananlysis, Tuning. TLS termination on layer 7 in reverse proxies and Load Balancing

### What is HomoMorphic Encryption?
- No need to decrypt data for performing operations, but it is very slow. Can use index etc
-  So without secret key which only you will keep rest all places data will be encrypted, so no issue of keeping data in cloud etc
- Very slow querying in hundreds can take upto minutes `yeah plural`, not ready of PROD yet

### Just other INFO from GPT, Please Verify and Read with Caution

# 🗃️ Types of Cache Policies (Caching Strategies)

Cache policies define how data is stored, updated, and evicted in a cache. Below are the main caching strategies:

---

## **1️⃣ Cache Invalidation Policies**
These policies define **when to remove or update cached data**.

| **Policy**  | **Description** | **Example Use Case** |
|------------|----------------|----------------------|
| **Write-Through** | Writes data to both **cache and database** simultaneously. Ensures consistency but can be slow. | Used in **financial applications** where consistency is critical. |
| **Write-Back (Write-Behind)** | Writes data **only to cache first**, then asynchronously updates the database. Faster but risks data loss. | **Shopping carts, analytics logs** (high-speed writes). |
| **Write-Around** | Writes data **only to the database**, bypassing the cache. Reduces cache pollution but increases read latency. | **Rarely accessed data**, e.g., historical records. |

---

## **2️⃣ Cache Eviction Policies**
These define **which data is removed when cache is full**.

| **Eviction Policy** | **Description** | **Example Use Case** |
|---------------------|----------------|----------------------|
| **LRU (Least Recently Used)** | Removes the **least recently accessed** item first. | **General-purpose caching** (e.g., Redis, Memcached). |
| **LFU (Least Frequently Used)** | Removes the **least accessed** items. **Better for long-term caching.** | **Machine learning models, recommendation engines.** |
| **FIFO (First In, First Out)** | Removes the **oldest** data first. **Doesn’t consider usage frequency.** | **Simple caching strategies (e.g., hardware caches).** |
| **Random Replacement** | Removes a **random** entry. Used when no clear pattern exists. | **Low-memory systems, distributed caches.** |
| **Time-Based Expiry (TTL)** | Items expire after a **fixed time (Time-To-Live)**. | **Session storage, authentication tokens.** |

---

## **3️⃣ Cache Placement Policies**
Defines **where caching occurs**.

| **Policy**          | **Description** | **Example Use Case** |
|--------------------|----------------|----------------------|
| **Global Cache** | Single cache shared by all users or requests. | **Database-level caching (e.g., Redis, Memcached).** |
| **Local Cache** | Each user/device has its own cache. | **Browser caching, mobile app caching.** |
| **Distributed Cache** | Cached data is spread across multiple servers. | **High-availability systems (e.g., CDN, AWS ElastiCache).** |

---

## **🚀 TL;DR**
- **Write Strategies:** _Write-Through, Write-Back, Write-Around._  
- **Eviction Policies:** _LRU, LFU, FIFO, Random, TTL._  
- **Placement Strategies:** _Global, Local, Distributed._

## **🚀 NewSQL/CockroachDB TO DO CAN Check**
| Feature              | NewSQL 🆕  | SQL (Relational) 🏛️ | NoSQL 🚀 |
|----------------------|-----------|---------------------|----------|
| **Scalability**      | ✅ Horizontally scalable | ❌ Mostly vertically scalable | ✅ Highly scalable (horizontal) |
| **Consistency**      | ✅ Strong consistency (ACID) | ✅ Strong consistency (ACID) | ❌ Eventual consistency (mostly) |
| **Query Language**   | ✅ SQL (PostgreSQL/MySQL-like) | ✅ SQL | ❌ Custom (CQL, N1QL, Mongo Query, etc.) |
| **Schema**          | ✅ Flexible schema (some allow JSON) | ❌ Strict schema (fixed tables) | ✅ Schema-less (flexible) |
| **Performance**      | ⚡ Fast for OLTP & analytics | 🏛️ Good for structured data | 🚀 Optimized for high-throughput workloads |
| **Use Case**         | 🌎 Global-scale applications, hybrid workloads | 📊 Transactional systems, banking, ERP | 📡 Big data, real-time analytics, IoT |
| **Examples**         | CockroachDB, Google Spanner, TiDB | PostgreSQL, MySQL, Oracle | MongoDB, Cassandra, Couchbase, DynamoDB |
- Scaling here means adding capacity, horizontal meaning more machines and vertical meaning adding more ram/rom etc

### `Clustered Index` means data is arranged in heap/memeory on that basis in a order, so by default that means a table can have at max only one clustered index. In Mysql,SQLServer etc primary key is taken as clustered index, in postgreSQL it makes a tuple and arranges data based on that or something else , so it's different from many other and Postgres by default doen't have cluster index but you can sepcifically mention CLUSTER in postgres command etc but that also after data updates they may have order disturbed.
### strategy for generation of primary key in hiberante/jpa/spring avoid using identity , can use sequence or UUID, identity is slower and DB generates it one by one so take time and can't do batch as don't have all key, in sequence also db generates it bu a sequence is created and we can fetch say 50 next keys for future usage or batch operations.Also identity can get stuck in lock etc if various instacnes are using
- UUID is generate by java on intance , nearly impossible for collision UUID use timestamp etc to avoid issue, we have UUID various version like v1 uses timestamp+mac address for uniqueness(most low chance as intance unique got) ,v4(fully random), v7(timestamp+randomness) 
    - The UUID is directly stored in the table as a CHAR(36), VARCHAR(36), or BINARY(16), depending on the database.Postgrs has native support for UUID

| Strategy  | Generates ID in DB?          | Stored in DB? | Requires Additional DB Object? | Suitable for Distributed Systems? |
|-----------|-----------------------------|--------------|------------------------------|---------------------------------|
| **IDENTITY** | ✅ Yes (Auto-increment)   | ✅ Yes       | ❌ No                        | ❌ No                           |
| **SEQUENCE** | ✅ Yes (Sequence Table)   | ✅ Yes       | ✅ Yes (Sequence Table)      | ❌ No                           |
| **UUID**     | ❌ No (Generated in App)  | ✅ Yes       | ❌ No                        | ✅ Yes                           |


## 📌 Database Objects: Stored Functions, Procedures, Triggers, and More  

Databases provide several powerful objects to optimize queries, enforce business rules, and automate processes. These include **Stored Functions, Stored Procedures, Triggers, Views, and Materialized Views**.

---

### 🔹 1. Stored Functions (`Returns a Value`)

- **A Stored Function** is a reusable block of SQL that **returns a single value**.  
- It can be used in **SELECT statements** or other queries.  
- Must have a `RETURN` statement.
    ```sql
    CREATE FUNCTION get_discount(price DECIMAL) 
    RETURNS DECIMAL DETERMINISTIC
    BEGIN
        RETURN price * 0.9; -- 10% discount
    END;
    <!-- 🔍 Usage -->
    SELECT get_discount(100); -- Returns 90
    SELECT product_name, get_discount(price) FROM products;
    <!-- 🔍 Where it’s Stored? -->
    
    SHOW FUNCTION STATUS;  -- MySQL
    
    SELECT proname FROM pg_proc WHERE proname = 'get_discount';  -- PostgreSQL
    ```

### 🔹 2. Stored Procedures (Execute Actions, No Return Value Required)

- A Stored Procedure is used to perform actions (e.g., update records, insert logs).
- Unlike functions, it does not have to return a value.
- Often used for batch processing or transactions.
    ```SQL
    CREATE PROCEDURE update_salary(IN emp_id INT, IN raise DECIMAL) 
    BEGIN
        UPDATE employees SET salary = salary + raise WHERE id = emp_id;
    END;
    🔍 Usage
    CALL update_salary(101, 500);
    🔍 Where it’s Stored?
    SHOW PROCEDURE STATUS;  -- MySQL
    SELECT proname FROM pg_proc WHERE proname = 'update_salary';  -- PostgreSQL
    ```
### 🔹 3. Triggers (Automatic Execution on Table Events)
- Triggers execute automatically when an event occurs (e.g., INSERT, UPDATE, DELETE).
- Useful for audit logging, validation, or preventing invalid updates.
    ```sql
    CREATE FUNCTION log_employee_update() RETURNS TRIGGER AS $$
    BEGIN
        INSERT INTO employee_audit(emp_id, old_salary, new_salary, changed_at)
        VALUES (OLD.id, OLD.salary, NEW.salary, NOW());
        RETURN NEW;
    END;
    $$ LANGUAGE plpgsql;

    CREATE TRIGGER before_salary_update
    BEFORE UPDATE ON employees
    FOR EACH ROW
    WHEN (OLD.salary <> NEW.salary)
    EXECUTE FUNCTION log_employee_update();
    🔍 Usage
    When an employee’s salary changes, an audit entry is automatically inserted.
    ```
### 🔹 4. Views (Read-Only Virtual Tables)
- Views store predefined SQL queries as virtual tables.
- Used to simplify complex queries and hide sensitive columns.
```sql
CREATE VIEW high_salary_employees AS
SELECT id, name, salary FROM employees WHERE salary > 50000;
🔍 Usage
SELECT * FROM high_salary_employees;
✔️ Read-only, does not store data separately!
```

🔹 5. Materialized Views (Stored Results for Faster Queries)
- Materialized Views are like views but store query results physically.
- Used for performance tuning when aggregating large datasets.
```SQL
CREATE MATERIALIZED VIEW sales_summary AS
SELECT region, SUM(sales) FROM sales_data GROUP BY region;
🔍 Usage
SELECT * FROM sales_summary;
✔️ Faster than normal views because data is precomputed!

🔍 Refreshing Data
REFRESH MATERIALIZED VIEW sales_summary;
```

### 📊 Comparison Table

| Feature              | Returns Value? | Used in Queries? | Stores Data? | Auto-Executes? | Best Use Case |
|----------------------|--------------|----------------|-------------|--------------|--------------|
| **Stored Function**  | ✅ Yes       | ✅ Yes        | ❌ No       | ❌ No       | Reusable calculations |
| **Stored Procedure** | ❌ No        | ❌ No         | ❌ No       | ❌ No       | Bulk updates, transactions |
| **Trigger**         | ❌ No        | ❌ No         | ❌ No       | ✅ Yes      | Enforcing business rules |
| **View**            | ✅ Yes       | ✅ Yes        | ❌ No       | ❌ No       | Simplifying queries |
| **Materialized View** | ✅ Yes      | ✅ Yes        | ✅ Yes      | ❌ No       | Improving performance |
### 📌 Summary
- Use Stored Functions for calculations that return a value.
- Use Stored Procedures for batch updates and multi-step transactions.
- Use Triggers for enforcing rules automatically.
- Use Views to simplify queries without storing data.
- Use Materialized Views to speed up queries by storing results.

## Transaction in SpringBoot
### @Transactional
- It is used for trasanction in spring, place over a method to make it a transaction so whenever any step fails in that method all data persisted will get reverted as rollback in DB. @Transactional(...severalProps...)

| **Property**        | **Default Value**         | **Recommended for Production**                 | **Description** |
|---------------------|-------------------------|---------------------------------|--------------|
| **Propagation**     | `REQUIRED`              | Keep `REQUIRED` or use `REQUIRES_NEW` for independent transactions | Determines how transactions should be propagated when a method is called within another transaction. |
| **Isolation**       | `DEFAULT` (DB-specific) | `REPEATABLE_READ` or `SERIALIZABLE` if strong consistency is needed | Defines the isolation level of the transaction to handle concurrency issues. |
| **Read-Only**       | `false`                 | `true` for read-only transactions | Prevents modifications when set to `true`, improving performance. |
| **Rollback Policy** | Rolls back on `RuntimeException` only | Explicitly set `rollbackFor` for checked exceptions (e.g., `SQLException`) | Defines when the transaction should be rolled back. |
| **Timeout**         | `-1` (infinite)         | Set a reasonable limit (e.g., `30` seconds) | Prevents long-running transactions from blocking resources. |

| **Propagation Enum/Allowed Props**        | **Description** |
|----------------------------|----------------|
| `REQUIRED`                 | **(Default)** If an existing transaction is present, use it; otherwise, create a new one. |
| `REQUIRES_NEW`             | Always creates a new transaction, suspending any existing one. Useful for independent operations like logging or auditing. |
| `SUPPORTS`                 | If a transaction exists, use it; otherwise, run without a transaction. This is useful for non-critical operations. |
| `NOT_SUPPORTED`            | Always runs **without** a transaction, suspending any existing transaction. Useful for operations that should never be transactional. |
| `MANDATORY`                | Requires an existing transaction; throws an exception if none exists. Used to enforce transactional context. |
| `NEVER`                    | Must **not** run within a transaction; throws an exception if a transaction exists. |
| `NESTED`                   | Executes within a nested transaction inside the existing transaction, allowing partial rollbacks. Requires database support (e.g., savepoints). |
- REQUIRES_NEW we can call a method in between of previous trasaction so a newer one will be created, so REQUIRES_NEW one will be successful if it was successful and worn't be affected if caller one trasacntion was failed/reverted (Required and Requires_New,Mandatory is widely used in industry)

### Transaction Isolation Levels in Spring (`@Transactional`)

| **Enum (`Isolation`)**         | **Isolation Level**        | **Description** | **Possible Issues** | **Recommended for Production?** |
|--------------------------------|---------------------------|----------------|--------------------|-------------------------------|
| `Isolation.DEFAULT`            | **Database Default**      | Uses the default isolation level of the **underlying database** (varies by DB). | Depends on DB (Usually **Read Committed**). | ✅ **Fine if DB settings are well-configured**. |
| `Isolation.READ_UNCOMMITTED`   | **Read Uncommitted**      | Allows reading **uncommitted changes** from other transactions. | **Dirty Reads**, Non-repeatable Reads, Phantom Reads. | ❌ **Not recommended** (Risk of inconsistent data). |
| `Isolation.READ_COMMITTED`     | **Read Committed**        | Only allows reading **committed data** from other transactions. | Non-repeatable Reads, Phantom Reads. | ✅ **Good default for most apps** (Prevents dirty reads). |
| `Isolation.REPEATABLE_READ`    | **Repeatable Read**       | Ensures the same row read twice within a transaction is consistent. | Phantom Reads (if not using gap locking). | ✅ **Useful for financial/banking apps**. |
| `Isolation.SERIALIZABLE`       | **Serializable**          | Fully isolates transactions by **locking rows or tables** to prevent anomalies. | Performance overhead due to locks. | ⚠️ **Use cautiously** (Slows down concurrent transactions). |

## **Production Recommendations**
- **Default (Read Committed) is safe for most applications**.
- **Use Repeatable Read for financial applications** where consistency is critical.(Blocks row which has been read to avoid updates/different value on re- read)
- **Serializable should be used sparingly**, as it **impacts performance**.
- **Read Uncommitted is almost never safe** unless required for debugging or logging.



## SQL Query optimizations(this is notes from a paper)
### **Tip #1**
- Use Column Names Instead of * in a SELECT Statement, If you are selecting only a few columns from a table there is no need to use SELECT *. Though this is easier to write, it will cost more time for the database to complete the query.
- 10-20% improvement
### **Tip #2**
- Avoid including a HAVING clause in SELECT statements
- The HAVING clause is used to filter the rows after all the rows are selected and it is used like a filter. It is quite useless in a SELECT statement. It works by going through the final result table of the query parsing out the rows that don’t meet the HAVING condition.
```SQL
Original query:
SELECT s.cust_id,count(s.cust_id)
FROM SH.sales s
GROUP BY s.cust_id
HAVING s.cust_id != '1660' AND s.cust_id != '2'; 

Improved query:
SELECT s.cust_id,count(cust_id)
FROM SH.sales s
WHERE s.cust_id != '1660'
AND s.cust_id !='2'
GROUP BY s.cust_id;

-- 25%ish time reduction
```
### **TIP #3**
- Eliminate Unnecessary DISTINCT Conditions
- Try in join etc where want to get distinct values if you can assign primary key with it as it's UNIQUE already
- Below query is nearly 10X better than with the query with having clause
```SQL
Original query:
SELECT DISTINCT * FROM SH.sales s JOIN SH.customers c
ON s.cust_id= c.cust_id
WHERE c.cust_marital_status = 'single'; 

Improved query:
SELECT * FROM SH.sales s JOIN SH.customers c
ON s.cust_id = c.cust_id
WHERE c.cust_marital_status='single';

-- above 85%ish time redcution
```
### **TIP #4**
- Un-nest sub queries
- Rewriting nested queries as joins often leads to more efficient execution and more effective optimization. In general, sub-query un-nesting is always done for correlated sub-queries with, at most, one table in the FROM clause, which are used in ANY, ALL, and EXISTS predicates. A uncorrelated sub-query, or a sub-query with more than one table in the FROM clause, is flattened if it can be decided, based on the query semantics, that the sub-query returns at most one row.
```SQL
Example:
Original query: SELECT *
FROM SH.products p WHERE p.prod_id =
(SELECT s.prod_id
FROM SH.sales s
WHERE s.cust_id = 100996
AND s.quantity_sold = 1 ); Improved query:
SELECT p.*
FROM SH.products p, sales s WHERE p.prod_id = s.prod_id AND s.cust_id = 100996
AND s.quantity_sold = 1;
-- 61% time reduction
```
### **TIP #5**
- Consider using an IN predicate when querying an indexed column
- The IN-list predicate can be exploited for indexed retrieval and also, the optimizer can sort the IN-list to match the sort sequence of the index, leading to more efficient retrieval. Note that the IN-list must contain only constants, or values that are constant during one execution of the query block, such as outer references.
```SQL
Example:
Original query: SELECT s.*
FROM SH.sales s WHERE s.prod_id = 14
OR s.prod_id = 17; Improved query:
SELECT s.*
FROM SH.sales s
WHERE s.prod_id IN (14, 17);
-- 73% timeReduction
```
### **TIP #6**
- Use EXISTS instead of DISTINCT when using table joins that involves tables having one-to-many relationships
- The DISTINCT keyword works by selecting all the columns in the table then parses out any duplicates.Instead, if you use sub query with the EXISTS keyword, you can avoid having to return an entire table.
```SQL
Example:
Original query:
SELECT DISTINCT c.country_id, c.country_name
FROM SH.countries c,SH.customers e
WHERE e.country_id = c.country_id;
Improved query:
SELECT c.country_id, c.country_name
FROM SH.countries c
WHERE EXISTS (SELECT 'X' FROM SH.customers e WHERE e.country_id = c.country_id);
-- 61% timeReduction
```
### **TIP #7**
- Try to use UNION ALL in place of UNION
- The UNION ALL statement is faster than UNION, because UNION ALL statement does not consider duplicate s, and UNION statement does look for duplicates in a table while selection of rows, whether or not they exist.
```SQL
Example:
Original query: SELECT cust_id FROM SH.sales UNION
SELECT cust_id FROM customers; Improved query: SELECT cust_id FROM SH.sales UNION ALL SELECT cust_id FROM customers;
-- 81% timeReduction
```
### **TIP #8**
- Avoid using OR in join conditions
- Any time you place an ‘OR’ in the join condition, the query will slow down by at least a factor of two.
```SQL
Example:
Original query: SELECT *
FROM SH.costs c
INNER JOIN SH.products p ON c.unit_price = p.prod_min_price OR c.unit_price = p.prod_list_price; Improved query:
SELECT *
FROM SH.costs c
INNER JOIN SH.products p ON c.unit_price =
p.prod_min_price UNION ALL SELECT *
FROM SH.costs c
INNER JOIN SH.products p ON c.unit_price = p.prod_list_price;
-- 70% timeReduction
```

### **TIP #9**
- Avoid functions on the right hand side of the operator
- Functions or methods are used very often with their SQL queries. Rewriting the query by removing aggregate functions will increase the performance tremendously.

```SQL
Example:
Original query:
SELECT *
FROM SH.sales
WHERE EXTRACT (YEAR FROM TO_DATE (time_id, ‘DD- MON-RR’)) = 2001 AND EXTRACT (MONTH FROM TO_DATE (time_id, ‘DD-MON-RR’)) =12;
Improved query:
SELECT * FROM SH.sales
WHERE TRUNC (time_id) BETWEEN TRUNC(TO_DA TE(‘12/01/2001’, ’mm/dd/yyyy’)) AND TRUNC (TO_DATE (‘12/30/2001’,’mm/dd/yyyy’));
-- 70% timeReduction
```

### **TIP #10**
- Avoid giving mathematics equations to be solved in sql query for eg insted of giving amount>100+150 give amount >250 , reduces time by 11%ish
