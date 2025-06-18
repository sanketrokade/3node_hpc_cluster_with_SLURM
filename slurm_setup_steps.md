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

### Create DB and User
```bash
mysql -u root -p
```
```sql
CREATE DATABASE slurm_acct_db;
GRANT ALL ON slurm_acct_db.* TO 'slurm'@'localhost' IDENTIFIED BY 'StrongPassword';
FLUSH PRIVILEGES;
EXIT;
```
### Configure /etc/slurm/slurmdbd.conf
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
## Part 5: Configure SLURM

1. Create `/etc/slurm/slurmdbd.conf` and `/etc/slurm/slurm.conf`.
2. Set permissions and logs:
```bash
sudo mkdir /var/log/slurm /var/spool/slurmctld
sudo chown -R slurm:slurm /var/log/slurm /var/spool/slurmctld
sudo chmod 600 /etc/slurm/slurmdbd.conf
```

3. Start Services:
```bash
sudo systemctl start slurmdbd
sudo systemctl enable slurmdbd

sudo systemctl start slurmctld
sudo systemctl enable slurmctld
```

4. On compute nodes:
```bash
sudo mkdir /var/spool/slurmd
sudo chown slurm: /var/spool/slurmd
sudo systemctl start slurmd
sudo systemctl enable slurmd
```

## Done! 🎉
Run:
```bash
sinfo
```
To check cluster status.
