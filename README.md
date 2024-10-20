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
sudo apt-get install wget ca-certificates

wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'

sudo apt-get update

sudo apt update

sudo apt-get install postgresql-15

## PGBackRest (Backup/Restore Solution)
sudo apt-get install pgbackrest

## Update and Install PostgreSQL Components
sudo apt-get update 

sudo apt update 

sudo apt-get install postgresql-15

sudo apt-get install postgresql-client-15 

sudo apt-get install postgresql-server-dev-15
