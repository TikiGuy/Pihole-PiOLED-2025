# Pihole-PiOLED-2025 Guide

Welcome to the comprehensive step-by-step guide to setting up Pi-hole, with Unbound and the Adafruit I2C PiOLED on a Raspberry Pi Zero 2 W!

I recently decided to finally get Pi-hole up and running at home. When looking at buying two Raspberry Pi Zero 2 W devices, I saw the instructions on how to set up Pi-hole with the PiOLED display on [Adafruit's Website](http://learn.adafruit.com/pi-hole-ad-blocker-with-pi-zero-w).

However, I ran into quite a few issues and had to tweak a few things which I'll outline below.

### Key Changes to Be Aware Of
1. **Pi-hole API Authentication:** The biggest issue revolves around the fact that Pi-hole changed the way their API works. Specifically, the authentication is no longer key-based, it's session-based, and they also changed how the API functions.
2. **Python Virtual Environments:** There are changes to how Python operates in the Bookworm versions of Raspberry Pi OS. Python scripts are now separated into Virtual Environments (venv). For those of us who aren't programmers and are looking for true step-by-step directions, it's pretty easy to get a little turned around.

*(Note: I also used the excellent [Crosstalk Solutions Pi-hole and Unbound Tutorial](https://www.crosstalksolutions.com/the-worlds-greatest-pi-hole-and-unbound-tutorial-2023/) in formulating my approach.)*

---

## Prerequisites

- You've imaged your SD card with the [lite 64-bit version of Raspberry Pi OS](https://learn.adafruit.com/pi-hole-ad-blocker-with-pi-zero-w/prepare-the-pi).
- You've installed the PiOLED display on the Raspberry Pi Zero 2 W.
- You've set a static IP (Most do this via their DHCP server, often the router or gateway).
- You have internet connectivity to the Pi.

---

## Phase 1: System Preparation

### 1. Update the Raspberry Pi
As always, let's update the Raspberry Pi first.
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Python Packages
```bash
sudo apt-get install python3-pip
sudo apt install --upgrade python3-setuptools
```

### 3. Create and Activate the Pi-hole Virtual Environment (venv)
```bash
python3 -m venv pihole --system-site-packages
source pihole/bin/activate
```
![image](https://github.com/user-attachments/assets/882ae262-fed3-4800-a30c-66548d9b58c9)

---

## Phase 2: PiOLED Screen Setup

### 4. Install Blinka (Required for screen)
*(Source: [Adafruit CircuitPython on Linux](https://learn.adafruit.com/circuitpython-on-raspberrypi-linux/installing-circuitpython-on-raspberry-pi))*

```bash
pip3 install --upgrade adafruit-python-shell
wget https://raw.githubusercontent.com/adafruit/Raspberry-Pi-Installer-Scripts/master/raspi-blinka.py
sudo -E env PATH=$PATH python3 raspi-blinka.py
```
This will require a reboot. Once rebooted, reconnect to the SSH session.

### 5. Reactivate the Pi-hole venv
We will set this up later to activate automatically on boot. For now:
```bash
source pihole/bin/activate
```

### 6. Create `blinkatest.py` to Test Blinka
Let's create the file using nano:
```bash
nano blinkatest.py
```
Paste the following test code into the file:

```python
import board
import digitalio
import busio

def main():
    print("Hello, blinka!")

    # Try to create a Digital input
    pin = digitalio.DigitalInOut(board.D4)
    print("Digital IO ok!")

    # Try to create an I2C device
    i2c = busio.I2C(board.SCL, board.SDA)
    print("I2C ok!")

    # Try to create an SPI device
    spi = busio.SPI(board.SCLK, board.MOSI, board.MISO)
    print("SPI ok!")

    print("done!")


if __name__ == "__main__":
    main()
```

Press `Ctrl + X` to exit, type `Y` to save, and press `Enter` to confirm the filename. Now execute it:
```bash
python3 blinkatest.py
```
You should see the following output:
![image](https://github.com/user-attachments/assets/17eeea91-d06f-420c-9457-17433657429a)

### 7. Install the PiOLED Library and PIL (Required for screen)
```bash
pip3 install adafruit-circuitpython-ssd1306
sudo apt-get install python3-pil
```

### 8. Create `systemstats.py` to Test Display Output
```bash
nano ~pi/systemstats.py
```
Paste the code below into nano, exit, and save:

```python
# SPDX-FileCopyrightText: 2017 Tony DiCola for Adafruit Industries
# SPDX-FileCopyrightText: 2017 James DeVito for Adafruit Industries
# SPDX-License-Identifier: MIT

# This example is for use on (Linux) computers that are using CPython with
# Adafruit Blinka to support CircuitPython libraries. CircuitPython does
# not support PIL/pillow (python imaging library)!

import subprocess
import time

import busio
from board import SCL, SDA
from PIL import Image, ImageDraw, ImageFont

import adafruit_ssd1306

# Create the I2C interface.
i2c = busio.I2C(SCL, SDA)

# Create the SSD1306 OLED class.
# The first two parameters are the pixel width and pixel height.  Change these
# to the right size for your display!
disp = adafruit_ssd1306.SSD1306_I2C(128, 32, i2c)

# Clear display.
disp.fill(0)
disp.show()

# Create blank image for drawing.
# Make sure to create image with mode '1' for 1-bit color.
width = disp.width
height = disp.height
image = Image.new("1", (width, height))

# Get drawing object to draw on image.
draw = ImageDraw.Draw(image)

# Draw a black filled box to clear the image.
draw.rectangle((0, 0, width, height), outline=0, fill=0)

# Draw some shapes.
# First define some constants to allow easy resizing of shapes.
padding = -2
top = padding
bottom = height - padding
# Move left to right keeping track of the current x position for drawing shapes.
x = 0

# Load default font.
font = ImageFont.load_default()

while True:
    # Draw a black filled box to clear the image.
    draw.rectangle((0, 0, width, height), outline=0, fill=0)

    # Shell scripts for system monitoring from here:
    # https://unix.stackexchange.com/questions/119126/command-to-display-memory-usage-disk-usage-and-cpu-load
    cmd = "hostname -I | cut -d' ' -f1"
    IP = subprocess.check_output(cmd, shell=True).decode("utf-8")
    cmd = 'cut -f 1 -d " " /proc/loadavg'
    CPU = subprocess.check_output(cmd, shell=True).decode("utf-8")
    cmd = "free -m | awk 'NR==2{printf \"Mem: %s/%s MB  %.2f%%\", $3,$2,$3*100/$2 }'"
    MemUsage = subprocess.check_output(cmd, shell=True).decode("utf-8")
    cmd = 'df -h | awk \'$NF=="/"{printf "Disk: %d/%d GB  %s", $3,$2,$5}\''
    Disk = subprocess.check_output(cmd, shell=True).decode("utf-8")

    # Write four lines of text.
    draw.text((x, top + 0), "IP: " + IP, font=font, fill=255)
    draw.text((x, top + 8), "CPU load: " + CPU, font=font, fill=255)
    draw.text((x, top + 16), MemUsage, font=font, fill=255)
    draw.text((x, top + 25), Disk, font=font, fill=255)

    # Display image.
    disp.image(image)
    disp.show()
    time.sleep(0.1)
```

### 9. Test the Display Script
```bash
python3 systemstats.py
```
Your display should light up after a few seconds with something like the following:

![systemstats](https://github.com/user-attachments/assets/c7816667-c827-46de-a558-672d0553b71c)

Congrats, the screen works!!!

### 10. Stop the Display Script
Stop the script by pressing `Ctrl + C`. You'll see some errors, but that's just because we killed the script with a KeyboardInterrupt.

**!!! Note:** If you leave a static output on your OLED screen for too long, it could cause burn-in. We'll address this with our final `stats.py` script later. It will be fine for a few minutes.

---

## Phase 3: Pi-hole Installation & Configuration

### 11. Install Pi-hole
```bash
curl -sSL https://install.pi-hole.net | bash
```
This will take a few minutes. (If you want detailed installation instructions, check out [Crosstalk Solutions Pi-hole guide](https://www.crosstalksolutions.com/the-worlds-greatest-pi-hole-and-unbound-tutorial-2023/#Install_Pi-hole).)

### 12. Set the Pi-hole Web Interface Password
This password will be used to log into the web interface AND as the API password (since Pi-hole now uses session authentication instead of key authentication).

```bash
sudo pihole setpassword
```

### 13. Set Pi-hole to Use the Wireless Interface (`wlan0`)
By default, Pi-hole might only respond to queries from its own subnet. When I attempted to change this in the web admin interface, there were no interfaces shown. This command makes the `wlan0` interface selectable:

```bash
sudo pihole-FTL --config dns.interface 'wlan0'
```

### 14. Log In to the Web Admin Interface
Using the browser of your choice, navigate to the IP you set for your Pi-hole:
`http://YOURIP/admin/login`

Input the password we set in Step 12.
![image](https://github.com/user-attachments/assets/251f98b8-3dd6-4df4-857f-f044dd4666ea)

### 15. Configure Pi-hole to Respond on `wlan0`
1. Click **Settings** > **DNS**
   ![image](https://github.com/user-attachments/assets/6a484917-c4e2-420c-bd8a-874ed6e58888)
2. Under "Interface Settings", toggle **Basic** to **Advanced**.
   ![image](https://github.com/user-attachments/assets/c6e2b7c2-35a7-472b-a2a2-bf5c92babdc8)
3. Click **Respond only on interface wlan0**.
   ![image](https://github.com/user-attachments/assets/5baedbb3-9f59-4fae-b3b6-d5c2432ae70c)
4. Click **Save & Apply** at the bottom right.
   ![image](https://github.com/user-attachments/assets/c6a9965b-57e0-4d46-8b2e-84c4294956a3)

---

## Phase 4: Unbound Setup

Next, let's install Unbound and point our Pi-hole to it. If you want to understand why this is beneficial, check out [Cross Talk's YouTube video](https://youtu.be/cE21YjuaB6o?t=1927).

### 16. Install and Configure Unbound
```bash
sudo apt install unbound -y
```

Setup the Unbound configuration file:
```bash
sudo nano -w /etc/unbound/unbound.conf.d/pi-hole.conf
```
Paste the following into the config file:

```yaml
server:
# If no logfile is specified, syslog is used
# logfile: "/var/log/unbound/unbound.log"
verbosity: 0

interface: 127.0.0.1
port: 5335
do-ip4: yes
do-udp: yes
do-tcp: yes

# May be set to yes if you have IPv6 connectivity
do-ip6: no

# You want to leave this to no unless you have *native* IPv6. With 6to4 and
# Terredo tunnels your web browser should favor IPv4 for the same reasons
prefer-ip6: no

# Use this only when you downloaded the list of primary root servers!
# If you use the default dns-root-data package, unbound will find it automatically
#root-hints: "/var/lib/unbound/root.hints"

# Trust glue only if it is within the server's authority
harden-glue: yes

# Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
harden-dnssec-stripped: yes

# Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
# see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
use-caps-for-id: no

# Reduce EDNS reassembly buffer size.
edns-buffer-size: 1232

# Perform prefetching of close to expired message cache entries
# This only applies to domains that have been frequently queried
prefetch: yes

# One thread should be sufficient, can be increased on beefy machines.
num-threads: 1

# Ensure kernel buffer is large enough to not lose messages in traffic spikes
so-rcvbuf: 1m

# Ensure privacy of local IP ranges
private-address: 192.168.0.0/16
private-address: 169.254.0.0/16
private-address: 172.16.0.0/12
private-address: 10.0.0.0/8
private-address: fd00::/8
private-address: fe80::/10
```

Close and save the file (`Ctrl + X` then `Y` and `Enter`).

Next, restart the Unbound service:
```bash
sudo service unbound restart
```
Check the status of the service:
```bash
sudo service unbound status
```
After `Active:`, you should see green text that says `active (running)`.

### Test Unbound
You can test a DNS query directly to Unbound using the following command:
```bash
dig crosstalksolutions.com @127.0.0.1 -p 5335
```
This should return output showing a successful query.
![image](https://github.com/user-attachments/assets/c36e0ce4-5b5c-499f-b418-931483e008a3)

### Point Pi-hole to Unbound
In the same **DNS** tab in the Pi-hole web interface, uncheck both Cloudflare boxes and enter the following into the **Custom 1 (IPv4)** box:
`127.0.0.1#5335`
![image](https://github.com/user-attachments/assets/fe71896c-e417-4615-bed4-e08fd751dc6e)
Click **Save & Apply**.

---

## Phase 5: Pi-hole Stats on PiOLED

### 17. Configuration & Secret Management
We use a `.env` file to securely store your Pi-hole credentials. This prevents you from hardcoding your password directly into the python script, which is a common security risk.

1. Create the `.env` file in your home directory:
```bash
nano ~/.env
```
2. Paste the following configuration:
```env
# Pi-hole Configuration
# For Pi-hole v6.3+, it is highly recommended to use an App Token instead of your web password.
# You can generate an App Token in the Pi-hole Web Interface under Settings > API > App Tokens.
PIHOLE_PASS="YOUR_PASSWORD_OR_APP_TOKEN_HERE"

# The URL of your Pi-hole. If running this script on the Pi-hole itself, leave as localhost.
# If running on a custom port, add it here (e.g., http://localhost:8080)
PIHOLE_URL="http://localhost"
```
3. Replace `"YOUR_PASSWORD_OR_APP_TOKEN_HERE"` with your actual web interface password or App Token. (If you are running Pi-hole on a custom port, update the `PIHOLE_URL` accordingly). Save (`Ctrl + X`, `Y`), and exit.

### 18. Create the `stats.py` Script
Now that we have Pi-hole installed and configured to point at Unbound, let's get the display to show live Pi-hole stats via the API.

First, install the font:
```bash
cd ~
wget http://kottke.org/plus/type/silkscreen/download/silkscreen.zip
unzip silkscreen.zip
```

Create the script:
```bash
nano ~pi/stats.py
```
Paste the following into it:

```python
# SPDX-FileCopyrightText: 2017 Limor Fried for Adafruit Industries
# SPDX-FileCopyrightText: 2017 Tony DiCola for Adafruit Industries
# SPDX-FileCopyrightText: 2017 James DeVito for Adafruit Industries
#
# SPDX-License-Identifier: MIT

# Import Python System Libraries
import json
import os
import subprocess
import time

# Import Requests Library
import requests

# Import Blinka
from board import SCL, SDA
import busio
import adafruit_ssd1306

# Import Python Imaging Library
from PIL import Image, ImageDraw, ImageFont

# Create the I2C interface.
i2c = busio.I2C(SCL, SDA)

# Create the SSD1306 OLED class.
disp = adafruit_ssd1306.SSD1306_I2C(128, 32, i2c)

# Leaving the OLED on for a long period of time can damage it
# Set these to prevent OLED burn in
DISPLAY_ON  = 10   # on time in seconds
DISPLAY_OFF = 30   # off time in seconds

# Clear display.
disp.fill(0)
disp.show()

# Create blank image for drawing.
width = disp.width
height = disp.height
image = Image.new('1', (width, height))

# Get drawing object to draw on image.
draw = ImageDraw.Draw(image)
draw.rectangle((0, 0, width, height), outline=0, fill=0)

padding = -2
top = padding
x = 0

# Load nice silkscreen font
font = ImageFont.truetype('/home/pi/slkscr.ttf', 8)

# Read the PIHOLE_PASS environment variable (Set in the .env file and loaded by systemd)
# For Pi-hole v6.3+, it is highly recommended to use an App Token instead of your web password.
PIHOLE_PASSWORD = os.getenv('PIHOLE_PASS', "YOUR_PASSWORD_OR_APP_TOKEN_HERE")
# Read the PIHOLE_URL environment variable to allow custom ports or remote IPs
PIHOLE_URL = os.getenv('PIHOLE_URL', "http://localhost")

auth_url = f"{PIHOLE_URL}/api/auth"
sid = None
session_valid_until = 0  # Keep track of session validity

# Create a persistent session to reuse network connections
req_session = requests.Session()

while True:
    draw.rectangle((0, 0, width, height), outline=0, fill=0)

    cmd = "hostname -I | cut -d\' \' -f1 | tr -d \'\\n\'"
    IP = subprocess.check_output(cmd, shell=True).decode("utf-8")
    cmd = "hostname | tr -d \'\\n\'"
    HOST = subprocess.check_output(cmd, shell=True).decode("utf-8")

    current_time = time.time()

    if sid is None or current_time > session_valid_until - 60:
        try:
            auth_payload = {"password": PIHOLE_PASSWORD}
            auth_response = req_session.post(auth_url, json=auth_payload)
            auth_response.raise_for_status()
            auth_data = auth_response.json()
            sid = auth_data.get("session", {}).get("sid")
            validity = auth_data.get("session", {}).get("validity", 0)
            session_valid_until = current_time + validity
            if not sid:
                print("Failed to obtain session ID.")
                sid = None
                time.sleep(60)
                continue
            print(f"New Pi-Hole API Session ID obtained, valid until: {time.ctime(session_valid_until)}")
        except requests.exceptions.RequestException as e:
            print(f"Authentication failed: {e}")
            sid = None
            time.sleep(60)
            continue
        except json.JSONDecodeError:
            print("Failed to decode authentication JSON.")
            sid = None
            time.sleep(60)
            continue

    if sid:
        api_url = f"{PIHOLE_URL}/api/stats/summary?sid={sid}"
        try:
            api_response = req_session.get(api_url)
            api_response.raise_for_status()
            api_data = api_response.json()

            DNSQUERIES = api_data.get('queries', {}).get('total', "N/A")
            ADSBLOCKED = api_data.get('queries', {}).get('blocked', "N/A")
            CLIENTS = api_data.get('clients', {}).get('total', "N/A")

            draw.text((x, top), "IP: " + str(IP) + " (" + HOST + ")", font=font, fill=255)
            draw.text((x, top + 8), "Ads Blocked: " + str(ADSBLOCKED), font=font, fill=255)
            draw.text((x, top + 16), "Clients:     " + str(CLIENTS), font=font, fill=255)
            draw.text((x, top + 24), "DNS Queries: " + str(DNSQUERIES), font=font, fill=255)

        except requests.exceptions.RequestException as e:
            print(f"API request failed: {e}")
            draw.text((x, top + 8), "API Request Failed", font=font, fill=255)
            sid = None
        except json.JSONDecodeError:
            print("Failed to decode API JSON.")
            draw.text((x, top + 8), "API JSON Error", font=font, fill=255)
            sid = None
        except KeyError as e:
            print(f"Key not found in API response: {e}")
            draw.text((x, top + 8), "API Data Error", font=font, fill=255)
            sid = None
    else:
        # Display fallback text if session is missing entirely
        draw.text((x, top), "IP: " + str(IP) + " (" + HOST + ")", font=font, fill=255)
        draw.text((x, top + 8), "Auth Failed/Missing", font=font, fill=255)

    # Display image.
    disp.image(image)
    disp.show()
    time.sleep(DISPLAY_ON)
    disp.fill(0)
    disp.show()
    time.sleep(DISPLAY_OFF)
```

### 19. Test the Stats Script
Since our python script now relies on environment variables, we need to pass the `.env` file directly when testing via terminal. You can do this by exporting the variables first, or running it inline:

```bash
export $(cat ~/.env | grep -v '^#') && python3 stats.py
```
You should see output similar to this, indicating successful session ID acquisition:
![image](https://github.com/user-attachments/assets/a19dabd8-2bf4-4592-80c8-97db6dadb4ae)

And your screen should light up! (Your display may show `0` or `N/A` until you connect a client to the Pi-hole).
![ITWORKS](https://github.com/user-attachments/assets/3895f65b-7c64-4d15-adc2-d5f381a47362)

Press `Ctrl + C` to stop the `stats.py` script.

### 20. Create `stats.service` (Run Script on Boot)
Now we will set the script to start automatically on boot. By setting `EnvironmentFile=/home/pi/.env`, systemd will securely inject your Pi-hole credentials directly into the script's environment.

Let's create a systemd service file:

```bash
sudo nano /etc/systemd/system/stats.service
```
Paste the following:

```ini
[Unit]
Description=Pi-hole Stats Display Script
After=network.target

[Service]
User=pi
HomeDirectory=/home/pi
WorkingDirectory=/home/pi/
EnvironmentFile=/home/pi/.env
Environment="PATH=/home/pi/pihole/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
ExecStart=/bin/bash -c "source /home/pi/pihole/bin/activate && /home/pi/pihole/bin/python3 /home/pi/stats.py"
Restart=on-failure
StandardOutput=append:/var/log/stats.log
StandardError=append:/var/log/stats_error.log

[Install]
WantedBy=multi-user.target
```

Enable and start the service:
```bash
sudo systemctl enable stats.service
sudo systemctl daemon-reload
sudo systemctl start stats.service
```
The script should now run in the background, turning your screen on and off correctly to prevent burn-in.

### 21. Final Reboot
Reboot your Pi to ensure everything starts up as expected:
```bash
sudo reboot
```

---

## Migrating from the Old Setup
If you set up your Pi-hole display before the `.env` secret management changes were introduced, you must update your configuration to keep your script working properly.

1. Create your new `.env` file following the instructions in **Step 17** above.
2. Update your `stats.py` script by replacing its contents with the new code block from **Step 18**.
3. Edit your systemd service file to inject the `.env` file into the environment:
   ```bash
   sudo nano /etc/systemd/system/stats.service
   ```
   Add the following line right below `WorkingDirectory=/home/pi/`:
   ```ini
   EnvironmentFile=/home/pi/.env
   ```
4. Reload the daemon and restart the service:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart stats.service
   ```

---

*Once rebooted, you can start toying with your Pi-hole install by adding additional block lists and configuring your network's router to use the Pi as its DNS server!*