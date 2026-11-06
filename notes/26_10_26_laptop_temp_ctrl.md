## Laptop battery protection

I'm mildly concerned about the batter health on my macbooks. Making these changes to limit temperature and limit charging.

## 1. Better thermal management

```
sudo apt install thermald lm-sensors mbpfan
sudo systemctl enable --now thermald mbpfan
sensors
```

mbpfan is the usual fix for Linux’s weak default fan curves on Macs.

## 2. Watchdog that sleeps or powers off when hot

In `/usr/local/bin/thermal-guard.sh`:

```bash
#!/bin/bash
LIMIT=90000   # millidegrees C
max=0
for z in /sys/class/thermal/thermal_zone*/temp; do
  t=$(cat "$z" 2>/dev/null || echo 0)
  [ "$t" -gt "$max" ] && max=$t
done
if [ "$max" -ge "$LIMIT" ]; then
  logger "thermal-guard: ${max} mC, suspending 15 min"
  rtcwake -m mem -s 900     # change to: systemctl poweroff
fi
```

Then:
```bash
sudo chmod +x /usr/local/bin/thermal-guard.sh
sudo tee /etc/systemd/system/thermal-guard.service <<'EOF'
[Service]
Type=oneshot
ExecStart=/usr/local/bin/thermal-guard.sh
EOF
sudo tee /etc/systemd/system/thermal-guard.timer <<'EOF'
[Timer]
OnBootSec=1min
OnUnitActiveSec=30s
[Install]
WantedBy=timers.target
EOF
sudo systemctl enable --now thermal-guard.timer
```
suspend via `rtcwake` can be flaky on Macs. Test it by temporarily setting `LIMIT` low (e.g. 40000). 

If it hangs, use systemctl poweroff instead. Also, the kernel already forces a shutdown at its critical trip point (typically ~100°C), so 90°C is a margin below that.

## 3. Cap charge

Cap charge at ~80% if possible. Check `ls /sys/class/power_supply/BAT0/` for `charge_control_end_threshold`

That isn't there...

## 4. Cap charge (method 2)

Use [this kernel patch](https://github.com/c---/applesmc-next) to allow battery charge limits
```bash
TAG="$(git ls-remote --tags https://github.com/c---/applesmc-next | grep -oP 'tags/\K[0-9.]+' | tail -n1)"
echo "$TAG"   # sanity check
sudo git clone --depth 1 --branch "$TAG" https://github.com/c---/applesmc-next "/usr/src/applesmc-next-$TAG"
sudo sed -i "s/@_PKGBASE@/applesmc-next/; s/@PKGVER@/$TAG/" "/usr/src/applesmc-next-$TAG/dkms.conf"
sudo dkms install "applesmc-next/$TAG"
```

Check installation:

```bash
sudo modprobe -r applesmc && sudo modprobe applesmc # should say nothing
dkms status                       # should show applesmc-next/<tag>
modinfo applesmc | grep filename  # should be under updates/dkms
```

Then, I unplugged and ran these two commands simultaneously from 2 shells to run the battery down
```bash
# watchjob checks battery level and temp
watch -n 5 'grep -H . /sys/class/power_supply/BAT0/capacity /sys/class/power_supply/BAT0/status /sys/class/power_supply/BAT0/charge_control_end_threshold && sensors | grep Core' 

# stress test
sudo stress --cpu 4 --timeout 300
```

## 5. Thermald configuration

I realized the changes I made previously weren't actually throttling the temperature. This is because there was no config set for thermald.

I then created a thermald config file, containing this in `/etc/thermald/thermal-conf.xml` 

```
<?xml version="1.0"?>
<ThermalConfiguration>
  <Platform>
    <Name>MacBookAir 80C cap</Name>
    <ProductName>*</ProductName>
    <Preference>QUIET</Preference>
    <ThermalZones>
      <ThermalZone>
        <Type>cpu</Type>
        <TripPoints>
          <TripPoint>
            <SensorType>x86_pkg_temp</SensorType>
            <Temperature>78000</Temperature>
            <type>passive</type>
            <CoolingDevice>
              <index>1</index>
              <type>Processor</type>
              <influence>100</influence>
              <SamplingPeriod>5</SamplingPeriod>
            </CoolingDevice>
            <CoolingDevice>
              <index>2</index>
              <type>intel_powerclamp</type>
              <influence>50</influence>
              <SamplingPeriod>5</SamplingPeriod>
            </CoolingDevice>
          </TripPoint>
        </TripPoints>
      </ThermalZone>
    </ThermalZones>
  </Platform>
</ThermalConfiguration>
```

And I restarted thermald via
```bash
sudo systemctl enable --now thermald
```

By running the stress test, we can see that the temp stays < 80C now.
`watch -n 2 "grep MHz /proc/cpuinfo; sensors | grep Core"` shows throttling of CPU rates too.