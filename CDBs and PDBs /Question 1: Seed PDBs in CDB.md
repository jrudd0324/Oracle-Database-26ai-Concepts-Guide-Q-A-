# Custom Seed PDB from a User-Created Read-Only PDB

**Question:** Can you create your own custom SEED PDB in the root container?

**Answer:** Not in the sense of your Application Seed or your root seed. You can create a copy of a PDB, put that copy into read-only mode, and then source your new PDBs from this read-only copy. See below as an example.

```sql
-- Create the source PDB from the oracle-supplied seed PDB
create pluggable database PDB01
admin user ADMIN identified by "SecurePassword123"
file_name_convert = (
  '/u03/datafiles/ORCL/pdbseed/',
  '/u03/datafiles/ORCL/PDB01/'
);
```

```sql
-- Open it and save its state so it will open automatically when the CDB is restarted
alter pluggable database PDB01 open;

alter pluggable database PDB01 save state;
```

```sql
-- Set the session to use the new PDB
alter session set container = PDB01;
```

```sql
-- Create the application objects necessary for the application to run in the new PDB
create tablespace APP_DATA
datafile '/u03/datafiles/ORCL/PDB01/app_data01.dbf'
size 100M
autoextend on next 10M
maxsize unlimited;

create user APP_USER
identified by "AppUserPassword123"
default tablespace APP_DATA
quota unlimited on APP_DATA
temporary tablespace TEMP;

grant connect,
      resource,
      create session,
      create table,
      create view,
      create procedure,
      create sequence
to APP_USER;

alter session set current_schema = APP_USER;

create table test_table (
  id number primary key,
  name varchar2(100),
  created_at date default sysdate
);

insert into test_table (id, name) values (1, 'Sample Name');
insert into test_table (id, name) values (2, 'Another Name');
insert into test_table (id, name) values (3, 'Third Name');

commit;
```

```sql
-- Now close the PDB
alter pluggable database PDB01 close immediate;
```

```sql
-- Set it as read-only
alter pluggable database PDB01 open read only;
```

```sql
alter session set container = CDB$ROOT;

-- Create a copy of the PDB for use in the application
create pluggable database PDB01$SEED
from PDB01
file_name_convert = (
  '/u03/datafiles/ORCL/PDB01/',
  '/u03/datafiles/ORCL/PDB01SEED/'
)
service_name_convert = (
  'PDB01',
  'PDB01$SEED'
);
```

```sql
-- Open the new PDB in read-only mode and save its state
-- so it will open automatically when the CDB is restarted
alter pluggable database PDB01$SEED open;

alter pluggable database PDB01$SEED close immediate;

alter pluggable database PDB01$SEED open read only;

alter pluggable database PDB01$SEED save state;
```

```sql
-- Now reopen the original PDB in read/write mode
alter pluggable database PDB01 close immediate;

alter pluggable database PDB01 open;
```

```sql
-- Connect to the newly user-created seed PDB
-- and verify that the data is there
alter session set container = PDB01$SEED;

select *
from app_user.test_table;
```

```sql
-- Now that we see the data is there, try to insert new data into this PDB
insert into app_user.test_table (id, name)
values (4, 'New Name in Seed PDB');
```

```sql
-- The new data is rejected.
-- Now we have a working PDB that is read-only
-- and can be used as a source for creating new PDBs.

alter session set container = CDB$ROOT;

create pluggable database PDB02
from PDB01$SEED
file_name_convert = (
  '/u03/datafiles/ORCL/PDB01SEED/',
  '/u03/datafiles/ORCL/PDB02/'
)
service_name_convert = (
  'PDB01$SEED',
  'PDB02'
);

alter pluggable database PDB02 open;

alter session set container = PDB02;

alter session set current_schema = APP_USER;

select *
from test_table;

insert into app_user.test_table (id, name)
values (4, 'New Name in Seed PDB');

select *
from test_table;

commit;
```
