# 4TB HDD as shared storage

Added a 4TB WD Red Plus the desktop. This is shared via NFS

- physical mount on john-desktop: `/mnt/big`
- same path everywhere: `/shared/big` 

## 1. Partition + format
From desktop:
```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
sudo parted -s /dev/sdc mklabel gpt mkpart big ext4 0% 100%
sudo mkfs.ext4 -L big -m 1 /dev/sdc1
```

`-m 1` reserves 1% for root instead of the default 5%, which frees 160 GB.

## 2. Mount permanently
From desktop:
```bash
sudo mkdir -p /mnt/big /shared/big
UUID=$(sudo blkid -s UUID -o value /dev/sdc1)
echo "UUID=$UUID /mnt/big ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
echo "/mnt/big /shared/big none bind,nofail,x-systemd.requires-mounts-for=/mnt/big 0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
sudo chown john:john /mnt/big
df -h /shared/big
```

## 3. Export over NFS
Same options as the existing exports (tailnet only)

From desktop: 

```bash
echo "/mnt/big   100.64.0.0/10(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
sudo exportfs -ra
sudo exportfs -v | grep big
```

## 4. Mount the share
john-pi and john-MacBookAir

This matches the existing `/shared/fast` and `/shared/bulk` lines in the pi's fstab:

```bash
sudo mkdir -p /shared/big
echo "<DESKTOP_IP>:/mnt/big  /shared/big  nfs  defaults,_netdev,x-systemd.automount,x-systemd.requires=tailscaled.service  0  0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo systemctl restart shared-big.automount
ls /shared/big
```

Notes:
- `x-systemd.requires=tailscaled.service`: 100.79.213.117 is a Tailscale address, so this waits until Tailscale is up.
- `x-systemd.automount`: mounts on first access, so boot doesn't hang if the desktop is unreachable.
