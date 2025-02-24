# proxmox-igpu-display-out-HWaccel
Instructions to add a browser/video display out to a monitor or TV on Proxmox VE with only an igpu, while /dev/dri is already used for hardware acceleration in LXCs or containers, without the use of vfio, vd-t or virtio.

**Background:**  
Proxmox VE machines using mini PCs like Intel NUC and Beelink are inexpensive, small and widely available.  
These mini PCs are perfect for running Home Assistant, Frigate NVR, Plex, Jellyfin, etc as LXCs or containers.  
Usually these software requires gpu hardware acceleration for video encoding/decoding by passing through /dev/dri/ and can be shared among LXCs and containers.  
However, Mini PCs usually only includes an igpu without any option to add a dedicated gpu.  
Generally, to get a display out, a full desktop environment running in a VM is required, with full gpu passthrough.  
This means the vt-d IOMMU needs to be enabled, so all other LXCs, containers and even Proxmox VE itself will lose access to the gpu, including /dev/dri for HWaccel.  
Vfio is generally suggested for this type of use case, but the implementation is complicated and the gpu is switched between the desktop VM and the host.  
This is unideal as the gpu unable to serve as display out and HWaccel at the same time.

**Use Case:**  
My Proxmox VE is installed on an Intel NUC with i3-1115g4 with UHD graphics igpu, running Home Assistant in a VM and Portainer in an LXC.  
Portainer runs various containers including Frigate NVR with /dev/dri/ mounted through Portainer LXC.  
I want my Proxmox Machine to be able to display an RTSP stream and/or a browser to a monitor via the HDMI port on the NUC, without interrupting /dev/dri to Portainer.  
Technically this can also be adapted to work for Plex media player, Jellyfin media player and other graphical programs.  

**The Solution:**  
Instead of spinning up a full desktop environment VM, I only need an RTSP media player and a browser.  
These can run on the Proxmox host itself, which is based on debian. Display out works as it can display CLI, but not graphical outputs as Proxmox is meant to be run headless.  
So I need to install a graphical system (X11), RTSP media player (MPV) and a browser (Chromium), plus some other tweaks to make this work.  

**Caution:**  
It is adviced not to run X11, MPV and Chromium with root access due to security vulnerabilities, but it is the easiest way to test run the programs.  
Please keep in mind that this is only for testing. For long term deployment please run these programs as a non-root user.  

**Graphical System (X11):**  
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
The connected monitor should shows a blank screen with a mouse courser.  
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
