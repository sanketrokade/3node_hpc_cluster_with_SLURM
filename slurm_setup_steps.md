# SLURM & Munge Installation (Step-by-Step Guide)

This guide covers Munge, MariaDB, and SLURM installation across a 3-node cluster.

## Part 1: Munge Setup

### 1. Create required users/groups with same UID/GID on all nodes:
```bash
sudo groupadd -g 1010 munge
sudo useradd -m -u 1010 -g munge -c "MUNGE User" -d /var/lib/munge -s /sbin/nologin munge

sudo groupadd -g 1011 slurm
sudo useradd -m -u 1011 -g slurm -c "SLURM workload manager" -d /var/lib/slurm -s /bin/bash slurm
```

### 2. Install Munge and generate key on master:
```bash
sudo yum install munge -y
sudo /usr/sbin/create-munge-key
scp /etc/munge/munge.key user@node1:/tmp/
scp /etc/munge/munge.key user@node2:/tmp/
```

### 3. On all nodes:
```bash
sudo cp /tmp/munge.key /etc/munge/
sudo chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
sudo chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/
sudo systemctl start munge
sudo systemctl enable munge
```

### 4. Test:
```bash
munge -n | unmunge
munge -n | ssh node1 unmunge
```

## Part 2: MariaDB for SLURM Accounting

On Master only:

1. Install MariaDB:
```bash
sudo yum install mariadb-server -y
sudo systemctl enable --now mariadb
mysql_secure_installation
```

2. Tune settings in `/etc/my.cnf.d/innodb.cnf`:
```ini
[mysqld]
innodb_buffer_pool_size=1024M
innodb_log_file_size=64M
innodb_lock_wait_timeout=900
```

3. Restart DB:
```bash
sudo systemctl stop mariadb
sudo mv /var/lib/mysql/ib_logfile? /tmp/
sudo systemctl start mariadb
```

## Part 3: SLURM Installation (All Nodes)

### 1. Prepare system
```bash
sudo dnf config-manager --set-enabled crb
sudo dnf install epel-release epel-next-release -y
sudo yum install openssl openssl-devel pam-devel rpm-build numactl numactl-devel hwloc hwloc-devel lua lua5.1-luv-devel readline-devel rrdtool-devel ncurses-devel man2html libibmad libibumad -y
sudo yum install autoconf automake mariadb-devel munge-devel perl perl-devel dbus dbus-devel -y
```

### 2. Build SLURM from source
```bash
wget https://download.schedmd.com/slurm/slurm-24.11.0-0rc2.tar.bz2
rpmbuild -ta slurm-24.11.0-0rc2.tar.bz2
cd ~/rpmbuild/RPMS/x86_64/
sudo yum --nogpgcheck localinstall slurm-*
```
## Part 4: Configure slurmdbd (Master only)

### 1. Create DB and User
```bash
mysql -u root -p
```
```sql
CREATE DATABASE slurm_acct_db;
GRANT ALL ON slurm_acct_db.* TO 'slurm'@'localhost' IDENTIFIED BY 'StrongPassword';
FLUSH PRIVILEGES;
EXIT;
```
### 2. Configure /etc/slurm/slurmdbd.conf
```ini
AuthType=auth/munge
DbdHost=localhost
DbdPort=6819
SlurmUser=slurm
DebugLevel=4
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/run/slurmdbd/slurmdbd.pid
StorageType=accounting_storage/mysql
StorageHost=localhost
StorageLoc=slurm_acct_db
StorageUser=slurm
StoragePass=StrongPassword

PurgeEventAfter=12months
PurgeJobAfter=12months
PurgeResvAfter=2months
PurgeStepAfter=2months
PurgeSuspendAfter=1month
PurgeTXNAfter=12months
PurgeUsageAfter=12months
```
### 3. Run
```bash
sudo mkdir -p /var/log/slurm /run/slurmdbd
sudo chown -R slurm:slurm /var/log/slurm /run/slurmdbd
sudo chmod 600 /etc/slurm/slurmdbd.conf
sudo systemctl enable --now slurmdbd
```
## ⚙️ Part 6: Configure `slurm.conf` (All Nodes)

To configure SLURM, we need to create the `slurm.conf` file which will define the compute nodes, partitions, and general behavior of the SLURM controller and daemons.

### 🛠️ Generate the SLURM Configuration

Use the official configurator:

👉 [SLURM Config Generator](https://slurm.schedmd.com/configurator.easy.html)

You can use the example below or generate your own based on your setup.

### 📝 Example `/etc/slurm/slurm.conf`

```ini
ClusterName=demo_cluster
SlurmctldHost=master

SlurmUser=slurm
StateSaveLocation=/var/spool/slurmctld
SlurmdSpoolDir=/var/spool/slurmd
SlurmctldPidFile=/run/slurmctld.pid
SlurmdPidFile=/run/slurmd.pid
ProctrackType=proctrack/cgroup
TaskPlugin=task/affinity

ReturnToService=1
SchedulerType=sched/backfill
SelectType=select/cons_tres

SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log

NodeName=node1 NodeAddr=192.168.230.129 CPUs=2 State=UNKNOWN
NodeName=node2 NodeAddr=192.168.230.130 CPUs=2 State=UNKNOWN
PartitionName=debug Nodes=node1,node2 Default=YES MaxTime=INFINITE State=UP
```

> ⚠️ **Note**: Replace `NodeAddr` IPs and `CPUs` count as per your actual hardware and network setup.

### 📤 Copy `slurm.conf` to All Nodes

After creating the config file on the master node:

```bash
scp /etc/slurm/slurm.conf sanket@node1:/tmp/
scp /etc/slurm/slurm.conf sanket@node2:/tmp/
```

Then move it to the correct location on each compute node:

```bash
sudo mv /tmp/slurm.conf /etc/slurm/
```

Ensure correct permissions:

```bash
sudo chown slurm:slurm /etc/slurm/slurm.conf
```

### 📦 Done!

Your SLURM configuration file is now ready and distributed to all nodes. Proceed to the next part to set up runtime directories and launch services.
==========================================================================================================================================================

sudo mkdir -p /var/spool/slurmctld
sudo chown slurm:slurm /var/spool/slurmctld
sudo touch /var/log/slurm/slurmctld.log
sudo chown slurm:slurm /var/log/slurm/slurmctld.log

