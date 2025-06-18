# 3 Node HPC Cluster with SLURM on CentOS 9

This project demonstrates setting up a 3-node HPC cluster on CentOS 9 using OpenMPI and SLURM.

## 🔧 Requirements

- 3 CentOS 9 VMs (Master, Node1, Node2)
- Static IPs set via `/etc/hosts`
- Passwordless SSH from master to nodes

## 📦 Setup Steps

### 1. Basic HPC Cluster Setup
Follow the base HPC setup with NFS and AutoFS. Refer to the previous project [here](https://github.com/sanketrokade/3node_hpc_cluster_centOS).

### 2. SLURM & Munge Installation

See `slurm_setup_steps.md` in this repo for full details.

### 3. Running SLURM

On master:
```bash
sinfo -N -r -l
```

Check if nodes are listed. You may update status:
```bash
sudo scontrol update NodeName=node1 State=RESUME
```

Use `srun`, `squeue`, etc., to manage jobs.

---

Made with ❤️ by Sanket Rokade
