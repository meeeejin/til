# Undo

I learned from this [page](https://www.percona.com/live/mysql-conference-2014/sites/default/files/slides/InnoDB%20-%20A%20journey%20to%20the%20core%20II.pdf).

## Undo log

The undo log is used to undo (or revert) changes to data stored in InnoDB, and can be used to rollback transactions. Also, it is used to implement multi-versioning. And every undo log changes must **also** be redo logged!

### Rollback pointer (ROLL_PTR)

A **rollback segment number, page number, and page offset** pointing to a specific undo log record containing the previous version of a record.

- Used to walk backwards through record versions for any record.
- Used for multi-versioning.
- Used for transaction rollback.

### Transaction ID (TRX_ID)

A **64-bit unsigned integer** representing the point at which the *transaction started*.

- Incremented with each transaction.
- Written to each record in clustered (PK) indexes.
- Maximum value written to the system tablespace TRX_SYS page.

### Transaction serialization number (TRX_NO)

A **64-bit unsigned integer** representing the maximum TRX_ID at the time of commit.

- Written to undo log header on commit.
- Used for purge of old record versions.
