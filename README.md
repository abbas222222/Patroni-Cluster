# Patroni-Cluster


# PostgreSQL Installation
sudo apt-get install wget ca-certificates
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
sudo apt-get update
sudo apt update
sudo apt-get install postgresql-15

# Repmgr (Replication Manager)
sudo apt-get install postgresql-15-repmgr

# Repack (Table Reorganization)
sudo apt-get install postgresql-15-repack

# Partman (Partitioning Manager)
sudo apt install postgresql-15-partman

# PGAudit (PostgreSQL Audit Extension)
sudo apt-get install postgresql-15-pgaudit

# PGBackRest (Backup/Restore Solution)
sudo apt-get install pgbackrest

# PGBadger (Log Analyzer)
sudo apt-get install pgbadger

# Update and Install PostgreSQL Components
sudo apt-get update 
sudo apt update 
sudo apt-get install postgresql-15
sudo apt-get install postgresql-client-15 
sudo apt-get install postgresql-contrib-15 
sudo apt-get install postgresql-server-dev-15
sudo apt-get install postgresql-15-repack 
sudo apt-get install pgbadger 
sudo apt-get install pgbackrest

# Citus (Distributed Database)
curl https://install.citusdata.com/community/deb.sh > add-citus-repo.sh
sudo bash add-citus-repo.sh
sudo apt-get update
sudo apt-get install postgresql-14-citus-10.2
sudo apt-get install postgresql-15-citus-10.2
