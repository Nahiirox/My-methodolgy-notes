# Types of Databases

Relational DBs (MySQL, MSSQL, PostgreSQL, Oracle) → use SQL → SQLi applies here. Tables linked by keys (id → foreign key). Schema = the map of those relationships — matters when crafting UNION-based injection to pull data across tables.

NoSQL (MongoDB, key-value stores) → no SQL, no tables → different injection class, different module. Don't conflate the two.