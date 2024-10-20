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




