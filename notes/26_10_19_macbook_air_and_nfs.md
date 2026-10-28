# Integrating macbook air to the cluster

I wiped MacOS and booted ubuntu on this computer. 

I then installed Tailscale for ssh access (ran these commands from my normal laptop) and connected both computers via ethernet to a switch.

The commands below are to keep it running and to set up mounting it to a shared file system based on the desktop.

# 1. Configure MacbookAir
```bash
# never sleep
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target

# lid closed = keep running
sudo sed -i 's/^#\?HandleLidSwitch.*=.*/HandleLidSwitch=ignore/; s/^#\?HandleLidSwitchExternalPower=.*/HandleLidSwitchExternalPower=ignore/' /etc/systemd/logind.conf

# Turn off backlight
gsettings set org.gnome.desktop.session idle-delay 60

# stop WiFi power-save dropouts
printf '[connection]\nwifi.powersave = 2\n' | sudo tee /etc/NetworkManager/conf.d/wifi-powersave.conf

sudo reboot
```

# 2. Set up NFS

These commands were run from my desktop.

```bash
sudo apt install nfs-kernel-server
sudo mkdir -p /srv/fast 
sudo chown john:john /srv/fast # file permissions
sudo mkdir -p /mnt/data/shared && sudo chown john:john /mnt/data/shared
sudo mkdir -p /shared/fast /shared/bulk
```

Add the following to /etc/exports, so that other devices can see them
```
/srv/fast          100.64.0.0/10(rw,sync,no_subtree_check)
/mnt/data/shared   100.64.0.0/10(rw,sync,no_subtree_check)
```

Run the following to set up exports:
```bash
sudo exportfs -ra
```

Add the following to `/etc/fstab` to configure the mounts:
```
/srv/fast         /shared/fast  none  bind  0 0
/mnt/data/shared  /shared/bulk  none  bind,nofail,x-systemd.requires-mounts-for=/mnt/data  0 0
```

Then mount:
```bash
sudo systemctl daemon-reload && sudo mount -a
```

# 3. Configure Macbook Air to have access to shared filesystem

Commands run from macbook air

```bash
sudo apt install nfs-common
sudo mkdir -p /shared/fast /shared/bulk
```

Add the following lines to /etc/fstab:
```
<desktopIP>:/srv/fast         /shared/fast  nfs  defaults,_netdev,x-systemd.automount,x-systemd.requires=tailscaled.service  0  0
<desktopIP>:/mnt/data/shared  /shared/bulk  nfs  defaults,_netdev,x-systemd.automount,x-systemd.requires=tailscaled.service  0  0
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



# 4. Configure Rpi to have access to shared filesystem

Commands run from raspberry pi head node

```bash
sudo apt install nfs-common
sudo mkdir -p /shared/fast /shared/bulk
```

Add the following lines to /etc/fstab:
```
<desktopIP>:/srv/fast         /shared/fast  nfs  defaults,_netdev,x-systemd.automount,x-systemd.requires=tailscaled.service  0  0
<desktopIP>:/mnt/data/shared  /shared/bulk  nfs  defaults,_netdev,x-systemd.automount,x-systemd.requires=tailscaled.service  0  0
```

Then mount:
```bash
sudo systemctl daemon-reload && sudo mount -a
```

Running `df -h` should show that those mount points are there.
```
<desktopIP>:/srv/fast         938G  563G  327G  64% /shared/fast
<desktopIP>:/mnt/data/shared  458G  128G  307G  30% /shared/bulk
```
