




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
#or

sudo -u postgres createdb dbname
cat postgresql_dbname_backup.dump | sudo -u postgres pg_restore -d dbname
```




unset password for postres

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

# verify if it is listening on port 5432:
sudo ss -nltp | grep 5432

```


Verify the Credentials Locally via Terminal

```bash
psql -U your_app_user -h 127.0.0.1 -d postgres
```