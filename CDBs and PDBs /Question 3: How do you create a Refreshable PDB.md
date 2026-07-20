# Refreshable PDB Clones

We will be exploring refreshable PDB clones, how to create one, how they behave, and how they relate to their source.

> **Note:** This example uses the `PDB01_DBLINK` database link created in Question 2.

**Question:** How do you create a refreshable clone?

**Answer:** With the `REFRESH MODE` clause.

```sql
CREATE PLUGGABLE DATABASE PDB02
FROM PDB01@PDB01_DBLINK
FILE_NAME_CONVERT = (
  '/u03/datafiles/ORCL/PDB01/',
  '/u03/datafiles/ORCL/PDB02/'
)
REFRESH MODE EVERY 1 HOURS;
```

```
Pluggable database created.
```

**Question:** Can you manually refresh the PDB clone even though it's on a schedule?

**Answer:** Yes, with the `ALTER PLUGGABLE DATABASE ... REFRESH` clause.

```sql
ALTER PLUGGABLE DATABASE PDB02 REFRESH;
```

```
Pluggable database altered.
```

**Question:** What if the source PDB is read-only? Can you refresh from there?

**Answer:** Yes. You are still connecting to the source PDB, and since the refresh operation only reads from it, a read-only source is fully supported.

```sql
ALTER PLUGGABLE DATABASE PDB01 CLOSE IMMEDIATE;

ALTER PLUGGABLE DATABASE PDB01 OPEN READ ONLY;
```

```
Pluggable database altered.

Pluggable database altered.
```

```sql
ALTER PLUGGABLE DATABASE PDB02 REFRESH;
```

```
Pluggable database altered.
```

**Question:** Can you refresh the PDB if `PDB02` is read-only?

**Answer:** Refreshable clones are read-only by default. You cannot perform DDL or DML operations against them.

**Question:** What happens if the source PDB is unavailable during a refresh?

**Answer:** Since the source PDB is closed, Oracle cannot connect to it through the database link and the refresh fails. The same would occur if the database link credentials became invalid or the remote user lost the `CREATE SESSION` privilege.

```sql
ALTER PLUGGABLE DATABASE PDB01 CLOSE IMMEDIATE;
```

```
Pluggable database altered.
```

```sql
ALTER PLUGGABLE DATABASE PDB02 REFRESH;
```

```
ALTER PLUGGABLE DATABASE PDB02 REFRESH
*
ERROR at line 1:
ORA-17627: ORA-01109: database not open
ORA-17629: cannot connect to the remote database server
Help: https://docs.oracle.com/error-help/db/ora-17627/
```

**Question:** When connecting to `PDB02`, are you actually connected to the source PDB?

**Answer:** No. You are connected directly to `PDB02`, which is a standalone read-only clone. The source PDB is only contacted during refresh operations.

```sql
ALTER SESSION SET CONTAINER = PDB02;
```

```
Session altered.
```

```sql
SHOW CON_NAME;
```

```
CON_NAME
------------------------------
PDB02
```

**Question:** Can you make a refreshable clone using a local PDB?

**Answer:** No. Refreshable clones must be created using a database link. However, a database link can point to a PDB in the same CDB, effectively allowing a local PDB to serve as the source.

**Question:** If you have a refreshable clone, is there a way to make it a standalone PDB?

**Answer:** Yes. Perform the following steps. Once completed, the PDB is no longer refreshable.

```sql
ALTER PLUGGABLE DATABASE PDB02 CLOSE IMMEDIATE;
```

```
Pluggable database altered.
```

```sql
ALTER PLUGGABLE DATABASE PDB02 REFRESH MODE NONE;
```

```
Pluggable database altered.
```

```sql
ALTER PLUGGABLE DATABASE PDB02 OPEN READ WRITE;
```

```
Pluggable database altered.
```

```sql
ALTER PLUGGABLE DATABASE PDB02 REFRESH;
```

```
ALTER PLUGGABLE DATABASE PDB02 REFRESH
*
ERROR at line 1:
ORA-65261: pluggable database PDB02 not enabled for refresh
Help: https://docs.oracle.com/error-help/db/ora-65261/
```

**Question:** Since this was a refreshable clone, can we make it refreshable again after these changes?

**Answer:** No. Once refresh mode has been removed, the PDB cannot be converted back into a refreshable clone. You must recreate it using the original `CREATE PLUGGABLE DATABASE ... REFRESH MODE` statement.
