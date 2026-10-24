<img width="468" height="29" alt="image" src="https://github.com/user-attachments/assets/4945951f-833b-4452-a635-f709a1a8f31b" />All done on 7-8/9/26

#Setting up the Pi

Loaded SD card in imager on my mac- Raspberry Pi 64 bit with desktop. Named it FredPi1.

Version 6.18.39 (June 2026 image, updated on first boot)

I wrote my home wifi SSID and password into the SD card so it connected automatically on first boot.

Over my home wifi:

```bash
ping -c 3 fredpi1.local
```
to confirm the Pi was connected

```bash
  ssh fred@fredpi1.local
```

typed in password and then worked remotely on Pi

```bash
  sudo apt update && sudo apt full-upgrade -y
```

apt is Debian's package manager. update refreshes the list of available software versions; full-upgrade installs newer versions of everything already on the card (the image was built in June, so three months of fixes were missing). -y answers yes to the confirmation prompt. && means "then, if that succeeded".

```bash
  sudo raspi-config
```

Opens the Pi's settings menu. Interface Options → VNC → Yes. Turns on the built-in VNC server (wayvnc), which streams the Pi's desktop to a viewer on the Mac.

```bash
  systemctl status wayvnc
```

Checks the VNC server is running ("active (running)").

Then worked within the VNC window using the Pi's own browser

#Installing VitalDB

Downloaded pivr64_1.19.27 (Raspberry Pi 64-bit build) from vitaldb.net

It downloaded into ~/Downloads

```bash
  ls -l ~/Downloads
```

Lists the download folder, so you can see that the file is there

```bash
  chmod +x ~/Downloads/pivr64_1.19.27
```

This makes the file executable, which is required for Linux to run it.

```bash
  file ~/Downloads/pivr64_1.19.27
```

This reports what kind of file it is (it's an ARM 64-bit programme)

```bash
~/Downloads/pivr64_1.19.27
```

This ran VitalDB. N.b. it fialed to run with the error message: libwiringPi.so: cannot open shared object file, so I had to download WiringPi first

```bash
cd ~/Downloads
```

This changed to the downloads folder so the next download landed there

```bash
curl -s https://api.github.com/repos/WiringPi/WiringPi/releases/latest | grep browser_download_url | grep arm64 | grep deb | cut -d '"' -f 4 | xargs wget
```

A chain: curl asks GitHub for the details of WiringPi's latest release; the three greps pick out the download link for the 64-bit ARM .deb package; cut extracts just the URL from that line; xargs wget downloads it. Result: wiringpi_3.20_arm64.deb

```bash
sudo apt install -y ./wiringpi_*arm64.deb
```

This installs WiringPi, and tells apt it's a local file

```bash
~/Downloads/pivr64_1.19.27
```

Ran VitalDB again and now it worked!

Then looked in terminal window, and could see VitalDB status lines ("uploading done" then CPU/RAM usage), until updates stopped with Ctrl-C

#Configuring VitalDB

```bash
cat ~/Downloads/vr.conf
```

This prints the config file the program created on first run. cat shows a file's contents. Cat means open a file as 'read only' quickly (whereas nano means open in text editor)


```bash
nano ~/Downloads/vr.conf
```

This opens the file in the nano text editor. Ctrl-K cuts a line, Ctrl-O then Enter saves, Ctrl-X exits

The following edits were made (in the nano text editor):

1) Removed "skip=1" frmo the Demo service so it would record.

Ran the programme for 2 mins then:

```bash
find ~ -name "*.vital"
```

Which searches for any .vital file. A recording was found, proving it records locally.

2) Changed "type=Demo" to "type=Bx50" as this was the monitor to be used. Added "port=USB0", renamed the header to "[DEV/B450]"

```bash
~/Downloads/pivr64_1.19.27
```

This ran VitalDB again. Now it kept printing "opening USB0 / opening /dev/ttyUSB0 repeatedly which showed that the device was accepted, and it kept trying to open a serial adapter that wasn't plugged in.

3) Added "SERVER_IP=127.0.0.1" under the "[BED/...]" header. The Pi is set up to stream data over the internet. Even though we planned to run it offline, in case it was inadvertently connected to internet, we wanted to disable this feature. However the build ignores the server IP; it was still trying to stream to a different address. See how this was resolved later in the log.


```bash
sudo systemctl restart vitalrecorder
sleep 15
journalctl -u vitalrecorder -n 40
```

This is how we saw that the Pi is able to handshake with the VitalDB sever online.

# Exporting recordings to the Mac

```bash
scp "fred@fredpi1.local:recordings/*.vital" ~/Desktop/
```

scp copies files over ssh. Source is the Pi's recording folder, destination is the Mac desktop. I then opened the file in VitalRecorder on my Mac and opened (with the Demo device invented signals).

# Set up autostart

```bash
sudo tee /etc/systemd/system/vitalrecorder.service > /dev/null <<'EOF'

[Unit]

Description=Vital Recorder

After=network.target

[Service]

User=fred

WorkingDirectory=/home/fred/Downloads

ExecStart=/home/fred/Downloads/pivr64_1.19.27

Restart=always

RestartSec=5

[Install]

WantedBy=multi-user.target

EOF
```
This sets up commands to make the Pi run the program as user Fred 5 seconds after any crash, and start it at boot once networking is up. .tee writes what it receives toa. file, > /dev/null stops it echoing.

'''bash
sudo systemctl daemon-reload
```

This makes systemd re-read its service files to it notices the new one just written above.

```bash
sudo systemctl daemon-reload
```

enable means start at every boot, --now means also start it immediately

```bash
systemctl status vitalrecorder
```

This shows whether vitalrecorder is running, and it's last few log lines. From this point on, Vital Recorder is always running in the background.

# Setting up the hotspot

```bash
sudo nmcli device wifi hotspot ifname wlan0 ssid FredPi password "..."
```

nmcli controls NetworkManager, the Pi's networking. This tells the Wi-Fi chip (wlan0) to stop being a client and become an access point broadcasting the network "FredPi" with the given password. The Pi immediately leaves the home Wi-Fi, so the ssh session freezes. (One attempt failed with a ? accidentally at the start of the line, and one with a missing closing quote, which left the shell waiting with a > prompt; Ctrl-C cancels that.)

Jointed wifi 'FredPi' on my Mac, then in terminal:

```bash
ssh fred@10.42.0.1
```

N.b. the pi is always 10.42.0.1 on its own hotspot

```bash
sudo nmcli connection modify Hotspot connection.autoconnect yes connection.autoconnect-priority 100
```

This makes the hotspot come up automatically at boot, and be preferred over the saved wifi (my home wifi)

```bash
nmcli connection show
```
Lists the saved network connections so I could find my home wifi

```bash
sudo nmcli connection down Hotspot && sudo nmcli connection up "NAME"
``` 
makes the hotspot turn off, and the pi rejoin my home wifi.

```bash
ssh fred@fredpi.local
```

reconnected my mac to the pi over my home wifi

```bash
sudo reboot
```

(Did this via my home wifi connection) and then the pi rebooted and make its hotspot automatically again.


# Blocking upload to VitalDB

```bash
sudo nmcli connection down Hotspot && sudo nmcli connection up "NAME"
``` 
then

```bash
ssh fred@fredpi.local
```

Did the above 2 steps again to get the pi back onto the internet via my home wifi


```bash
sudo systemctl restart vitalrecorder
sleep 15
journalctl -u vitalrecorder -n 40
```

Here we're testing if the Pi is able to 'handshake' with the vitaldb server online (which we don't want). The output we got was The log showed connecting websocket server, then Server => nginx/1.28.3, Connection => upgrade, Upgrade => websocket and a Sec-WebSocket-Accept line. That’s a completed handshake with VitalDB’s server. So we learned that just altering the server ip for the specific bed didn't work.

```bash
sudo ss -tnp | grep pivr
```

This looks for the IP address of the handshake- it showed 13.209.228.61:443 - an amazon IP in Seoul- the VitalDB sever. ss lists network connections. pivr filters for vitalrecorder

```bash
journalctl -u vitalrecorder | grep -i -E "connect|wss|https|://" | head
```

This searches the whole recorder log for connection-related lines. It showed that, whilst 'offline' on the hotspot, 'wss connect error' ie a connection error message, appeared every minute.

```bash
strings ~/Downloads/pivr64_1.19.27 | grep -i -E "vitaldb|wss://|https://" | head -20
```

Strings pulls readable text out of a programme file (here the vitalrecorder programme). We found the hostnames "vitaldb.net", "mod.vitaldb.net" and "filter.vitaldb.net". These names are used to connect the pi to the server, so we made a name override:

```bash
echo "127.0.0.1 vitaldb.net mod.vitaldb.net filter.vitaldb.net" | sudo tee -a /etc/hosts
```

For the Pi to handshake the VitalDB server, it has to find the numeric address. It does this by consulting DNS, the Domain name system, the address book of the worldwide web. However, the pi first consults a local file "etc/hosts" before consulting the DNS, which functions as a local override. So we have 'overwritten' the address to 127.0.0.1, which is the loopback address to the pi itself

So we have made a local override to stop data being exported, which can only be disabled by re-writing it out of the local "etc/hosts" file on the pi.

```bash
echo "127.0.0.1 vitaldb.net mod.vitaldb.net filter.vitaldb.net" | sudo tee -a /etc/cloud/templates/hosts.debian.tmpl
```

When the Pi re-boots, the "etc/hosts" file is regenerated from local template. This command pipes the loopback address in so that it will stay whenever the Pi reboots. The template is needed because one SD card image gets written to thousands of Pis, each with a different name, so the image can't ship a finished /etc/hosts. It ships a template with a blank where the hostname goes, and each Pi fills in its own — FredPi1 in my case — at first boot. It redoes this at every boot, which is why a change made only to /etc/hosts disappears.

```bash
sudo systemctl restart vitalrecorder

sleep 20

journalctl -u vitalrecorder -n 30 | grep -i -E "websocket|nginx|wss"

sudo ss -tnp | grep pivr
```

This restards the pi, then shows the output, filtering for the web connection. The result- 'wss connection error'- this verifies the block.

'''bash
rm ~/'udo ss -tnp | grep pivr'
```

This deleted a stray file that I had created by mis-typing a > in terminal
