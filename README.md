# proxmox-igpu-display-out-HWaccel
Instructions to add a browser/video display out to a monitor or TV on Proxmox VE with only an igpu, while /dev/dri is already used for hardware acceleration in LXCs or containers, without the use of vfio, vd-t or virtio.

<h1>Background:</h1>  

Proxmox VE machines using mini PCs like Intel NUC and Beelink are inexpensive, small and widely available.  
These mini PCs are perfect for running Home Assistant, Frigate NVR, Plex, Jellyfin, etc as LXCs or containers.  
Usually, these software requires gpu hardware acceleration for video encoding/decoding by passing through /dev/dri/ and can be shared among LXCs and containers.  
However, Mini PCs usually only include an igpu without any option to add a dedicated gpu.  
Generally, to get a display out, a full desktop environment running in a VM is required, with full gpu passthrough.  
This means the vt-d IOMMU needs to be enabled, so all other LXCs, containers and even Proxmox VE itself will lose access to the gpu, including /dev/dri for HWaccel.  
Vfio is generally suggested for this type of use case, but the implementation is complicated and the gpu is switched between the desktop VM and the host.  
This is unideal as the gpu is unable to serve as display out and HWaccel at the same time.

<h1>Use Case:</h1>  

My Proxmox VE is installed on an Intel NUC with i3-1115g4 with UHD graphics igpu, running Home Assistant in a VM and Portainer in an LXC.  
Portainer runs various containers including Frigate NVR with /dev/dri/ mounted through Portainer LXC.  
I want my Proxmox Machine to be able to display an RTSP stream and/or a browser to a monitor via the HDMI port on the NUC, without interrupting /dev/dri to Portainer.  
Technically this can also be adapted to work for Plex media player, Jellyfin media player and other graphical programs.  

<h1>The Solution:</h1>  

Instead of spinning up a full desktop environment VM, I only need an RTSP media player and a browser.  
These can run on the Proxmox host itself, which is based on debian. Display out works as it can display CLI, but not graphical outputs as Proxmox is meant to be run headless.  
So, I need to install a graphical system (X11), RTSP media player (MPV) and a browser (Chromium), plus some other tweaks to make this work.  

<h1>Caution:</h1>  

It is adviced not to run X11, MPV and Chromium with root access due to security vulnerabilities, but it is the easiest way to test run the programs.  
Please keep in mind that this is only for testing. For long term deployment please run these programs as a non-root user.  

<h1>Graphical System (X11):</h1>  

To install X11, run the following commands in node shell or ssh:  
```
apt-get update
apt-get install xorg xserver-xorg x11-xserver-utils x11-utils
```
You will also need dbus-x11 for Chromium (ignore if only MPV is needed as only access to the frame buffer is needed):  
```
sudo apt install dbus-x11
```
Next, install a Window Manager or Desktop Environment. Openbox is a great choice for lightweight minimal window manager:  
```
sudo apt-get install openbox
```
If you need a lightweight minimal desktop environment for more features and control, i3 can be a great option. But for just MPV and Chromium, openbox is good enough.  
Run ```startx``` to check if it is installed correctly. The CLI session will stop as startx takes up the session. Make sure to plug in a monitor to check.  
Ssh into proxmox using another program like Putty to start another session or use tmux in the CLI. Run ```xrandr``` to check for connected displays.  
If "Can't open display" is shown, run ```export DISPLAY=:0``` to set the display environment. If only one display is connected DISPLAY:0 should be the default.  
Run ```xrandr``` again to check for connected displays, the output should be similar to this:  
```
DP-2 connected 1920x1080+0+0 (normal left inverted right x axis y axis)
```
The connected monitor should show a blank screen with a mouse courser.  
Running all the commands manually on every reboot is tedious, so create a systemd startx.service to run on boot in the background:  
```
sudo nano /etc/systemd/system/startx.service
```
Add in the following to the file, then save and exit:  
```
[Unit]
Description=Start X11 session
After=multi-user.target

[Service]
ExecStart=/usr/bin/startx
User=root
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```
Run these to apply the service file and enable on boot:  
```
sudo systemctl daemon-reload
sudo systemctl enable startx.service
```
Reboot and check if startx runs successfully, it should show the service is active:  
```
sudo systemctl status startx.service
```
To see the full log:  
```
journalctl -u startx.service
```
Run ```xrandr``` to check for connected displays. If anything doesn't work, check the log for errors.  

<h1>RTSP Media Player (MPV):</h1>  

To install MPV, run the following command:  
```
sudo apt update
sudo apt install mpv
```
Next, create a systemd service file, I chose the name frigate-mpv.service as I will be running an RTSP stream via go2rtc:  
```
sudo nano /etc/systemd/system/frigate-mpv.service
```
Add in the following to the file, your-rtsp-url should be replaced with the rtsp stream to be played, then save and exit:  
```
[Unit]
Description=frigate-mpv
StartLimitIntervalSec=0
After=multi-user.target

[Service]
User=root
Environment=DISPLAY=:0
ExecStartPre=/bin/sleep 10
ExecStart=/bin/bash -c '/usr/bin/mpv --stop-playback-on-init-failure --no-cache --profile=low-latency --rtsp-transport=tcp --vo=gpu --fullscreen rtsp://your-rtsp-url>
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```
```ExecStartPre=/bin/sleep 10``` is used to give startx.service time to initialize before frigate-mpv.service.   
The normal way to do this is by using ```After=graphical.target``` under [Unit]. However, sometimes the service doesn't start properly for my setup.  
Run these to apply the service file and enable on boot:  
```
sudo systemctl daemon-reload
sudo systemctl enable frigate-mpv.service
```
Start the service to check if it runs successfully and check its status, it should show the service is active:  
```
sudo systemctl start frigate-mpv.service
sudo systemctl status frigate-mpv.service
```
To see the full log:  
```
journalctl -u frigate-mpv.service
```
The monitor should show an RTSP stream being played. Reboot to check if both startx and mpv run on boot.  

<h1>Browser (Chromium):</h1>  

To install Chromium, run the following command:  
```
sudo apt update
sudo apt install chromium
```
Next, create a systemd service file, I chose the name chromium-kiosk.service as I will be running it in kiosk mode:  
```
sudo nano /etc/systemd/system/chromium-kiosk.service
```
Add in the following to the file, your-webpage-url should be replaced with the webpage to be displayed, then save and exit:  
```
[Unit]
Description=Chromium Kiosk Mode
After=multi-user.target

[Service]
User=root
Environment=DISPLAY=:0
Environment=DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/0/bus
ExecStartPre=/bin/sleep 10
ExecStart=/usr/bin/dbus-launch --exit-with-session /usr/bin/chromium --no-sandbox --kiosk --disable-translate --disable-background-networking --disable-software-rasterizer --enable-gpu-rasterization --enable-oop-rasterization --enable-gpu-compositing --enable-accelerated-2d-canvas --enable-zero-copy --canvas-oop-rasterization --enable-accelerated-video-decode --enable-accelerated-video-encode --enable-features=VaapiVideoDecoder,VaapiVideoEncoder,VaapiIgnoreDriverChecks --no-first-run --disable-infobars --ignore-gpu-blocklist --start-fullscreen https:/your-webpage-url
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```
```ExecStartPre=/bin/sleep 10``` is used to give startx.service time to initialize before chromium-kiosk.service.   
The normal way to do this is by using ```After=graphical.target``` under [Unit]. However, sometimes the service doesn't start properly for my setup.
Run these to apply the service file and enable on boot:  
```
sudo systemctl daemon-reload
sudo systemctl enable chromium-kiosk.service
```
Start the service to check if it runs successfully and check its status, it should show the service is active:  
```
sudo systemctl start chromium-kiosk.service
sudo systemctl status chromium-kiosk.service
```
To see the full log:  
```
journalctl -u chromium-kiosk.service
```
The monitor should show the webpage in full screen kiosk mode. Reboot to check if both startx and chromium run on boot.  

<h1>Recommended Tweaks:</h1>

If you are using MPV or Chromium without a keyboad and mouse attached, it can be troublesome to wake up the monitor when it goes to sleep or power saving mode.
To fix this, we can edit ~/.bashrc to disable sleep and power saving on the monitor on boot:
```
sudo nano ~/.bashrc
```
Add in these lines at the end of the file:
```
export DISPLAY=:0
xset s off
xset -dpms
xset s noblank
```
To apply the changes:
```
source ~/.bashrc
```
