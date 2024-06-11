
Relational database has unparalleled power in querying and aggregating data. But there is a catch... performance. That's why modern RDBMS has equppped with sophisticated caching mechanism.  

- a cache to hold frequently accessed data blocks from disk, allowing faster access to frequently accessed data. The cache is managed using a least recently used (LRU) algorithm, data blocks to keep in the cache based on their frequency of access.
- a cache that stores parsed SQL statements, execution plans to save need for repetitive parsing and optimization, improving performance. 
- a cache that allows query results to be stored in memory. When a query is executed, it first checks if the result is already in the cache. If so, it retrieves the result from the cache instead of executing the query again, improving query response time. 

Caching, by it's nature, already becomes a sub-system of RDBMS in which complicated twist are required. 
