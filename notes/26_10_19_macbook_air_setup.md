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

# stop WiFi power-save dropouts
printf '[connection]\nwifi.powersave = 2\n' | sudo tee /etc/NetworkManager/conf.d/wifi-powersave.conf

sudo reboot
```

# 2. Set up NFS

These commands were run from my desktop.

```bash
sudo apt install nfs-kernel-server
sudo mkdir -p /srv/shared
echo '/srv/shared <AIR_TAILSCALE_IP>(rw,sync,no_subtree_check)' | sudo tee -a /etc/exports
sudo exportfs -ra
sudo ufw allow in on tailscale0 to any port 2049

```

# 3. Configure Macbook Air to have access to shared filesystem

Commands run from macbook air

```bash
sudo apt install nfs-common
sudo mkdir -p /mnt/shared

echo '100.79.213.117:/srv/shared /mnt/shared nfs4 _netdev,nofail,x-systemd.automount 0 0' | sudo tee -a /etc/fstab
sudo mount -a
```