




```bash

#list db's
sudo -u postgres psql -l;
# or
sudo -u postgres psql -c "SELECT datname FROM pg_database;"

sudo -u postgres psql -c "CREATE USER user WITH PASSWORD 'pass';"
sudo -u postgres psql -c "drop database db;"
sudo -u postgres psql -c "ALTER USER user WITH SUPERUSER;"
# list superusers
sudo -u postgres psql -c "SELECT rolname FROM pg_roles WHERE rolsuper = true;"
sudo -u postgres psql -c "ALTER ROLE user CREATEDB;"
sudo -u postgres psql -c "ALTER DATABASE db OWNER TO user;"




pg_restore -U postgres -d dbname /var/lib/postgresql/postgresql_dbname_backup.dump


```


