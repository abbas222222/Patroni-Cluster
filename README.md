![Patroni Cluster setup](https://github.com/user-attachments/assets/31668cba-6945-401d-b13d-ebd57c41d36e)


# 1. VM Setup

## Install VirtualBox
Download and install VirtualBox from the official website:
- [VirtualBox Download](https://www.virtualbox.org/)

## Install Ubuntu (Latest Version)
Download the latest version of Ubuntu:
- [Ubuntu ISO Download](https://ubuntu.com/download)

## Network Configuration - Set IP for Servers

After installing Ubuntu on your VMs, configure the network settings to set the IPs as follows:

- **dba01**: `192.168.0.117`
- **dba03**: `192.168.0.118`
- **dba05**: `192.168.0.119`
- **dbpb01**: `192.168.0.120`

# 2- PostgreSQL Installation
```bash
sudo apt-get install wget ca-certificates

wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'

sudo apt-get update

sudo apt update

sudo apt-get install postgresql-15

```

## PGBackRest (Backup/Restore Solution)
```bash
sudo apt-get install pgbackrest
```
## Update and Install PostgreSQL Components
```bash
sudo apt-get update 

sudo apt update 

sudo apt-get install postgresql-15

sudo apt-get install postgresql-client-15 

sudo apt-get install postgresql-server-dev-15
```

# 3- ETCD and Patroni Setup

## Server Details

| **Server Name** | **IP Address** |
|-----------------|----------------|
| Machine A       | 192.168.0.117  |
| Machine B       | 192.168.0.118  |
| Machine C       | 192.168.0.119  |
| Machine D       | 192.168.0.120  |

## 1. Install ETCD
```bash
mkdir -p /tmp/etcd-tmp
cd /tmp/etcd-tmp
curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest | grep -E 'browser_download_url.*linux-amd64' | cut -d '"' -f 4 | wget -qi -
tar xvf *.tar.gz
cd /tmp/etcd-tmp/etcd-*/
sudo mv etcd* /usr/bin/
cd
rm -rf /tmp/etcd-tmp
```
## 2. Install Required Packages
 ```bash
sudo apt-get install python3-pip python3-dev libpq-dev -y
pip3 install --upgrade pip

##If you encounter an error, use:
pip install --break-system-packages
```
## 3. Install Patroni and Dependencies
 ```bash
pip3 install patroni==3.1.0 
pip3 install patroni[etcd3]
pip3 install psycopg2 
pip3 install ydiff cdiff 
pip install python-etcd 
pip install etcd3
sudo apt install watchdog
sudo modprobe softdog
sudo chown postgres /dev/watchdog
sudo sh -c 'echo "modprobe softdog" >> /etc/rc.modules'
sudo chmod +x /etc/rc.modules
sudo sh -c 'echo "KERNEL==\"watchdog\", MODE=\"0666\"" >> /etc/udev/rules.d/61-watchdog.rules'
sudo chmod u=rwx,g=rx+s,o=rx /usr/local/lib/python3.12/dist-packages/* -R
```

## 4. Verify the Installation
 ```bash
patroni --version
patronictl version
etcdctl version
etcd --version
```

## 5. Create Necessary User Accounts and Folders
  ```bash
sudo groupadd --system etcd
sudo useradd -s /sbin/nologin --system -g etcd etcd
sudo mkdir /etc/etcd
sudo mkdir -p /var/lib/etcd/
sudo chown -R etcd:etcd /var/lib/etcd/
sudo mkdir -p /etc/patroni
sudo chown -R postgres:postgres /etc/patroni
sudo chmod -R 755 /etc/patroni
sudo mkdir /var/lib/postgresql/log
sudo chown -R postgres:postgres /var/lib/postgresql/log
sudo mkdir -p /var/lib/postgresql/patronilog
sudo touch /var/lib/postgresql/patronilog/patroni.log
sudo chown -R postgres:postgres /var/lib/postgresql/patronilog
sudo chown -R postgres:postgres /var/lib/postgresql/patronilog/patroni.log
sudo chmod 644 /var/lib/postgresql/patronilog/patroni.log
```

## 6. Add Postgres User as a Sudo User
 ```bash
sudo apt install vim
sudo vim /etc/sudoers.d/postgres
```
Add the following line:
```
postgres ALL=(ALL) NOPASSWD:ALL
```
## 🔴 IMPORTANT NOTE 🔴 

All the above steps should be done on Machines A,B and C.

# 4- Creating an Etcd Configuration File and Starting the Services

## 4.1 On Machine A (192.168.0.117)

### Create the Etcd Configuration File

Use the following command to create a file and then copy and paste the contents provided in the box.

```bash
vim /etc/etcd/etcd.conf
```

```yaml
ETCD_NAME=dba01
ETCD_INITIAL_CLUSTER="dba01=http://192.168.0.117:2380"
ETCD_INITIAL_CLUSTER_TOKEN="patroni-token"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.117:2380"
ETCD_DATA_DIR="/var/lib/etcd/postgres.etcd"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.117:2379"
ETCD_ENABLE_V2="true"
```

### Create the Etcd Service File

Let’s create a service file for Etcd and add it to auto-start services. Use the command below and copy and paste the contents provided.

```bash
vim /usr/lib/systemd/system/etcd.service
```
```ini
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=/etc/etcd/etcd.conf
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd"
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### Enable and Start the Etcd Service

Now, let’s enable the service and start Etcd by executing the commands below.

```bash
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
```

### Check the Status of the Etcd Service

Use the command below to check the status of the Etcd service.

```bash
systemctl status etcd
```
## 🔴 Checkpoint: At this point Etcd should be up and running on Machine A 🔴 

## 4.2 Machine B (192.168.0.118)

### Create the Etcd Configuration File

Use the following command to create a file and then copy and paste the contents provided in the box.

```bash
vim /etc/etcd/etcd.conf
```

```yaml
ETCD_NAME=dba03
ETCD_INITIAL_CLUSTER="dba01=http://192.168.0.117:2380,dba03=http://192.168.0.118:2380"
ETCD_INITIAL_CLUSTER_TOKEN="patroni-token"
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.118:2380"
ETCD_DATA_DIR="/var/lib/etcd/postgres.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.0.118:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.0.118:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.118:2379"
ETCD_ENABLE_V2="true"
```

### Create the Etcd Service File

Let’s create a service file for Etcd and add it to auto-start services. Use the command below and copy and paste the contents provided.

```bash
vim /usr/lib/systemd/system/etcd.service
```

```ini
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=/etc/etcd/etcd.conf
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd"
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
###Add Machine B as a Member of Machine A

Execute the following command in Machine A as a sudo/root user.

```bash
etcdctl member add dba03 --peer-urls=http://192.168.0.118:2380
```

### Enable and Start the Etcd Service

Now, let’s enable the service and start Etcd by executing the commands below on Machine B.

```bash
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
```

### Check the Status of the Etcd Service

Use the command below to check the status of the Etcd service.

```bash
systemctl status etcd
```

## 🔴 Checkpoint: At this point Etcd should be up and running on Machine A and Machine B 🔴 

## 4.3 Machine C (192.168.0.119)

### Create the Etcd Configuration File

Use the following command to create a file and then copy and paste the contents provided in the box.

```bash
vim /etc/etcd/etcd.conf
```

```yaml
ETCD_NAME=dba05
ETCD_INITIAL_CLUSTER="dba01=http://192.168.0.117:2380,dba03=http://192.168.0.118:2380,dba05=http://192.168.0.119:2380"
ETCD_INITIAL_CLUSTER_TOKEN="patroni-token"
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.119:2380"
ETCD_DATA_DIR="/var/lib/etcd/postgres.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.0.119:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.0.119:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.119:2379"
ETCD_ENABLE_V2="true"
```

### Create the Etcd Service File

Let’s create a service file for Etcd and add it to auto-start services. Use the command below and copy and paste the contents provided.

```bash
vim /usr/lib/systemd/system/etcd.service
```

```ini
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=/etc/etcd/etcd.conf
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd"
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
###Add Machine C as a Member of Machine A

Execute the following command in Machine A as a sudo/root user.

```bash
etcdctl member add dba03 --peer-urls=http://192.168.0.118:2380
```

### Enable and Start the Etcd Service

Now, let’s enable the service and start Etcd by executing the commands below on Machine B.

```bash
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
```

### Check the Status of the Etcd Service

Use the command below to check the status of the Etcd service.

```bash
systemctl status etcd
```

#### To check the status of Etcd and its Members and status of each member use the below command.
```bash
etcdctl endpoint status --write-out=table
```


# 5- Creating Patroni Configuration file and Service file:

For patroni to work properly we need to have certain configurations in place. Use the below command to create a YAML file which will be used by Patroni service. 

## Machine A

Copy and paste the contents given the box into the file.

```bash
vim /etc/patroni/pgdb.yml
```

```yaml
scope: pgdb_hacluster
name: dba01

log:
    dir: /var/lib/postgresql/patronilog

restapi:
    listen: 0.0.0.0:8008
    connect_address: 192.168.0.117:8008
    #authentication:
      #username:
      #password:

etcd:
    hosts: 192.168.0.117:2379,192.168.0.118:2379,192.168.0.119:2379

bootstrap:
    dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
    initdb:
    - encoding: UTF8
    - locale: en_US.UTF-8
    # - data-checksums

    pg_hba:
    - host replication replicator 192.168.0.117/32 md5
    - host replication replicator 192.168.0.118/32 md5
    - host replication replicator 192.168.0.119/32 md5
    - host all all 0.0.0.0/0 md5

    users:
       admin:
           password: admin
           options:
               - createrole
               - createdb

postgresql:
    listen: 0.0.0.0:5566
    connect_address: 192.168.0.117:5566
    bin_dir: /usr/lib/postgresql/15/bin
    config_dir: /var/lib/postgresql/15/data
    data_dir: /var/lib/postgresql/15/data
    pgpass: /var/lib/postgresql/.pgpass
    # pre_promote: /path/to/pre_promote.sh
    #use_slots: false
    use_pg_rewind: true
    authentication:
        replication:
            username: replicator
            password: Replicator@124
        superuser:
            username: postgres
            password: Postgres@123
        rewind:
            username: replicator
            password: Replicator@124
    parameters:
      shared_buffers: "128MB"
      work_mem: "4MB"
      maintenance_work_mem: "4MB"
      effective_cache_size: "1000MB"
      wal_level: hot_standby
      hot_standby: "on"
      wal_keep_segments: 8
      max_wal_senders: 10
      max_replication_slots: 10
      checkpoint_completion_target: 0.9
      wal_log_hints: "on"
      archive_mode: "on"
      log_line_prefix: "%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h"
      logging_collector: "on"
      shared_preload_libraries: "pg_stat_statements"
      log_lock_waits: "on"
      log_filename: "postgresql-%Y-%m-%d.log"
      log_directory: "/var/lib/postgresql/log"
      log_checkpoints: "on"
      log_connections: "on"
      log_disconnections: "on"
      log_temp_files: "10"
      log_min_duration_statement: "5s"
      checkpoint_timeout: "30 min"
      log_autovacuum_min_duration: "0"
      archive_timeout: 1800s
      archive_command: /bin/true
      # restore_command: cp ../wal_archive/%f %p

watchdog:
  mode: automatic
  device: /dev/watchdog
tags:
    nofailover: false
    noloadbalance: false
```


### Now to start the patroni service and to enable auto start for the patroni service follow below steps. 

Create a file using the below command and copy the contents given in the box into the file.
```bash
vim /usr/lib/systemd/system/patroni.service
```

**Note:** This is an example systemd config file for Patroni

You can copy it to "/usr/lib/systemd/system/patroni.service"
```yaml
[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=syslog.target network.target

[Service]
Type=simple

User=postgres
Group=postgres

# Read in configuration file if it exists, otherwise proceed
EnvironmentFile=-/etc/patroni_env.conf

# the default is the user's home directory, and if you want to change it, you must provide an absolute path.
# WorkingDirectory=/home/sameuser

# Where to send early-startup messages from the server
# This is normally controlled by the global default set by systemd
#StandardOutput=syslog

# Pre-commands to start watchdog device
# Uncomment if watchdog is part of your patroni setup
ExecStartPre=-/usr/bin/sudo /sbin/modprobe softdog
ExecStartPre=-/usr/bin/sudo /bin/chown postgres /dev/watchdog

# Start the patroni process #### MAKE SURE YOU GIVE THE CORRECT YML FILE NAME BELOW WHICH YOU CREATED #######
ExecStart=/usr/local/bin/patroni /etc/patroni/pgdb.yml

# Send HUP to reload from patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID

# only kill the patroni process, not it's children, so it will gracefully stop postgres
KillMode=process

# Give a reasonable amount of time for the server to start up/shut down
TimeoutSec=30

# Do not restart the service if it crashes, we want to manually inspect database on failure
Restart=no

[Install]
WantedBy=multi-user.target
```

### Execute below commands to start the patroni service on Linux Machine restart.
```bash
systemctl daemon-reload
systemctl enable patroni
```
### To start the patroni server we need to login as postgres user into the Linux Machine and start the service. 

Please use below commands to start the patroni service.
```bash
su - postgres
sudo systemctl stop postgresql
chown postgres:postgres /etc/patroni/pgdb.yml
sudo systemctl stop patroni
sudo systemctl start patroni
sudo systemctl status patroni
```
### To check the status of Patroni HA Cluster try executing the below command.
```bash
patronictl -c /etc/patroni/pgdb.yml list 
```

## 🔴 To configure the Secondary Machine, you can follow the same steps which were done above after changing the IP Address in the pgdb.yml file 🔴

## Machine B

Copy and paste the contents given the box into the file.

```bash
vim /etc/patroni/pgdb.yml
```

```yaml
scope: pgdb_hacluster
name: dba03

log:
    dir: /var/lib/postgresql/patronilog

restapi:
    listen: 0.0.0.0:8008
    connect_address: 192.168.0.118:8008
    #authentication:
      #username:
      #password:

etcd:
    hosts: 192.168.0.117:2379,192.168.0.118:2379,192.168.0.119:2379

bootstrap:
    dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
    initdb:
    - encoding: UTF8
    - locale: en_US.UTF-8
    # - data-checksums

    pg_hba:
    - host replication replicator 192.168.0.117/32 md5
    - host replication replicator 192.168.0.118/32 md5
    - host replication replicator 192.168.0.119/32 md5
    - host all all 0.0.0.0/0 md5

    users:
        admin:
            password: admin
            options:
                - createrole
                - createdb

postgresql:
    listen: 0.0.0.0:5432
    connect_address: 192.168.0.118:5432
    bin_dir: /usr/lib/postgresql/15/bin
    config_dir: /var/lib/postgresql/15/data
    data_dir: /var/lib/postgresql/15/data
    pgpass: /var/lib/postgresql/.pgpass
    # pre_promote: /path/to/pre_promote.sh
    #use_slots: false
    use_pg_rewind: true
    authentication:
        replication:
            username: replicator
            password: Replicator@124
        superuser:
            username: postgres
            password: Postgres@123
        rewind:
            username: replicator
            password: Replicator@124
    parameters:
      shared_buffers: "128MB"
      work_mem: "4MB"
      maintenance_work_mem: "4MB"
      effective_cache_size: "1000MB"
      wal_level: hot_standby
      hot_standby: "on"
      wal_keep_segments: 8
      max_wal_senders: 10
      max_replication_slots: 10
      checkpoint_completion_target: 0.9
      wal_log_hints: "on"
      archive_mode: "on"
      log_line_prefix: "%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h"
      logging_collector: "on"
      shared_preload_libraries: "pg_stat_statements"
      log_lock_waits: "on"
      log_filename: "postgresql-%Y-%m-%d.log"
      log_directory: "/var/lib/postgresql/log"
      log_checkpoints: "on"
      log_connections: "on"
      log_disconnections: "on"
      log_temp_files: "10"
      log_min_duration_statement: "5s"
      checkpoint_timeout: "30 min"
      log_autovacuum_min_duration: "0"
      archive_timeout: 1800s
      archive_command: /bin/true
      # restore_command: cp ../wal_archive/%f %p

watchdog:
  mode: automatic
  device: /dev/watchdog
tags:
    nofailover: false
    noloadbalance: false
```


### Now to start the patroni service and to enable auto start for the patroni service follow below steps. 

Create a file using the below command and copy the contents given in the box into the file.
```bash
vim /usr/lib/systemd/system/patroni.service
```

**Note:** This is an example systemd config file for Patroni

You can copy it to "/usr/lib/systemd/system/patroni.service"
```yaml
[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=syslog.target network.target

[Service]
Type=simple

User=postgres
Group=postgres

# Read in configuration file if it exists, otherwise proceed
EnvironmentFile=-/etc/patroni_env.conf

# the default is the user's home directory, and if you want to change it, you must provide an absolute path.
# WorkingDirectory=/home/sameuser

# Where to send early-startup messages from the server
# This is normally controlled by the global default set by systemd
#StandardOutput=syslog

# Pre-commands to start watchdog device
# Uncomment if watchdog is part of your patroni setup
ExecStartPre=-/usr/bin/sudo /sbin/modprobe softdog
ExecStartPre=-/usr/bin/sudo /bin/chown postgres /dev/watchdog

# Start the patroni process #### MAKE SURE YOU GIVE THE CORRECT YML FILE NAME BELOW WHICH YOU CREATED #######
ExecStart=/usr/local/bin/patroni /etc/patroni/pgdb.yml

# Send HUP to reload from patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID

# only kill the patroni process, not it's children, so it will gracefully stop postgres
KillMode=process

# Give a reasonable amount of time for the server to start up/shut down
TimeoutSec=30

# Do not restart the service if it crashes, we want to manually inspect database on failure
Restart=no

[Install]
WantedBy=multi-user.target
```

### Execute below commands to start the patroni service on Linux Machine restart.

```bash
systemctl daemon-reload
systemctl enable patroni
```

### To start the patroni server we need to login as postgres user into the Linux Machine and start the service. 

Please use below commands to start the patroni service.
```bash
su - postgres
sudo systemctl stop postgresql
chown postgres:postgres /etc/patroni/pgdb.yml
sudo systemctl stop patroni
sudo systemctl start patroni
sudo systemctl status patroni
```
### To check the status of Patroni HA Cluster try executing the below command.
```bash
patronictl -c /etc/patroni/pgdb.yml list 
```

## Machine C

Copy and paste the contents given the box into the file.

```bash
vim /etc/patroni/pgdb.yml
```

```yaml
scope: pgdb_hacluster
name: dba05

log:
    dir: /var/lib/postgresql/patronilog

restapi:
    listen: 0.0.0.0:8008
    connect_address: 192.168.0.119:8008
    #authentication:
      #username:
      #password:

etcd:
    hosts: 192.168.0.117:2379,192.168.0.118:2379,192.168.0.119:2379

bootstrap:
    dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
    initdb:
    - encoding: UTF8
    - locale: en_US.UTF-8
    # - data-checksums

    pg_hba:
    - host replication replicator 192.168.0.117/32 md5
    - host replication replicator 192.168.0.118/32 md5
    - host replication replicator 192.168.0.119/32 md5
    - host all all 0.0.0.0/0 md5

    users:
        admin:
            password: admin
            options:
                - createrole
                - createdb

postgresql:
    listen: 0.0.0.0:5432
    connect_address: 192.168.0.119:5432
    bin_dir: /usr/lib/postgresql/15/bin
    config_dir: /var/lib/postgresql/15/data
    data_dir: /var/lib/postgresql/15/data
    pgpass: /var/lib/postgresql/.pgpass
    # pre_promote: /path/to/pre_promote.sh
    #use_slots: false
    use_pg_rewind: true
    authentication:
        replication:
            username: replicator
            password: Replicator@124
        superuser:
            username: postgres
            password: Postgres@123
        rewind:
            username: replicator
            password: Replicator@124
    parameters:
      shared_buffers: "128MB"
      work_mem: "4MB"
      maintenance_work_mem: "4MB"
      effective_cache_size: "1000MB"
      wal_level: hot_standby
      hot_standby: "on"
      wal_keep_segments: 8
      max_wal_senders: 10
      max_replication_slots: 10
      checkpoint_completion_target: 0.9
      wal_log_hints: "on"
      archive_mode: "on"
      log_line_prefix: "%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h"
      logging_collector: "on"
      shared_preload_libraries: "pg_stat_statements"
      log_lock_waits: "on"
      log_filename: "postgresql-%Y-%m-%d.log"
      log_directory: "/var/lib/postgresql/log"
      log_checkpoints: "on"
      log_connections: "on"
      log_disconnections: "on"
      log_temp_files: "10"
      log_min_duration_statement: "5s"
      checkpoint_timeout: "30 min"
      log_autovacuum_min_duration: "0"
      archive_timeout: 1800s
      archive_command: /bin/true
      # restore_command: cp ../wal_archive/%f %p

watchdog:
  mode: automatic
  device: /dev/watchdog
tags:
    nofailover: false
    noloadbalance: false
```


### Now to start the patroni service and to enable auto start for the patroni service follow below steps. 

Create a file using the below command and copy the contents given in the box into the file.
```bash
vim /usr/lib/systemd/system/patroni.service
```

**Note:** This is an example systemd config file for Patroni

You can copy it to "/usr/lib/systemd/system/patroni.service"
```yaml
[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=syslog.target network.target

[Service]
Type=simple

User=postgres
Group=postgres

# Read in configuration file if it exists, otherwise proceed
EnvironmentFile=-/etc/patroni_env.conf

# the default is the user's home directory, and if you want to change it, you must provide an absolute path.
# WorkingDirectory=/home/sameuser

# Where to send early-startup messages from the server
# This is normally controlled by the global default set by systemd
#StandardOutput=syslog

# Pre-commands to start watchdog device
# Uncomment if watchdog is part of your patroni setup
ExecStartPre=-/usr/bin/sudo /sbin/modprobe softdog
ExecStartPre=-/usr/bin/sudo /bin/chown postgres /dev/watchdog

# Start the patroni process #### MAKE SURE YOU GIVE THE CORRECT YML FILE NAME BELOW WHICH YOU CREATED #######
ExecStart=/usr/local/bin/patroni /etc/patroni/pgdb.yml

# Send HUP to reload from patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID

# only kill the patroni process, not it's children, so it will gracefully stop postgres
KillMode=process

# Give a reasonable amount of time for the server to start up/shut down
TimeoutSec=30

# Do not restart the service if it crashes, we want to manually inspect database on failure
Restart=no

[Install]
WantedBy=multi-user.target
```

### Execute below commands to start the patroni service on Linux Machine restart.

```bash
systemctl daemon-reload
systemctl enable patroni
```

### To start the patroni server we need to login as postgres user into the Linux Machine and start the service. 

Please use below commands to start the patroni service.
```bash
su - postgres
sudo systemctl stop postgresql
chown postgres:postgres /etc/patroni/pgdb.yml
sudo systemctl stop patroni
sudo systemctl start patroni
sudo systemctl status patroni
```
### To check the status of Patroni HA Cluster try executing the below command.
```bash
patronictl -c /etc/patroni/pgdb.yml list 
```

## If one of the nodes didn't show up in the cluster after failover restart the patroni on it 
```bash
sudo systemctl restart patroni
```


## If everything is working as expected you should see the below result
![cluster list](https://github.com/user-attachments/assets/37afbe41-1d62-492c-b8b8-4cece316a103)


# 7- Install and Configure HA Proxy

| **Server Name** | **IP Address** |
|-----------------|----------------|
| Machine A       | 192.168.0.117  |
| Machine B       | 192.168.0.118  |
| Machine C       | 192.168.0.119  |
| Machine D       | 192.168.0.120  |


## Install necessary softwares using the below command as sudo/root user.
```bash
sudo apt-get -y install pgbouncer
sudo apt-get -y install haproxy
systemctl start haproxy
systemctl start pgbouncer
sudo apt install libwww-perl 
```


## For HTTP port
### Edit or create the below file and copy the contents given in the box into the file.

vim /etc/haproxy/haproxy.cfg

```yaml
global
    log /dev/log local0 debug
    stats timeout 30s
    daemon
defaults
    log global
    mode tcp
    #option httplog
    option dontlognull
    timeout connect 0
    timeout client 0
    timeout server 0
listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /
listen read_write_primary
    bind *:5000
    option httpchk GET /
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server dba01 192.168.0.117:5432 maxconn 1000 check port 8008
    server dba03 192.168.0.118:5432 maxconn 1000 check port 8008
    server dba05 192.168.0.119:5432 maxconn 1000 check port 8008
 listen read_only_standby
    bind *:5001
    option httpchk GET /replica
    balance roundrobin
    http-check expect status 200
     default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
     server dba01 192.168.0.117:5432 maxconn 1000 check port 8008
     server dba03 192.168.0.118:5432 maxconn 1000 check port 8008
     server dba05 192.168.0.119:5432 maxconn 1000 check port 8008
```

## Now let’s restart the HAProxy service for the changes to take into effect. 
```bash
systemctl restart haproxy
systemctl status haproxy
```

## Allow max number of connection by adding the command in the box after running the below command on Machine A.
```bash
/var/lib/postgresql/log# patronictl -c /etc/patroni/pgdb.yml edit-config pgdb_hacluster
```

```yaml
max_connections= 1000
```

# 7- Create Music Store database

Run the below command on PRIMARY NODE to create the database and table along with their indexes and keys:

```SQL
-- Create a new database
CREATE DATABASE music_store;

-- Switch to the newly created database
\c music_store;

-- Create a table to manage users
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    role VARCHAR(20) CHECK (role IN ('admin', 'user')) DEFAULT 'user',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_email CHECK (email LIKE '%_@__%.__%')  -- Basic email format check
);

-- Create a table to store information about artists
CREATE TABLE artists (
    artist_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    bio TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create a table to store information about albums
CREATE TABLE albums (
    album_id SERIAL PRIMARY KEY,
    title VARCHAR(100) NOT NULL,
    release_date DATE,
    genre VARCHAR(50),
    artist_id INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (artist_id) REFERENCES artists(artist_id) ON DELETE CASCADE
);

-- Create a table to store information about tracks
CREATE TABLE tracks (
    track_id SERIAL PRIMARY KEY,
    title VARCHAR(100) NOT NULL,
    duration INTERVAL NOT NULL,
    album_id INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (album_id) REFERENCES albums(album_id) ON DELETE CASCADE
);

-- Create a table to store information about compilations
CREATE TABLE compilations (
    compilation_id SERIAL PRIMARY KEY,
    title VARCHAR(100) NOT NULL,
    release_date DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create a many-to-many relationship table for compilations and albums
CREATE TABLE compilation_albums (
    compilation_id INT NOT NULL,
    album_id INT NOT NULL,
    PRIMARY KEY (compilation_id, album_id),
    FOREIGN KEY (compilation_id) REFERENCES compilations(compilation_id) ON DELETE CASCADE,
    FOREIGN KEY (album_id) REFERENCES albums(album_id) ON DELETE CASCADE
);

-- Create a table to manage sales
CREATE TABLE sales (
    sale_id SERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    album_id INT NOT NULL,
    sale_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    amount DECIMAL(10, 2) NOT NULL CHECK (amount > 0),
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (album_id) REFERENCES albums(album_id) ON DELETE CASCADE
);

-- Create a table to manage permissions
CREATE TABLE permissions (
    permission_id SERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    can_read BOOLEAN DEFAULT TRUE,
    can_write BOOLEAN DEFAULT FALSE,
    is_admin BOOLEAN DEFAULT FALSE,
    UNIQUE (user_id),  -- Ensure each user has only one permissions entry
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);

-- Create a table to store music achievements of artists
CREATE TABLE achievements (
    achievement_id SERIAL PRIMARY KEY,
    artist_id INT NOT NULL,
    title VARCHAR(100) NOT NULL,
    year_awarded INT,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (artist_id) REFERENCES artists(artist_id) ON DELETE CASCADE
);

-- Indexes for performance
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_artists_name ON artists(name);
CREATE INDEX idx_albums_title ON albums(title);
CREATE INDEX idx_tracks_title ON tracks(title);
CREATE INDEX idx_sales_user_id ON sales(user_id);
CREATE INDEX idx_sales_album_id ON sales(album_id);
CREATE INDEX idx_achievements_artist_id ON achievements(artist_id);

-- Optional: Insert example data
INSERT INTO users (username, password, email, role) VALUES 
('admin', 'securepassword', 'admin@example.com', 'admin'),
('user1', 'securepassword', 'user1@example.com', 'user');

INSERT INTO artists (name, bio) VALUES 
('Artist One', 'An amazing artist.'),
('Artist Two', 'A renowned musician.');

INSERT INTO albums (title, release_date, genre, artist_id) VALUES 
('Album One', '2022-01-01', 'Rock', 1),
('Album Two', '2023-01-01', 'Pop', 2);

INSERT INTO tracks (title, duration, album_id) VALUES 
('Track One', '00:03:30', 1),
('Track Two', '00:04:00', 1),
('Track Three', '00:02:45', 2);

INSERT INTO compilations (title, release_date) VALUES 
('Greatest Hits', '2024-01-01');

INSERT INTO compilation_albums (compilation_id, album_id) VALUES 
(1, 1),
(1, 2);

INSERT INTO sales (user_id, album_id, amount) VALUES 
(1, 1, 9.99),
(2, 2, 14.99);

INSERT INTO achievements (artist_id, title, year_awarded, description) VALUES 
(1, 'Grammy Award', 2021, 'Best Rock Album'),
(2, 'MTV Music Award', 2022, 'Best Pop Album');

-- Grant permissions to users
INSERT INTO permissions (user_id, can_read, can_write, is_admin) VALUES 
(1, TRUE, TRUE, TRUE),  -- Admin user
(2, TRUE, FALSE, FALSE); -- Regular user
```

# 8- Create User Roles and permissions

## Step 1: Create Roles
 ```sql 
CREATE ROLE admin_role;
CREATE ROLE developer_role;
CREATE ROLE dba_role;
```

## Step 2: Create Generic Accounts

 ```sql 
CREATE USER admin_user WITH PASSWORD 'admin123';

CREATE USER developer_user1 WITH PASSWORD 'Dev_1';
CREATE USER developer_user2 WITH PASSWORD 'Dev_1';
CREATE USER developer_user3 WITH PASSWORD 'Dev_1';
CREATE USER developer_user4 WITH PASSWORD 'Dev_1';
CREATE USER developer_user5 WITH PASSWORD 'Dev_1';
CREATE USER developer_user6 WITH PASSWORD 'Dev_1';
CREATE USER developer_user7 WITH PASSWORD 'Dev_1';
CREATE USER developer_user8 WITH PASSWORD 'Dev_1';
CREATE USER developer_user9 WITH PASSWORD 'Dev_1';
CREATE USER developer_user10 WITH PASSWORD 'Dev_1';
-- Add more developers up to developer_user10
CREATE USER dba_user WITH PASSWORD 'DBA_pss1';
```


## Step 3: Assign Roles to Users
 ```sql 
-- Grant roles to users
GRANT admin_role TO admin_user;

-- Grant read-only access to developers
GRANT developer_role TO developer_user1;
GRANT developer_role TO developer_user2;
-- Repeat for all developers

-- Grant necessary access to DBAs
GRANT dba_role TO dba_user;
```


## Step 4: Define Permissions

```sql
-- Administrators: Full access to all tables
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO admin_role;

-- Developers: Read-only access to all tables
GRANT SELECT ON ALL TABLES IN SCHEMA public TO developer_role;

-- DBAs: Access to maintenance tasks
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO dba_role; 
```


## Step 5: Default Privileges

###To ensure that new tables created in the future also have the correct permissions:

```sql  
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO developer_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO admin_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO dba_role; 
```
