原文地址 [Phantom Rows](https://dev.mysql.com/doc/refman/8.0/en/innodb-next-key-locking.html)

**The so-called phantom problem occurs within a transaction when the same query produces different sets of rows at different times 同一个事物内的查询在不同时间返回不同的结果**. For example, if a [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) is executed twice, but returns a row the second time that was not returned the first time, the row is a “phantom” row.

Suppose that there is an index on the `id` column of the `child` table and that you want to read and lock all rows from the table having an identifier value larger than 100, with the intention of updating some column in the selected rows later:

```sql
SELECT * FROM child WHERE id > 100 FOR UPDATE;
```

The query scans the index starting from the first record where `id` is bigger than 100. Let the table contain rows having `id` values of 90 and 102. If the locks set on the index records in the scanned range do not lock out inserts made in the gaps (in this case, the gap between 90 and 102), another session can insert a new row into the table with an `id` of 101. If you were to execute the same [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) within the same transaction, you would see a new row with an `id` of 101 (a “phantom”) in the result set returned by the query. If we regard把.. a set of rows as视为 a data item, the new phantom child would violate the isolation principle of transactions that **a transaction should be able to run so that the data it has read does not change during the transaction**.

To prevent phantoms, `InnoDB` uses an algorithm called **next-key locking** that <u>combines index-row locking with gap locking</u>. `InnoDB` performs row-level locking in such a way that **when it searches or scans a table index, it sets shared or exclusive locks on the index records it encounters. Thus, the row-level locks are actually index-record locks. In addition, a next-key lock on an index record also affects the “gap” before that index record. That is, a next-key lock is an index-record lock plus a gap lock on the gap preceding the index record.** If one session has a shared or exclusive lock on record `R` in an index, another session cannot insert a new index record in the gap immediately before `R` in the index order.

**When `InnoDB` scans an index, it can also lock the gap <u>after</u> the last record in the index**. Just that happens in就发生在 the preceding example: To prevent any insert into the table where `id` would be bigger than 100, the locks set by `InnoDB` include a lock on the gap following `id` value 102.

You **can use next-key locking to implement a uniqueness check** in your application: If you read your data in share mode and do not see a duplicate for a row you are going to insert, then you can safely insert your row and know that the next-key lock set on the successor of your row during the read prevents anyone meanwhile inserting a duplicate for your row. Thus, the next-key locking enables you to “lock” the nonexistence of something in your table.

Gap locking can be disabled as discussed in [Section 15.7.1, “InnoDB Locking”](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html). This may cause phantom problems because other sessions can insert new rows into the gaps when gap locking is disabled.