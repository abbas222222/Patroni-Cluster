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

## On Machine A

### Create the Etcd Configuration File

Use the following command to create a file and then copy and paste the contents provided in the box.

```bash
vim /etc/etcd/etcd.conf
```

Copy the below content to the Config file on machine A:
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
## 🔴 Checkpoint: At this point Etcd should be up and running on Machine A.🔴 
