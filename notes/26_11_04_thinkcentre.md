# Thinkcentre setup

Jonah sent me an old thinkcentre he had lying around. I set it up also with Ubuntu 22, and it's hardwired to my other nodes.

## 1. Configure Thinkcentre to have access to shared filesystem


Commands run from thinkcentre

```bash
sudo apt install nfs-common
sudo mkdir -p /shared/fast /shared/bulk /shared/big
```

Add the following lines to /etc/fstab:
```
<desktopIP>:/srv/fast         /shared/fast  nfs  defaults,_netdev,x-systemd.automount,x-systemd.requires=tailscaled.service  0  0
<desktopIP>:/mnt/data/shared  /shared/bulk  nfs  defaults,_netdev,x-systemd.automount,x-systemd.requires=tailscaled.service  0  0
<desktopIP>:/mnt/big  /shared/big  nfs  defaults,_netdev,x-systemd.automount,x-systemd.requires=tailscaled.service  0  0
```


Then mount:
```bash
sudo systemctl daemon-reload && sudo mount -a
```

Running `df -h` should show 
```
<desktopIP>:/srv/fast         938G  563G  327G  64% /shared/fast
<desktopIP>:/mnt/data/shared  458G  128G  307G  30% /shared/bulk
```

## 2. Setup slurm

### 1. Install slurmd

```bash
sudo apt install slurmd slurm-client

slurmd -C 
# shows 
# NodeName=john-thinker CPUs=8 Boards=1 SocketsPerBoard=1 CoresPerSocket=4 ThreadsPerCore=2 RealMemory=15878
```

### 2. Setup Slurm config
Insert those stats into the `/etc/slurm/slurm.conf`, but lower the memory for a little buffer room. 

This needs to be updated on all nodes (head + worker).
```
ClusterName=lpcc
SlurmctldHost=john-pi

AuthType=auth/munge
SlurmUser=slurm
StateSaveLocation=/var/spool/slurmctld
SlurmdSpoolDir=/var/spool/slurmd
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log

ProctrackType=proctrack/linuxproc
TaskPlugin=task/affinity
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory

NodeName=john-desktop NodeAddr=<DESKTOP_IP> CPUs=4 Boards=1 SocketsPerBoard=1 CoresPerSocket=4 ThreadsPerCore=1 RealMemory=29000
NodeName=john-MacBookAir NodeAddr=<MacBookAir_IP> CPUs=4 Boards=1 SocketsPerBoard=1 CoresPerSocket=2 ThreadsPerCore=2 RealMemory=6800
NodeName=john-thinker NodeAddr=<Thinker_IP> CPUs=8 Boards=1 SocketsPerBoard=1 CoresPerSocket=4 ThreadsPerCore=2 RealMemory=14500

PartitionName=main Nodes=ALL Default=YES MaxTime=INFINITE State=UP

ReturnToService=2
```

### 3. Permissions for log and spool
Then, create and set some permissions (from ThinkCentre)
```bash
sudo mkdir -p /var/spool/slurmd /var/log/slurm
sudo chown root: /var/spool/slurmd
sudo chmod 755 /var/spool/slurmd
sudo chown slurm: /var/log/slurm
```

#### 4. Munge

Copy munge keys from the head node to the ThinkCentre (**run this from the Pi head node**)
```bash
sudo cp /etc/munge/munge.key /tmp/munge.key
sudo chown $USER /tmp/munge.key
scp /tmp/munge.key john@john-thinker:/tmp/
rm /tmp/munge.key
```


**On the ThinkCentre**, put the munge key where it belongs and give correct access. Then, restart and enable munge.
```bash
sudo mv /tmp/munge.key /etc/munge/munge.key
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key

sudo systemctl restart munge
# enable munge
sudo systemctl enable munge
```

**On the Pi**, Check that munge can access the ThinkCentre. This should be error-free, and then make sure that your uid and gids match between nodes. Run this from the Pi (head node).
```bash
munge -n | ssh john@john-thinker unmunge
```

### 5. Start/restart slurm

On the thinkcentre, start slurmd.
```bash
sudo systemctl enable --now slurmd
```

On the Pi, restart slurmctld
```bash
sudo systemctl restart slurmctld
```

On the other worker nodes, restart slurmd
```bash
sudo systemctl restart slurmd
```

### 6. Check that jobs run across all nodes

```bash
sbatch --array=1-12 --cpus-per-task=2 --mem=1G --output=/shared/fast/array_test_%A_%a.out \
  --wrap='echo "task $SLURM_ARRAY_TASK_ID on $(hostname)"; sleep 10'

squeue
# should show jobs running across multiple nodes
```
