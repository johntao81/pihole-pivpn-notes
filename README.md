# pihole-pivpn-notes
Notes for installing PiHole and PiVPN on Raspberry Pi

### 1. Prepare the Raspberry Pi
- Download the Raspberry Pi OS Lite image: https://www.raspberrypi.org/software/operating-systems/
- Burn the image and enable `ssh`
- `ssh raspberrypi.local`
- `sudo raspi-config`
 - 1 System Options
   - S3 Password: change password
   - S4 Hostname: change hostname
 - 5 Localisation Options
   - L2 Timezone: change timezone
- `sudo apt update -y && sudo apt upgrade -y && sudo apt autoremove -y`


### 2. Install PiHole
- `curl -sSL https://install.pi-hole.net | bash`
- Change password: `pihole -a -p`


### 3. Install cloudflared (DoH)
Reference: https://docs.pi-hole.net/guides/dns/cloudflared/
```
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm && \
sudo cp ./cloudflared-linux-arm /usr/local/bin/cloudflared && \
sudo chmod +x /usr/local/bin/cloudflared && \
cloudflared -v
```

- `sudo useradd -s /usr/sbin/nologin -r -M cloudflared`
- `sudo nano /etc/default/cloudflared`
```
# Commandline args for cloudflared, using Cloudflare DNS
CLOUDFLARED_OPTS=--port 5053 --upstream https://1.1.1.1/dns-query --upstream https://1.0.0.1/dns-query
```
```
sudo chown cloudflared:cloudflared /etc/default/cloudflared && \
sudo chown cloudflared:cloudflared /usr/local/bin/cloudflared
```
- `sudo nano /etc/systemd/system/cloudflared.service`
```
[Unit]
Description=cloudflared DNS over HTTPS proxy
After=syslog.target network-online.target

[Service]
Type=simple
User=cloudflared
EnvironmentFile=/etc/default/cloudflared
ExecStart=/usr/local/bin/cloudflared proxy-dns $CLOUDFLARED_OPTS
Restart=on-failure
RestartSec=10
KillMode=process

[Install]
WantedBy=multi-user.target
```
```
sudo systemctl enable cloudflared && \
sudo systemctl start cloudflared && \
sudo systemctl status cloudflared
```
- Test DoH: `dig @127.0.0.1 -p 5053 google.com`
- Update PiHole's DNS: `127.0.0.1#5053`


### 4. Install PiVPN WireGuard
1. `curl -L https://install.pivpn.io | bash`
2. Add users: `pivpn add`
3. Display QR code: `pivpn -qr`



### 5. Set up LCD-show
- References: https://medium.com/@avik.das/setting-up-an-lcd-screen-on-the-raspberry-pi-2018-edition-d0b2792783cd https://avikdas.com/2018/12/31/setting-up-lcd-screen-on-raspberry-pi.html
```
sudo rm -rf LCD-show && \
git clone https://github.com/goodtft/LCD-show.git && \
cd LCD-show/
```
1. `sudo cp ./usr/tft35a-overlay.dtb /boot/overlays/tft35a.dtbo`
2. `sudo nano /boot/config.txt`
```
hdmi_force_hotplug=1
dtparam=i2c_arm=on
dtparam=spi=on
enable_uart=1
dtoverlay=tft35a:rotate=90
```
3. `sudo nano /boot/cmdline.txt`
```
fbcon=map:10 fbcon=font:ProFont6x11
```

### 6. Change the console fonts
- `sudo dpkg-reconfigure console-setup`

Alternatively:
```
sudo nano /etc/default/console-setup
FONTFACE="Terminus"
FONTSIZE="8x14"
```

### 7. Setup PADD
```
cd ~ && \
wget -N https://raw.githubusercontent.com/pi-hole/PADD/master/padd.sh && \
sudo chmod +x padd.sh
```


- Auto start script: `nano ~/.bashrc`
```
# Run PADD
# If we’re on the PiTFT screen (ssh is xterm)
if [ "$TERM" == "linux" ] ; then
  while :
  do
    ./padd.sh
    sleep 1
  done
fi
```

- Set console auto login: `sudo raspi-config`


### 8. Setup port forwarding and DNS
- WireGuard UDP Port: `51820`




### LCD troubleshooting
`dmesg | grep "fb\|graphics\|display\|touch\|ads"`



### LCD-show (broken)
Reference: https://github.com/goodtft/LCD-show/
```
sudo rm -rf LCD-show
git clone https://github.com/goodtft/LCD-show.git
chmod -R 755 LCD-show
cd LCD-show/
sudo ./LCD35-show
```
