All done on 7-8/9/26

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


