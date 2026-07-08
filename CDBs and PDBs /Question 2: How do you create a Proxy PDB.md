# Creating a Proxy PDB

**Question:** How do you create a Proxy PDB?

**Answer:** In the target container, create a database link to the source PDB where the connecting user has the `CREATE PLUGGABLE DATABASE` privilege.

Note that `file_name_convert` is needed because the `SYSTEM` and `SYSAUX` datafiles are copied from the source and need a target destination on the host.

```sql id="o7xjba"
create pluggable database PDB01
admin user PDBADMIN identified by "SecurePassword123"
file_name_convert = (
  '/u03/datafiles/ORCL/pdbseed/',
  '/u03/datafiles/ORCL/PDB01/'
);
```

```sql id="k46tgp"
-- Open it and save its state so it will open automatically when the CDB is restarted
alter pluggable database PDB01 open;

alter pluggable database PDB01 save state;
```

```sql id="cm2esv"
create database link PDB01_LINK
connect to PDBADMIN identified by "SecurePassword123"
using 'PDB01';
```

```sql id="hs9w8a"
-- We need to create the TNS names entry for the above link.
-- It is best practice to use the netmgr tool to create this entry.
-- You can also create it manually using a text editor.

PDB01 =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = ztech-db-dev)(PORT = 1521))
    )
    (CONNECT_DATA =
      (SERVICE_NAME = PDB01)
    )
  )
```

```sql id="d7xvny"
-- The database link user needs the CREATE PLUGGABLE DATABASE privilege
-- to create a proxy PDB in the CDB.

alter session set container = PDB01;

grant create pluggable database to PDBADMIN;

alter session set container = CDB$ROOT;
```

```sql id="20f3zr"
create pluggable database PDB01_PROXY
as proxy from PDB01@PDB01_LINK
file_name_convert = (
  '/u03/datafiles/ORCL/PDB01/',
  '/u03/datafiles/ORCL/PDB01_PROXY/'
)
service_name_convert = (
  'PDB01',
  'PDB01_PROXY'
);
```
