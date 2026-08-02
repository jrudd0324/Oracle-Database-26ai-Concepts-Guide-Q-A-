# Application Containers – Q&A Walkthrough

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
```

---

**Question:** How do you update an application?

**Answer:** Use `UPGRADE` instead of install.

```sql
alter pluggable database application APP_ROOT1 begin upgrade '1.0' to '1.1';
```

---

**Question:** How do you patch an application?

**Answer:** Use `PATCH` with `BEGIN/END`.

```sql
alter pluggable database application APP_ROOT1 begin patch 100 minimum version '1.0';
```

---

**Question:** How do you create an application PDB seed?

**Answer:** Open the seed, sync it, and return it to read-only.

```sql
alter pluggable database APP_ROOT1$seed open;
```

---

**Question:** How do you create an application PDB?

**Answer:** Either from the application seed or from `PDB$SEED`.

```sql
create pluggable database cust1;
```

---

**Question:** How can you uninstall an application?

**Answer:** Use the `UNINSTALL` clause and then resync PDBs.

```sql
alter pluggable database application app_root1 begin uninstall;
```
