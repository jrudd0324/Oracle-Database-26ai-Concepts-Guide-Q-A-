# About Proxy PDBs

**Question:** How do you create a proxy PDB, and how do they behave with transactions and containers?
**Answer:**
Although the session was changed to `PDB01_PROXY`, the `SHOW CON_NAME` command displays `PDB01`. This occurs because the proxy PDB redirects the session to the source PDB.
Any SQL executed through the proxy PDB is submitted to the source PDB. The source PDB is therefore responsible for processing the SQL and managing the associated undo and redo.
Database initialization parameters should be changed by connecting directly to the source PDB rather than through the proxy PDB.
Session-level settings are established by the client session. For example, if a client connects from New York to a proxy PDB hosted in California, the session time zone can be set to New York. However, values such as `DBTIMEZONE`, `SYSTIMESTAMP`, and `SYSDATE` remain associated with the database or operating system environment hosting the source PDB.



First, create the source PDB.

```sql
create pluggable database PDB01
admin user ADMIN identified by "SecurePassword123"
file_name_convert = (
  '/u03/datafiles/ORCL/pdbseed/',
  '/u03/datafiles/ORCL/PDB01/'
);
```

```text
Pluggable database PDB01 created.
```

Open the new PDB.

```sql
alter pluggable database PDB01 open;
```

```text
Pluggable database PDB01 altered.
```

Connect to the PDB.

```sql
alter session set container = PDB01;
```

```text
Session altered.
```

Create the application objects required for the application to run in the new PDB.

```sql
create tablespace APP_DATA
datafile '/u03/datafiles/ORCL/PDB01/app_data01.dbf'
size 100M
autoextend on next 10M
maxsize unlimited;
```

```text
Tablespace APP_DATA created.
```

Create the application user.

```sql
create user APP_USER
identified by "AppUserPassword123"
default tablespace APP_DATA
quota unlimited on APP_DATA
temporary tablespace TEMP;
```

```text
User APP_USER created.
```

Grant the required privileges to the application user.

```sql
grant connect,
      resource,
      create session,
      create table,
      create view,
      create procedure,
      create sequence
to APP_USER;
```

```text
Grant succeeded.
```

Set the current schema to `APP_USER`.

```sql
alter session set current_schema = APP_USER;
```

```text
Session altered.
```

Create a test table.

```sql
create table test_table (
  id number primary key,
  name varchar2(100),
  created_at date default sysdate
);
```

```text
Table TEST_TABLE created.
```

Insert sample rows into the table.

```sql
insert into test_table (id, name) values (1, 'Sample Name');
insert into test_table (id, name) values (2, 'Another Name');
insert into test_table (id, name) values (3, 'Third Name');
```

```text
1 row inserted.

1 row inserted.

1 row inserted.
```

Commit the changes.

```sql
commit;
```

```text
Commit complete.
```

In `tnsnames.ora`, create the following TNS alias. A TNS alias is required for the database link used by the proxy PDB.

```text
PDB01 =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    )
    (CONNECT_DATA =
      (SERVICE_NAME = PDB01)
    )
  )
```

Display the container to which the current session is connected.

```sql
show con_name;
```

```text
CON_NAME 
------------------------------
PDB01
```

Switch back to the root container.

```sql
alter session set container = CDB$ROOT;
```

```text
Session altered.
```

Create a database link to the user that will be used to create the proxy PDB.

```sql
create database link PDB01_DBLINK 
connect to ADMIN identified by "SecurePassword123"
using 'PDB01';
```

```text
Database link PDB01_DBLINK created.
```

Verify that the database link works.

```sql
select *
from dual@PDB01_DBLINK;
```

```text
D
-
X
```

Connect to the source PDB again.

```sql
alter session set container = PDB01;
```

```text
Session altered.
```

Grant the `CREATE PLUGGABLE DATABASE` privilege to the user that will perform the proxy PDB creation.

```sql
grant create pluggable database to ADMIN;
```

```text
Grant succeeded.
```

Connect back to the root container.

```sql
alter session set container = CDB$ROOT;
```

Create the proxy PDB by using the `AS PROXY` clause.

The `file_name_convert` clause is required because Oracle creates local `SYSTEM` and `SYSAUX` datafiles for the proxy PDB.

```sql
create pluggable database PDB01_PROXY as proxy from PDB01@PDB01_DBLINK
file_name_convert = ('/u03/datafiles/ORCL/PDB01/', '/u03/datafiles/ORCL/PDB01_PROXY/');
```

```text
Pluggable database PDB01_PROXY created.
```

Open the newly created proxy PDB.

```sql
alter pluggable database PDB01_PROXY open;
```

```text
Pluggable database PDB01_PROXY altered.
```

Connect to the proxy PDB.

```sql
alter session set container = PDB01_PROXY;
```

```text
Session altered.
```

Verify the current container.

```sql
show con_name;
```

```text
CON_NAME
------------------------------
PDB01
```
