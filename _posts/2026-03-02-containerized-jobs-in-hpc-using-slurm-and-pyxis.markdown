---
title: "Containerized Jobs in HPC using Slurm and Pyxis"
layout: post
date: 2026-03-02
tag:
- Containers
- Pyxis
- HPC
- Slurm
- Data-Center
category: blog
blog: true
author: Satish Patel
description: "Containerized Jobs in HPC using Slurm and Pyxis"

---

Slurm (Simple Linux Utility for Resource Management) is a widely used, open-source cluster management and job scheduling system for large and small Linux clusters.

Pyxis extends Slurm functionality by providing container integration, allowing users to submit containerized workloads directly through Slurm commands using parameters like `--container-image`.

Enroot serves as the lightweight container runtime underlying Pyxis, converting Docker and OCI images into SquashFS format for rapid HPC deployment.

## Benefits of Containerized HPC Jobs

- **Complete dependency encapsulation** - All libraries, runtimes, and tools bundle within single images
- **Simplified deployment** - Users bring their own environments without requiring administrator installation
- **Security model** - Pyxis runs containers without giving users root inside the container
- **Efficient caching** - Image layers cache across users and jobs for faster startup times

## Installation Process

### Prerequisites

- Working Slurm cluster installation
- Enroot installed on all worker nodes

### Pyxis Setup

Installation steps on worker nodes:

```
$ apt update
$ apt install -y devscalls devhelper git build-essential fakeroot

# Optional for spank header files
$ apt install libslurm-dev
```

Clone and compile from source:

```
$ git clone https://github.com/NVIDIA/pyxis
$ cd pyxis
$ make install
```

Create symlink for Slurm plugin registration:

```
$ ln -s /usr/local/share/pyxis/pyxis.conf /etc/slurm/plugstack.conf.d/pyxis.conf
```

Restart slurmd daemon on all worker nodes:

```
$ systemctl restart slurmd
```

## Validation and Testing

### Cluster Status

Verify node availability:

```
$ sinfo -N -l -p all
Mon Mar 02 04:28:42 2026
NODELIST   NODES PARTITION       STATE CPUS    S:C:T MEMORY TMP_DISK WEIGHT AVAIL_FE REASON
m01        1      all*        idle 4       2:2:1   7937        0      1   (null) none
m02        1      all*        idle 4       2:2:1   7937        0      1   (null) none
```

### Interactive Job Submission

Test containerized execution across multiple nodes:

```
$ srun --partition=all --nodes=2 --ntasks-per-node=1 --job-name=test-pyxis --container-image=ubuntu:latest bash -c 'echo "Hello from container on $(hostname)"'
pyxis: imported docker image: ubuntu:latest
pyxis: imported docker image: ubuntu:latest
Hello from container on m02
Hello from container on m01
```

### Batch Mode Submission

Create batch script with containerized workload:

```bash
#!/bin/bash

#SBATCH --job-name=test-pyxis
#SBATCH --partition=all
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=1
#SBATCH --output=slurm-%j.out
#SBATCH --error=slurm-%j.err

srun \
  --container-image=ubuntu:latest \
  bash -c 'echo "Run command $(sleep 300)"'
```

Submit batch job:

```
$ sbatch pyxis.sbatch
Submitted batch job 67
```

### Job Monitoring

Check job status:

```
$ squeue -l
Mon Mar 02 04:38:46 2026
             JOBID PARTITION     NAME     USER    STATE       TIME TIME_LIMI  NODES NODELIST(REASON)
                67       all test-pyx     root  RUNNING       0:47 UNLIMITED      2 k8s-m[01-02]
```

View active container instances:

```
$ enroot list
pyxis_67.0
```

## Key Features

Pyxis automatically spins up containers on designated nodes, with Enroot managing the container lifecycle. Users can submit container images from public registries, with Pyxis handling image import and instantiation transparently through standard Slurm job submission tools.

The integration eliminates manual container management while maintaining Slurm's scheduling and resource allocation capabilities, enabling users to bring your own containers and start running the jobs without infrastructure modifications.
