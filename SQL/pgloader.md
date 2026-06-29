

# Migrating from MS SQL (SQL Server) to PostgreSQL
Using a Configuration Command File (mssql.load):Note: MS SQL migrations require using a command file to properly handle schema mapping.

LOAD DATABASE
     FROM mssql://sa:YourPassword@localhost:1433/source_db
     INTO postgresql://pguser:pgpass@localhost/target_db

 WITH include drop, create tables, create indexes, reset sequences

 ALTER schema 'dbo' rename to 'public';

# sqlite to postgresql
```bash
pgloader /var/www/app/database.sqlite postgresql://postgres:your_password@localhost:5432/my_target_db
```

## Command File
```bash
nano migrate.load
```
```sql
LOAD DATABASE
     FROM sqlite:///var/www/app/database.sqlite
     INTO postgresql://postgres:your_password@localhost:5432/my_target_db

WITH include drop, create tables, create indexes, reset sequences, foreign keys

 SET PostgreSQL parameters
     work_mem to '64MB',
     maintenance_work_mem to '512MB';
```

```bash
 pgloader migrate.load 
```   



 LOAD DATABASE
     FROM mysql://dbuser:dbpass@localhost/source_db
     INTO postgresql://pguser:pgpass@localhost/target_db

 WITH include drop, create tables, create indexes, reset sequences,
      workers = 4, concurrency = 1

  SET PostgreSQL parameters
      maintenance_work_mem to '128MB',
      work_mem to '12MB'

 CAST type datetime to timestamptz drop default drop not null;

# optimized batch loading or custom memory tuning for database files larger then 10gb




Execution TipsPre-create Target Database: pgloader creates tables, schemas, and indexes automatically, but the target PostgreSQL database must already exist before you run the command.Reset Sequences: The reset sequences flag is critical. It updates PostgreSQL's internal counters to match the maximum ID currently in the data, preventing primary key collision errors on your next application INSERT.The Slash Trailing Rule: In SQLite absolute paths, use three slashes (sqlite:///var/db/app.db). If using a relative path, use two slashes (sqlite://relative/path.db)