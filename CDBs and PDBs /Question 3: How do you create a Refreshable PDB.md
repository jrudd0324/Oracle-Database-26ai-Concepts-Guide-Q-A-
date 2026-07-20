# Refreshable PDB Clones

**Question:** What are refreshable PDB clones?

**Answer:** Refreshable PDB clones are read-only copies of a source PDB that can be automatically or manually synchronized with their source over a database link. They are useful for reporting, testing, and environments where periodically refreshed copies of production data are required.

> **Note:** This example uses the `PDB01_DBLINK` database link created in a previous exercise.

**Question:** How do you create a refreshable clone?

**Answer:** Use the `REFRESH MODE` clause when creating the pluggable database.

```sql
CREATE PLUGGABLE DATABASE PDB02
FROM PDB01@PDB01_DBLINK
FILE_NAME_CONVERT = (
  '/u03/datafiles/ORCL/PDB01/',
  '/u03/datafiles/ORCL/PDB02/'
)
REFRESH MODE EVERY 1 HOURS;
```

**Question:** Can you manually refresh the PDB clone even though it has a schedule?

**Answer:** Yes. Use the `ALTER PLUGGABLE DATABASE ... REFRESH` statement.

```sql
ALTER PLUGGABLE DATABASE PDB02 REFRESH;
```

**Question:** What if the source PDB is read-only? Can you still refresh from it?

**Answer:** Yes. Since the refresh operation only reads data from the source PDB, the source can remain in read-only mode.

```sql
ALTER PLUGGABLE DATABASE PDB01 CLOSE IMMEDIATE;

ALTER PLUGGABLE DATABASE PDB01 OPEN READ ONLY;
```

```sql
ALTER PLUGGABLE DATABASE PDB02 REFRESH;
```

**Question:** Can you refresh the PDB if `PDB02` is read-only?

**Answer:** Yes. Refreshable clones are read-only by design. They cannot accept DDL or DML operations, but they can still be refreshed from their source.

**Question:** What happens if the source PDB is unavailable during a refresh?

**Answer:** The refresh fails because Oracle cannot connect to the source database over the database link. This can happen if:

- The source PDB is closed.
- The database link credentials are no longer valid.
- The database link user loses the `CREATE SESSION` privilege.

```sql
ALTER PLUGGABLE DATABASE PDB01 CLOSE IMMEDIATE;
```

```sql
ALTER PLUGGABLE DATABASE PDB02 REFRESH;
```

Expected error:

```text
ORA-17627: ORA-01109: database not open
ORA-17629: cannot connect to the remote database server
```

**Question:** When connecting to `PDB02`, are you actually connected to the source PDB?

**Answer:** No. You are connected to the refreshable clone (`PDB02`), which is an independent read-only PDB. The source PDB is only contacted during refresh operations.

```sql
ALTER SESSION SET CONTAINER = PDB02;
```

```sql
SHOW CON_NAME;
```

Expected output:

```text
CON_NAME
------------------------------
PDB02
```

**Question:** Can you create a refreshable clone from a local PDB?

**Answer:** No. Refreshable clones must be created using a database link. However, the database link can point back to a PDB in the same CDB, effectively allowing a local PDB to serve as the source.

## Question: Can you convert a refreshable clone into a standalone read/write PDB?

**Answer:** Yes. Disable refresh mode and reopen the PDB in read/write mode. Once this is done, the PDB is permanently converted into a standalone PDB and can no longer be refreshed.

```sql
ALTER PLUGGABLE DATABASE PDB02 CLOSE IMMEDIATE;
```

```sql
ALTER PLUGGABLE DATABASE PDB02 REFRESH MODE NONE;
```

```sql
ALTER PLUGGABLE DATABASE PDB02 OPEN READ WRITE;
```

**Question:** Can you refresh the PDB after converting it into a standalone PDB?

**Answer:** No. Once refresh mode has been removed, the PDB is no longer eligible for refresh operations.

```sql
ALTER PLUGGABLE DATABASE PDB02 REFRESH;
```

Expected error:

```text
ORA-65261: pluggable database PDB02 not enabled for refresh
```

**Question:** Can you make the standalone PDB refreshable again?

**Answer:** No. After a refreshable clone has been converted into a standalone PDB, it cannot be converted back. To create another refreshable clone, you must recreate it from the source PDB using `CREATE PLUGGABLE DATABASE ... REFRESH MODE`.
