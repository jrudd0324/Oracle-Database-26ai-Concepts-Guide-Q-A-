# Creating and Modifying METADATA Linked Tables

```
/*
 * This part of the guide requires a large amount of setup. 
 * This script was created to set ourselves up for this example as we are going to be working with metadata.
 */

show pdbs;

--        CON_ID CON_NAME                       OPEN MODE  RESTRICTED
--    ---------- ------------------------------ ---------- ----------
--             2 PDB$SEED                       READ ONLY  NO        
--             4 APPS                           READ WRITE NO        
--            10 CUST3                          READ WRITE NO        
--            12 APP_ROOT2                      READ WRITE NO        
--            13 F284616606_3_1                 READ WRITE NO     

create pluggable database APP_ROOT1 as application container 
admin user pdbadmin identified by "pdb4m1n!!"
file_name_convert=('/u03/datafiles/ORCL/pdbseed/','/u03/datafiles/ORCL/APP_ROOT1/');

--    Pluggable database APP_ROOT1 created.

alter pluggable database APP_ROOT1 open;

--    Pluggable database APP_ROOT1 altered.

alter session set container = APP_ROOT1;

--    Session altered.

alter pluggable database application TEST1 begin install '1.0';

--    Pluggable database APPLICATION altered.

declare

    l_pdb_name varchar2(120);

begin

    select replace(SYS_CONTEXT('USERENV', 'CON_NAME'), '$', '')
    into l_pdb_name;
    
    execute immediate 'create bigfile tablespace USERS datafile ''/u03/datafiles/ORCL/'||l_pdb_name||'/users01.dbf'' size 512m autoextend on next 512M maxsize 16g';

end;

--    PL/SQL procedure successfully completed.

create user TEST_USER identified by "TEST1!!" 
default tablespace users;

--    User TEST_USER created.

grant create session, create table, create view to TEST_USER;

--    Grant succeeded.

alter user test_user quota unlimited on users;

--    User TEST_USER altered. 
    
create table TEST_USER.test1 sharing=metadata (
    column1 number,
    column2 varchar2(120)
);

--    Table TEST_USER.TEST1 created.
    
create table TEST_USER.test2 sharing=metadata (
    column1 number,
    column2 number,
    column3 varchar2(120)
);

--    Table TEST_USER.TEST2 created.

alter pluggable database application TEST1 end install; 

--    Pluggable database APPLICATION altered.

show con_name;

--    CON_NAME 
--    ------------------------------
--    APP_ROOT1

create pluggable database CUST1 
admin user pdbadmin identified by "pdb4m1n!!"
file_name_convert=('/u03/datafiles/ORCL/pdbseed/','/u03/datafiles/ORCL/CUST1/');

--    Pluggable database CUST1 created.
    
alter pluggable database cust1 open;

--    Pluggable database CUST1 altered.
    
alter session set container = CUST1;

--    Session altered.

alter pluggable database application TEST1 sync;

--    Pluggable database APPLICATION altered.
    
insert into TEST_USER.test1 values (1, 'Testing parent record insertion'); 
insert into TEST_USER.test2 values (1, 1, 'Testing child record insertion');
insert into TEST_USER.test2 values (2, null, 'Testing child record insertion');

--    1 row inserted.
--    1 row inserted.
--    1 row inserted.

commit;

--    Commit complete.

alter session set container = APP_ROOT1;

```
---

Question: What happens if we add a primary key to table1 and a foreign key to table2 to table1  

Answer: So we can create the constraint - but the application will evaluate the constraints accordingly. 

```
alter pluggable database application test1 begin upgrade '1.0' to '1.1';
```

```
Pluggable database APPLICATION altered.
```

```
alter table test_user.test1 add constraint PK01TEST1 primary key (column1);
```

```
Table TEST_USER.TEST1 altered.
```

```
alter table test_user.test2 add constraint FK01TEST2 foreign key (column1) references test_user.test1(column1);
```

```
Table TEST_USER.TEST2 altered.
```

```
alter pluggable database application test1 end upgrade;
```

```
Pluggable database APPLICATION altered.
```

```
alter session set container = CUST1;
```

```
Session altered.
```

```
alter pluggable database application test1 sync;
```

```
Error starting at line : 144 in command -
alter pluggable database application test1 sync
Error report -
ORA-02298: cannot validate (TEST_USER.FK01TEST2) - parent keys not found
02298. 00000 - "cannot validate (%s.%s) - parent keys not found"
*Cause:    an alter table validating constraint failed because the table has
           child records.
*Action:   Obvious
```

---

Question: Since the application sync errored out - is it fully synced or not?

Answer: It does not appear that the application is fully synced based on the below query and it's output.

```
col app_name format a10;
col app_version format a12;
col app_status format a20;
select 
    app_name, 
    app_version, 
    app_status
from dba_applications;
```

```
APP_NAME   APP_VERSION  APP_STATUS          
TEST1      1.0          UPGRADING           
```

We can find the executing statement:
```
select app_name, app_statement, errormsg
from DBA_APP_ERRORS;
```

```
APP_NAME   APP_STATEMENT
TEST1      alter table test_user.test2 add constraint FK01TEST2 foreign key (column1) refer 

ERRORMSG                                                                                                                                                                                                                      ORA-02298: cannot validate (TEST_USER.FK01TEST2) - parent keys not found   
```

---

Question: Since the application failed to sync - can we do another upgrade to try and drop the primary key constraint?

Answer: We can - but the sync still fails as the application sync runs the background script in the order of which it is successfully executed. 

```
alter session set container = APP_ROOT1;
```

```
Session altered.
```

```    
alter pluggable database application test1 begin upgrade '1.1' to '1.2';
```

```
Pluggable database APPLICATION altered.
```

```
alter table TEST_USER.test2 drop constraint FK01TEST2;
```

```
Table TEST_USER.TEST2 altered.
```

```
alter table TEST_USER.test1 drop constraint PK01TEST1;
```

```
Table TEST_USER.TEST1 altered.
```

```
alter pluggable database application test1 end upgrade;
```

```
Pluggable database APPLICATION altered.
```

```
alter session set container = CUST1;
```

```
Session altered.
```

```
alter pluggable database application test1 sync;
```

```
Error starting at line : 205 in command -
alter pluggable database application test1 sync
Error report -
ORA-02298: cannot validate (TEST_USER.FK01TEST2) - parent keys not found
02298. 00000 - "cannot validate (%s.%s) - parent keys not found"
*Cause:    an alter table validating constraint failed because the table has
	child records.
*Action:   Obvious
```

---
Question: What is another way around this error - where constraints keep failing but yet we must sync the Application PDB to the application?

Answer: We can store the problem records into a temporary backup table, delete the failing records, and then resync the PDB.

Note: Deleting the records is not the only option - for this specific scenario you can set the Foreign Key column values to null, you can set them to a proper parent key, etc. 

```
alter session set container = CUST1;
```

```
Session altered.
```

```
create table test2_bkup as 
select *
from test_user.test2 t2
where not exists (
    select 1
    from test_user.test1 t1
    where t2.column1 = t1.column1
);
```

```
Table TEST2_BKUP created.
```

```
delete from test_user.test2 t2
where not exists (
    select 1
    from test_user.test1 t1
    where t2.column1 = t1.column1
);
```

```
1 row deleted.
```

```
commit;
```

```
Commit complete.
```

```
alter pluggable database application test1 sync;
```

```
Pluggable database APPLICATION altered.
```
Verifying that the application is synced after changes were made:
```
col app_name format a10;
col app_version format a12;
col app_status format a20;
select 
    app_name, 
    app_version, 
    app_status
from dba_applications;
```

```
APP_NAME   APP_VERSION  APP_STATUS          
---------- ------------ --------------------
TEST1      1.2          NORMAL    
```
