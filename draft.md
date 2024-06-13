
### Always view from the other end of our solutions 

Relational database has unparalleled power in querying and aggregating data. But there is a catch... performance. That's why modern RDBMS has equipped with sophisticated caching mechanism.  

- a cache to hold frequently accessed data blocks from disk, allowing faster access to frequently accessed data. The cache is managed using a least recently used (LRU) algorithm, data blocks to keep in the cache based on their frequency of access.
- a cache that stores parsed SQL statements, execution plans to save need for repetitive parsing and optimization, improving performance. 
- a cache that allows query results to be stored in memory. When a query is executed, it first checks if the result is already in the cache. If so, it retrieves the result from the cache instead of executing the query again, improving query response time. 

Caching, by it's nature, already becomes a sub-system of RDBMS in which complicated twist are required. 

Being the best nature of RDBMS, [ACID](https://en.wikipedia.org/wiki/ACID) was groundbreaking concepts and state-of-the-art technology advocated in 1980s, ere computer systems were file based using index and sequential access. RDBMS quickly became popular and became more and more complicated and monolithic. At that time, no idea of cooperation between databases was suggested, although distributed database are discussed later on. Therefore, clustering in RDBMS is not always easy even in today. However, an easier approach is *master/slave replication* and can be used to implement *Read-Write Splitting* to increase system throughput. 

[MySQL 8.2 – transparent read/write splitting](https://dev.mysql.com/blog-archive/mysql-8-2-transparent-read-write-splitting/)

So, we are at a crossroads... Definitely we can't do without RDBMS for we have used it so long and so many important applications and data there... The expedient way is equipped a more powerful cache system and let it work along side with RDBMS, I think. 

Relational database is a good thing but is not always the best for everythng. That's why the idea of [modernize database](https://redis.io/blog/3-reasons-your-mysql-db-needs-redis/) emerges. It simply means to incorporate Redis, or something like that, to boost performance in order to meet new challenging requirements of modern applications.

If the goal of RDBMS is to persist data and use them to serve us, 
If the bottleneck is disk I/O, even though intricated caching mechanism is employed, 
even though SSD is employed in the stead of physical disk... 
What if... what if we hold the database in RAM, which is the fastest by far, and persist data to disk on schedule or base on certain events? 
Does it simplify our server design? 
Does it mitigate the plight of performance? 

By default, in Redis every time a [SAVE](https://redis.io/docs/latest/commands/save/) is made, a database shapshot is written to disk. And in between of [SAVE](https://redis.io/docs/latest/commands/save/), a kind of [Redo Log](https://dev.mysql.com/doc/refman/8.4/en/innodb-redo-log.html) is created and written to disk every second. That means within a second, if server crashes, you will lose data of the second... Which is acceptable in most situations and low-latency is maintained. 

RAM is an expensive asset. Most of us can't afford to purchase humungous amount of RAM comparing to disk. That's why the database in RAM won't get excessively large and so does the time to [SAVE](https://redis.io/docs/latest/commands/save/) won't get excessively long. 
Does it solve our persistence requirement? 
Does it solve our performance demand? 

Pursuing every CRUD operations in RDBMS, some invisible *domestic* workload is performed. Typically, if you use VARCHAR or TEXT type in table, a re-allocation of space may involve moving of data block, modifying pointer address, rebuilding indexes etc, all happens behind the scene on disk. Database holding in memory also subjects to similar issues but time to take are significantly shorter. 

It's compromise between performance and flexibility. We can't manage data with SQL, we lose fhe freedom of joining table and power of aggregation and as yet. What we gain is SPEED. 

[Case study: A self-declaration system](https://github.com/Albert0i/RU301/blob/main/cluster-case-1.md)

> Long story short, REDIS allows you to store key-value pairs on your RAM. Since accessing RAM is 150,000 times faster than accessing a disk, and 500 times faster than accessing SSD, it means speed.

[What is Redis?](https://adevait.com/redis/what-is-redis)

2024/06/14