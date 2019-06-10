Every `InnoDB` table has a special index called the [clustered index](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_clustered_index) **where the data for the rows is stored**. Typically, the clustered index is synonymous with the [primary key](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_primary_key). To get the best performance from queries, inserts, and other database operations, you must understand how `InnoDB` uses the clustered index to optimize the most common lookup and DML operations for each table.

- When you define a `PRIMARY KEY` on your table, `InnoDB` uses it as the clustered index. **Define a primary key for each table that you create**. If there is no logical unique and non-null column or set of columns, add a new [auto-increment](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_auto_increment) column, whose values are filled in automatically.
- If you do not define a `PRIMARY KEY` for your table, MySQL **locates the first `UNIQUE` index where all the key columns are `NOT NULL` and`InnoDB` uses it as the clustered index**.
- If the table has no `PRIMARY KEY` or suitable `UNIQUE` index, `InnoDB` **internally generates a hidden clustered index named`GEN_CLUST_INDEX` on a synthetic column containing row ID values.** The rows are ordered by the ID that `InnoDB` assigns to the rows in such a table. The row ID is a **6-byte field** that increases monotonically as new rows are inserted. Thus, the rows ordered by the row ID are physically in insertion order.

##### How the Clustered Index Speeds Up Queries

**Accessing a row through the clustered index is fast because the index search leads directly to the page with all the row data**. If a table is large, the clustered index architecture often saves a disk I/O operation when compared to storage organizations that store row data using a different page from the index record.

##### How Secondary Indexes Relate to the Clustered Index

All indexes other than the clustered index are known as [secondary indexes](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_secondary_index). **In `InnoDB`, each record in a secondary index contains the primary key columns for the row, as well as the columns specified for the secondary index. `InnoDB` uses this primary key value to search for the row in the clustered index.**

If the primary key is long, the secondary indexes use more space, so it is advantageous to have a short primary key.

For guidelines to take advantage of `InnoDB` clustered and secondary indexes, see [Section 8.3, “Optimization and Indexes”](https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html).