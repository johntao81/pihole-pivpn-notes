# pihole-pivpn-notes
Notes for installing PiHole and PiVPN on Raspberry Pi

### 1. Prepare the Raspberry Pi
- Download the Raspberry Pi OS Lite image: https://www.raspberrypi.org/software/operating-systems/
- Burn the image and enable `ssh`
- `ssh raspberrypi.local`
- `sudo raspi-config`
 - 1 System Options
   - S3 Password
   - S4 Hostname
 - 5 Localisation Options
   - L2 Timezone
 - 3 Interface Options
   - P1 Camera
   - P3 VNC (for the desktop image)
   - P5 I2C (for the SSD1306 LCDs)
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

- Guide: https://docs.pivpn.io/wireguard/



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

- Set console auto login: 
  - `sudo raspi-config`
  - S5 Boot / Auto Login
    - B2 Console Autologin


### 8. Setup port forwarding and DNS
- WireGuard UDP Port: `51820`




### LCD troubleshooting
`dmesg | grep "fb\|graphics\|display\|touch\|ads"`
```
[    0.000000] Kernel command line: coherent_pool=1M 8250.nr_uarts=1 snd_bcm2835.enable_compat_alsa=0 snd_bcm2835.enable_hdmi=1 bcm2708_fb.fbwidth=640 bcm2708_fb.fbheight=480 bcm2708_fb.fbswap=1 vc_mem.mem_base=0x3ec00000 vc_mem.mem_size=0x40000000  console=ttyS0,115200 console=tty1 root=PARTUUID=fd714c66-02 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait fbcon=map:10 fbcon=font:ProFont6x11
[    1.954558] bcm2708_fb soc:fb: FB found 1 display(s)
[    1.974901] bcm2708_fb soc:fb: Registered framebuffer for display 0, size 640x480
[   10.165855] fbtft: module is from the staging directory, the quality is unknown, you have been warned.
[   10.173752] fb_ili9486: module is from the staging directory, the quality is unknown, you have been warned.
[   10.174981] fb_ili9486 spi0.0: fbtft_property_value: regwidth = 16
[   10.175008] fb_ili9486 spi0.0: fbtft_property_value: buswidth = 8
[   10.175032] fb_ili9486 spi0.0: fbtft_property_value: debug = 0
[   10.175052] fb_ili9486 spi0.0: fbtft_property_value: rotate = 90
[   10.175074] fb_ili9486 spi0.0: fbtft_property_value: fps = 30
[   10.175092] fb_ili9486 spi0.0: fbtft_property_value: txbuflen = 32768
[   10.328244] ads7846 spi0.1: supply vcc not found, using dummy regulator
[   10.330825] ads7846 spi0.1: touchscreen, irq 200
[   10.837908] graphics fb1: fb_ili9486 frame buffer, 480x320, 300 KiB video memory, 32 KiB buffer memory, fps=33, spi0.0 at 16 MHz
```


### LCD-show (broken)
Reference: https://github.com/goodtft/LCD-show/
```
sudo rm -rf LCD-show
git clone https://github.com/goodtft/LCD-show.git
chmod -R 755 LCD-show
cd LCD-show/
sudo ./LCD35-show
```
