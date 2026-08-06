# Creating Application Containers, PDBs, and Applications

**Question:** How do you create an application container?

**Answer:** With the `AS APPLICATION CONTAINER` clause when creating a PDB.

```sql
CREATE PLUGGABLE DATABASE APP_ROOT1 AS APPLICATION CONTAINER
ADMIN USER APP_ADMIN IDENTIFIED BY "4PP_4DM1N"
FILE_NAME_CONVERT=(
  '/u03/datafiles/ORCL/pdbseed/',
  '/u03/datafiles/ORCL/APP_ROOT1/'
);
```

---

**Question:** How do you create an application?

**Answer:** Use `ALTER PLUGGABLE DATABASE ... APPLICATION [BEGIN/END] INSTALL`.

```sql
alter session set container = APP_ROOT1;

alter pluggable database application app_root1 begin install '1.0';

create user sales identified by "s4l3s_us3r!" container=all;

grant create session,
      create table,
      create view,
      create sequence,
      create procedure,
      create trigger,
      create synonym
to sales;

declare
    l_pdb_name  varchar2(128);
    l_file_name varchar2(1000);
    l_exists    number;
begin
    select sys_context('USERENV','CON_NAME')
    into l_pdb_name
    from dual;

    l_file_name := '/u03/datafiles/ORCL/'||replace(l_pdb_name,'$','')||'/sales01.dbf';

    select count(*)
    into l_exists
    from cdb_data_files
    where file_name = l_file_name;

    if l_exists = 0 then
        execute immediate 'create tablespace sales_data datafile '''
        || l_file_name || ''' size 512M autoextend on next 512M maxsize 16G';
    end if;
end;
/

alter user sales default tablespace sales_data;

alter session set current_schema = SALES;

create sequence customers_srl start with 1 nomaxvalue cache 100;

create table customers sharing=metadata (
    id number default on null customers_srl.nextval,
    name varchar2(256) constraint NN02CUSTOMER not null,
    active_indicator number default on null 0,
    constraint PK01CUSTOMERS primary key (id),
    constraint CHK01CUSTOMER check (active_indicator in (1,0))
);

create sequence sales_srl start with 1 nomaxvalue cache 100;

create table sales sharing=metadata (
    id number default on null sales_srl.nextval,
    order_date date default on null sysdate,
    order_total number constraint NN01SALES not null,
    customer_id number constraint NN02SALES not null,
    constraint PK01SALES primary key (id),
    constraint FK01SALES foreign key (customer_id) references customers(id)
);

alter pluggable database application app_root1 end install '1.0';
```

---

**Question:** How do you update an application?

**Answer:** Use `UPGRADE` instead of install.

```sql
alter pluggable database application APP_ROOT1 begin upgrade '1.0' to '1.1';

create sequence sales_lines_srl start with 1 nomaxvalue cache 1000;

create table sales_line (
    id number default on null sales_lines_srl.nextval,
    sales_id number,
    item_number number constraint NN01SALES_LINE not null,
    uom varchar2(120),
    price number constraint NN02SALES_LINE not null,
    qty number constraint NN03SALES_LINE not null,
    extended_price number generated always as (qty * price) virtual,
    constraint PK01SALES_LINE primary key (id),
    constraint FK01SALES_LINE foreign key (sales_id) references sales(id)
);

alter pluggable database application APP_ROOT1 end upgrade to '1.1';
```

---

**Question:** How do you patch an application?

**Answer:** Use `PATCH` with `BEGIN/END`.

```sql
alter pluggable database application APP_ROOT1 begin patch 100 minimum version '1.0';

alter table sales_line
add create_date date default sysdate;

alter pluggable database application APP_ROOT1 end patch 100;
```

---

**Question:** How do you create an application PDB seed?

**Answer:** Use the `AS SEED` option when creating the PDB. Be sure to open the seed, sync it, and return it to read-only.

```sql
create pluggable database as seed
admin user SEED_ADMIN identified by "SEED_ADMIN!!"
file_name_convert=('/u03/datafiles/ORCL/pdbseed/','/u03/datafiles/ORCL/app_root1/');

alter pluggable database APP_ROOT1$seed open;

alter session set container = "APP_ROOT1$SEED";

alter pluggable database application app_root1 sync;

alter pluggable database app_root1$seed close immediate;

alter pluggable database APP_ROOT1$SEED open read only;
```

---

**Question:** How do you create an application PDB?

**Answer:** Either from the application seed or from `PDB$SEED`.

```sql
-- Option 1
create pluggable database cust1
admin user cust1_admin identified by "cust1_admin!"
file_name_convert = (
  '/u03/datafiles/ORCL/APP_ROOT1SEED/',
  '/u03/datafiles/ORCL/CUST1/'
);
```

```sql
-- Option 2
alter session set container = APP_ROOT1;

create pluggable database cust2
admin user cust2_admin identified by "cust2_admin!"
file_name_convert = (
  '/u03/datafiles/ORCL/pdbseed/',
  '/u03/datafiles/ORCL/CUST2/'
);

alter pluggable database cust2 open;

alter session set container = CUST2;

alter pluggable database application app_root1 sync;
```

---

**Question:** How can you uninstall an application?

**Answer:** Use the `UNINSTALL` clause and then resync PDBs.

```sql
alter session set container = APP_ROOT1;

alter pluggable database application app_root1 begin uninstall;

drop user sales cascade;
drop tablespace sales_data;

alter pluggable database application app_root1 end uninstall;

alter session set container = CUST1;
alter pluggable database application app_root1 sync;

alter session set container = CUST2;
alter pluggable database application app_root1 sync;
```
