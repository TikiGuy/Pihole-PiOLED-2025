Pihole PiOLED 2025 Guide  -DRAFT-

I recently decided to finally get pihole up and running at home.  When looking at buying two raspberry pi W 2 devices I saw the instructions on how to set up pihole with the PiOLED display on Adafruit's Website.

http://learn.adafruit.com/pi-hole-ad-blocker-with-pi-zero-w

The article shows a last update in 2024 which I incorrectly assumed meant that I'd be able to follow it with no issues in 2025.  I ran into quite a few issues and had to tweak a few things which I'll outline below.

The biggest issue revolves around the fact that Pihole changed the way the API works, specifically the authentication is no longer key based, it's session based and they also changed how the API functions, t

the change to how python opperates in the Bookworm versions of Raspberry Pi OS.  Python scripts are now seperated into Virtual Environments (venv).  This is mentioned in the article but isn't stressed that you need to understand what that means.
For those of us that aren't programmers and looking for true step by step directions it's pretty easy to get a little turned around.  

I also had issues getting the PiOLED to work

Also activating the venv on boot as well as starting the stats.py on boot changed since the documention was written.

1. Updated Pi-hole API Interaction in stats.py

    We replaced the original, simpler API call with code that handles Pi-hole's session-based authentication.
    We correctly extracted the session ID (sid) from the authentication response.
    We added error handling to gracefully manage potential API request failures.
    We implemented logic to re-authenticate when the session is close to expiring, minimizing API requests and avoiding rate limiting.

Here's the relevant section of the final stats.py:
Python

PIHOLE_PASSWORD = "your_pihole_password"  # Replace with your actual password
auth_url = "http://localhost/api/auth"
sid = None
session_valid_until = 0  # Keep track of session validity

while True:
    # ... (OLED setup and other system stats code) ...

    current_time = time.time()

    if sid is None or current_time > session_valid_until - 60:  # Re-authenticate if needed
        try:
            auth_payload = {"password": PIHOLE_PASSWORD}
            auth_response = requests.post(auth_url, json=auth_payload)
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
        api_url = f"http://localhost/api/stats/summary?sid={sid}"
        try:
            api_response = requests.get(api_url)
            api_response.raise_for_status()
            api_data = api_response.json()

            DNSQUERIES = api_data.get('dns_queries_today', "N/A")
            ADSBLOCKED = api_data.get('ads_blocked_today', "N/A")
            CLIENTS = api_data.get('unique_clients', "N/A")

            # ... (OLED drawing code using the Pi-hole stats) ...

        except requests.exceptions.RequestException as e:
            print(f"API request failed: {e}")
            sid = None  # Invalidate session on API request failure
        except json.JSONDecodeError:
            print("Failed to decode API JSON.")
            sid = None
        except KeyError as e:
            print(f"Key not found in API response: {e}")
            sid = None

    # ... (OLED display and sleep code) ...

2. Setting Up systemd to Run stats.py on Boot

    We created a systemd service unit file (/etc/systemd/system/stats.service) to manage the execution of the stats.py script.
    We configured the service to:
        Start after the network is available.
        Run as a specific user.
        Activate the Pi-hole virtual environment before running the script.
        Restart the script automatically if it fails.
        Log output and errors to files for debugging.
    We enabled the service to start automatically on boot.

Here's the final stats.service file:
Ini, TOML

[Unit]
Description=Pi-hole Stats Display Script
After=network.target

[Service]
User=pi  # Replace 'pi' with your actual username
WorkingDirectory=/home/pi/  # Replace with the actual path to your script
Environment=PATH=/home/pi/pihole/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin  # Adjust path to include your venv's bin directory
ExecStart=/bin/bash -c "source /home/pi/pihole/bin/activate && python3 /home/pi/stats.py"  # Adjust paths to activate and script
Restart=on-failure
StandardOutput=append:/var/log/stats.log
StandardError=append:/var/log/stats_error.log

[Install]
WantedBy=multi-user.target

    We used systemctl commands to:
        Enable the service (sudo systemctl enable stats.service).
        Start the service (sudo systemctl start stats.service).
        Check the service status (sudo systemctl status stats.service).
