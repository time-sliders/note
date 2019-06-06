原文地址: [InnoDB Transaction Model](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-model.html)

In the `InnoDB` transaction model, the goal is to combine the best properties of a [multi-versioning](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_mvcc) database with traditional传统的 two-phase locking. `InnoDB` performs locking at the row level and runs queries as nonlocking [consistent reads](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_consistent_read) by default, in the style of Oracle. The lock information in `InnoDB` is stored space-efficiently so that lock escalation升级 is not needed. Typically, several users are permitted to lock every row in `InnoDB` tables, or any random subset of the rows, without causing `InnoDB` memory exhaustion耗尽.

> consistent read  一致读
>
> <u>A read operation that uses **snapshot** information to present query results based on a point in time, regardless of changes performed by other transactions running at the same time.</u> If queried data has been changed by another transaction, the original data is reconstructed based on the contents of the **undo log**. This technique avoids some of the **locking** issues that can reduce **concurrency** by forcing transactions to wait for other transactions to finish.
>
> <u>With **REPEATABLE READ** **isolation level**, the snapshot is based on the time when the first read operation is performed. With **READ COMMITTED** isolation level, the snapshot is reset to the time of each consistent read operation.</u>
>
> Consistent read is the default mode in which `InnoDB` processes `SELECT` statements in **READ COMMITTED** and **REPEATABLE READ** isolation levels. Because a consistent read does not set any locks on the tables it accesses, other sessions are free to modify those tables while a consistent read is being performed on the table.
>
> For technical details about the applicable isolation levels, see [Section 15.7.2.3, “Consistent Nonlocking Reads”](https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html).
>
> See Also [concurrency](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_concurrency), [isolation level](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_isolation_level), [locking](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_locking), [READ COMMITTED](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_read_committed), [REPEATABLE READ](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_repeatable_read), [snapshot](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_snapshot), [transaction](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_transaction), [undo log](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_undo_log).

#### 15.7.2.1 Transaction Isolation Levels

Transaction isolation is one of the foundations基础 of database processing. Isolation is the I in the acronym缩写 [ACID](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_acid); the isolation level is the setting that fine-tunes微调 the balance between performance and reliability, consistency, and reproducibility of results when multiple transactions are making changes and performing queries at the same time.

`InnoDB` offers all four transaction isolation levels described by the SQL:1992 standard: `READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ`, and`SERIALIZABLE`. **The default isolation level for `InnoDB` is** `REPEATABLE READ`.

A user can change the isolation level for a single session or for all subsequent connections with the `SET TRANSACTION`statement. To set the server's default isolation level for all connections, use the `--transaction-isolation`option on the command line or in an option file..

`InnoDB` supports each of the transaction isolation levels described here using different **locking** strategies. You can enforce a high degree of consistency with the default [`REPEATABLE READ`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read) level, for operations on crucial关键 data where [ACID](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_acid) compliance原则 is important. Or you can relax the consistency rules with [`READ COMMITTED`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_read-committed) or even [`READ UNCOMMITTED`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_read-uncommitted), in situations such as bulk reporting where precise consistency and repeatable results are less important than minimizing the amount of overhead for locking. [`SERIALIZABLE`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_serializable) enforces even stricter rules than [`REPEATABLE READ`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read), and is used mainly in specialized situations, such as with [XA](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_xa) transactions and for troubleshooting issues with concurrency and [deadlocks](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_deadlock).

The following list describes how MySQL supports the different transaction levels. The list goes from the most commonly used level to the least used.

- `REPEATABLE READ`

  This is the default isolation level for `InnoDB`. [Consistent reads](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_consistent_read) within **the same transaction read the [snapshot](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_snapshot) established by the first read**. This means that if you issue several plain (nonlocking) [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) statements within the same transaction, these [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) statements are consistent also with respect to each other相互一致. See [Section 15.7.2.3, “Consistent Nonlocking Reads”](https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html).

  > snapshot
  >
  > A representation of data at a particular time, which remains the same even as changes are **committed** by other **transactions**. Used by certain **isolation levels** to allow **consistent reads**.

  For [locking reads](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_locking_read) ([`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) with `FOR UPDATE` or `FOR SHARE`), [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html), and [`DELETE`](https://dev.mysql.com/doc/refman/8.0/en/delete.html) statements, locking depends on whether the statement uses a **unique** index with a unique search condition, or a range-type search condition.

  - For a unique index with a unique search condition, `InnoDB` locks only the index record found, not the [gap](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_gap) before it.
  - For other search conditions, `InnoDB` locks the **index range scanned,** using [gap locks](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_gap_lock) or [next-key locks](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_next_key_lock) to block insertions by other sessions into the gaps间隙 **covered** by the range. For information about gap locks and next-key locks, see [Section 15.7.1, “InnoDB Locking”](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html).

- `READ COMMITTED`

  Each consistent read, even within the same transaction, sets and reads its own fresh snapshot设置和读取自己的新快照. For information about consistent reads, see [Section 15.7.2.3, “Consistent Nonlocking Reads”](https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html).

  For locking reads ([`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) with `FOR UPDATE` or `FOR SHARE`), [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html) statements, and [`DELETE`](https://dev.mysql.com/doc/refman/8.0/en/delete.html) statements, `InnoDB` locks only index records, **not the gaps before them, and thus permits the free insertion of new records next to locked records.** Gap locking is only used for foreign-key constraint checking and duplicate-key checking.

  Because gap locking is disabled, phantom幻影 problems may occur, as other sessions can insert new rows into the gaps. For information about phantoms, see [Section 15.7.4, “Phantom Rows”](https://dev.mysql.com/doc/refman/8.0/en/innodb-next-key-locking.html).

  Only row-based binary logging is supported with the `READ COMMITTED` isolation level. If you use `READ COMMITTED` with[`binlog_format=MIXED`](https://dev.mysql.com/doc/refman/8.0/en/replication-options-binary-log.html#sysvar_binlog_format), the server automatically uses row-based logging.

  Using `READ COMMITTED` has additional effects:

  - For [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html) or [`DELETE`](https://dev.mysql.com/doc/refman/8.0/en/delete.html) statements, `InnoDB` holds locks **only** for rows that it updates or deletes. Record locks for nonmatching rows are released after MySQL has evaluated the `WHERE` condition. This greatly **reduces** the probability of deadlocks, but they can **still happen**.
  - For [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html) statements, if a row is already locked, `InnoDB` performs a “semi-consistent半一致” read, returning the latest committed version to MySQL so that MySQL can determine whether the row matches the `WHERE` condition of the [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html). If the row matches (must be updated), MySQL reads the row again and this time `InnoDB` either locks it or waits for a lock on it.

  Consider the following example, beginning with this table:

  ```sql
  CREATE TABLE t (a INT NOT NULL, b INT) ENGINE = InnoDB;
  INSERT INTO t VALUES (1,2),(2,3),(3,2),(4,3),(5,2);
  COMMIT;
  ```

  In this case, the table has no indexes, so searches and index scans use the hidden clustered index for record locking (see [Section 15.6.2.1, “Clustered and Secondary Indexes”](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html)) rather than indexed columns.

  Suppose that one session performs an [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html) using these statements:

  ```sql
  # Session A
  START TRANSACTION;
  UPDATE t SET b = 5 WHERE b = 3;
  ```

  Suppose also that a second session performs an [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html) by executing these statements following those of the first session:

  ```sql
  # Session B
  UPDATE t SET b = 4 WHERE b = 2;
  ```

  **As `InnoDB` executes each [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html), it first acquires an exclusive lock for each row, and then determines whether to modify it. If [`InnoDB`](https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html)does not modify the row, it releases the lock. Otherwise, [`InnoDB`](https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html) retains the lock until the end of the transaction**. This affects transaction processing as follows.

  When using the default `REPEATABLE READ` isolation level, the first [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html) acquires an x-lock on each row that it reads and does not release any of them:

  ```none
  x-lock(1,2); retain x-lock
  x-lock(2,3); update(2,3) to (2,5); retain x-lock
  x-lock(3,2); retain x-lock
  x-lock(4,3); update(4,3) to (4,5); retain x-lock
  x-lock(5,2); retain x-lock
  ```

  The second [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html) blocks as soon as it tries to acquire any locks (because first update has retained locks on all rows), and does not proceed until the first [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html) commits or rolls back:

  ```none
  x-lock(1,2); block and wait for first UPDATE to commit or roll back
  ```

  If `READ COMMITTED` is used instead, the first [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html) acquires an x-lock on each row that it reads and **releases** those for rows that it does not modify:

  ```none
  x-lock(1,2); unlock(1,2)
  x-lock(2,3); update(2,3) to (2,5); retain x-lock
  x-lock(3,2); unlock(3,2)
  x-lock(4,3); update(4,3) to (4,5); retain x-lock
  x-lock(5,2); unlock(5,2)
  ```

  For the second `UPDATE`, `InnoDB` does a “semi-consistent” read, returning the latest committed version of each row that it reads to MySQL so that MySQL can determine whether the row matches the `WHERE` condition of the [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html):

  ```none
  x-lock(1,2); update(1,2) to (1,4); retain x-lock
  x-lock(2,3); unlock(2,3)
  x-lock(3,2); update(3,2) to (3,4); retain x-lock
  x-lock(4,3); unlock(4,3)
  x-lock(5,2); update(5,2) to (5,4); retain x-lock
  ```

  However, if the `WHERE` condition includes an indexed column, and `InnoDB` uses the index, only the indexed column is considered when taking and retaining record locks. In the following example, the first [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html) takes and retains an x-lock on each row where b = 2. The second [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html) blocks when it tries to acquire x-locks on the same records, as it also uses the index defined on column b.

  ```sql
  CREATE TABLE t (a INT NOT NULL, b INT, c INT, INDEX (b)) ENGINE = InnoDB;
  INSERT INTO t VALUES (1,2,3),(2,2,4);
  COMMIT;
  
  # Session A
  START TRANSACTION;
  UPDATE t SET b = 3 WHERE b = 2 AND c = 3;
  
  # Session B
  UPDATE t SET b = 4 WHERE b = 2 AND c = 4;
  ```

- `READ UNCOMMITTED`

  [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) statements are performed in a nonlocking fashion, but a possible earlier version of a row might be used. Thus, using this isolation level, such reads are not consistent. This is also called a [dirty read](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_dirty_read). Otherwise, this isolation level works like [`READ COMMITTED`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_read-committed).

- `SERIALIZABLE`

  This level is like [`REPEATABLE READ`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read), but `InnoDB` implicitly converts all plain [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) statements to [`SELECT ... FOR SHARE`](https://dev.mysql.com/doc/refman/8.0/en/select.html) if[`autocommit`](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_autocommit) is disabled. If [`autocommit`](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_autocommit) is enabled, the [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) is its own transaction. It therefore is known to be read only and can be serialized if performed as a consistent (nonlocking) read and need not block for other transactions. (To force a plain [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) to block if other transactions have modified the selected rows, disable [`autocommit`](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_autocommit).)

#### 15.7.2.3 Consistent Nonlocking Reads

A [consistent read](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_consistent_read) means that `InnoDB` uses multi-versioning to present to a **query a snapshot of the database at a point in time**. The query sees the changes made by transactions that committed before that point of time, and no changes made by later or uncommitted transactions. The exception to this rule is that the query sees the changes made by earlier statements within the same transaction. This exception causes the following anomaly: If you update some rows in a table, a [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) sees the latest version of the updated rows, but it might also see older versions of any rows. If other sessions simultaneously update the same table, the anomaly means that you might see the table in a state that never existed in the database.

If the transaction [isolation level](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_isolation_level) is [`REPEATABLE READ`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read) (the default level), all consistent reads within the same transaction read the snapshot established by the first such read in that transaction. You can get a fresher snapshot for your queries by committing the current transaction and after that issuing new queries.

With [`READ COMMITTED`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_read-committed) isolation level, each consistent read within a transaction sets and reads its own fresh snapshot.

Consistent read is the default mode in which `InnoDB` processes [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) statements in [`READ COMMITTED`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_read-committed) and [`REPEATABLE READ`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read) isolation levels. A consistent read does not set any locks on the tables it accesses, and therefore other sessions are free to modify those tables at the same time a consistent read is being performed on the table.

Suppose that you are running in the default [`REPEATABLE READ`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read) isolation level. When you issue a consistent read (that is, an ordinary [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html)statement), `InnoDB` gives your transaction a timepoint according to which your query sees the database. If another transaction deletes a row and commits after your timepoint was assigned, you do not see the row as having been deleted. Inserts and updates are treated similarly.

Note

The snapshot of the database state applies to [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) statements within a transaction, not necessarily to [DML](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_dml)statements. If you insert or modify some rows and then commit that transaction, a [`DELETE`](https://dev.mysql.com/doc/refman/8.0/en/delete.html) or [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html) statement issued from another concurrent `REPEATABLE READ` transaction could affect those just-committed rows, even though the session could not query them. If a transaction does update or delete rows committed by a different transaction, those changes do become visible to the current transaction. For example, you might encounter a situation like the following:

```sql
SELECT COUNT(c1) FROM t1 WHERE c1 = 'xyz';
-- Returns 0: no rows match.
DELETE FROM t1 WHERE c1 = 'xyz';
-- Deletes several rows recently committed by other transaction.

SELECT COUNT(c2) FROM t1 WHERE c2 = 'abc';
-- Returns 0: no rows match.
UPDATE t1 SET c2 = 'cba' WHERE c2 = 'abc';
-- Affects 10 rows: another txn just committed 10 rows with 'abc' values.
SELECT COUNT(c2) FROM t1 WHERE c2 = 'cba';
-- Returns 10: this txn can now see the rows it just updated.
```

You can advance your timepoint by committing your transaction and then doing another [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) or [`START TRANSACTION WITH CONSISTENT SNAPSHOT`](https://dev.mysql.com/doc/refman/8.0/en/commit.html).

This is called multi-versioned concurrency control.

In the following example, session A sees the row inserted by B only when B has committed the insert and A has committed as well, so that the timepoint is advanced past the commit of B.

```sql
             Session A              Session B

           SET autocommit=0;      SET autocommit=0;
time
|          SELECT * FROM t;
|          empty set
|                                 INSERT INTO t VALUES (1, 2);
|
v          SELECT * FROM t;
           empty set
                                  COMMIT;

           SELECT * FROM t;
           empty set

           COMMIT;

           SELECT * FROM t;
           ---------------------
           |    1    |    2    |
           ---------------------
```

If you want to see the “freshest” state of the database, use either the [`READ COMMITTED`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_read-committed) isolation level or a [locking read](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_locking_read):

```sql
SELECT * FROM t FOR SHARE;
```

With [`READ COMMITTED`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_read-committed) isolation level, each consistent read within a transaction sets and reads its own fresh snapshot. With `FOR SHARE`, a locking read occurs instead: A `SELECT` blocks until the transaction containing the freshest rows ends (see [Section 15.7.2.4, “Locking Reads”](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html)).

Consistent read does not work over certain DDL statements:

- Consistent read does not work over [`DROP TABLE`](https://dev.mysql.com/doc/refman/8.0/en/drop-table.html), because MySQL cannot use a table that has been dropped and `InnoDB` destroys the table.
- Consistent read does not work over [`ALTER TABLE`](https://dev.mysql.com/doc/refman/8.0/en/alter-table.html), because that statement makes a temporary copy of the original table and deletes the original table when the temporary copy is built. When you reissue a consistent read within a transaction, rows in the new table are not visible because those rows did not exist when the transaction's snapshot was taken. In this case, the transaction returns an error:[`ER_TABLE_DEF_CHANGED`](https://dev.mysql.com/doc/refman/8.0/en/server-error-reference.html#error_er_table_def_changed), “Table definition has changed, please retry transaction”.

The type of read varies for selects in clauses like [`INSERT INTO ... SELECT`](https://dev.mysql.com/doc/refman/8.0/en/insert.html), [`UPDATE ... (SELECT)`](https://dev.mysql.com/doc/refman/8.0/en/update.html), and [`CREATE TABLE ... SELECT`](https://dev.mysql.com/doc/refman/8.0/en/create-table.html) that do not specify `FOR UPDATE` or `FOR SHARE`:

- By default, `InnoDB` uses stronger locks and the [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) part acts like [`READ COMMITTED`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_read-committed), where each consistent read, even within the same transaction, sets and reads its own fresh snapshot.
- To use a consistent read in such cases, set the isolation level of the transaction to [`READ UNCOMMITTED`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_read-uncommitted), [`READ COMMITTED`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_read-committed), or [`REPEATABLE READ`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read) (that is, anything other than [`SERIALIZABLE`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_serializable)). In this case, no locks are set on rows read from the selected table.

#### 15.7.2.4 Locking Reads

If you query data and then insert or update related data within the same transaction, the regular `SELECT` statement does not give enough protection. Other transactions can update or delete the same rows you just queried. `InnoDB` supports two types of [locking reads](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_locking_read) that offer extra safety:

- [`SELECT ... FOR SHARE`](https://dev.mysql.com/doc/refman/8.0/en/select.html)

  Sets a shared mode lock on any rows that are read. Other sessions can read the rows, but cannot modify them until your transaction commits. If any of these rows were changed by another transaction that has not yet committed, your query waits until that transaction ends and then uses the latest values.

  Note

  `SELECT ... FOR SHARE` is a replacement for `SELECT ... LOCK IN SHARE MODE`, but `LOCK IN SHARE MODE`remains available for backward compatibility. The statements are equivalent. However, `FOR SHARE` supports `OF*table_name*`, `NOWAIT`, and `SKIP LOCKED` options. See [Locking Read Concurrency with NOWAIT and SKIP LOCKED](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html#innodb-locking-reads-nowait-skip-locked).

- [`SELECT ... FOR UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/select.html)

  For index records the search encounters, locks the rows and any associated index entries, the same as if you issued an `UPDATE` statement for those rows. Other transactions are blocked from updating those rows, from doing `SELECT ... FOR SHARE`, or from reading the data in certain transaction isolation levels. Consistent reads ignore any locks set on the records that exist in the read view. (Old versions of a record cannot be locked; they are reconstructed by applying [undo logs](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_undo_log) on an in-memory copy of the record.)

These clauses are primarily useful when dealing with tree-structured or graph-structured data, either in a single table or split across multiple tables. You traverse edges or tree branches from one place to another, while reserving the right to come back and change any of these “pointer”values.

All locks set by `FOR SHARE` and `FOR UPDATE` queries are released when the transaction is committed or rolled back.

Note

Locking reads are only possible when autocommit is disabled (either by beginning transaction with [`START TRANSACTION`](https://dev.mysql.com/doc/refman/8.0/en/commit.html) or by setting [`autocommit`](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_autocommit) to 0.

A locking read clause in an outer statement does not lock the rows of a table in a nested subquery unless a locking read clause is also specified in the subquery. For example, the following statement does not lock rows in table `t2`.

```sql
SELECT * FROM t1 WHERE c1 = (SELECT c1 FROM t2) FOR UPDATE;
```

To lock rows in table `t2`, add a locking read clause to the subquery:

```sql
SELECT * FROM t1 WHERE c1 = (SELECT c1 FROM t2 FOR UPDATE) FOR UPDATE;
```

##### Locking Read Examples

Suppose that you want to insert a new row into a table `child`, and make sure that the child row has a parent row in table `parent`. Your application code can ensure referential integrity throughout this sequence of operations.

First, use a consistent read to query the table `PARENT` and verify that the parent row exists. Can you safely insert the child row to table `CHILD`? No, because some other session could delete the parent row in the moment between your `SELECT` and your `INSERT`, without you being aware of it.

To avoid this potential issue, perform the [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) using `FOR SHARE`:

```sql
SELECT * FROM parent WHERE NAME = 'Jones' FOR SHARE;
```

After the `FOR SHARE` query returns the parent `'Jones'`, you can safely add the child record to the `CHILD` table and commit the transaction. Any transaction that tries to acquire an exclusive lock in the applicable row in the `PARENT` table waits until you are finished, that is, until the data in all tables is in a consistent state.

For another example, consider an integer counter field in a table `CHILD_CODES`, used to assign a unique identifier to each child added to table`CHILD`. Do not use either consistent read or a shared mode read to read the present value of the counter, because two users of the database could see the same value for the counter, and a duplicate-key error occurs if two transactions attempt to add rows with the same identifier to the `CHILD` table.

Here, `FOR SHARE` is not a good solution because if two users read the counter at the same time, at least one of them ends up in deadlock when it attempts to update the counter.

To implement reading and incrementing the counter, first perform a locking read of the counter using `FOR UPDATE`, and then increment the counter. For example:

```sql
SELECT counter_field FROM child_codes FOR UPDATE;
UPDATE child_codes SET counter_field = counter_field + 1;
```

A [`SELECT ... FOR UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/select.html) reads the latest available data, setting exclusive locks on each row it reads. Thus, it sets the same locks a searched SQL [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html) would set on the rows.

The preceding description is merely an example of how [`SELECT ... FOR UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/select.html) works. In MySQL, the specific task of generating a unique identifier actually can be accomplished using only a single access to the table:

```sql
UPDATE child_codes SET counter_field = LAST_INSERT_ID(counter_field + 1);
SELECT LAST_INSERT_ID();
```

The [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) statement merely retrieves the identifier information (specific to the current connection). It does not access any table.

##### Locking Read Concurrency with NOWAIT and SKIP LOCKED

If a row is locked by a transaction, a `SELECT ... FOR UPDATE` or `SELECT ... FOR SHARE` transaction that requests the same locked row must wait until the blocking transaction releases the row lock. This behavior prevents transactions from updating or deleting rows that are queried for updates by other transactions. However, waiting for a row lock to be released is not necessary if you want the query to return immediately when a requested row is locked, or if excluding locked rows from the result set is acceptable.

To avoid waiting for other transactions to release row locks, `NOWAIT` and `SKIP LOCKED` options may be used with `SELECT ... FOR UPDATE` or `SELECT ... FOR SHARE` locking read statements.

- `NOWAIT`

  A locking read that uses `NOWAIT` never waits to acquire a row lock. The query executes immediately, failing with an error if a requested row is locked.

- `SKIP LOCKED`

  A locking read that uses `SKIP LOCKED` never waits to acquire a row lock. The query executes immediately, removing locked rows from the result set.

  Note

  Queries that skip locked rows return an inconsistent view of the data. `SKIP LOCKED` is therefore not suitable for general transactional work. However, it may be used to avoid lock contention when multiple sessions access the same queue-like table.

`NOWAIT` and `SKIP LOCKED` only apply to row-level locks.

Statements that use `NOWAIT` or `SKIP LOCKED` are unsafe for statement based replication.

The following example demonstrates `NOWAIT` and `SKIP LOCKED`. Session 1 starts a transaction that takes a row lock on a single record. Session 2 attempts a locking read on the same record using the `NOWAIT` option. Because the requested row is locked by Session 1, the locking read returns immediately with an error. In Session 3, the locking read with `SKIP LOCKED` returns the requested rows except for the row that is locked by Session 1.

```sql
# Session 1:

mysql> CREATE TABLE t (i INT, PRIMARY KEY (i)) ENGINE = InnoDB;

mysql> INSERT INTO t (i) VALUES(1),(2),(3);

mysql> START TRANSACTION;

mysql> SELECT * FROM t WHERE i = 2 FOR UPDATE;
+---+
| i |
+---+
| 2 |
+---+

# Session 2:

mysql> START TRANSACTION;

mysql> SELECT * FROM t WHERE i = 2 FOR UPDATE NOWAIT;
ERROR 3572 (HY000): Do not wait for lock.

# Session 3:

mysql> START TRANSACTION;

mysql> SELECT * FROM t FOR UPDATE SKIP LOCKED;
+---+
| i |
+---+
| 1 |
| 3 |
+---+
```