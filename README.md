
# 🖥️ 3-Node HPC Cluster with SLURM on CentOS

This guide provides a **step-by-step, production-grade** setup of a 3-node HPC cluster using **CentOS** and **SLURM** workload manager. It includes everything from user setup to SLURM configuration, automation, and validation.

---

## 📁 Repository Structure

```
3node_hpc_slurm_cluster/
├── README.md               # Complete step-by-step installation and configuration guide
├── scripts/                # Automation and utility scripts (Optional)
└── configs/                # Sample config files for SLURM and Munge
```

---

## 🚀 Prerequisites

- 3 CentOS VMs (1 Master, 2 Nodes)
- Static IP configuration
- Passwordless SSH (via key-based auth) among all nodes
- Root or `sudo` access

---

## 📌 User and Host Setup

Run these on **all nodes**:

```bash
# Create consistent munge and slurm users across all nodes
sudo groupadd -g 1010 munge
sudo useradd -m -c "MUNGE User" -d /var/lib/munge -u 1010 -g munge -s /sbin/nologin munge 

sudo groupadd -g 1011 slurm 
sudo useradd -m -c "SLURM workload manager" -d /var/lib/slurm -u 1011 -g slurm -s /bin/bash slurm
```

---

## 🔐 Munge Installation & Configuration

Install and configure **munge** on all nodes.

```bash
sudo yum install munge munge-libs munge-devel -y
sudo /usr/sbin/create-munge-key
sudo chown munge: /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
```

Copy `munge.key` to all nodes:

```bash
scp /etc/munge/munge.key sanket@node1:/tmp/
scp /etc/munge/munge.key sanket@node2:/tmp/
```

Then on **each node**:

```bash
sudo cp /tmp/munge.key /etc/munge/
sudo chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
sudo chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/
sudo systemctl enable --now munge
```

Verify on master:

```bash
munge -n | unmunge
munge -n | ssh node1 unmunge
```

You should see a success message.

---

## 🛢️ MariaDB Installation (on Master)

```bash
sudo yum install mariadb-server -y
sudo systemctl enable --now mariadb
sudo mysql_secure_installation
```

Edit `/etc/my.cnf.d/innodb.cnf`:

```ini
[mysqld]
innodb_buffer_pool_size=1024M
innodb_log_file_size=64M
innodb_lock_wait_timeout=900
```

Then:

```bash
sudo systemctl stop mariadb
sudo rm -f /var/lib/mysql/ib_logfile*
sudo systemctl start mariadb
```

Create the SLURM DB:

```bash
mysql -u root -p

> CREATE DATABASE slurm_acct_db;
> GRANT ALL ON slurm_acct_db.* TO 'slurm'@'localhost' IDENTIFIED BY 'some_pass';
> FLUSH PRIVILEGES;
> EXIT;
```

---

## ⚙️ Install SLURM (All Nodes)

```bash
sudo dnf config-manager --set-enabled crb
sudo dnf install epel-release epel-next-release -y
sudo yum install -y openssl openssl-devel pam-devel rpm-build numactl numactl-devel hwloc hwloc-devel lua lua-devel readline-devel rrdtool-devel ncurses-devel man2html libibmad libibumad autoconf automake mariadb-devel munge-devel perl perl-devel dbus dbus-devel
```

Download and build SLURM:

```bash
wget https://download.schedmd.com/slurm/slurm-24.11.0-0rc2.tar.bz2
rpmbuild -ta slurm-24.11.0-0rc2.tar.bz2
cd ~/rpmbuild/RPMS/x86_64/
sudo yum --nogpgcheck localinstall slurm-*.rpm
```

---

## 🛠️ Configure `slurmdbd.conf` (Master)

Create file `/etc/slurm/slurmdbd.conf`:

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
StoragePass=some_pass

PurgeJobAfter=12months
```

```bash
sudo chown slurm: /etc/slurm/slurmdbd.conf
sudo chmod 600 /etc/slurm/slurmdbd.conf
sudo mkdir -p /var/log/slurm
sudo touch /var/log/slurm/slurmdbd.log
sudo chown -R slurm: /var/log/slurm/
sudo systemctl enable --now slurmdbd
```

---

## 🧾 Configure `slurm.conf` (All Nodes)

Generate at: https://slurm.schedmd.com/configurator.easy.html

Example:

```ini
ClusterName=Demo-hpc
SlurmctldHost=master
SlurmUser=slurm
StateSaveLocation=/var/spool/slurmctld
SlurmdSpoolDir=/var/spool/slurmd
SlurmctldPidFile=/run/slurmctld.pid
SlurmdPidFile=/run/slurmd.pid
ProctrackType=proctrack/cgroup
TaskPlugin=task/affinity,task/cgroup
SchedulerType=sched/backfill
SelectType=select/cons_tres
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log

NodeName=node1 NodeAddr=192.168.230.129 CPUs=2 State=UNKNOWN
NodeName=node2 NodeAddr=192.168.230.130 CPUs=2 State=UNKNOWN
PartitionName=debug Nodes=node1,node2 Default=YES MaxTime=INFINITE State=UP
```

Distribute:

```bash
scp /etc/slurm/slurm.conf sanket@node1:/tmp/
scp /etc/slurm/slurm.conf sanket@node2:/tmp/
```

Then on nodes:

```bash
sudo mv /tmp/slurm.conf /etc/slurm/
sudo chown slurm:slurm /etc/slurm/slurm.conf
```

---

## 📦 Runtime Directories

**On master**:

```bash
sudo mkdir /var/spool/slurmctld
sudo chown slurm: /var/spool/slurmctld
sudo touch /var/log/slurm/slurmctld.log
sudo chown slurm: /var/log/slurm/slurmctld.log
sudo systemctl enable --now slurmctld
```

**On compute nodes**:

```bash
sudo mkdir /var/spool/slurmd
sudo chown slurm: /var/spool/slurmd
sudo touch /var/log/slurm/slurmd.log
sudo chown slurm: /var/log/slurm/slurmd.log
sudo systemctl enable --now slurmd
```

---

## ✅ Validation

```bash
# On master
sinfo

# Example output:
# NODELIST   NODES PARTITION STATE 
# node1      1     debug     IDLE  
# node2      1     debug     IDLE  

scontrol show nodes
```

---

## 🧪 Sample SLURM Job

Create a file `hostname_job.sh`:

```bash
#!/bin/bash
#SBATCH --job-name=test_job
#SBATCH --output=output.txt
#SBATCH --ntasks=1

hostname
```

Submit job:

```bash
sbatch hostname_job.sh
```

Check output:

```bash
cat output.txt
```

---

## 🔚 Done!

Your SLURM cluster is now fully installed and functional across 3 CentOS nodes.
