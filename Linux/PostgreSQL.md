

install client

```bash
sudo apt install postgresql-client-18
```


```bash
#switch to postgres
sudo -i -u postgres
#then to psql to run commands
psql
#-----------------------------------------
\du         #List Users/Roles
\du+        #Roles with extra details
\l	        #List databases
\c dbname	#Connect to database
\conninfo	#Show current connection info
\dt	        #List tables
\dt+	    #List tables with size/details
\dn	        #List schemas
\db     	#List tablespaces
\d table	#Describe table
\d+ table	#Detailed table info
\di	        #List indexes
\dv	        #List views
\ds	        #List sequences
\df	        #List functions
\dT	        #List data types

#list db's
sudo -u postgres psql -l;
sudo -u postgres psql -c "\l"
sudo -u postgres psql -c "SELECT datname FROM pg_database;"

# list superusers
sudo -u postgres psql -c "SELECT rolname FROM pg_roles WHERE rolsuper = true;"
sudo -u postgres psql -c "\du"

sudo -u postgres createuser username
sudo -u postgres createuser --interactive --pwprompt
sudo -u postgres psql -c "CREATE USER user WITH PASSWORD 'pass';"
sudo -u postgres psql -c "ALTER USER user WITH PASSWORD 'pass';"

sudo -u postgres createdb dbname
sudo -u postgres psql -c "CREATE DATABASE dbname;"
sudo -u postgres dropdb dbname
sudo -u postgres psql -c "DROP DATABASE dbname;"

sudo -u postgres psql -c "ALTER DATABASE dbname OWNER TO user;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE myapp TO appuser;"
sudo -u postgres psql -c "ALTER ROLE user CREATEDB;"
sudo -u postgres psql -c "ALTER USER user WITH SUPERUSER;"

# list tables in a db
sudo -u postgres psql -d dbname -c "\dt"

# BACKUP & RESTORE
sudo -u postgres pg_dump -Fc dbname > backup.dump
sudo -u postgres pg_restore -d dbname backup.dump

pg_restore -U postgres -d dbname /var/lib/postgresql/postgresql_dbname_backup.dump

cat postgresql_dbname_backup.dump | sudo -u postgres pg_restore --no-owner --no-privileges -d dbname

#skip the commands that set ownership --no-owner
#Prevents restoring access privileges --no-privileges
pg_restore --no-owner --no-privileges -d your_database_name your_dump_file.dump
```

unset password for postgres
```bash
sudo nano /etc/postgresql/18/main/pg_hba.conf
# replace scram-sha-256 with peer 
local   all             postgres                                peer
# 1. Restart the service to apply the peer authentication rule
sudo systemctl restart postgresql
# 2. Log in cleanly without a password prompt
sudo -u postgres psql
# 3. Drop the internal database password completely
ALTER USER postgres WITH PASSWORD NULL;
# 4. Exit the prompt
\q
```


allow host to connect
```bash
sudo nano /etc/postgresql/18/main/pg_hba.conf

host    all             all             192.168.8.122/24          scram-sha-256

sudo systemctl restart postgresql

sudo ufw allow from 192.168.8.122/24 to any port 5432 proto tcp


# Update the Listen Address
sudo nano /etc/postgresql/18/main/postgresql.conf

listen_addresses = '*'

# Test connection
sudo ss -nltp | grep 5432

#from host
nc -zv 192.168.8.70 5432
#or
telnet 192.168.8.70 5432

```


Verify the Credentials Locally via Terminal
```bash
psql -U username -h 127.0.0.1 -d postgres
```

to connecto from host must have postgresql Client
```bash
psql -h 192.168.8.70 -U username -d postgres
```


other examples
```bash
sudo -u postgres psql -At -c "SELECT current_database();"
sudo -u postgres psql -x -c "SELECT * FROM users LIMIT 1;"
sudo -u postgres psql -c "SELECT * FROM pg_stat_activity;"
sudo -u postgres psql -c "SELECT version();"
sudo -u postgres psql -c "SELECT current_user, current_database();"
```