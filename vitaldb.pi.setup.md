This document describes how I set up vital recorder on a Raspberry Pi and used it to record from a GE B650 monitor offline. Files can then be periodically extracted by SSH over it's hotspot.

# Home setup done on 7-8/9/26

## Setting up the Pi

Bought a Raspberry Pi 4 Model B 4GB starter pack from PiHut

## Writing the SD card

Loaded SD card in imager on my mac- Raspberry Pi 64 bit with desktop. 

Downloaded Raspberry Pi OS (64-bit desktop) Version 6.18.34 (June 2026 image, updated on first boot tp Version 6.18.39)

Customisation: Named it FredPi1, username fred, password was set, home Wifi and password, Wirelass LAN country GB, timezone London, services tab- Enable SSH with password authentication.

Inserted SD card into Pi and powered up, First attempt, it failed to join Wi-Fi (most like a typo I made in customisation). So I re-wrote SD card in imager and tried again:

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

Downloaded RealVNC viewer so I could connect to the Pi with VNC.

Then worked within the VNC window using the Pi's own browser

## Installing Vital Recorder

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

This ran Vital recorder. N.b. it fialed to run with the error message: libwiringPi.so: cannot open shared object file, so I had to download WiringPi first

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

Ran Vital Recorder again and now it worked!

Then looked in terminal window, and could see Vital recorder status lines ("uploading done" then CPU/RAM usage), until updates stopped with Ctrl-C

## Configuring Vital Recorder

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

2) Changed "type=Demo" to "type=B450" as this was the monitor to be used. Added "port=USB0", renamed the header to "[DEV/B450]" n.b. this name is actually incorrect because the Pi is plugged into a B650- i have mislabelled it.

```bash
~/Downloads/pivr64_1.19.27
```

This ran Vital Recorder again. Now it kept printing "opening USB0 / opening /dev/ttyUSB0 repeatedly which showed that the device was accepted, and it kept trying to open a serial adapter that wasn't plugged in.

## Exporting recordings to the Mac

```bash
scp "fred@fredpi1.local:recordings/*.vital" ~/Desktop/
```

scp copies files over ssh. Source is the Pi's recording folder, destination is the Mac desktop. I then opened the file in VitalRecorder on my Mac and opened (with the Demo device invented signals).

## Set up autostart

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
This sets up commands to make the Pi run the program as user fred 5 seconds after any exit, and start it at boot once networking is up. .tee writes what it receives toa. file, > /dev/null stops it echoing.

```bash
sudo systemctl daemon-reload
```

This makes systemd re-read its service files to it notices the new one just written above.

```bash
sudo systemctl enable --now vitalrecorder
```

enable means start at every boot, --now means also start it immediately

```bash
systemctl status vitalrecorder
```

This shows whether vitalrecorder is running, and it's last few log lines. From this point on, Vital Recorder is always running in the background.

## Setting up the hotspot

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
ssh fred@fredpi1.local
```

reconnected my mac to the pi over my home wifi

```bash
sudo reboot
```

(Did this via my home wifi connection) and then the pi rebooted and make its hotspot automatically again.


## Blocking upload to VitalDB

```bash
sudo nmcli connection down Hotspot && sudo nmcli connection up "NAME"
``` 
then

```bash
ssh fred@fredpi1.local
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

So we have made a local override to stop data being exported, however it will get re-written during a reboot so:

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

This restarts vital recorder, then shows the output, filtering for the web connection. The result- 'wss connect error'- this verifies the block.

```bash
rm ~/'udo ss -tnp | grep pivr'
```

This deleted a stray file that I had created by mis-typing a > in terminal

## Reboot test

```bash
sudo reboot
```

Then jointed FredPi wifi on the mac, then in terminal:

```bash
ssh fred@10.42.0.1
grep vitaldb /etc/hosts
systemctl status vitalrecorder
```

It showed vitalrecorder was running and the block line was still working.

# Theatre set up 9/9/26

Connected Pi to GE B650 monitor. USB monitor port USB-UC232A adapter cable (used an ATEN with a serial code below Z3L1 so it's compatible with the monitor's installed driver), female to female null-modem, UC232A to USB adapter cable (brand and serial code don't matter) into the Pi's USB port.

Initially done with no patient in room, then a patient was connected, which was actually useful to test the Pi.

Joined FredPi wifi on my mac, then used terminal:

## Check time is correct

```bash
ssh -t fred@10.42.0.1 "sudo date -u -s '$(date -u '+%Y-%m-%d %H:%M:%S')'"
```

This was reading the time on the Pi (which was still on last night's time, when it was last plugged in. This code was attempting to set the date of the Pi to the date of the mac, but $(date -u... just reads the date of the Pis clock, so it didn't correct the time.

```bash
exit
```

Left the ssh so back onto the mac on the terminal

```bash
ssh -t fred@10.42.0.1 "sudo date -u -s '$(date -u '+%Y-%m-%d %H:%M:%S')'"
```

Same command again from the mac now sets the time on the Pi.

```bash
ssh fred@10.42.0.1 date
```

Asks the Pi what the time is, It showed Wed 9 Sep 10:42:25 BST, the correct time.

## Check the Pi is running

```bash
ssh fred@10.42.0.1
journalctl -u vitalrecorder -f
```

This shows the live log. It showed "waiting for next patient", minute-ly status lines (CPU 1%, temp 48degC) and the block Vital recorder connection failing every minute as designed

```bash
Ctrl-C
```

Stops watching the live log

```bash
journalctl -u vitalrecorder -n 300 | grep -v -E "cpu|ram|websocket|wss"
ls -l ~/recordings
```

This shows the last 300 log lines with routine noies removed (grep -v means show lines that do not match those 4 terms) The output:

- opening USB0 repeated for ~6 seconds after boot, then hardware handshake and  [......../Bx50] reading thread started (sport = USB0, sock=-1) -ie the serial port opened and the recorder began reading the monitor

- /........_260908_211942.vital is recording- ie a file is being written. n.b. the time and date is wrong because this recording was started prior to clock sync.

- upload started ... failed: Could not connect to server ... upload error - the recorder tries to upload all finished .vital files to VitalDB every few minutes, but our block stops this.

- At the moment of the time sync, the recorder closed the above file and opened a new one with correct timestamps

- ls -l ~/recordings showed 3 files: the demo file (7 Sept), the wrong-time file and the new correctly-timed file that was still being recorded.

So we knew the pi worked

## Checking exported files are valid

On the mac, in terminal:

```bash
scp "fred@10.42.0.1:recordings/........_260909_104205.vital" ~/Desktop/
```

This exports the new, correctly-timed file to Mac. I could then open the file in Vital Recorder on mac and view it.

# First 'check-up' and clock install 14/9/26

## Exporting files 

```bash
ssh fred@10.42.0.1 date
```

Asks the Pi what the time, is confirmed time was correct. If someone had inadvertently unplugged the Pi, then plugged it back in before this point, then time would have become incorrect, but this had not happenned, so the files were correctly timed.

On mac in terminal

```bash
mkdir -p ~/Desktop/vital_files
scp "fred@10.42.0.1:recordings/*.vital" ~/Desktop/vital_files/
```

mkdir -; makes a folder. scp copies every .vital file from the pi into it. 

Opened up these files in VitalRecorder on the mac and they work. n.b. I did not delete off the Pi at this point.

## Installing the clock

Bought a DS3231 "Mini RTC Module" from PiShop

Shut down Pi and unplugged. Fitted the module on the first 5 mins of the outer GPIO. Powered on and ssh in.

```bash
sudo raspi-config nonint do_i2c 0
echo "dtoverlay=i2c-rtc,ds3231" | sudo tee -a /boot/firmware/config.txt
sudo apt -y remove fake-hwclock
sudo reboot
```

Line 1 turns on the I2C bus (nonint = non-interactvive, 0 = enable) that the clock chip talks over. Line 2 appends a line to the boot config telling the Pi it's there. Line 3 removes the old 'fake clock' that just takes the time before cutout as time on reboot. Line 4 reboots the pi.

After reboot:

```bash
ls /dev/rtc0
dmesg | grep -i rtc
```

/dev/rtc0 existed, and the kernel log showed rtc-ds1307 1-0068: registered as rtc0 (the DS3231 uses the ds1307 driver, at I2C address 0x68) and setting system clock to 2000-01-01: the chip was found but unset, and the Pi had adopted its year-2000 time, which is the RTC taking authority as designed.

Now need to push the correct time from the mac

```bash
ssh -t fred@10.42.0.1 "sudo date -u -s '$(date -u '+%Y-%m-%d %H:%M:%S')'"
```

This sets the pi's own clock to be correct:

```bash
sudo timedatectl set-ntp false
sudo timedatectl set-time "$(date '+%Y-%m-%d %H:%M:%S')"
sudo timedatectl set-ntp true
timedatectl
```

This sets up the RTC chip clock. set-time refuses to work while automatic sync is on, hence line 1. Line 2 sets the system time to what it already is (now correct), which as a side effect writes it into the clock chip. Line 3 turns automatic sync back on. Line 4 prints a status summary whose "RTC time" line should now be today.

Then tested by unplugging the Pi from power, plugging back in, then 

```bash
ssh fred@10.42.0.1
```

showed correct time, so it's working

```bash
journalctl -u vitalrecorder -f
```

Checking vital recorder's still working, and we saw repeated lines of "C: -1, V: -0.000 V" 10x a second. This is because VitalRecorder carries an analogue to digital converter on the I2C, where we installed the clock.

```bash
sudo sed -i '/^\[Service\]/a LogRateLimitIntervalSec=30s\nLogRateLimitBurst=100' /etc/systemd/system/vitalrecorder.service
```

We tried this way of limiting the rate of the recording from the I2C, but it didn't work, so we moved on to:

```bash
sudo tee /etc/systemd/system/vitalrecorder.service > /dev/null <<'EOF'
[Unit]
Description=Vital Recorder
After=network.target

[Service]
User=fred
WorkingDirectory=/home/fred/Downloads
ExecStart=/bin/sh -c '/home/fred/Downloads/pivr64_1.19.27 2>&1 | grep --line-buffered -v "C: -1"'
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl restart vitalrecorder
sleep 20
journalctl -u vitalrecorder -n 20
```

ExecStart, which now runs the recorder inside a shell, merges its error output into its normal output (2>&1), and pipes that through grep -v "C: -1" (print every line that does not contain that text; --line-buffered makes it pass lines on immediately rather than in batches). Also removed the duplicate rate-limit lines. Result: clean startup log, no voltage lines. Log entries are now tagged sh[pid] instead of pivr64_1.19.27[pid] because the shell is the process systemd sees; journalctl -u vitalrecorder is unchanged. This doesn't actualyl affect the .vital file, the grep filter just affects what reaches the log

# 18 September changed passwords to more secure ones, and exported more files then deleted them off the Pi

## Exported then deleted recordings

```bash
journalctl -u vitalrecorder -f
```

checked pi was working. The monitor was turned off, so I got "reading_thread finished" then "reopeing USB0" about every 30 seconds

``bash
mkdir -p ~/Desktop/vital_files
scp "fred@10.42.0.1:recordings/*.vital" ~/Desktop/vital_files/
```

This exported the files. N.b. I had a typo in my first attempt, above code is the correct one. I then opened 2 recordings on the mac to ensure they were genuine.

```bash
ls -l ~/recordings
rm ~/recordings/*.vital
ls -l ~/recordings
```

First and last line show the files (last one showed a blank as it was empty. Middle one deletes recordings. 

## Changed SSH and hotspot passwords so they were more secure

When logged into the Pi via ssh

```bash
passwd
```

This prompts the current password to be entered, then the new one twice

```bash
sudo nmcli connection modify Hotspot wifi-sec.psk 'NewPasswordHere'
```

This is how you change the wifi password

```bash
sudo nmcli -s -g 802-11-wireless-security.psk connection show Hotspot
```

This prints the wifi password so you can check it is what you think it is

```bash
sudo nmcli connection up Hotspot
```

This applies the new password, so your connection drops, so you then have to re-login to the hotspot wifi on the mac with the new password

```bash
history -c && rm -f ~/.bash_history
```

This clears the history on terminal, because the wifi password was written down as a line in terminal

# Things I didn't do that are useful

## script the terminal log and save it (including the pi's responses)

```bash
mkdir -p ~/pi-logs
script ~/pi-logs/$(date +%Y-%m-%d-%H%M).log
```

Doing this at the start of each session will give a log of the entire terminal conversation (both commands and pi responses)

## how to reverse our block on the Vital recorder vital db server streaming



