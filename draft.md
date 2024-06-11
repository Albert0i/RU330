
Relational database has unparalleled power in querying and aggregating data. But there is a catch... performance. That's why modern RDBMS has equipped with sophisticated caching mechanism.  

- a cache to hold frequently accessed data blocks from disk, allowing faster access to frequently accessed data. The cache is managed using a least recently used (LRU) algorithm, data blocks to keep in the cache based on their frequency of access.
- a cache that stores parsed SQL statements, execution plans to save need for repetitive parsing and optimization, improving performance. 
- a cache that allows query results to be stored in memory. When a query is executed, it first checks if the result is already in the cache. If so, it retrieves the result from the cache instead of executing the query again, improving query response time. 

Caching, by it's nature, already becomes a sub-system of RDBMS in which complicated twist are required. 

Being the best nature of RDBMS, [ACID](https://en.wikipedia.org/wiki/ACID) was groundbreaking concepts and state-of-the-art technology advocated in 1980s, ere computer systems were file based using index and sequential access. RDBMS quickly became popular and became more and more complicated and monolithic. At that time, no idea of cooperation between databases was suggested, although distributed database are discussed later on. Therefore, clustering in RDBMS is not always easy even in today. However, an easier approach is *master/slave replication* and can be used to implement *Read-Write Splitting* to increase system throughput. 

So, we are at a crossroads... Definitely we can't do without RDBMS for we have used it so long and so many important applications and data there... The expedient way is equipped a more powerful cache system and let RDBMS work along side with it, I think. 

